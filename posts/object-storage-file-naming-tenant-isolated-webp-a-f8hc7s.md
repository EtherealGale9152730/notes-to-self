# Object Storage File Naming: Tenant-Isolated WebP and AVIF Image Sizes

Short answer: use an immutable, tenant-scoped key for each uploaded original, put every thumbnail under that original's revision, and encode the complete transform recipe in each WebP or AVIF derivative key; backups belong in a separate retention system, not in a `backup/` filename.

For a fintech product catalog, the least complex safe layout is `{tenant}/{asset}/{revision}/original/{generated-name}.{ext}` for the source and `{tenant}/{asset}/{revision}/derived/{recipe}/{generated-name}.{format}` for each output. A database row should identify the currently published revision. Uploading a replacement creates a new revision and changes that pointer only after validation and derivative generation succeed. It doesn't overwrite yesterday's bytes.

That distinction matters more than punctuation. A neat path cannot enforce tenant authorization, prove durability, or make a mutable object into a backup. The naming scheme's job is narrower: give every logical generation and every transformation a collision-free identity that an operator can inspect.

## Code the tenant boundary into every key

Start with the isolation boundary. The first path component should be an internal, opaque tenant identifier, never a display name, email domain, or customer-supplied string. Authorization still has to come from the application or storage policy; the prefix makes that policy easier to express and makes accidental cross-tenant reads easier to spot in logs. Treat any key received from a browser as untrusted input. OWASP's upload guidance recommends application-generated filenames, extension allowlists, file-type validation, size limits, and storing uploads away from the web root.

The second component is a stable asset identifier. The third is an immutable revision identifier, generated before the upload begins. Together they separate “this catalog image” from “these exact source bytes.” A human filename such as `card-front-final.png` can remain metadata, but it shouldn't be the storage identity: names collide, users rename files, Unicode has multiple representations, and punctuation creates unpleasant escaping problems.

The derivative portion should identify the transformation, not merely the output width. `300.webp` is ambiguous once crop mode, height, quality policy, color handling, or encoder behavior changes. A canonical recipe such as `w=640_h=640_fit=cover_fmt=avif_enc=3` is longer, but it says why two objects differ. Keep the parameter order fixed, allow only known values, and bump the encoder or pipeline version whenever the same input and nominal dimensions may produce different bytes.

A concrete layout looks like this:

```python
from dataclasses import dataclass
from enum import Enum
from uuid import UUID


class Format(str, Enum):
    WEBP = "webp"
    AVIF = "avif"


@dataclass(frozen=True)
class Variant:
    width: int
    height: int
    fit: str
    format: Format
    encoder_version: int

    def recipe(self) -> str:
        if not (1 <= self.width <= 4096 and 1 <= self.height <= 4096):
            raise ValueError("dimensions outside the approved range")
        if self.fit not in {"contain", "cover"}:
            raise ValueError("unsupported fit mode")
        return (
            f"w={self.width}_h={self.height}_fit={self.fit}_"
            f"fmt={self.format.value}_enc={self.encoder_version}"
        )


def original_key(tenant_id: UUID, asset_id: UUID, revision_id: UUID, ext: str) -> str:
    allowed = {"jpg", "jpeg", "png", "webp", "avif"}
    normalized = ext.lower()
    if normalized not in allowed:
        raise ValueError("extension is not allowed")
    return f"tenants/{tenant_id}/assets/{asset_id}/revisions/{revision_id}/original/source.{normalized}"


def variant_key(
    tenant_id: UUID,
    asset_id: UUID,
    revision_id: UUID,
    variant: Variant,
) -> str:
    return (
        f"tenants/{tenant_id}/assets/{asset_id}/revisions/{revision_id}/"
        f"derived/{variant.recipe()}/image.{variant.format.value}"
    )
```

The example deliberately accepts UUID objects rather than raw path strings. It also bounds dimensions and enumerates crop modes. MIME signatures, decoding, malware scanning, authorization, and decompression limits remain separate upload-pipeline responsibilities; an allowed extension alone does not establish that a file is a valid image.

Small detail, large consequence.

## Evaluate content negotiation against tenant isolation

Clients commonly ask for “the best supported format,” but negotiation belongs above storage. A manifest can map a logical role such as `catalog-grid-2x` to exact WebP and AVIF keys, while HTML or an API response chooses between them. Baking browser capability into an object name creates a moving target; storing format, dimensions, fit, and pipeline version creates a finite set of reproducible artifacts.

There are two reasonable ways to identify a revision: a random identifier assigned by the application, or a digest derived from validated source bytes. A digest offers natural deduplication and can help verify content, but it may reveal that two tenants uploaded identical material if keys or logs cross isolation boundaries. A random revision avoids that signal and works before the full stream is read. For a tenant-scoped fintech catalog, the conservative default is a random revision under an opaque tenant ID, with a cryptographic digest retained as protected metadata for integrity checks rather than exposed as a global key.

I'm not sure a single recipe vocabulary will survive every imaging library upgrade; that depends on which transforms the service permits and whether exact byte reproduction is required. Resolve that uncertainty with golden fixtures: a small, reviewed corpus of source images, expected dimensions and formats, plus a rule that any behavior-changing pipeline release increments `encoder_version`. The version is not decoration. Without it, a retry after deployment can target the same key with different bytes, defeating the no-overwrite rule even when the requested width never changed.

Do not derive the current revision by listing a prefix and selecting the lexicographically greatest key. Object names are storage identities, not publication state. Keep the active revision in a transactional metadata record, and return the manifest for that revision after its required variants exist.

## How should object storage handle original image thumbnail retries?

Immutability requires write behavior as well as naming. Each upload moves through explicit application states such as `staged`, `validated`, `rendered`, and `published`. The service accepts a source only into a fresh revision, validates it according to the upload policy, generates the required sizes into that revision, verifies the manifest, then atomically changes the asset's active-revision pointer. Readers see either the old complete set or the new complete set. Never a half-built mixture.

Retries must be idempotent. If a worker sees that the expected derivative key already exists, it should verify the stored checksum and declared metadata against the intended result; an exact match is success, while a mismatch is a collision that stops publication. At the API boundary, a conditional create can be represented as a conflict rather than silently replacing bytes. The particular precondition mechanism varies by object store, so confirm its semantics in the selected service and test concurrent writers instead of assuming every “put” is create-only.

This is where attractive directory diagrams often fail. Imagine revision `r2` needs six outputs: three sizes in both WebP and AVIF. Five renders complete, the AVIF encoder rejects the largest source because it exceeds the application's decoded-pixel limit, and a cleanup worker starts at the same time. If publication is inferred from object presence, some requests can select `r2` while one variant is absent; if cleanup is inferred from age, the still-active `r1` may be removed before rollback. A manifest and a transactional pointer turn those races into ordinary state transitions: `r2` stays unpublished, `r1` stays readable, and cleanup ignores every revision referenced by an active or retained manifest. The rejection is a policy outcome, not a reason to mutate `r1`.

No silent replacement.

## Budget for immutable revisions and retention

The best pattern is the one whose unsafe operations are difficult to express. Readability helps on call, but it comes after isolation, immutability, and deterministic derivation.

| Pattern | Useful property | Failure mode | Appropriate use |
| --- | --- | --- | --- |
| `{tenant}/{asset}/original.jpg` | Short and readable | Replacement overwrites history; derivatives can refer to different source bytes | Only for disposable caches with authoritative data elsewhere |
| `{tenant}/{asset}/{revision}/...` | Separates uploads and supports pointer-based publication | Requires metadata and garbage-collection rules | Default for durable originals and derived images |
| `{tenant}/{digest}/...` | Detects identical content within a tenant | Hashing precedes identity; careless global deduplication can weaken isolation | Controlled deduplication with tenant-scoped keys |
| Date folders plus a filename | Convenient manual browsing | Clock and name collisions; rename semantics are unclear | Exports or reports, not canonical asset identity |
| A `backup/` prefix in the same mutable namespace | Easy to create | Shares credentials and deletion paths with production | Not suitable as the only recovery copy |

The catch is operational overhead. Revisioned keys consume more objects, need a metadata store, and require deliberate garbage collection. Stick with a simpler mutable key when every derivative is a rebuildable cache, the original is authoritative elsewhere, and stale or lost thumbnails have an accepted service impact. For regulated or business-critical originals, don't call a copied prefix a backup: select retention and recovery controls based on the required recovery point, recovery time, deletion threat, and legal policy, then test restoration into an isolated location.

Cost belongs in that decision, but no naming convention makes storage free. Model original retention, derivative count, request volume, egress, metadata operations, and temporary overlap during revision changes. The numbers depend on image distribution and access patterns; a representative trace and a month of measured object counts will answer more than a nominal per-gigabyte comparison.

## Migrate by revision instead of renaming in place

Introduce the new layout on writes first. Dual-read old assets through an explicit legacy mapping, backfill each source into a fresh revision, generate its required variants, verify the manifest, and switch only that asset's pointer. Record counts and checksums before retiring an old key; never interpret a successful copy request as proof that the application can decode and serve the result.

Deployment gates should cover cross-tenant authorization attempts, duplicate concurrent uploads, worker retries, malformed images, transform-version changes, partial variant sets, pointer rollback, and restore drills. Observe rejected uploads by reason, render latency, missing-manifest lookups, checksum mismatches, unpublished revision age, and cleanup candidates. Keep tenant identifiers in structured audit fields, but exclude original filenames and sensitive metadata unless the operational need is explicit.

The migration is complete when reads no longer depend on legacy names and a restore test can reconstruct both the active pointer and every referenced object. Until then, legacy deletion is premature.

## References

- OWASP, “File Upload Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html

## Further reading

- Vercel Blob documentation: https://vercel.com/docs/vercel-blob
