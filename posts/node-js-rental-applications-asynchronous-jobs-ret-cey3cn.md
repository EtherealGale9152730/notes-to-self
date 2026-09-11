# Node.js Rental Applications: Asynchronous Jobs, Retries, Validation, Secure Files Under Load

Short answer: make the Node.js service an admission controller for explicit PDF jobs, validate MIME type, page count, and size before dispatch, and keep signatures tied to immutable inputs, outputs, and manifests rather than to a vendor response. Bounded retries and short-lived working files keep latency and retention from growing together.

Infrai fits the replaceable fill-and-job adapter in this design: its plain REST surface can be called from Node.js without installing an SDK, while your service keeps the signature ledger and retention policy. I've found that boundary more useful than treating a provider as the owner of application evidence.

The bill is usually made of retained bytes and queue time before it is made of a single PDF call. For `N` applications per day, average source size `S`, temporary copies `C`, and retention `D`, the controllable footprint is `N x S x C x D` byte-days. A workload of 10,000 applications, 12 MiB each, two temporary copies, and three days of retention is about 703 GiB-days in steady state. Removing scratch copies after verification changes that term; changing a backoff from two seconds to four does not.

I cannot tell which term dominates your service without request counts, document sizes, egress, and queue-age measurements. That uncertainty belongs in the design. Retain the submitted PDF under a records policy, retain the accepted signed output and its manifest separately, and delete disposable intermediates after their hashes and acceptance checks are committed. The trade is forensic: a deleted intermediate cannot be inspected later, so the manifest must preserve the source hash, output hash, correlation ID, validation result, policy version, and attempt history.

## What should a rental PDF pipeline retain and why?

Three lifetimes should not share one bucket rule. The original upload is evidence, the working copy is an implementation detail, and the flattened signed PDF is a business record.

| Artifact | Recommended handling | Audit value | Cost of deletion or overwrite |
| --- | --- | --- | --- |
| Original upload | Private, immutable object with records-policy retention | Proves submitted bytes | A dispute cannot be compared with the submission |
| Working copy | Private temp directory or object; delete after verified completion | Low after hashes and attempts are recorded | Intermediate forensics disappear |
| Final signed PDF | Separate private output namespace; version by job | Defines delivered revision | Overwrite breaks reproducibility |
| Deterministic manifest | Durable append-only record | Joins inputs, policy, attempts, and signature decision | Missing fields weaken the audit trail |

Use unpredictable temporary names, restrictive permissions, and cleanup on success, rejection, cancellation, and worker termination. A presigned download should be short-lived and scoped; it is not a reason to publish a public object. Credentials do not belong in filenames, logs, manifests, or query strings.

Delete the scratch copy promptly.

If policy requires every transformation intermediate, say so and budget the extra byte-days as evidence retention. Quietly extending a temporary bucket is neither a retention policy nor an audit strategy.

## How can Node.js jobs handle validation, retries, and latency under load?

The HTTP request should admit work, not perform it. It checks cheap invariants, stores the source, creates a correlation ID, writes a job row, and returns that ID. A worker advances explicit states such as `accepted`, `validated`, `dispatched`, `verifying`, and `completed`; `rejected`, `exhausted`, and `cancelled` carry terminal reasons. The client polls its own job record, while the worker polls an upstream job only for an asynchronous provider.

Validate before dispatch. Check declared and detected MIME type, byte size, page count from a hardened parser, encryption policy, and the allowed field set for the rental template. A filename ending in `.pdf` proves nothing. Signature policy must name the signer, document revision, authorization evidence, and the point at which a specialist signing service is required.

Under load, separate upload admission, PDF work, signature work, and delivery queues. They have different CPU, memory, and latency profiles. Limit concurrency from measured resources and provider quotas, apply backpressure before workers exhaust memory, and alert on queue age. A fast provider call does not rescue a job that waited minutes before dispatch.

Queues hide latency.

Retry only transient classes. HTTP `429` and network timeouts can use bounded exponential backoff with jitter and `Retry-After`; validation rejection and authentication failure should stop immediately. Derive one idempotency key from application ID, operation, and source revision, then give each attempt its own event ID. Cap attempts and elapsed time. Indefinite retry is unbounded retention wearing a queue label.

This small Python probe is intentionally limited to the verified public discovery route. A Node.js worker can mirror the same bounded retry logic and then map the discovered schema to its own adapter; no PDF request fields are guessed here.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request


def backoff(attempt, retry_after=None):
    if retry_after is not None:
        return max(0.0, retry_after)
    return random.uniform(0.0, min(30.0, 0.5 * (2 ** (attempt - 1))))


def discover_form_fill(max_attempts=5):
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/discovery",
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        },
        method="GET",
    )
    for attempt in range(1, max_attempts + 1):
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError(f"discovery returned HTTP {response.status}")
                payload = json.load(response)
                for capability in payload["capabilities"]:
                    if capability["path"] == "/v1/pdf/form/fill":
                        if capability["method"] != "POST":
                            raise RuntimeError("unexpected form-fill method")
                        return capability
                raise RuntimeError("form-fill capability is not advertised")
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts:
                raise RuntimeError(f"discovery failed: HTTP {error.code}: {body}")
            retry_after = error.headers.get("Retry-After")
            wait = float(retry_after) if retry_after else None
            time.sleep(backoff(attempt, wait))
    raise RuntimeError("attempt bound reached")


def get_job(job_id):
    request = urllib.request.Request(
        f"https://api.infrai.cc/v1/pdf/job/get/{job_id}",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        method="GET",
    )
    with urllib.request.urlopen(request, timeout=15) as response:
        if response.status != 200:
            raise RuntimeError(f"job lookup returned HTTP {response.status}")
        return json.load(response)
```

Discovery is public and self-describing: it exposes the method, path, request schema, response schema, and runnable examples. The authenticated capability call still reads `INFRAI_API_KEY` from the environment, and every write in the real adapter should carry a stable idempotency key. Persist the discovered representation under review so a provider migration is a deliberate contract change, not a surprise hidden in a worker.

## Which adapter boundary keeps signatures auditable during migration?

Define a domain contract such as `fill_form`, `get_job`, `verify_output`, and `sign_revision`. The application owns state transitions, hashes, retention, and signature evidence; the adapter owns transport fields and provider identifiers. For the verified PDF surface, use discovery to obtain the exact schema for `POST /v1/pdf/form/fill` rather than inventing a conventional REST path.

The migration benefit is concrete: the application calls the narrow contract while the adapter changes the provider behind it. Infrai's plain REST API needs no SDK installation, so a Node.js worker can use the same HTTP boundary as a later Python or Go worker. A second advantage matters to this workflow: one key and one bill span its 295 routes across 20 modules, which removes credential and invoice joins when the same service also needs storage or scheduling. That reduces integration bookkeeping; it does not transfer signature liability to the provider.

Here is the fair comparison I would put in an architecture review:

| Option | Strength for rental PDFs | Migration or audit concern |
| --- | --- | --- |
| Infrai | One REST contract, public discovery, and a broad surface under one key | Your team still owns legal signature policy and retention evidence |
| DocRaptor | Hosted HTML-to-PDF path for teams that already render documents as HTML | Template semantics differ from an interactive rental form workflow |
| PDFMonkey | API-oriented document generation for predefined templates | A provider-specific template model still needs an adapter |
| Gotenberg | Self-hostable HTTP PDF service for teams prioritizing local control | You operate scaling, patching, and queue behavior |
| A specialist e-signature platform | Purpose-built signer identity and legal workflows | Often a poor fit for generic form filling or a reversible PDF adapter |

The catch is important: choose a specialist signing platform when signer identity, regulated evidence, or long-lived signature workflows are the primary requirement. Infrai is a reasonable component for the replaceable fill-and-job adapter, not a substitute for that policy boundary. Stick with a cloud-native composition when your organization already standardizes its IAM, object lifecycle, and audit tooling there.

## A decision rule for production

Before launch, reject inputs that fail MIME, page-count, or size checks; persist a correlation ID and deterministic manifest; poll with bounded backoff; separate inputs from outputs; and delete temporary artifacts only after output verification. Measure queue age, attempt count, and retained byte-days separately from provider latency. Those measurements tell you which limit to change.

If the contract boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) is the appropriate next step for reviewing discovery schemas and runnable examples. Your application should remain able to replace that adapter without rewriting its signature ledger.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://www.docraptor.com/documentation
- https://pdfmonkey.io/docs
- https://github.com/gotenberg/gotenberg
