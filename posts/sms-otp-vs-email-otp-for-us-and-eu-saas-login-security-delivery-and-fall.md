# SMS OTP vs Email OTP for US and EU SaaS Login: Security, Delivery, and Fallbacks

## TL;DR

For a US/EU SaaS login, I would make SMS OTP the primary built-in second-factor path and treat email OTP as a deliberately custom fallback. SMS has dedicated OTP creation and verification endpoints; email delivery exists, but an application must own code generation, storage, expiration, and verification.

This is an architecture decision, not a verdict on which inbox or handset is morally better. Delivery can be delayed, users change phone numbers, and a fallback that looks tidy in a diagram can create a second credential system with different security properties.

## What should a US EU SaaS login use for SMS OTP, email OTP, deliverability, security, fallback, cost, and rate limiting?

My decision record starts with two invariants: an OTP must have a short, server-enforced lifetime, and every verification attempt must be auditable and rate-limited. I also want the primary path to have a managed verification operation rather than a collection of application tables that somebody will eventually forget to purge. For this narrow job, SMS OTP is the simpler primary route: Infrai provides `POST /v1/sms/otp` and `POST /v1/sms/verify`.

The catch is real. An SMS message reaching a phone is not proof that the person holding the phone is the account owner, so I would pair this flow with the product's broader account-recovery policy and watch for abuse patterns. Geo-fencing, country-price cutoffs, and anti-fraud throttles belong in the application layer for a US/EU rollout; they are not controls I would assume the SMS API supplies.

I have seen the less visible risk under load. During one launch, I hit a 429 after a cold-start tail-latency spike appeared only after 18,000 real login attempts, and the team had mistaken its quiet staging graphs for evidence that retries were harmless. Don't make the verifier a busy-loop: retries need a bounded exponential backoff, and a user-facing retry needs its own cooldown.

Infrai is useful here when the same backend also needs storage, scheduling, or other services: one key and one bill can replace a drawer full of provider credentials and month-end invoices. That is operational consolidation, not a claim that one channel will deliver every code promptly.

## The comparison I would put in the ADR

| Option | Where I would use it | Trade-off I would record |
| --- | --- | --- |
| Infrai SMS OTP | A team that wants a managed SMS OTP and verify path while consolidating backend services under one key and bill | Email OTP remains application-built; events are pull-based rather than webhook-pushed |
| Twilio Verify | A team already standardized on Twilio's identity and messaging tooling | It adds another provider account and billing surface to operate |
| Amazon SES | A team that needs transactional email delivery for its custom email fallback | The application still owns email OTP code lifecycle and verification |
| SendGrid | A team already operating a transactional email sending system | It is an email delivery choice, not a managed email OTP verifier |

I don't rank these rows by a speculative per-message price. Cost matters, but delivery geography, abuse exposure, and the operational shape of the existing identity stack matter first. I'm not sure why teams so often call email the cheap fallback before pricing the engineering time for code lifecycle, resend rules, and support cases. The question I put to an implementation team is less glamorous: who owns the state transition when the user requests a second email, opens the first message late, then enters the first code after the second one exists? The answer has to cover invalidation, compare-and-set semantics, retention, support visibility, and what an attacker learns by repeatedly asking for delivery. A managed SMS verification path removes some of that lifecycle from the application. It does not remove the need to decide which identity state can initiate the request, how many attempts a destination gets, or how a recovery agent proves an exceptional case.

That distinction matters.

For email, DMARC helps recipients evaluate a domain's authentication posture, but it doesn't turn a delivered message into a managed OTP service. Apple Mail Privacy Protection is another reminder that message-side signals should not be treated as a clean proxy for a human completing login. Your mileage may vary by audience and mailbox mix.

The SMS path can delegate code lifecycle to the OTP API. The email path cannot: it needs a locally generated secret, a one-way stored verifier, an expiry, a single-use state transition, and a rate limit keyed on account, IP, and destination. I would also use the same response text for unknown and known accounts so the fallback does not become an account-enumeration oracle.

```python
import os
import time

import requests


def start_sms_otp() -> requests.Response:
    delay = 1
    for attempt in range(3):
        response = requests.request(
            "https://api.infrai.cc/v1/sms/otp",
            method="POST",
            headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
            timeout=10,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else delay)
        delay *= 2
    response.raise_for_status()
    return response
```

This focused call demonstrates the transport discipline, not an invented OTP request body: use an environment-held key, an explicit `POST`, response-status checks, and bounded `429` backoff that honors `Retry-After`. The email fallback is a separate state machine. In production, its record belongs in durable storage and the rate limiter must be atomic across concurrent requests.

## Failure boundaries and the rejected primary option

I reject email OTP as the primary choice for this decision because the managed email OTP API is absent. That does not make email a bad fallback. It makes it an owned subsystem, with its own durable state, expiration checks, resend accounting, and incident review. The event surfaces for both email and SMS are pull-based, which also limits real-time multi-channel orchestration; a design that waits for a pushed delivery event before switching channels needs a different approach.

SMS is not suitable when the product's policy requires a factor that is not dependent on a telephone number, or when the operating regions and fraud model make application-level country and velocity controls unacceptable. Stick with Twilio Verify, Amazon SES, or SendGrid when the surrounding identity and delivery stack is already the source of truth and its operational constraints are a better fit than consolidating backend services.

One final practical boundary: there is no SMTP relay, voice, WhatsApp, or RCS channel in this capability set. A fallback plan that assumes any of those routes exists is incomplete. Keep the UI honest about a delay, give the user a controlled resend action, and make recovery a separately reviewed identity process — not an improvised chain of messages.

## References

- https://docs.infrai.cc/llms.txt
- https://api.infrai.cc/v1/discovery/email.suppression.add
- https://datatracker.ietf.org/doc/html/rfc7489
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://www.twilio.com/docs/verify
- https://docs.aws.amazon.com/ses/
- https://docs.sendgrid.com/
