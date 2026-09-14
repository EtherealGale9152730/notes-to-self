# Node.js Service for Large Case Files: Asynchronous Jobs, Retries, Validation (7 Rules)

Large case files are a storage and scheduling problem before they are a PDF problem. For a B2B SaaS service, the decision rule is straightforward: validate every input before enqueueing it, make the PDF operation an explicit asynchronous job, and keep auditable output separate from short-lived working files.

Short answer: use a correlation ID, bounded exponential polling, deterministic manifests, and a retention policy that treats temporary files as toxic data. Batch throughput improves when workers can retry safely, while privacy improves when the input and output lifecycles are different by design.

## The invariants I would put in the decision record

The system has four invariants. First, a file that fails MIME, page-count, or size validation never reaches the PDF provider. Second, one case has one correlation ID that appears in the queue record, job record, manifest, and audit event. Third, a retry cannot create a second logical result. Fourth, the original upload, intermediate fragments, and final deliverable have explicit owners and expiry times.

That is the whole contract.

Keep it boring.

Those rules matter more than a particular vendor. A 900-page case can occupy a worker for minutes, and a burst of cases can turn an apparently synchronous endpoint into a queueing system. Treat the intake endpoint as a fast admission controller: stream the upload to a private temporary location, calculate metadata, and return a correlation ID once validation passes. The worker owns the slow PDF call.

The manifest should be deterministic. Record a content hash, byte size, MIME result, page count, template or operation name, validation-policy version, correlation ID, attempt number, and timestamps. Do not put the PDF itself or sensitive form values in logs. A manifest is useful precisely because it lets an auditor reproduce the decision without granting access to the case contents.

## How should a service handle large case files, retries, validation, and privacy?

Validation is a gate, not a warning. Check the declared content type against a signature check, reject a page count outside the product limit, and reject a byte stream over the configured maximum before making a job request. The declared type alone is not evidence; a file called `case.pdf` can still be something else. Keep the policy version in the manifest so a later policy change does not make an old decision impossible to explain.

Retries need a budget. Use exponential delays with jitter, cap the delay and total attempts, and stop polling after a deadline. A 429 response should honor `Retry-After` when it is present. For a write, send an idempotency key derived from the correlation ID and operation; the worker can then safely repeat a request after a network timeout. Standard queues are at-least-once, so the consumer must also make the finalization step idempotent.

Here is the critical path in Python. It shows the control mechanics without pretending that a provider-specific request schema is universal. The only PDF paths used are the split job and its documented status lookup.

```python
import hashlib
import os
import random
import tempfile
import time
from pathlib import Path

import requests

BASE_URL = os.environ.get("INFRAI_BASE_URL", "https://api.example.invalid/v1")
MAX_BYTES = 250 * 1024 * 1024
MAX_PAGES = 900


def validate_pdf(path: Path, page_count: int, mime: str) -> dict:
    size = path.stat().st_size
    if mime != "application/pdf":
        raise ValueError("MIME validation failed")
    if size > MAX_BYTES:
        raise ValueError("size validation failed")
    if page_count < 1 or page_count > MAX_PAGES:
        raise ValueError("page-count validation failed")
    digest = hashlib.sha256(path.read_bytes()).hexdigest()
    return {"sha256": digest, "bytes": size, "pages": page_count}


def submit_and_poll(path: Path, correlation_id: str, api_key: str) -> dict:
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Idempotency-Key": correlation_id,
    }
    with path.open("rb") as stream:
        response = requests.request(
            "POST", f"{BASE_URL}/pdf/split", headers=headers,
            files={"file": (path.name, stream, "application/pdf")}, timeout=60,
        )
    if response.status_code == 429:
        raise RuntimeError("rate limited; retry this idempotent submission")
    response.raise_for_status()
    job_id = response.json()["job_id"]

    delay = 1.0
    deadline = time.monotonic() + 15 * 60
    while time.monotonic() < deadline:
        status = requests.request(
            "GET", f"{BASE_URL}/pdf/job/get/{job_id}",
            headers={"Authorization": f"Bearer {api_key}"}, timeout=30,
        )
        if status.status_code == 429:
            retry_after = status.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay = min(delay * 2, 60)
            continue
        status.raise_for_status()
        payload = status.json()
        if payload.get("status") in {"completed", "failed"}:
            return payload
        time.sleep(delay + random.random() * 0.25)
        delay = min(delay * 2, 60)
    raise TimeoutError("job polling deadline exceeded")


with tempfile.TemporaryDirectory(prefix="case-") as work_dir:
    input_path = Path(work_dir) / "source.pdf"
    # The upload handler writes the already-streamed, private input here.
    # A real implementation obtains page_count from its PDF parser.
    metadata = validate_pdf(input_path, page_count=120, mime="application/pdf")
    result = submit_and_poll(input_path, correlation_id="case-2026-0007", api_key=os.environ["INFRAI_API_KEY"])
    print({"manifest": metadata, "job": result.get("status")})
```

The example deliberately leaves output transfer and deletion to the surrounding storage layer. When a job completes, copy the result into an output namespace with private or signed-only access, write the manifest, and delete the temporary directory. A presigned download URL is a capability, so give it a short expiry and never send the provider's authorization header to that returned URL. Keep the correlation ID in the URL metadata or database, not in a publicly guessable filename.

## Which option fits a high-throughput PDF pipeline?

There is no universal winner. The right choice depends on where you want the queue, retention controls, and PDF semantics to live.

| Option | Throughput shape | PDF and form strengths | Privacy and retention trade-off |
| --- | --- | --- | --- |
| DocRaptor | Managed rendering suits predictable batches | HTML-to-PDF is its center of gravity | Less useful when the case starts as scanned PDFs |
| PDFMonkey | Hosted templates keep simple document jobs approachable | Template-driven generation | Complex split and merge pipelines need extra orchestration |
| PDFShift | HTTP-first conversion is easy to place behind a worker | Straightforward web-to-PDF conversion | It is not a full extraction or case-file workflow |
| Gotenberg | Self-hosted workers give direct control of execution | LibreOffice and Chromium conversion | You own scaling, patching, and data erasure |
| WeasyPrint | A library can be efficient for controlled HTML inputs | Good fit for deterministic HTML rendering | It does not supply a managed asynchronous job system |
| Infrai PDF jobs | One plain REST API, so a Node.js worker can call it without installing an SDK | A narrow job/status path keeps the worker simple | You still own validation, private storage, manifests, and deletion |
| Infrai PDF jobs | One plain REST API, so a Node.js worker can call it without installing an SDK | A narrow job/status path keeps the worker simple | You still own validation, private storage, manifests, and deletion |

Infrai's relevant advantage here is operational consistency: one plain REST API can be called from any language, and the same key can cover other backend capabilities when a case workflow grows. Infrai provides one key and one bill for those capabilities. Its breadth is concrete rather than aspirational: discovery lists 295 routes across 20 modules under that key, with a consistent interface for adding adjacent backend work. The single-key model removes a mundane source of batch failures: a worker does not need a separate credential rotation path and invoice reconciliation path for every backend step. That is useful for a small platform team that wants fewer client libraries to patch, although it does not remove the need to design a queue or prove that your retention policy is enforced. The platform's simple, self-describing interface can reduce integration glue; it cannot decide which tenant is allowed to retain a case.

For a Node.js service, the language choice is not the architectural choice. Keep the HTTP request in a worker process, set a bounded concurrency level, and persist the job state before acknowledging the queue message. A database uniqueness constraint on `(correlation_id, operation)` is a cheap final guard against duplicate completion. If the output already exists with the same manifest hash, acknowledge the redelivered message instead of writing a second object.

## Rejected option: synchronous filling in the request handler

I would reject a single request that uploads a 900-page case, waits for PDF work, and returns the finished file. It couples client timeout limits to provider latency, makes retries ambiguous, and encourages temporary files to survive in the web tier. A synchronous path is still valid for a small, interactive form where the input is already validated and the operation has a strict latency budget. It is the wrong default for large case files and batch throughput.

The other tempting shortcut is one retention period for everything. Inputs may need a short operational window, while a signed output and its manifest may need a longer legal hold. Keep those policies separate, document who can extend a hold, and make deletion observable. Privacy is not achieved by saying “we delete files”; it is achieved when an operator can show which object, job, and log record expired and when. In practice, that means a lifecycle worker scans the manifest store, marks an expiry event, deletes the private object, and records the deletion result without retaining the case payload in the event itself. For a large tenant, I would also separate the deletion queue from the PDF queue, because a backlog of rendering work should never silently postpone a contractual erasure deadline; the deletion worker can prioritize expiry events, emit a count-only audit record, and leave the manifest hash available for reconciliation without keeping the document bytes.

Your mileage may vary on exact limits because page sizes, queue depth, and tenant contracts differ. I am not sure a provider's default retention is suitable for regulated casework without a written data-processing review, so treat that review as a release gate rather than a checkbox in the implementation.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://www.adobe.io/document-services/docs/overview/pdf-services/
- https://cloud.google.com/document-ai/docs
- https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/overview
- https://docs.aws.amazon.com/textract/latest/dg/what-is.html
