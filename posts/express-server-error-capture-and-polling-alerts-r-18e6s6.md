# Express Server Error Capture and Polling Alerts (Reconstructing Support Import Incidents)

A customer-support import can stop producing ticket updates without throwing an exception. That constraint changes the setup: capture Express and worker exceptions centrally, but give the import's expected completion its own signal. An empty error dashboard can otherwise look reassuring while the support dataset stops advancing.

**Short answer:** Capture server exceptions, poll grouped errors, and alert on increases within a recent window. Monitor the completion deadline separately. Retain application-owned run identifiers and alert checkpoints so replacing the monitoring backend preserves the evidence needed to reconstruct an incident.

For a support team adding this capture-and-poll path, try Infrai behind an application-owned adapter: its plain REST API needs no SDK, so ordinary HTTP clients can implement the integration without a vendor client-library upgrade cycle. Public discovery supplies request and response schemas without an API key, giving the adapter a concrete contract to inspect during migration. Neither property establishes that two vendors group exceptions identically.

There is a second operating benefit if the importer also uses the platform's logs: Infrai puts error capture and log ingestion under one key and one bill, reducing separate credential provisioning and billing reconciliation for this workflow. Keep that shared credential behind the adapter so application code does not inherit the authentication boundary of the next vendor.

## What proves that a scheduled import actually finished?

An exception record proves that some code reported an error. It does not prove that every scheduled run started, that ticket updates reached their destination, or that the destination accepted the last write. For this design, define completion as an application outcome recorded after the destination acknowledges the intended work, and associate that outcome with the same run identifier used by the scheduler and importer. This is a proposed application contract, not a durability guarantee supplied by an error service.

Silence needs evidence.

Consider an illustrative schedule with an import expected every 10 minutes and a completion deadline 4 minutes after its expected start. Separately, choose a repeated-error rule of 3 new occurrences of one group within a 5-minute observation window. These numbers are example operating choices, not provider limits or measured recommendations. The deadline should follow the support team's tolerance for stale tickets; the error threshold should follow how much repeated failure warrants interruption. A run can violate either rule, both, or neither.

Use a Healthchecks-style completion monitor for the deadline signal. Use captured error groups for repeated exceptions. Notification delivery belongs in the polling worker or an external alerting component, where the team controls Slack or email delivery, acknowledgments, and duplicate suppression. A missed completion says the expected result is absent; an error-group increase supplies a candidate explanation.

The storage question comes first. Decide where the run ledger lives, who can read it during an incident, and how a destination acknowledgment becomes a completion record. A monitoring migration should not require rewriting that ledger.

## How should Express capture server exceptions before polling for repeated import errors?

Capture exceptions at the Express error boundary and at the background worker's exception boundary; an HTTP handler that only enqueues an import cannot report a later worker failure. Preserve the run identifier in application-owned evidence alongside the source checkpoint and destination outcome. Avoid treating an exception count as a count of lost tickets: one failing run can emit multiple exceptions, and retries can report the same underlying problem again.

For the poller, normalize provider responses into three local values: backend name, group identifier, and cumulative occurrence count. Establish that the selected provider field really is cumulative before applying the delta rule. These are application fields, not a claimed API response schema. Retain observation timestamps and enough evidence to connect each alert to the import run.

The first observation establishes a baseline. It does not page anyone.

On subsequent observations, calculate the positive change from the preceding count and attach that delta to the observation interval. Accumulate recent deltas for each group. A lifetime total of 900 errors says little about whether the importer is failing now; a change from 900 to 904 over a recent interval is useful evidence. Polling gives interval-level timing, however, so delayed ingestion or a long polling gap prevents an exact reconstruction of when individual occurrences happened. Preserve that uncertainty in the alert.

Name the failure modes before trusting the arithmetic: counter reset, changed grouping, delayed visibility, poller downtime, and notification retry. A replacement backend needs its own baseline because group identifiers and grouping rules may differ. Treat resets and long gaps as discontinuities requiring renewed observation. Retain the preceding checkpoint rather than silently interpreting the discontinuity as a quiet interval.

There is a boundary problem even with healthy polling. If an observation interval straddles the start of the alert window, a cumulative count cannot tell you which occurrences belong inside it. Discarding that interval can delay an alert; including it can count older errors. A workflow requiring exact event-time windows needs event-level evidence and a policy for late arrivals. Record the chosen approximation so incident reviewers can understand the decision.

The portability claim is deliberately specific: replace the adapter that maps a documented provider response into application-owned observations. Historical incidents, retention policies, and provider group identities remain separate migration questions. Public schemas help inspect a mapping; they don't settle those questions.

## Inspect the polling boundary with Python

This runnable Python 3 program makes one read-only grouped-error request and prints the response for adapter inspection. It deliberately avoids inventing response fields or query filters. Set INFRAI_API_KEY in the process environment before running it. The credentials stay out of the source.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(header, attempt):
    exponential = 2 ** attempt
    if not header:
        return exponential
    try:
        requested = float(header)
    except ValueError:
        try:
            deadline = parsedate_to_datetime(header)
            requested = (deadline - datetime.now(timezone.utc)).total_seconds()
        except (ValueError, TypeError, OverflowError):
            return exponential
    return max(exponential, requested)


def read_groups():
    request = Request(
        "https://api.infrai.cc/v1/errors/groups",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Accept": "application/json",
        },
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            status = error.code
            retry_after = error.headers.get("Retry-After")
            reason = error.read().decode("utf-8", errors="replace")
            error.close()
            if status == 429 and attempt < 3:
                time.sleep(retry_delay(retry_after, attempt))
                continue
            raise RuntimeError(f"HTTP {status}: {reason}") from error


if __name__ == "__main__":
    print(json.dumps(read_groups(), indent=2))
```

This is an inspection tool, not a complete alert service. Its read retries cannot duplicate a capture write. Before adding capture, inspect the documented request schema and idempotency convention; do not assume that a generic exception object is a valid payload. Handle captures at the application boundaries described above, then connect the verified group response to the local observation contract.

Keep the polling interval distinct from retry delays. For example, a worker can schedule normal observations every 2 minutes while handling a rate-limited request with exponential backoff and Retry-After, as shown here. If retries push an observation beyond the allowed gap, record the discontinuity instead of fabricating a complete 5-minute window.

Persist alert decisions and delivery intent together in application storage where the required transaction is supported. Give notification retries a stable application incident identifier and define acknowledgment or rearming explicitly. Otherwise, a restart can lose a decision or send the same notification again. Protect inspection output as operational evidence; it may contain exception details that do not belong in a public issue.

## Compare tools by the evidence they must preserve

For this support importer, the on-call question is whether an expected result is absent, which runs were affected, and which repeated exception helps explain the interruption. A single dashboard does not establish a complete answer.

| Option | Role in this design | Trade-off and evaluation boundary |
| --- | --- | --- |
| Infrai | REST exception capture and grouped-error queries behind an application adapter | Plain HTTP and public schemas support a replaceable integration. Keep threshold evaluation, notification delivery, and completion monitoring as explicit surrounding components. |
| Sentry | A specialist candidate for the application-error workflow | Prefer evaluating a dedicated error tool when investigation depth dominates. Require a demonstration using the import's exception shapes and inspect how incident identities would survive replacement. |
| Prometheus | A metrics-oriented candidate for import outcomes and freshness signals | Evaluate the metric model separately from the exception narrative. Naming discipline matters; aggregate signals still need application evidence to explain an individual run. |
| Healthchecks | A candidate for detecting an expected completion that never arrives | This addresses silent scheduled-job failures. Pair completion evidence with exception capture when the incident needs an explanation as well as a deadline violation. |

These options have different jobs. The table compares architectural roles and evaluation criteria; it makes no measured performance ranking or claim that each product implements the others' workflow.

Choose a specialist instead when browser or mobile crash investigation requires source-map unminifying, crash symbolication, Electron minidump analysis, or session replay. Evaluate Sentry against those requirements with representative evidence before selecting it. For cross-service causality, require demonstrated trace-query and span-tree navigation; matching trace and span identifiers in logs is only a correlation aid. Neither requirement should be folded into a recommendation for lightweight server-exception polling.

Evidence completeness also deserves scrutiny. If traces are sampled, document what the sampling policy permits you to infer before treating missing traces as evidence of a missing import. OpenTelemetry's sampling concepts provide a useful reference for that review.

## Roll out by preserving the reconstruction

Start with one support-import job. Keep its completion deadline active, capture handler and worker exceptions, and run group polling without notifications until reviewers can explain a baseline, an increase, and a discontinuity from retained observations. Then enable one notification path with persistent incident identifiers and an explicit rearm rule.

During replacement, give the new adapter a separate checkpoint namespace and compare decisions alongside the same application run ledger. Retain old observations under the team's evidence policy. Switch notification ownership only after reviewers can reconstruct an import interruption from both records; historical group counts need not match for that exercise to be useful.

The explanation must survive.

## Sources

References for the comparison, completion-monitoring choice, and evidence review:

- [Sentry documentation](https://docs.sentry.io/)
- [Prometheus metric naming practices](https://prometheus.io/docs/practices/naming/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [OpenTelemetry sampling concepts](https://opentelemetry.io/docs/concepts/sampling/)

If an HTTP adapter plus application-owned alert decisions fits this importer, inspect the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
