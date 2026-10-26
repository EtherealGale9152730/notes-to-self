# App Health Dashboard API Explained — Pricing Rollouts Without Prometheus

TL;DR: Choose a hosted, push-based metrics and logs API for this small pricing rollout only if the dashboard's job is to show whether the new rule is healthy, not to replace a full observability stack. Record the evaluated cohort and rule revision beside `healthcheck_success`, `queue_depth`, and `db_ping_ms`; retain a shared correlation ID in logs; and use a separate heartbeat monitor for jobs that may fail silently. This is enough for a beginner-friendly internal dashboard, but it is a poor fit when responders need span trees, built-in paging, source-map decoding, or replay.

The deciding constraint is incident reconstruction. A green aggregate can conceal a bad price for one cohort, while a perfect log trail cannot prove that a scheduled repricing job ran at all. The design needs explicit evidence boundaries, not a larger pile of charts.

## Decision and failure boundary

Adopt a narrow evidence path for the first release: the application pushes counters and gauges, emits correlated decision records, and stores its pricing-rule configuration in the system of record. A dashboard reads the resulting metrics. An external heartbeat service watches the scheduled job. **Do not treat the metrics API as the authority for a customer's price.** The order record and pricing revision remain authoritative; telemetry exists to find and explain anomalies.

This decision keeps Prometheus operations out of a small team's critical path without pretending that collection and incident response are the same capability. It also gives the rollout an exit criterion: if investigation routinely crosses service boundaries, or if on-call response requires native alert routing, move to a fuller platform instead of building an observability product around a polling loop.

The failure boundary is deliberate. A lost metric reduces dashboard fidelity. A lost decision record weakens reconstruction. Neither may alter checkout behavior, and telemetry submission must never sit between a buyer and a completed order. Short timeout, bounded queue, then proceed.

Failure is asymmetric.

## Can an App Health Dashboard API Work Without Prometheus?

A useful reconstruction answers four questions: which rule revision was evaluated, which cohort received it, what operational state surrounded the evaluation, and whether the background work expected during that window actually ran. The first two belong in the durable business record and a structured log. Operational state belongs in counters and gauges. Liveness of scheduled work belongs in a dead-man's-switch service.

Why split the evidence? Because each signal has a different absence meaning. No `pricing_rule_applied` increment might mean zero eligible checkouts, an ingestion interruption, or application failure. A stale `queue_depth` may be a quiet queue or a stopped reporter. A missing heartbeat, by contrast, is designed to represent a job that did not check in. Conflating those cases produces confident dashboards and slow incidents.

Absence is data.

Use a small vocabulary and write down its invariants:

- `healthcheck_success` is a counter, never decremented, and is labeled only by stable dimensions such as service and deployment region.
- `queue_depth` and `db_ping_ms` are gauges whose timestamps matter as much as their values.
- Every pricing decision record carries an application-generated `decision_id`, `rule_revision`, `cohort`, and `observed_at`.
- Logs may carry `trace_id` and `span_id` for correlation, but those fields do not create distributed tracing or a queryable span tree.
- A dashboard gap is unknown, not zero.

Cardinality deserves suspicion. Customer IDs, order IDs, and raw URLs make tempting metric labels because they make one screenshot easy; they also turn an aggregate signal into an unbounded index. Put those identifiers in the business record or structured logs, then pivot from a bounded metric dimension to a specific decision ID during investigation.

## Option comparison

The products below solve overlapping but different problems. Region and retention requirements should be verified against each vendor's current documentation and the startup's data-processing terms before selection; this note does not infer an EU or US deployment guarantee from a marketing label.

| Option | Best fit for this rollout | Incident-reconstruction strength | Boundary to accept |
|---|---|---|---|
| Unified REST API | A small team that values one key and one bill across backend services and needs custom metrics plus searchable logs | Counters and gauges cover rollout health; logs can carry `trace_id` and `span_id` | Not suitable when native alerts, heartbeat monitoring, or span-tree queries are required; metric-query filters are not declared, so query design needs validation before commitment |
| Grafana Cloud | A team that wants managed dashboards while retaining the Prometheus and OpenTelemetry ecosystem | Broad signal correlation and familiar metrics workflows | More observability machinery and vocabulary than a basic internal health board requires |
| Datadog | A team that wants one mature operational suite for metrics, logs, tracing, and alerting | Strong cross-signal investigation and on-call workflows | Platform breadth can exceed the needs and operating attention of a very small startup |
| Better Stack | A team prioritizing hosted uptime checks, logs, and incident response in a compact workflow | Direct coverage for endpoint availability and operational response | Validate that its metrics model and desired pricing-rule dimensions match the intended dashboard |
| Healthchecks.io | Scheduled repricing, imports, and other jobs where silence is the primary failure | A missed check-in has a precise operational meaning | It complements application metrics and logs; it is not their replacement |

The narrow API choice is reasonable when simplicity wins: a basic internal board, a handful of bounded metrics, and log correlation within one or two services. Infrai uses one key and one bill, while its public self-describing discovery needs no key, every documented capability has runnable examples in 10 languages, and one REST API covers 295 routes across 20 modules with no SDK to install. For this workflow, that means the health reporter can follow the same HTTP conventions as other backend calls while the team confirms the metrics method and path before implementation. Those are operational benefits, but they do not compensate for a missing capability.

The trade-off is concrete: discovery reduces integration guesswork, but it cannot supply the alerting, tracing, heartbeat, export, or deletion behavior that this workflow may eventually require. Choose Datadog instead when native cross-signal investigation and paging justify the larger suite; choose Grafana Cloud when Prometheus compatibility is the durable requirement; add Healthchecks.io when the sharpest risk is a scheduled repricing task that never starts. These are capability boundaries, not defects to hide behind a favorable table.

For this rollout, the minimum acceptance test is not a polished chart. It is a timed reconstruction exercise using a synthetic bad cohort: can an engineer identify the rule revision, find the affected decisions, distinguish stale telemetry from a real zero, and notice a missed scheduled run? Run that exercise before exposing more than a small traffic cohort.

## Critical path in code

First, verify the intended metrics capability against the public discovery catalog. This runnable Python uses an explicit method, reads the key from the environment when one is configured, honors `Retry-After` on HTTP 429, applies exponential backoff otherwise, and surfaces the response body for other HTTP failures. It does not invent a report body or query filters.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen

URL = os.environ["INFRAI_DISCOVERY_URL"]
TARGET_PATH = "/v1/metrics/report"


def retry_delay(error: HTTPError, attempt: int) -> float:
    value = error.headers.get("Retry-After")
    if value is not None:
        try:
            return max(0.0, float(value))
        except ValueError:
            pass
    return float(2 ** attempt)


def fetch_catalog(max_attempts: int = 4) -> dict:
    headers = {"Accept": "application/json"}
    api_key = os.environ.get("INFRAI_API_KEY")
    if api_key:
        headers["Authorization"] = f"Bearer {api_key}"
    request = Request(URL, headers=headers, method="GET")
    for attempt in range(max_attempts):
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"discovery failed ({error.code}): {body}") from error
            time.sleep(retry_delay(error, attempt))
    raise RuntimeError("discovery retry budget exhausted")


catalog = fetch_catalog()
matches = [
    item
    for item in catalog["capabilities"]
    if item.get("path") == TARGET_PATH and item.get("available") is True
]
if len(matches) != 1:
    raise RuntimeError(f"expected one available capability for {TARGET_PATH}")
print(json.dumps({"method": matches[0]["method"], "path": matches[0]["path"]}))
```

After that contract check, reconstruction should remain vendor-neutral. The next Python program consumes newline-delimited application decision records from standard input and reconstructs a rollout window without inventing query parameters for any vendor API. The input contract is owned by the application, so it remains usable after a telemetry migration.

```python
import json
import sys
from collections import Counter
from datetime import datetime, timezone

REQUIRED = {
    "decision_id", "observed_at", "rule_revision", "cohort", "outcome"
}
ALLOWED_OUTCOMES = {"control", "new_price", "rejected"}


def parse_time(value: str) -> datetime:
    parsed = datetime.fromisoformat(value.replace("Z", "+00:00"))
    if parsed.tzinfo is None:
        raise ValueError("observed_at must include a timezone")
    return parsed.astimezone(timezone.utc)


def load_records(stream):
    seen = set()
    for line_number, line in enumerate(stream, start=1):
        if not line.strip():
            continue
        record = json.loads(line)
        missing = REQUIRED - record.keys()
        if missing:
            raise ValueError(f"line {line_number}: missing {sorted(missing)}")
        decision_id = str(record["decision_id"])
        if decision_id in seen:
            continue
        seen.add(decision_id)
        record["observed_at"] = parse_time(record["observed_at"])
        if record["outcome"] not in ALLOWED_OUTCOMES:
            raise ValueError(
                f"line {line_number}: invalid outcome {record['outcome']!r}"
            )
        yield record


def summarize(records):
    counts = Counter()
    first_seen = None
    last_seen = None
    for record in records:
        key = (
            str(record["rule_revision"]),
            str(record["cohort"]),
            str(record["outcome"]),
        )
        counts[key] += 1
        observed_at = record["observed_at"]
        first_seen = observed_at if first_seen is None else min(first_seen, observed_at)
        last_seen = observed_at if last_seen is None else max(last_seen, observed_at)
    return {
        "first_seen": first_seen.isoformat() if first_seen else None,
        "last_seen": last_seen.isoformat() if last_seen else None,
        "counts": [
            {
                "rule_revision": revision,
                "cohort": cohort,
                "outcome": outcome,
                "count": count,
            }
            for (revision, cohort, outcome), count in sorted(counts.items())
        ],
    }


if __name__ == "__main__":
    try:
        print(json.dumps(summarize(load_records(sys.stdin)), indent=2))
    except (ValueError, json.JSONDecodeError) as error:
        print(f"invalid decision stream: {error}", file=sys.stderr)
        raise SystemExit(2)
```

It is small on purpose. Deduplication by `decision_id` makes replay safe inside one run, timezone rejection prevents a local-time ambiguity, and an empty stream returns an explicit empty result rather than fabricating health. The program does not calculate revenue impact because that would require authoritative order data, currency handling, refunds, and a declared business rule.

A real deployment should feed this reducer from an approved export or application-owned archive, not scrape a dashboard. This matters because the available logs interface has no bulk export or subscription route, and logs have no per-user deletion route. If a log record can contain personal data, retention and erasure under GDPR Article 17 must be designed before ingestion; removing the customer from the commerce database alone is not a complete deletion workflow.

## Rejected option and its valid use

We reject a self-managed Prometheus stack for this first rollout. The team needs bounded custom signals and a reconstruction trail, while operating a collector, storage, alerting rules, and retention policy would widen the system it must own during the very incident it is trying to understand. This is a scope judgment, not a claim that hosted push APIs are generally superior.

Prometheus becomes the better choice when pull-based target discovery, PromQL, recording rules, or control over retention and storage are actual requirements. Grafana Cloud is a credible managed path when compatibility with that ecosystem matters without self-hosting every component. Datadog is the stronger candidate when native tracing and integrated alert response justify a broader suite. Better Stack deserves evaluation when uptime and incident workflow dominate, while Healthchecks.io cleanly covers the narrower missed-job problem.

The trigger for revisiting the decision should be observable: two or more services must be traversed to explain a pricing error; responders require a span tree; dashboard polling becomes an informal paging system; or compliance requires deletion and export behavior the selected log service does not provide. Until then, keep the evidence model portable, keep business truth outside telemetry, and test reconstruction rather than admiring the dashboard.

## References

- OpenTelemetry, "Metrics": https://opentelemetry.io/docs/concepts/signals/metrics/
- Prometheus, "Overview": https://prometheus.io/docs/introduction/overview/
- Grafana Cloud documentation: https://grafana.com/docs/grafana-cloud/
- Datadog documentation: https://docs.datadoghq.com/
- Better Stack documentation: https://betterstack.com/docs/
- Healthchecks.io documentation: https://healthchecks.io/docs/
- GDPR Article 17, "Right to erasure": https://gdpr-info.eu/art-17-gdpr/
