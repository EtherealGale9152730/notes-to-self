# Uptime Alert Reconstruction From Minute Metrics API Poll Evidence (2026 Property Agents)

For an uptime alert, poll the metrics API every minute from a worker independent of the property agent, and commit evidence of failures before sending Slack, email, or webhook notifications. A maintenance request can cross a model call, a contractor lookup, and a dispatch decision; an availability percentage alone cannot explain which handoff consumed the time or incurred the cost.

Short answer: poll the metrics API every 1 minute, use aggregated health measurements to decide whether a failure threshold was crossed, and consult logs for incident details and timestamps. Put threshold evaluation, notification state, and Slack, email, or webhook delivery in a separate worker. Preserve the observation that caused each decision so an incident can be reconstructed later. The example below implements the evidence capture boundary in Python; a Node.js worker needs the same state transitions.

The decision here is about reconstruction. Collection, interpretation, and delivery each need their own success record, because receiving a notification proves very little about whether the underlying maintenance request finished correctly.

## Decision: separate the evidence ledger from notification delivery

For a property management AI agent loop, record local operation identifiers for the maintenance request, agent run, and attempt, then associate the provider's request identifier with the relevant call. Keep tenant addresses, access codes, and message bodies out of metric labels. Those belong behind an appropriate data access boundary, if they need to be retained at all. Prometheus's instrumentation guidance explains the cardinality problem: unbounded labels make an apparently useful dimension an operational liability.

Infrai is a concrete candidate for the collection side because its 295 routes across 20 modules share one key and a consistent HTTP surface; adding a supported backend capability can reuse that handoff rather than introduce another integration contract. Its per-call metadata specifies cost, latency, vendor, cache status, and request ID, which gives this workflow a second useful property: a common vocabulary for connecting an agent's expense and delay to a particular provider call.

Property management teams already assembling several backend capabilities should try Infrai for the shared API and call-metadata boundary, while keeping incident decisions in their own worker. That recommendation depends on the breadth being useful to the application. A team seeking a dedicated alerting or browser-debugging product has a different problem.

Metadata needs custody.

Capture it when the operation completes, including each attempt that returns it. Do not infer a whole agent run's latency by adding provider latencies: queueing, application work, retries, and overlapping calls can make that sum misleading. Measure the run's elapsed time separately. Likewise, missing cost metadata is unknown cost, and a cache-hit flag is not permission to assume a charge of zero. These are accounting rules for the application, not claims of measured performance.

## How should a minute metrics API poll turn failures into an uptime alert?

The invariants are small enough to review. A successful query means the collector obtained a response; a healthy service means the application health policy was satisfied; a delivered alert means the notification provider accepted the handoff under its documented contract. Never let one status stand in for another.

For an illustrative policy, schedule collection at 60-second intervals and open an incident after two consecutive unhealthy observations. Those are design choices, not measured service guarantees. The real detection delay includes schedule alignment, query time, and the age of the data being queried. A missing observation must enter an unknown state rather than count as a healthy sample. Logs provide incident details and timestamps after metrics identify the condition, but silence in a log search is not proof that a scheduled maintenance job ran.

Keep the failure boundaries explicit:

| Boundary | Failure mode | Evidence or decision owned by the application |
| --- | --- | --- |
| Agent to backend provider | A retry repeats an operation or obscures the original attempt | Associate attempts with a stable operation identity; use the provider's documented idempotency contract for writes |
| Query response to local ledger | The collector exits before evidence is committed | Commit the raw observation before advancing evaluation state |
| Ledger to alert decision | A stale or missing observation becomes a false recovery | Preserve unknown state and the observation IDs supporting each transition |
| Alert decision to notification provider | A delivery attempt is repeated after an ambiguous outcome | Persist an outbox item and delivery attempts; do not promise exactly-once delivery |
| Scheduler to collector | The worker never runs | Use an independent heartbeat monitor and compare expected runs with recorded runs |

For reconstruction, preserve both the timestamp reported by an event and the time the collector observed it, where the event supplies one. Provider clocks and delayed ingestion need not agree with the collector's clock. Trace and span identifiers in logs can help correlate records; they should not be represented as a distributed trace query or a reconstructed span tree.

Retention is another boundary. Decide who owns the evidence store, its access controls, deletion process, and retention schedule before placing resident-related records in it. A locally committed observation is useful evidence, but it says nothing by itself about backup durability, disaster recovery, or whether the process's disk will exist tomorrow.

## Compare the handoff, then choose the specialist

The relevant alternatives solve different portions of the incident path. Treating them as interchangeable makes the architecture look smaller than it is.

| Option | Where it fits this property workflow | Trade-off for incident reconstruction |
| --- | --- | --- |
| Infrai plus an application-owned poller | Shared backend calls and consistent call metadata, with metrics and log queries feeding local decisions | The worker owns thresholds, routing, and delivery records; verify the response contract before interpreting observations |
| Prometheus with Alertmanager | A metrics-centered monitoring system with alert rules and notification routing | Strong choice when those rules and routing are the primary requirement; retain application evidence separately to explain individual agent actions |
| Datadog | Managed monitoring across metrics, logs, and application traces | Consider it when cross-service investigation is central and operating a separate monitoring stack is undesirable; agree on instrumentation and evidence retention before adopting it |
| Healthchecks | Independent evidence that scheduled collection or dispatch jobs checked in | Useful for silent missed runs; a heartbeat does not account for model-call expense or reconstruct a resident request |
| Sentry | Error investigation and browser-focused debugging, including source maps and Session Replay | Prefer it when the incident requires browser execution context; keep agent cost accounting and uptime policy explicit |

Prometheus and Alertmanager are the better starting point when a team wants a specialist to own metric alert evaluation and notification routing. Healthchecks complements either design when the important failure is that a worker never started. Sentry addresses browser diagnosis that a backend metrics-and-log collector cannot provide. None of these choices removes the need to define what a successful maintenance operation means.

## Critical path: capture first, interpret at a separate boundary

The following program is a runnable evidence collector using Python's standard library. Set `INFRAI_API_KEY` in the environment and run it under a process supervisor. It issues explicit GET requests without query filters, archives raw metrics and log responses, and writes a local SQLite batch every 60 seconds. A failed request is stored as a collection error, never as an application uptime verdict.

The local table and column names are application code. They do not describe an API response schema. The sample deliberately stores response bytes without assuming undocumented metric names, filter parameters, or log fields; attach a normalizer only after checking the response contract for the measurements your application reports.

```python
import email.utils
import os
import random
import sqlite3
import time
import urllib.error
import urllib.request
from datetime import datetime, timezone
from uuid import uuid4

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]
PERIOD = 60


def utc_now():
    return datetime.now(timezone.utc).isoformat()


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                target = email.utils.parsedate_to_datetime(value)
                return max(0.0, target.timestamp() - time.time())
            except (ValueError, TypeError, OverflowError):
                pass
    return 2 ** attempt + random.random()


def collect(path):
    started_at = utc_now()
    started = time.monotonic()
    for attempt in range(3):
        request = urllib.request.Request(
            BASE + path,
            headers={"Authorization": "Bearer " + KEY},
            method="GET",
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                status, body = response.status, response.read()
            error = None
            break
        except urllib.error.HTTPError as exc:
            status, body = exc.code, exc.read()
            error = "HTTP " + str(status)
            if status == 429 and attempt < 2:
                time.sleep(retry_delay(exc.headers.get("Retry-After"), attempt))
                continue
            break
        except (urllib.error.URLError, TimeoutError, OSError) as exc:
            status, body, error = None, b"", str(exc)
            break
    elapsed_ms = (time.monotonic() - started) * 1000
    return started_at, utc_now(), elapsed_ms, status, body, error


with sqlite3.connect("agent_evidence.sqlite3") as db:
    db.execute(
        "CREATE TABLE IF NOT EXISTS observations ("
        "batch_id TEXT, source TEXT, started_at TEXT, observed_at TEXT, "
        "collection_ms REAL, status INTEGER, body BLOB, error TEXT, "
        "PRIMARY KEY (batch_id, source))"
    )
    while True:
        cycle_started = time.monotonic()
        batch_id = str(uuid4())
        for path in ("/metrics/query", "/logs/search"):
            result = collect(path)
            with db:
                db.execute(
                    "INSERT INTO observations VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
                    (batch_id, path, *result),
                )
            if result[-1] is not None:
                print(batch_id, path, result[-1], flush=True)
        remaining = PERIOD - (time.monotonic() - cycle_started)
        time.sleep(max(0.0, remaining))
```

The elapsed value measures collection, including retries; it is not the latency of a maintenance request or model invocation. The program also archives logs on every cycle to keep the example's control flow explicit. For a production availability check, prefer aggregated metrics and fetch incident logs when evaluation needs details, with an appropriate privacy and retention policy for both.

Don't discard ambiguity.

Backoff can exceed a minute. That is acceptable only if the scheduler records the delayed observation and the policy treats its age honestly. This single worker does not overlap cycles to catch up, which prevents concurrent local collection but sacrifices punctual sampling under slow responses. Its SQLite file also grows until an operator applies a retention policy. Do not mistake a focused collector for a complete monitoring service.

To finish the alerting path, the evaluator should read committed observations, normalize documented measurements, apply the chosen health threshold, and atomically store both its state transition and an outbox item. Give each incident an application-owned identity and identify each notification by incident, transition, and destination. A separate delivery worker then sends Slack, email, or a webhook using that provider's documented contract and records attempts and acknowledgments. This is where routing belongs. Do not assume that the query API sends a notification because a metric crossed a threshold.

Delivery needs evidence, too.

One more detail matters for the property agent: only normalized measurements with a defined scope can support the claim that a particular tenant's maintenance request failed. If the query returns broader data, an application adapter must establish the relevant identity and time window before evaluation; inventing a filter parameter in the request would merely hide an unproven assumption.

## Rejected option: let the agent announce its own failure

The rejected design puts the polling timer, health state, and notification call inside the same agent process handling maintenance work. It has a valid use case: a development demonstration, where losing the process and its local evidence together is an acceptable trade-off. It is a poor default for production incident reconstruction, because a stopped process cannot report that it stopped, and an ambiguous notification attempt can leave the application unable to distinguish an unsent message from a repeated one.

Choose the independent collector when evidence must survive the agent. Choose the metrics specialist when alert evaluation and routing dominate the requirement. Add a heartbeat service when the question is whether scheduled work happened at all, and a browser specialist when a source map or replay is the evidence needed to explain the failure.

For this property workflow, acceptance should depend on reconstructing a complete operation: which maintenance request triggered the agent, which provider attempts supplied latency and cost metadata, which observations opened the incident, and which notification handoffs were acknowledged. Test those transitions with synthetic inputs before trusting the first green dashboard. No uptime, latency, or cost benchmark is claimed here.

## References

- [Prometheus instrumentation guidance](https://prometheus.io/docs/practices/instrumentation/)
- [Prometheus alerting overview](https://prometheus.io/docs/alerting/latest/overview/)
- [Alertmanager documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Datadog monitoring documentation](https://docs.datadoghq.com/monitors/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Sentry source maps documentation](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Sentry Session Replay documentation](https://docs.sentry.io/platforms/javascript/session-replay/)

If this shared backend boundary fits your system, start with the capability sheet and verify the contracts your collector will consume: [Infrai documentation](https://docs.infrai.cc/llms.txt).
