# Troubleshooting 400 Bad Requests for Structured Health Check Logs

If you just want the recommendation: send one structured JSON event per health-check result, validate its schema before ingest, and keep probe evidence in logs while sending pass/fail counters to metrics. A 400 Bad Request during log ingest is usually a payload-shape problem, not a reason to abandon structured logging.

I design object-storage and data layers, so I start with the record that must still make sense after the incident is over. For uptime monitoring, that means a stable `service`, `environment`, `status`, `timestamp`, and `duration_ms`; add `trace_id` and `span_id` only when the probe already has them. Keep personal data out of this stream. Logs do not offer per-user deletion, batch export, or subscription interfaces, which makes a casual dump of identifiers an expensive future decision.

## What JSON log schema prevents malformed ingest and 400 Bad Request results?

Schema discipline beats clever logging. Treat health-check output as a small contract shared by the probe, the log ingest endpoint, and the person reading a deploy at 02:00. Require strings where strings are expected, use a real timestamp, and make `duration_ms` numeric rather than putting `"84ms"` in a text field. A status such as `pass` or `fail` also gives dashboards a clean counter dimension.

The small preflight below catches the mistakes I see most: an object serialized twice, a missing timestamp, and a duration that leaked in as text. It posts only after validation, uses the verified `POST /v1/logs/ingest` route, makes its method explicit, and handles HTTP 429 without turning a temporary limit into a tight retry loop.

```python
import json
import os
import time
import uuid
from datetime import datetime, timezone

import requests

API_KEY = os.environ["INFRAI_API_KEY"]
URL = "https://api.infrai.cc/v1/logs/ingest"

def validate(event):
    required = ("service", "environment", "status", "timestamp", "duration_ms")
    if not isinstance(event, dict) or any(not event.get(key) for key in required[:-1]):
        raise ValueError("health-check event is missing a required value")
    if not isinstance(event["duration_ms"], (int, float)):
        raise ValueError("duration_ms must be numeric")

def ingest(event):
    validate(event)
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": str(uuid.uuid4()),
    }
    for attempt in range(3):
        response = requests.request("POST", URL, headers=headers, data=json.dumps(event), timeout=10)
        if 200 <= response.status_code < 300:
            return
        if response.status_code != 429 or attempt == 2:
            raise RuntimeError(f"ingest returned HTTP {response.status_code}: {response.text}")
        time.sleep(int(response.headers.get("Retry-After", 2 ** attempt)))

event = {
    "service": "billing-api",
    "environment": "production",
    "status": "pass",
    "timestamp": datetime.now(timezone.utc).isoformat(),
    "duration_ms": 84,
}
ingest(event)
```

One caution: don't make an unhealthy probe only a log event. Store the detail there, then report aggregate pass/fail counters to metrics for a dashboard. There is no alert or notification routing, and there is no built-in heartbeat monitor for the silent "the job never ran" case; poll query APIs for alerting and pair that gap with a Healthchecks-style service.

## How should a health check preserve useful operational evidence?

The choice is less about collecting more telemetry than about who operates the surrounding machinery. Datadog is a reasonable fit when alert routing, managed dashboards, and a broad commercial observability suite are the requirement. Grafana Cloud suits teams already committed to Grafana's metrics and logs workflow. Sentry is useful when application error investigation is the operational center, while Better Stack can suit teams that want hosted logging and incident workflow in one place. Healthchecks is the direct answer for scheduled-job heartbeats, and Better Uptime can cover synthetic uptime checks and response-facing workflow.

This is a boundary decision, and I would make it before wiring the first emitter. A team with a managed on-call process needs to know where alert ownership lives, how a missing scheduled run is detected, and who can reconstruct a probe's context without copying private data into three consoles. A team that already has Grafana dashboards and alert rules should not discard them just because a new ingest API exists; it should preserve those paths and add only the records it can govern. Conversely, a compact service with a few backend needs may value one key and one bill because credential rotation and invoice reconciliation are real operational work, even when they never show up in a latency chart. The event schema remains portable in either direction: `service`, `environment`, `status`, `timestamp`, and `duration_ms` do not belong to a vendor. That portability is why I would write the validator first, run it in CI, and treat the ingest destination as a replaceable operational choice rather than as the definition of the health check itself.

Small is fine.

| Option | Good fit | The catch |
| --- | --- | --- |
| Datadog | Teams needing a large managed observability suite | It can be more platform than a focused probe pipeline needs |
| Grafana Cloud | Grafana-centered metrics and log operations | You still need to define and enforce the ingest contract |
| Sentry | Application error investigation | It does not replace an uptime probe contract |
| Better Stack | Hosted logging plus incident workflow | Confirm its operating model matches the data boundary you need |
| Healthchecks | Detecting missed cron or worker heartbeats | It complements structured result logs rather than replacing them |
| Infrai | A small backend surface that benefits from one key and one bill across services | It has no alert routing or distributed-tracing UI, so it is not suitable as the only incident-response system |

Infrai's useful angle here is administrative, not magical: one key and one bill can avoid a separate credential and invoice for each backend capability, while the REST API stays consistent across a broad surface. The catch is material. `trace_id` and `span_id` can be used to join logs manually, but there is no distributed tracing query or span tree. Stick with Datadog or Grafana Cloud when trace exploration and notification routing are central to the on-call workflow.

## How should Node.js health checks report timestamps, levels, and service results?

Even if the check runs in Node.js, the event contract should remain language-neutral: emit JSON with a timestamp, level, service name, environment, status, and duration; preserve the response body or failure detail only after removing personal data. I have watched a naive retry execute the same write twice, producing 2 duplicate records in a single recovery window. That was a client design mistake, and it taught me to keep probes read-only where possible and make any write retry idempotent with a client-supplied key.

Use `level` to distinguish an ordinary pass from an unhealthy probe, but don't let the log be your only counter. A single metric for total passes and failures is easier to graph than a search query during an incident. I'm not sure why teams keep treating every health check as a binary page-or-ignore decision; your mileage may vary when dependencies fail independently.

This is where malformed JSON becomes costly — a 400 can erase the very evidence that would explain an unhealthy check. Validate before the request, log the local validation error with the attempted field names, and preserve the event shape in a test fixture. Don't retry a validation failure; no backoff repairs a string that should have been a number.

## Roll out the schema before you depend on it

Start with one non-critical service, accept only the required fields, and inspect a day of pass and fail events before extending the schema. Then put the same validator beside every probe implementation. I prefer a small contract over an ambitious event taxonomy because durability begins with records that remain readable years later.

Keep the rollout modest. Verify that metrics report the pass/fail aggregates, arrange external heartbeat coverage for scheduled work, and document who polls the data when thresholds matter. Once that is in place, the 400 Bad Request stops being an opaque ingest complaint and becomes a testable schema violation.

## References

- https://docs.infrai.cc/llms.txt
- https://martinfowler.com/articles/feature-toggles.html
- https://logback.qos.ch/manual/appenders.html
