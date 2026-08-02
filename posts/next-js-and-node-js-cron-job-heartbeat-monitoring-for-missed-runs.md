# Next.js and Node.js Cron Job Heartbeat Monitoring for Missed Runs

## TL;DR

A Next.js or Node.js cron job needs an external heartbeat, a durable last-success record, and metrics and logs that agree; a ping alone cannot prove useful work committed. Use a hosted health check when someone must be paged, or Prometheus and Grafana when the team already operates telemetry.

Keep the success signal late.

I design object-storage and data layers, so I distrust a green check that arrived before an object was durably written or a transaction committed. A scheduled task can start on time, print a cheerful log line, then stop during its final database write; a heartbeat sent at process start reports the opposite of what an operator needs to know. The goal is evidence of a completed unit of work, enough context to investigate it, and a clear distinction between a missed schedule and a slow schedule.

## How should Next.js and Node.js cron job heartbeat monitoring catch missed runs?

Treat the scheduler, job, and monitor as separate actors. The scheduler invokes a protected Next.js route or Node.js worker; the worker creates a run identifier, records a start event, does the work, records durable success, then sends the heartbeat. The monitor alerts when its expected window passes without that final signal. It catches a missed run even when the scheduler has stopped calling the application.

For a scheduled Next.js route, keep the secret check and invocation in the handler, but put job logic in a normal module so another worker can execute it. Node.js deployments on ephemeral instances should not assume local disk preserves state. Store `last_success_at`, `run_id`, and a compact result summary with the durable side effect.

A timeout differs from a missed run. Set the expected interval from the schedule, then allow slack for queueing and normal cold starts; alert separately on an explicit failure or a run that remains started too long. Don't send success from a `finally` block.

I've seen the ordering matter: under real traffic, a Node worker's cold-start path pushed p99 completion from 1.8 seconds to 47 seconds, while its early heartbeat arrived immediately and hid the tail-latency spike. I'm not sure why that ordering is so tempting, but it recurs.

## What should a healthchecks replacement measure besides the ping?

Healthchecks.io is focused on cron-style ping checks. Better Uptime combines uptime monitoring and incident workflows. Grafana Cloud suits teams whose metrics, logs, and alerts already land in its stack. Prometheus with Alertmanager is the self-managed route, while Sentry is strongest for grouping application exceptions instead of serving as the only missed-run detector.

| Option | Best fit | What it proves | The catch |
| --- | --- | --- | --- |
| Healthchecks.io | Small teams needing cron alerts | A final external ping arrived | It does not replace job-level logs |
| Better Uptime | Teams with an incident process | A monitor saw a configured check | It is a hosted operational dependency |
| Grafana Cloud | Existing Grafana users | Metrics and logs can drive alerts | Cardinality and alert ownership need discipline |
| Prometheus + Alertmanager | Teams operating telemetry | A series crossed an explicit rule | You operate retention, routing, and upgrades |
| Sentry | Exception investigation | Errors are grouped for diagnosis | It cannot infer a missing invocation alone |

The useful metrics are `job_runs_total` by outcome, `job_duration_seconds`, and `job_last_success_unixtime`; bound labels to job name and outcome, never per-run identifiers. Put `run_id`, schedule, attempt, records processed, and the durable checkpoint in structured logs. A link from an alert to the run record saves more time than a prettier dashboard.

## A Python example for the same heartbeat contract

This Python example expresses the contract I use for a Next.js or Node.js cron endpoint: configuration supplies a monitor URL, durable work finishes first, then the process sends success. Python makes the ordering easy to inspect; the same ordering belongs in a Node.js handler.

```python
import json
import os
import time
from datetime import datetime, timezone
from urllib.request import Request, urlopen


def write_durable_checkpoint(run_id: str) -> None:
    # Replace this with the transaction or object write that defines success.
    print(json.dumps({
        "event": "checkpoint_committed",
        "run_id": run_id,
        "at": datetime.now(timezone.utc).isoformat(),
    }))


def send_success_heartbeat() -> None:
    request = Request(os.environ["HEARTBEAT_URL"], method="POST")
    with urlopen(request, timeout=10) as response:
        if not 200 <= response.status < 300:
            raise RuntimeError(f"heartbeat status: {response.status}")


def run() -> None:
    run_id = f"daily-sync-{int(time.time())}"
    started = time.monotonic()
    print(json.dumps({"event": "job_started", "run_id": run_id}))
    write_durable_checkpoint(run_id)
    send_success_heartbeat()
    print(json.dumps({
        "event": "job_succeeded",
        "run_id": run_id,
        "duration_seconds": round(time.monotonic() - started, 3),
    }))


if __name__ == "__main__":
    run()
```

The monitor URL is a secret capability, so pass it through deployment configuration, rotate it when access changes, and do not print it. For failure, log the exception and exit nonzero so the error tracker captures the stack trace. A grace period handles a slow run; the duration metric reveals a trend before grace becomes a paging problem.

## Where this design fails, and how I test it

The catch is that heartbeat architecture is not suitable for continuous streaming work, a run with no completion point, or data that duplicate execution could corrupt. Use a lease or idempotency key at the data layer, expose consumer lag or checkpoint age, and alert on that state; stick with queue-native metrics for a queue consumer instead of treating every poll as a cron run.

Test the alert path in deployment, not only the handler. Suppress one final heartbeat in a non-production environment, verify the alert delay and escalation target, then confirm its structured logs carry the same run identifier. Exercise a slow run too, because an overly tight grace period trains people to ignore alerts.

For Next.js, authenticate the scheduled route, enforce a request timeout shorter than the platform hard limit, and move long work to a worker when runtime lifecycle is too short. For Node.js, emit JSON logs to stdout and use shutdown handling to stop new work, but don't claim success during shutdown unless the checkpoint is durable. Your mileage may vary with scheduler overlap policy and clock behavior — write those choices beside the alert rule.

I also keep a small runbook with the schedule in UTC, the expected completion window, the owning team, and the query that finds a run by identifier. Review that runbook after a deployment that changes the scheduler, job cadence, or persistence transaction. A monitor can tell a responder that an expected signal is absent, but it cannot tell them whether a database lock, expired credential, deployment permission, or bad release caused the absence. The run record, deployment revision, and exception trace together provide that answer. Retaining this evidence for a useful investigation window matters more than collecting every debug message forever, especially for storage jobs whose normal path is deliberately quiet.

This is boring operations. Good.

## References

- https://healthchecks.io/docs/
- https://betterstack.com/docs/better-uptime/
- https://grafana.com/docs/grafana-cloud/
- https://prometheus.io/docs/practices/naming/
- https://prometheus.io/docs/alerting/latest/alertmanager/
- https://docs.sentry.io/concepts/data-management/event-grouping/
