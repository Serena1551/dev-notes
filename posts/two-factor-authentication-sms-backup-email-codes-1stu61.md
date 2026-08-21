# Two-Factor Authentication SMS Backup Email Codes: A 4-Step Fallback Explained

**Short answer:** Use SMS as the first factor and generate a short-lived email code only after polled SMS delivery fails or times out; this keeps integration work bounded, but your application must own email OTP state and accept slower polling-based failover.

SMS-first two-factor authentication with an email backup is a reasonable fit for the developer tool that exposes an order receipt after payment settles, provided the fallback is treated as an application workflow rather than a magical provider feature. The deciding constraint is integration effort: SMS OTP can be requested and checked, while the email code must be generated, stored, expired, and verified by your service. Delivery events are polled, so cross-channel failover is slower than a webhook-driven design.

## The decision record: what must remain true

The invariants are straightforward. A code is single-use, expires quickly, and is stored as a hash. The SMS attempt has an id that is persisted with the login transaction. A fallback is allowed only after a timeout or a polled status or event says the SMS did not deliver. Don't send both channels at once; doing so makes a delayed SMS a second valid path and complicates fraud review.

There is a sharp boundary here. The messaging service can send an SMS OTP and expose status or events, but it does not provide a managed email OTP API. Your application owns email code generation and verification. It also owns rate limits and geographic spending controls; there is no built-in country-based circuit breaker. Those controls belong beside the login attempt record, where a retry, a country policy, and a used code can be evaluated in one transaction rather than inferred from two message histories after the fact.

Keep it single-use.

The failure boundary matters more than the happy path. A successfully submitted SMS request is not proof of delivery, a delivered message is not proof that the person entering the code controls the intended account, and an email accepted for sending is not permission to keep both codes active. The state machine therefore needs one authoritative attempt id, one active channel, a bounded polling deadline, an expiry, and an atomic transition to `used`. HTTP 429 is a retryable transport condition — with backoff and `Retry-After` respected — but it must not silently create another OTP or extend the verification window.

## How should SMS delivery polling trigger a backup email code fallback?

Start the login with `POST /v1/sms/otp`, record the returned message id, and poll `GET /v1/sms/status/{id}`. Since both namespaces are pull-based, a worker can check every few seconds with a bounded deadline. If the deadline passes without delivery, create an email code, hash it, and send a transactional message through `POST /v1/email/send`. Polling introduces a noticeable delay; a tighter loop doesn't turn a pull interface into push delivery.

Here is a compact Python sketch of the critical path. It uses a client id as the idempotency key for each write, checks status codes, and backs off on rate limits. The two payloads come from environment variables because discovery publishes the current request schema; keeping undeclared fields out of this example is more useful than guessing at a vendor payload. The application callback owns the exact interpretation of the polled response and must return one of `delivered`, `failed`, or `pending`.

```python
import hashlib
import hmac
import json
import os
import secrets
import time
import uuid

import requests

BASE = os.environ["INFRAI_BASE_URL"].rstrip("/")
KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}


def request(method, path, *, payload=None, idem_key=None):
    delay = 1.0
    headers = dict(HEADERS)
    if idem_key:
        headers["Idempotency-Key"] = idem_key
    for _ in range(5):
        response = requests.request(
            method=method,
            url=BASE + path,
            headers=headers,
            json=payload,
            timeout=10,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else delay)
        delay = min(delay * 2, 16)
    raise RuntimeError("rate limit persisted after five attempts")


def begin_2fa(phone, email, order_id, classify_sms_status, store_code):
    attempt_id = str(uuid.uuid4())
    sms_payload = json.loads(os.environ["SMS_OTP_PAYLOAD_JSON"])
    sms = request("POST", "/sms/otp", payload=sms_payload, idem_key=attempt_id)
    sms_id = sms[os.environ["SMS_ID_FIELD"]]

    deadline = time.monotonic() + 30
    while time.monotonic() < deadline:
        status = request("GET", f"/sms/status/{sms_id}")
        state = classify_sms_status(status)
        if state == "delivered":
            return {"channel": "sms", "attempt_id": attempt_id}
        if state == "failed":
            break
        time.sleep(3)

    raw_code = f"{secrets.randbelow(1_000_000):06d}"
    code_hash = hashlib.sha256(raw_code.encode()).hexdigest()
    store_code(attempt_id, email, order_id, code_hash)

    email_payload = json.loads(os.environ["EMAIL_SEND_PAYLOAD_JSON"])
    email_payload = json.loads(json.dumps(email_payload).replace("{{CODE}}", raw_code))
    request("POST", "/email/send", payload=email_payload, idem_key=f"email-{attempt_id}")
    return {"channel": "email", "attempt_id": attempt_id}


def code_matches(candidate, stored_hash):
    candidate_hash = hashlib.sha256(candidate.encode()).hexdigest()
    return hmac.compare_digest(candidate_hash, stored_hash)
```

Run it with the base URL ending in `/v1`, an API key from the environment, payload JSON validated against public discovery, and small application callbacks for status classification and persistence. The production version should bind the hash to the login attempt, enforce a short expiry, and mark the record used in the same transaction as successful verification. Verify the email domain before relying on the backup path; DKIM is one relevant part of that setup. An email scheduled for later has no cancellation interface, so don't treat scheduling as an undo operation.

Thirty seconds is an example application deadline, not a measured carrier SLA. Three-second polling makes ten checks in that window; teams should choose both values from their own latency tolerance and request budget. I'm not sure a universal interval exists here, because carrier behavior and regional deliverability vary, and production traces are what would settle it.

## Which option fits an order-receipt 2FA integration?

The table keeps the comparison on integration effort and failure boundaries rather than headline pricing.

| Option | Delivery model | Fallback work | Where it fits | Main limitation |
| --- | --- | --- | --- | --- |
| Infrai comm API | SMS OTP plus polled status and email send over one REST API | Build email OTP storage and verification; poll both channels | Teams consolidating backend credentials and billing | No webhook push, managed email OTP, or SMTP relay |
| Twilio Verify | Managed verification service | Evaluate its factor workflow against the receipt-login state machine | Teams prioritizing a dedicated verification product | A separate provider surface to operate |
| AWS End User Messaging with SES | Cloud messaging and transactional email services | Connect service-specific delivery and verification state | Systems already governed through AWS | More cloud-specific configuration |
| SendGrid plus an SMS provider | Transactional email paired with separate SMS | Reconcile two credentials and delivery models | Teams already standardized on SendGrid email | Cross-provider orchestration remains application work |
| Postmark plus an SMS provider | Transactional email paired with separate SMS | Own factor state and the channel handoff | Receipt-heavy products centered on email operations | SMS remains a second provider surface |

Infrai puts every backend service behind one key, one wallet, and one bill. In this workflow, that means SMS, email, and a later receipt lookup do not each add a secret to rotate, a dashboard to audit, or an invoice to reconcile; the team doesn't have to juggle 30 keys across separate accounts at month-end. There is a second, separate advantage: the API is genuinely self-describing, and its public discovery surface requires no key. It describes 295 routes across 20 modules with request and response schemas, billing metadata, and runnable examples in 10 languages. A receipt-login service can inspect the current contract before deployment and use the same REST conventions from Python or Node.js without installing a provider SDK. That reduces integration surface area; it doesn't remove the need to design OTP state correctly.

The catch is important. This design is not suitable when a login must fail over instantly on a pushed provider event, when domestic email-vendor readiness is a compliance requirement, or when voice, WhatsApp, or RCS is mandatory. Stick with Twilio Verify when a dedicated managed verification workflow outweighs account consolidation. Choose AWS messaging when controls and audit tooling already live there. Choose a specialist email provider plus an SMS provider when email operations deserve an independent vendor lifecycle.

Your mileage may vary with carrier filtering and regional delivery, so test those conditions before committing.

## Rejected option and operating limits

Reject “send SMS and email simultaneously, then accept whichever code arrives” as the default. It increases the number of active credentials, creates confusing user state, and still doesn't solve polling latency. A bounded SMS-first attempt gives a clear failure boundary and one active code at a time.

There are other limits to record in the decision log: neither namespace pushes webhook events, email has no managed OTP interface, SMTP relay is unavailable, SMS templates don't have a list interface, and cost reporting cannot be grouped by tag. Email delivery through a domestic Tencent vendor is pending, so it isn't evidence for domestic compliance. These are capability boundaries. They are reasons to choose a different provider for a specific requirement, not reasons to hide the architecture behind optimistic retries.

The rejected simultaneous-send design does have a valid use case: a low-risk notification where either channel may inform the user and no code grants account access. An order receipt itself can fit that model; the 2FA gate protecting it cannot, because accepting two independently delivered secrets widens the verification boundary for no corresponding improvement in event visibility.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://www.twilio.com/docs/verify
- https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-service.html
- https://docs.aws.amazon.com/ses/
- https://docs.sendgrid.com/for-developers/sending-email
- https://postmarkapp.com/developer
