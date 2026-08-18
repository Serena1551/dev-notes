# Event Notifications: How to Orchestrate Email Fallback, SMS, and Status Polling

For multi-channel event notifications, an email-first password reset with SMS fallback has a short useful life: delivery after the token expires is not recovery, even if a provider eventually labels the message delivered. That constraint changes the design.

Short answer: send email first, persist its message ID, poll for an acceptable email event until a business deadline, and let an idempotent application worker send SMS exactly once if that deadline passes; this is practical for SaaS notifications, but pull-only events make the fallback time approximate rather than real-time.

The database, not either delivery provider, should own the decision. It must remember that one reset attempt moved from `queued` to `emailed`, then either to `delivered`, `sms_fallback`, or `failed`. Without that durable state, a worker restart or a repeated poll can turn one reset request into several text messages.

## Failure drill: the six-minute race

Start with the user-visible deadline and work backward. Suppose the reset token is valid for ten minutes and the product allows email six minutes before trying SMS. The exact six-minute policy is a product choice, not a delivery guarantee. Poll cadence, provider event lag, worker scheduling, and database contention all add uncertainty, so the worker may decide at six minutes plus one polling interval. I'm not sure a tighter target is defensible without measuring those components in the deployed system.

Not enough.

For each notification, store an application-generated notification ID, the email message ID, the SMS message ID when one exists, the token expiry, the fallback deadline, the next poll time, the current state, and a version or equivalent compare-and-swap value. Keep the reset URL or verification secret out of logs and status tables. A worker should claim only due rows, check suppression before each channel send, and commit the state transition in the same database transaction that records its idempotency decision.

The acceptable email signal also needs a written definition. A generic `sent` or `accepted` event may mean the provider took responsibility; it may not mean the mailbox received the message. The supplied email event surface can be polled, but its event vocabulary and timing semantics are not specified here, so don't silently equate every non-error event with delivery. Choose the accepted event set from the live discovery schema and your support policy, then test it with controlled addresses. Now run the awkward race deliberately: worker A reads `emailed` one millisecond before the deadline, worker B reads the same row one millisecond after it, and the email event becomes visible between their provider reads. Both workers may make individually plausible decisions, yet only a conditional database transition may win. Worker A can close the row as `delivered`; worker B can claim `sms_fallback`; whichever commits first makes the other's old version invalid. If no acceptable signal appears by the business timeout, the winning state-machine transition, rather than a provider-side rule, authorizes SMS fallback.

Suppression is a gate, not cleanup. Check it before the initial email and again before SMS because recipient eligibility can change while the worker waits. The two channels have different suppression operations, so model the result as a channel-specific decision rather than one permanent boolean on the customer record.

## How can a practical SaaS workflow poll email status and trigger SMS fallback?

Use a monotonic state machine. `queued -> emailed -> delivered` is the preferred path; `emailed -> sms_fallback -> delivered` is the fallback path; either active state may become `failed` when the token expires or policy rejects further contact. Never move backward, and never infer state solely from the age of an in-memory task.

The following Python example is deliberately focused on the hard part: durable decisions around polling. It starts after the application's send adapters have stored provider message IDs, because the verified material does not publish the request or response fields needed to reproduce those writes without guessing. The injected callbacks stand for the documented email-event read, SMS-status read, transactional state update, and already-validated SMS send adapter. The send adapter should use its application notification ID as the idempotency key.

```python
from __future__ import annotations

import os
import random
import time
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Callable
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def infrai_get(path: str) -> str:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    delay = 1.0

    for attempt in range(5):
        request = Request(
            f"{base_url}{path}",
            method="GET",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                return response.read().decode("utf-8")
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"request rejected with HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            wait = float(retry_after) if retry_after else delay + random.random()
            time.sleep(wait)
            delay *= 2

    raise RuntimeError("retry budget exhausted")


def read_infrai_email_events(_message_id: str) -> str:
    return infrai_get("/v1/email/event/list")


def read_infrai_sms_status(message_id: str) -> str:
    return infrai_get(f"/v1/sms/status/{message_id}")


@dataclass(frozen=True)
class Notification:
    notification_id: str
    state: str
    email_message_id: str
    sms_message_id: str | None
    fallback_at: datetime
    expires_at: datetime


def poll_once(
    item: Notification,
    read_email_events: Callable[[str], str],
    read_sms_status: Callable[[str], str],
    email_delivered: Callable[[str], bool],
    sms_delivered: Callable[[str], bool],
    sms_allowed: Callable[[Notification], bool],
    send_sms_once: Callable[[Notification], str],
    transition_once: Callable[[str, str, str | None], None],
) -> None:
    now = datetime.now(timezone.utc)
    if now >= item.expires_at:
        transition_once(item.notification_id, "failed", None)
        return

    if item.state == "emailed":
        events_json = read_email_events(item.email_message_id)
        if email_delivered(events_json):
            transition_once(item.notification_id, "delivered", None)
        elif now >= item.fallback_at and sms_allowed(item):
            sms_id = send_sms_once(item)
            transition_once(item.notification_id, "sms_fallback", sms_id)
        return

    if item.state == "sms_fallback" and item.sms_message_id:
        status_json = read_sms_status(item.sms_message_id)
        if sms_delivered(status_json):
            transition_once(item.notification_id, "delivered", None)
```

There are three important boundaries, and they are intentional. First, `email_delivered` and `sms_delivered` must parse the response schemas returned by live discovery; treating an undocumented string as authoritative would be brittle. Second, `send_sms_once` must perform the SMS suppression check immediately before sending and use a stable idempotency key, so two workers observing the same timeout cannot produce two sends. Third, `transition_once` must reject stale transitions with a version check or row lock. Those are application invariants, not comments a retry loop can enforce.

Set `INFRAI_BASE_URL` to the documented v1 API base before running the worker, then pass `read_infrai_email_events` and `read_infrai_sms_status` into `poll_once`. At the HTTP adapter boundary, the example honors `Retry-After` on `429` and otherwise uses exponential backoff with jitter. It surfaces other `4xx` bodies because they carry the rejection reason, sets the request method explicitly, reads the bearer key from `INFRAI_API_KEY`, and keeps credentials out of source control.

Infrai fits this adapter boundary when a team wants plain REST calls without installing or tracking a client SDK, and the same key and bill can cover both communication capabilities. Its public discovery surface also exposes schemas and runnable examples, which is useful when implementing the omitted serializers. The catch is structural: email and SMS events are pull-based, so it is not suitable for sub-minute escalation with a firm timing promise.

## Evidence matrix: choosing the delivery boundary

Vendor selection comes after the state model because no provider removes the need to define expiry, suppression, duplicate handling, and what counts as delivered. Compare products against those invariants with a small proof, using current documentation and a test recipient set; marketing labels such as “multi-channel” say little about event timing or durability.

| Option | Integration shape to evaluate | Best fit in this design | Reason to choose something else |
|---|---|---|---|
| Infrai | One REST surface for email and SMS; events are polled | A small service that values an SDK-free HTTP boundary and can tolerate approximate fallback timing | Choose a push-capable option when a hard sub-minute escalation target matters |
| Twilio SendGrid plus Twilio Messaging | Two named communication products behind one application state machine | Teams already standardizing their delivery adapters around Twilio products | Re-evaluate if minimizing provider-specific integration surfaces is the primary constraint |
| Amazon SES plus Amazon SNS | Separate AWS email and messaging products coordinated by application code | An AWS-centered system prepared to own the orchestration layer | Re-evaluate when the team wants one direct communication API boundary |
| Mailgun plus an SMS provider | Email product paired with a separate text-message provider | Teams with an established Mailgun email integration and a preferred SMS vendor | Re-evaluate when key, billing, and event normalization across vendors create more operational work than desired |
| Postmark plus an SMS provider | Transactional email product paired with a separate text-message provider | Teams prioritizing their existing Postmark workflow | Re-evaluate when a second provider and adapter are unacceptable |

One row will not win every deployment.

This table intentionally avoids a synthetic winner. The evidence here establishes Infrai's pull model, REST shape, and capability boundary, but it does not establish equivalent event semantics, measured latency, or uptime for the alternatives. Your mileage may vary with region and recipient network. A defensible bake-off records duplicate rate, the distribution of event delay, suppression behavior, and terminal-state coverage under the same password-reset policy; until those measurements exist, a reliability ranking would be theater.

There are other boundaries to surface before signing off. Infrai has no email webhook or SMS webhook, no SMTP relay, and no voice, WhatsApp, or RCS channel. Email does not provide a managed OTP interface, so an email verification code belongs to the application, while SMS has an OTP operation. SMS geographic fencing and country-price circuit breakers also belong in application policy. For domestic China email compliance, a pending Tencent email vendor is not evidence of readiness.

## Rollout ledger: proving that fallback stays singular

Begin with shadow polling: send only the normal email, persist observed events, and calculate when SMS would have fired without actually sending it. This reveals the event vocabulary and delay distribution while keeping the customer experience unchanged. Do not log reset tokens.

Next, enable SMS fallback for a narrow cohort with one database-enforced idempotency key per notification and channel. Track state-transition conflicts, suppression denials, polls per notification, fallback count, and delivery signals before token expiry. A queue may deliver work more than once, so the worker must remain correct when `poll_once` is repeated concurrently — the durable compare-and-swap is the guardrail.

Finally, test the ugly paths on purpose: an HTTP `429`, a worker restart after the SMS send but before its state commit, an email event that arrives just as the fallback deadline passes, a recipient newly suppressed between channels, and token expiry during backoff. The expected outcome is one terminal state and no duplicate fallback. If the service-level objective still requires deterministic sub-minute escalation, stop tuning the poll interval and choose an architecture with push events; this workflow cannot manufacture immediacy from pull-only signals.

## Sources

- https://mustache.github.io/mustache.5.html
- https://senders.yahooinc.com/best-practices/
