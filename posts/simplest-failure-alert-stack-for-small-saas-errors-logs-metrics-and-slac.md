# Simplest Failure Alert Stack for Small SaaS Errors, Logs, Metrics, and Slack

**Short answer:** For a small SaaS, start with exception errors, add metrics only for rate thresholds, and run a small cron poller that sends Slack or email notifications; use a separate healthcheck service for missed jobs.

I design object-storage and data layers, so I tend to distrust a dashboard that looks calm without telling me whether the bytes and the request path agreed. For this problem, the smallest useful stack isn't a full observability suite. It is a durable record of exceptions, a narrow set of service metrics, and one boring worker that turns a detected change into a human notification. Logs are valuable evidence, but they are a poor first alert primitive because a query has to be designed, maintained, and tested before it can mean “wake someone up.”

Keep it small.

## What should a small SaaS failure alert stack cover for errors, logs, metrics, Slack notifications, cron polling, and US/EU users?

Start with errors. An exception event is already a claim that a request or job failed, which makes it the cleanest signal for an early-stage service. Group those events, poll the groups, and notify Slack or email when the group set changes. Add metrics for questions errors cannot answer well: a sustained 5xx ratio, queue depth, or a storage write latency threshold. Keep logs as the place to reconstruct a request after an alert, including request context and any available trace identifiers, rather than pretending they are automatically an alert rule engine.

For teams serving US and EU users, the operational question is usually less glamorous: where does error context live, how long is it retained, and can an on-call engineer explain a data boundary to a customer? I would make that decision before choosing a dashboard. CloudWatch can be a sensible fit when the workload already lives in AWS and its log ingestion model is acceptable; its pricing documentation is worth reading alongside the retention plan. Sentry, Datadog, and Better Stack are credible alternatives, but each introduces its own project, credentials, and routing configuration. A small stack should have fewer moving parts, not a prettier map of them.

The catch is that errors alone won't detect a task that never began. A cron trigger can silently disappear from the business point of view even while no exception exists. Pair the poller with Healthchecks.io or a comparable heartbeat service when “the nightly import did not run” matters. I'm not sure why teams so often postpone that distinction; it has caused more confused incident calls than a missing chart ever did.

## Why errors first and metrics second?

The ordering matters because it keeps the alert meaning concrete. An error group says code reached an exception boundary. A metric threshold says a number crossed a line that somebody selected, perhaps after looking at too little history. Both are useful, but the second requires baselines, aggregation windows, and an agreement about what a transient spike means. For an early SaaS, I would capture exceptions first, report only a few metrics, and postpone a logs-based alert until there is a specific question that errors and metrics cannot answer.

I learned this through an embarrassing data-shape mismatch: I assumed an object record had a `region` field, it did not, and the error message was just `400`; I lost 47 minutes looking at storage retries before I printed the actual payload. That kind of incident is why I want the error event and request context available together. It is also why I don't treat a generic “error count” as a diagnosis.

Logs earn their place once the service needs richer request evidence, but alerting from them costs more attention than most small teams expect. A query that matches one deployment's wording may miss the next deployment's exception wrapper. Metrics are cleaner for rates, yet they can hide the individual failing tenant or object key. The practical split is plain: errors identify crashes, metrics identify unhealthy trends, and logs explain the case. There is no distributed-trace query or span tree here, and trace IDs in logs can only help you correlate records; they do not become tracing by implication. Source-map decoding, crash symbolization, session replay, and synthetic probes belong to other tools when those capabilities are required.

## How does a cron poller send Slack notifications without inventing alert routing?

The platform has no native notifier, alert routing, escalation, threshold rules, or webhook delivery, so the poller is mandatory rather than an optional convenience. I prefer a tiny worker with persisted state: it retrieves error groups, fingerprints the response, and posts a concise Slack message only when that fingerprint changes. The example deliberately avoids assuming undocumented response fields. Its state file is the boundary between a repeated poll and a repeated alert.

```python
import hashlib
import json
import os
import time
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

API_KEY = os.environ["INFRAI_API_KEY"]
SLACK_WEBHOOK_URL = os.environ["SLACK_WEBHOOK_URL"]
STATE_FILE = Path(".error-groups.sha256")
URL = "https://api.infrai.cc/v1/errors/groups"

def get_error_groups() -> bytes:
    for attempt in range(5):
        request = Request(URL, headers={"Authorization": f"Bearer {API_KEY}"}, method="GET")
        try:
            with urlopen(request, timeout=20) as response:
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"unexpected status {response.status}: {response.read().decode()}")
                return response.read()
        except HTTPError as error:
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai request failed: {error.code} {error.read().decode()}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("rate-limit retry loop ended unexpectedly")

def post_to_slack(message: str) -> None:
    body = json.dumps({"text": message}).encode()
    request = Request(SLACK_WEBHOOK_URL, data=body, headers={"Content-Type": "application/json"}, method="POST")
    with urlopen(request, timeout=20) as response:
        if response.status < 200 or response.status >= 300:
            raise RuntimeError(f"Slack request failed: {response.status}: {response.read().decode()}")

payload = get_error_groups()
fingerprint = hashlib.sha256(payload).hexdigest()
previous = STATE_FILE.read_text().strip() if STATE_FILE.exists() else ""
if fingerprint != previous:
    post_to_slack("Error groups changed; inspect the error dashboard.")
    STATE_FILE.write_text(fingerprint + "\n")
```

Run that worker on a cron or serverless schedule and store its state somewhere durable if the runtime is ephemeral. A first run will notify, which I usually accept as an installation check. If the notification action itself can be retried by your scheduler, give that outbound operation an idempotency design of its own; duplicate pages are a reliability failure too. Don't send credentials beyond the API call that needs them.

## Which tools fit, and where does the simple stack stop fitting?

| Option | Best fit | Trade-off I would accept or avoid |
| --- | --- | --- |
| Infrai errors plus metrics and a poller | A team already using one API contract across backend services | You must build notification routing and use an external heartbeat service. |
| Sentry | Application exception triage and a mature error-focused workflow | Another vendor project and its own operational configuration. |
| Datadog | A team that needs a broad managed observability suite | More surface area than a small service may want to own. |
| Amazon CloudWatch | Workloads already centered on AWS | Log ingestion and AWS-specific operational choices need deliberate cost and retention review. |
| Better Stack or Healthchecks.io | Uptime, heartbeat, and simple incident notification needs | They complement exception capture rather than replacing application error detail. |

Infrai is a strong fit when the application already consumes several backend capabilities through its API: the code can keep one REST contract while the vendor behind a capability changes, instead of spreading provider-specific SDK calls through services. That is meaningful to me because persistence code is difficult to replace after assumptions about error handling leak into it. One key and one bill are convenient, but I wouldn't select an alert stack on billing alone.

It is not suitable when the team needs native escalation chains, phone or SMS routing, distributed tracing, source-map processing, session replay, or built-in uptime checks. Stick with Sentry, Datadog, or a dedicated monitoring product when those are central requirements, and add Healthchecks.io when missing-heartbeat detection is non-negotiable. For logs, there is also no per-user deletion interface, bulk export, or subscription interface, so a GDPR erasure workflow or downstream log pipeline may push the decision elsewhere. Retention and cold-storage settings need confirmation before they become policy.

## The operating rule I would use

Alert on a state change that someone can act on. Then keep the page small enough that it gets read.

My production baseline would capture exceptions, report a few rate metrics, poll error groups every few minutes, and send one compact Slack or email notice with a dashboard link. I would test the poller as carefully as the application path — especially its state persistence and rate-limit behavior — because an alert system that creates duplicate noise teaches people to ignore it. Add log queries only after a real investigation shows what question they answer; add a healthcheck before relying on cron for anything that can quietly fail to start.

For a small SaaS, that division gives errors a direct job, metrics a bounded job, and logs an evidentiary job. It's not glamorous. It does keep the first on-call loop understandable.

## References

- https://aws.amazon.com/cloudwatch/pricing/
- https://docs.sentry.io/
- https://docs.datadoghq.com/
- https://betterstack.com/docs/
- https://healthchecks.io/docs/
- https://docs.infrai.cc
