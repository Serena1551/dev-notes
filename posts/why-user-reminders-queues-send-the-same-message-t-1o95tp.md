# Why User Reminders Queues Send the Same Message Twice: Retry and Idempotency

## TL;DR

Decision rule: treat every user reminder as a logical occurrence with a stable idempotency key, expect the queue to deliver that occurrence more than once, and require the final sender to honor the same key across retries. A worker-side `processed` flag alone doesn't close the failure window between sending the message and acknowledging the queue.

This is the uncomfortable part: an at-least-once queue can preserve work, but it cannot by itself promise that an external side effect happens once. The design has to make a repeated delivery harmless. For a marketplace cleanup job, the same rule applies to each cleanup target; for user reminders, the externally visible effect is usually an email, push notification, or text, so duplicate suppression must extend all the way to that boundary.

## Why the same message is sent twice

A queue retry does not necessarily mean the first worker failed before doing useful work. Consider one reminder occurrence, `reminder_7f3`, scheduled for `2026-08-12T09:00:00Z`. Worker A claims it at 09:00:01, asks the notification service to send, and receives a receipt at 09:00:02. Before A commits that receipt, its process exits and the queue lease eventually expires. Worker B receives the same transport message, reads a ledger that still lacks a completed record, and sends again at 09:01:04. The queue has no durable evidence of the first success, while the notification service sees two ordinary requests unless both carry the same idempotency key. Both workers are behaving consistently with their local evidence; the user still sees two messages.

Nothing looks broken locally.

The dangerous sequence is short:

1. Receive the reminder.
2. Send the notification.
3. Record `sent`.
4. Acknowledge the queue item.

If execution stops between steps 2 and 3, retrying is necessary for reliability and risky for duplication. Reordering the steps only moves the risk: recording `sent` before the send can turn a crash into a permanently missing reminder. There is no atomic transaction spanning an ordinary database, a queue acknowledgment, and an unrelated notification provider unless those systems expose a shared transactional mechanism.

Duplicate publication is a separate failure mode. A scheduler can enqueue the same logical occurrence twice, each with a different transport message ID, so deduplicating only on the broker's delivery ID is too narrow. Conversely, deduplicating forever on `user_id + reminder_type` is too broad because tomorrow's legitimate reminder would be discarded. The identity belongs to the business occurrence, not the delivery attempt.

Retries can also race. A slow worker may hold an expired lease while a replacement starts; without a conditional state transition, both can send. Cancellation adds another edge: a reminder can be canceled after enqueue but before execution, which means the worker should read authoritative reminder state immediately before claiming the send. These are state-machine problems — the queue merely makes them visible.

No retry policy fixes identity.

## How should a user reminders queue handle at-least-once retry without duplicate processing?

Give each scheduled occurrence an immutable identifier at creation time. Derive the idempotency key from fields that define that occurrence, such as the reminder ID, schedule version, channel, and scheduled timestamp. Do not include an attempt number, random nonce, current time, or transport delivery ID; any of those would produce a new key precisely when a retry needs the old one.

If the raw identity contains information that should not cross service boundaries, derive a fixed token with HMAC. RFC 2104 defines HMAC as keyed hashing for message authentication. Here it provides a deterministic, opaque token as long as the canonical input and secret remain stable. It doesn't replace authorization, and key rotation needs an explicit version because changing the secret changes the token.

Then carry that key through three layers:

1. The producer stores it with the reminder occurrence and publishes it in the job payload.
2. The worker claims it through a unique constraint and a renewable lease, so concurrent attempts do not both proceed.
3. The notification sender accepts the same key and returns the prior result when an attempt repeats.

The third layer closes the ambiguous window. If the worker sends successfully and stops before marking its ledger row complete, the lease eventually expires and a retry sends with the identical key. A sender that enforces idempotency recognizes the prior operation rather than creating another notification. The worker can then mark the ledger complete and acknowledge the queue item.

There is a catch. If the downstream sender does not support an idempotency key, a worker-side ledger cannot distinguish “the send completed but the response was lost” from “the send never happened.” In that case, don't claim exactly-once user-visible delivery. Either accept a documented duplicate risk, move delivery behind a component whose database and dispatch operation you control, or choose a sender with a verifiable idempotency contract. A short retry delay changes frequency, not correctness.

## A focused Python implementation

The following example leaves storage and transport behind typed interfaces so the important contract stays visible. `claim` must be an atomic insert-or-conditional-update keyed by `key`; `complete` must conditionally match the lease token. The sender must make repeated calls with one idempotency key resolve to one logical send.

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timezone
import hashlib
import hmac
from typing import Literal, Protocol


@dataclass(frozen=True)
class ReminderJob:
    reminder_id: str
    schedule_version: int
    scheduled_for: datetime
    channel: Literal["email", "push", "sms"]
    recipient_ref: str
    template_ref: str


class Ledger(Protocol):
    def claim(self, key: str, lease_seconds: int) -> str | None:
        """Return a lease token, or None when another live claim owns the key."""

    def complete(self, key: str, lease_token: str, receipt: str) -> None:
        """Conditionally change the matching live claim to complete."""

    def is_complete(self, key: str) -> bool:
        """Return whether the logical occurrence has completed."""


class ReminderStore(Protocol):
    def is_sendable(self, reminder_id: str, schedule_version: int) -> bool:
        """Check current cancellation and version state."""


class Sender(Protocol):
    def send(self, job: ReminderJob, idempotency_key: str) -> str:
        """Return one stable receipt for repeated calls with the same key."""


def occurrence_key(job: ReminderJob, secret: bytes, key_version: str) -> str:
    scheduled = job.scheduled_for.astimezone(timezone.utc).isoformat()
    canonical = "\x1f".join(
        [
            key_version,
            job.reminder_id,
            str(job.schedule_version),
            scheduled,
            job.channel,
        ]
    ).encode("utf-8")
    digest = hmac.new(secret, canonical, hashlib.sha256).hexdigest()
    return f"{key_version}_{digest}"


def process_reminder(
    job: ReminderJob,
    secret: bytes,
    ledger: Ledger,
    reminders: ReminderStore,
    sender: Sender,
) -> Literal["sent", "already_sent", "busy", "canceled"]:
    key = occurrence_key(job, secret, key_version="k1")

    if ledger.is_complete(key):
        return "already_sent"
    if not reminders.is_sendable(job.reminder_id, job.schedule_version):
        return "canceled"

    lease_token = ledger.claim(key, lease_seconds=60)
    if lease_token is None:
        return "busy"

    receipt = sender.send(job, idempotency_key=key)
    ledger.complete(key, lease_token, receipt)
    return "sent"
```

The queue adapter should acknowledge `sent`, `already_sent`, and `canceled`. It should also acknowledge or briefly defer `busy` according to the queue's lease model; immediately hammering the same live claim adds load without increasing correctness. Exceptions should remain unacknowledged so the configured retry policy can redeliver them, while poison inputs that can never validate should go to a quarantine path rather than retry forever.

One subtlety deserves extra scrutiny: the sample checks cancellation before `claim`, so a cancellation racing after that check can still lose. A stricter system makes `claim` verify the current reminder version in the same database transaction, or has the sender consult a dispatch record created transactionally with the claim. Which boundary is sufficient depends on the product's cancellation promise. Without that promise and its timing definition, the correct cutoff remains uncertain.

For Node.js services, the state machine and key material should remain identical even though the syntax changes: use the platform HMAC primitive, a unique database key, conditional lease updates, and a sender API that accepts the stable token. Changing languages does not change the crash window.

## Trade-offs and operational proof

Before selecting a queue or sender, test the contract rather than the happy path. Product documentation may describe at-least-once delivery — Google Cloud Pub/Sub, for example, documents that delivery mode in its overview — but the application still owns business identity and the final side effect. Broker delivery guarantees and notification guarantees are different claims.

| Design choice | Prevents | Does not prevent | Best fit |
|---|---|---|---|
| Transport message-ID dedupe | Repeat delivery of one published item | Duplicate publication with new IDs | A narrow optimization, never the business key |
| Unique worker ledger | Concurrent processing for one known occurrence | Send-success/commit-loss duplication | Internal work with transactional side effects |
| Sender idempotency key | Repeated external send for one key | Bad key construction or overly broad identity | User-visible notifications |
| Record before send | Duplicate attempts | Missing sends after a crash | Not suitable for reminders that must arrive |
| Record after send only | Missing sends before the record | Duplicate sends after an ambiguous result | Only when duplicates are acceptable |

Testing should force each transition, not wait for a production accident. Publish the same occurrence twice under different transport IDs. Start two workers against the same key. Expire a lease while the first worker is paused. Make the sender complete while its response is withheld, then retry and verify that the receipt and user-visible message remain singular. Cancel between enqueue and claim. Advance `schedule_version` and confirm that the new occurrence gets a new key while a replay of the old version is rejected.

Observe the same business key across producer, worker, ledger, and sender, but avoid logging recipient addresses or message bodies. Useful counters include claims rejected as busy, completed-key replays, lease expirations, quarantined inputs, sender retries by outcome, and reminder age at completion. An increase in dedupe hits can indicate producer duplication or acknowledgment trouble even when users are protected from duplicate messages.

Retention is a correctness setting. Keep completed keys for at least as long as an old delivery or manual replay can return; deleting them earlier reopens duplication. Keeping them indefinitely may conflict with storage and privacy requirements, so define a replay horizon, enforce it at the queue and repair tools, and expire ledger records only after that horizon. Your mileage may vary because retry windows, support procedures, and regulatory constraints differ, but the retention period should be derived from an explicit maximum replay age rather than convenience.

## Roll out without gambling on reminders

Start by generating and logging the stable key without changing delivery. Next, enforce the unique ledger in observation mode and compare claims with actual sender receipts. Enable downstream idempotency for a small reminder category, inject the crash cases above, and expand only when duplicate-key retries consistently resolve to the original receipt.

Keep the old path available during migration, but never let old and new workers consume the same occurrence without sharing the key and ledger. The release criterion is compact: one logical reminder, one stable identity, and one externally visible effect even after the same queue message is processed twice.

## References

- https://www.rfc-editor.org/rfc/rfc2104
- https://cloud.google.com/pubsub/docs/overview
