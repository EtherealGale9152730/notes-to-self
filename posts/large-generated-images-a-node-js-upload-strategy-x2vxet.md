# Large Generated Images: A Node.js Upload Strategy for Private Buckets

Short answer: for a developer tool that stores large AI-generated images, use multipart object storage behind a commit protocol, and publish an image only after the storage completion and application record agree. The important design choice is the commit boundary, not a vendor's transfer slogan.

That sounds narrower than an upload strategy. It is narrower. A transfer that reaches the bucket but never reaches the database is not a usable artifact; a database row that says “ready” while the object is incomplete is worse. Private access, retries, and worker restarts all become manageable once the service treats upload as a state machine rather than as one long HTTP request.

This is an architecture decision record for a production developer tool. The tool generates images, stores the resulting files, and lets an authorized user retrieve them later. It does not assume that every image is large enough for multipart transfer. Small files have a simpler option, and the rejected option matters here.

## Retention and access belong in the artifact contract

The service should give each generated artifact an immutable object key, start a multipart session for files that justify it, record successful part acknowledgements, complete the session explicitly, and publish the application record only after completion succeeds. Retrieval should use short-lived signed authority against a private bucket. No public-read shortcut belongs in this contract.

The invariants are more useful than a particular SDK:

- A part acknowledgement is progress, not publication.
- A multipart session ends in an explicit completed or aborted state.
- A retry cannot replace an earlier generation under a friendly key such as `latest.png`.
- The application points to a completed immutable object, not to an upload session.
- Access to the object is checked at retrieval time and scoped to the intended user or job.

The object key is deliberately boring: `generations/{job_id}/{output_id}-{nonce}.png`. A unique key removes a storage-level overwrite race. The database can later choose which immutable result is current. That separation also makes a worker retry legible: it can inspect the job record, reconcile the existing session, and continue or abort instead of guessing what a previous process did.

Keep the record durable. If it disappears with the worker, the storage session has no owner and the system has lost the facts needed to finish or clean it up.

There is a second boundary that teams tend to miss. Unfinished multipart parts are not ordinary readable objects, so an object expiration policy is not a complete cleanup plan. Keep a record for every started session, give the owning worker a lease, and let a reconciler explicitly abort sessions whose leases have expired. A one-day cleanup rule may be a useful backstop, but it is not a substitute for application ownership.

## How can teams measure a Node.js upload strategy for large files in private object storage?

Multipart transfer buys a smaller retry unit. It also adds session identifiers, part numbers, completion metadata, abandoned-session cleanup, and another class of ambiguous outcomes. The following comparison is intentionally about boundaries rather than product rankings.

| Transfer shape | Good fit | Cost that remains with the team |
|---|---|---|
| One request to object storage | Small generated images on a stable backend path | A failed request repeats the whole payload, and the service still needs an idempotent publication decision |
| Multipart upload | Large images, bundles, or unreliable links where a part is a useful retry unit | Completion, part reconciliation, abort cleanup, and an application record that survives worker loss |
| Browser-direct upload | A product that can safely issue scoped upload authority to a browser | CORS policy, client-side resume behavior, abuse controls, and a server-side publication check |
| Media delivery platform | A product whose main problem is transformation and delivery rather than durable object persistence | The team must verify private delivery semantics, retention behavior, and the platform's ownership model |

The deciding axis for this case is large-file throughput, but throughput is a constraint, not a sufficient acceptance test. Measure the distribution of generated image sizes, the bandwidth between the worker and the storage service, the retry rate, and the time an abandoned session remains visible to the reconciler. I am not sure there is a universal part-size crossover; your mileage will vary with the deployment path and the output distribution. Establish it with a representative load test, then keep the threshold in configuration.

Do not let a successful part response publish the image. Do not let a successful completion response, by itself, select the product's current image. The first says storage accepted a transfer; the second says storage assembled an object. The application still has to commit the result under its own transaction and authorization rules.

## Compare transfer shapes by the failure they create

The first failure is worker loss after part 2 of 5. The durable record should say which session owns the object key and which parts were acknowledged. A new worker can verify that record, inspect storage's current multipart state through the provider adapter, and either send the missing parts or finish the session. It should not create a second “probably equivalent” object unless the original session has been deliberately abandoned. That sounds like bookkeeping until a retry races the original worker: one process may still be sending part 3 while another creates a new session, and both can produce files that look correct while only one has a valid publication record. The reconciliation query, lease expiry, and immutable key are the control points that turn that race into a decision rather than a mystery.

The second is an ambiguous completion. A timeout does not prove that completion failed. Retrying blindly can create confusing logs or race with a completion that actually succeeded. Reconcile the session and the application record before issuing another terminal operation. Terminal means terminal.

The third is cancellation. A canceled generation should stop publication and explicitly abort an unfinished session. A completed object may remain as an unreferenced artifact until retention policy removes it, but the product must not expose it as the result of a canceled job. This is why the object key, job state, and retrieval authorization cannot be collapsed into one string.

The fourth is a mutable key race. There is no general promise that two workers writing the same convenient key will produce the version your database intended. Use immutable keys and a transactional pointer, or put a real coordination mechanism around the mutable name. “The last request won” is not a publication policy.

The fifth is credential leakage. A signed retrieval URL is scoped authority with an expiry, not a replacement for authorization. Keep the storage credential on the server side, avoid placing bearer material in logs or object metadata, and make the application decide whether the requester may receive a URL for that particular artifact.

These are not exotic edge cases. They are the normal consequences of splitting one logical upload across a worker, a storage service, and a database. They don't disappear because the SDK returns a promise.

## The API implementation needs a durable commit ledger

The provider adapter can use the storage system's native multipart calls, while the application owns the state that determines publication. This small Python model shows the contract that a Node.js service should preserve across its adapter and worker implementations.

```python
from dataclasses import dataclass, field
from enum import Enum


class UploadState(str, Enum):
    UPLOADING = "uploading"
    COMPLETED = "completed"
    ABORTED = "aborted"


@dataclass
class UploadRecord:
    job_id: str
    object_key: str
    expected_parts: int
    state: UploadState = UploadState.UPLOADING
    acknowledged_parts: set[int] = field(default_factory=set)

    def acknowledge_part(self, part_number: int) -> None:
        if self.state is not UploadState.UPLOADING:
            raise RuntimeError("a terminal upload cannot accept another part")
        if not 1 <= part_number <= self.expected_parts:
            raise ValueError("part number is outside the expected range")
        self.acknowledged_parts.add(part_number)

    def complete(self) -> None:
        expected = set(range(1, self.expected_parts + 1))
        if self.acknowledged_parts != expected:
            raise RuntimeError("completion requires every expected part")
        self.state = UploadState.COMPLETED

    def abort(self) -> None:
        if self.state is UploadState.COMPLETED:
            raise RuntimeError("a completed upload cannot be aborted")
        self.state = UploadState.ABORTED


record = UploadRecord(
    job_id="generation-1842",
    object_key="generations/generation-1842/output-0-7f3a.png",
    expected_parts=3,
)
for part_number in (1, 2, 3):
    record.acknowledge_part(part_number)
record.complete()
assert record.state is UploadState.COMPLETED
```

In the real critical path, persist the part acknowledgement after the adapter confirms the transfer, with an idempotency key derived from the job, session, and part number. Use bounded backoff for rate-limited calls. When the outcome is uncertain, reconcile before changing state. Then, after the storage completion is confirmed, commit the application pointer in a transaction that cannot publish an already-canceled job.

The code does not choose a part size because that number is workload-specific. It does make one thing non-negotiable: the set of acknowledged parts must be complete before the application can cross its own publication boundary. Tests should cover duplicate acknowledgement, missing parts, worker restart, cancellation during completion, and a second worker acquiring an expired lease.

## When should the simpler one-request upload win?

A single request is valid for small files and stable private backend traffic. It has fewer moving pieces, no incomplete multipart sessions, and less coordination. I would use it when measured files are comfortably below the operational threshold and repeating a failed transfer is cheap.

Three words: keep it simple.

It is a poor default for large generated images. A transient connection loss can force the worker to resend the entire file, and the application still needs to distinguish a transport success from a published artifact. Multipart reduces the retry unit, but it does not remove the commit protocol or the cleanup worker.

The catch is operational complexity. If the team cannot monitor abandoned sessions, reconcile ambiguous completion, and keep upload records durable across deploys, stay with the simpler single-request path until those controls exist. If the product needs public image hosting, immutable regulatory retention, automatic cross-region replication, or browser uploads with independently managed CORS, choose storage infrastructure that explicitly provides those capabilities. Private persistence for generated artifacts is a different requirement.

The practical decision rule is therefore: start with a single request for measured small outputs; move to multipart when the retry cost and link behavior justify its state machine; publish only from a completed, authorized, immutable result. The rule survives a provider change because it belongs to the application contract.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://cloudinary.com/documentation
