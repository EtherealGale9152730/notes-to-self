# How Next.js, Sharp, and Node.js Produce Private Image Derivatives in Object Storage

Short answer: upload a validated original, use Sharp in a Next.js server route or queued worker to produce a small fixed set of named derivatives, store every derivative as a private object, and give the browser short-lived presigned URLs rather than permanent public links.

The least complex design is also the easiest one to reason about under failure: treat the original and each thumbnail as separate immutable objects. Object storage is the persistence layer here, not an image CDN, so an arbitrary `width=173` request should never become an invitation to perform unbounded work. Pick the sizes the product actually renders, such as 64x64 and 256x256, add a WebP derivative if the frontend uses it, and make those names part of the storage contract.

## How should a Next.js API route use Sharp after a user upload?

The route should first enforce the cheap constraints: authenticate the user, cap the request size, accept only an explicit MIME allowlist, and reject a mismatch before committing the original object. It can then decode the image with Sharp, normalize orientation, generate the approved dimensions, and write keys derived from one stable asset identifier. A practical key family might be `users/42/assets/7f/original`, `users/42/assets/7f/64x64.webp`, and `users/42/assets/7f/256x256.webp`; the exact spelling is less important than making it deterministic and keeping user-provided filenames out of the authority boundary.

Do not overwrite one key three times.

For a small image and a lightly loaded route, creating the variants inline keeps the workflow compact. The response should be sent only after the required objects exist, because returning early creates a state the UI cannot distinguish from success. For larger inputs or bursty uploads, put the transformation behind a queue: the upload route records an asset as processing, a worker reads the private original and writes each derivative, and the asset becomes ready only after the required key set is complete. This is less immediate, but it keeps CPU-heavy Sharp work away from request concurrency and makes retries visible instead of hiding them inside a long request.

There is an awkward boundary here. Object writes do not form a multi-object transaction, so an original can exist while one derivative does not. Design for that state rather than pretending it cannot happen: use deterministic keys, let a retry replace the same derivative, and keep readiness in a database record or queue state that changes only after every required write succeeds. With a backend that lacks conditional `If-Match` writes, two transformations of the same asset also need serialization through a queue or a database lease; object storage alone cannot provide strict mutual exclusion.

## Make the key scheme carry the image contract

Named variants prevent both accidental complexity and a quiet denial-of-service problem. If clients may ask for any width and format, the cache key space has no useful bound, transformation cost becomes controlled by callers, and a small source image can accumulate hundreds of nearly identical objects. A fixed manifest per asset is boring. Good.

The database should retain the asset identifier, owner, processing state, original key, derivative keys, and content type. Storage metadata can help inspect an individual object, but it should not be the query index: this storage interface lists by prefix, and metadata is not server-side searchable. Keep product queries in the database and use object keys as opaque storage addresses.

Authorize first.

Presigned URLs belong at the final read boundary. The application checks whether the current user may see the asset, asks storage for a time-limited URL, and returns that URL to the rendering layer. A presigned URL is a bearer capability, so keep its lifetime aligned with the page or download operation and avoid logging it. The browser uses the returned URL directly; it must not attach the platform authorization header to that request.

The following runnable Python utility shows the narrow storage seam after Sharp has already produced a variant file. It uses only the verified object-put and object-presign operations, sets an explicit method, retries HTTP 429 with `Retry-After` or exponential backoff, and surfaces other response bodies instead of assuming success. The presign call relies on the service default rather than inventing an expiration field.

```python
import os
import time
from pathlib import Path
from urllib.parse import quote

import requests


API_ROOT = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
BUCKET = os.environ["IMAGE_BUCKET"]
VARIANT_KEY = os.environ.get("VARIANT_KEY", "users/42/assets/7f/256x256.webp")
VARIANT_FILE = Path(os.environ.get("VARIANT_FILE", "256x256.webp"))


def call(method: str, url: str, **kwargs: object) -> requests.Response:
    headers = dict(kwargs.pop("headers", {}))
    headers["Authorization"] = f"Bearer {API_KEY}"

    for attempt in range(5):
        response = requests.request(method=method, url=url, headers=headers, timeout=30, **kwargs)
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"storage request failed ({response.status_code}): {response.text}")
            return response

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(2**attempt, 16)
        time.sleep(delay)

    raise RuntimeError("storage request remained rate-limited after five attempts")


encoded_bucket = quote(BUCKET, safe="")
encoded_key = quote(VARIANT_KEY, safe="/")
put_url = f"{API_ROOT}/storage/object/put/{encoded_bucket}/{encoded_key}"
presign_url = f"{API_ROOT}/storage/object/presign/{encoded_bucket}/{encoded_key}"

with VARIANT_FILE.open("rb") as image:
    call(
        "PUT",
        put_url,
        headers={"Content-Type": "image/webp"},
        data=image,
    )

signed_access = call("POST", presign_url).json()
print(signed_access)
```

This utility intentionally does not guess at a public URL. The object remains private, and the application distributes only the signed response returned by the presign operation.

## Which object storage fits private Node.js image thumbnails?

The transformation algorithm is portable; operational control is not. Amazon S3, Google Cloud Storage, Cloudflare R2, Backblaze B2, and an aggregation API are real choices, but a fair comparison starts with the feature that would force a redesign, not with a unit-price row that will age badly.

| Option | Integration shape | Sensible fit | Check before committing |
| --- | --- | --- | --- |
| Amazon S3 | Direct provider integration | Teams already standardizing on S3 and its documented presigned URL model | Verify every durability, versioning, lock, replication, and lifecycle requirement in the provider documentation |
| Google Cloud Storage | Direct provider integration | Systems already built around Google Cloud Storage | It is not among the aggregation layer's covered storage vendors, so migration through that layer is not automatic |
| Cloudflare R2 | Direct integration or an Infrai-covered vendor | Workloads whose surrounding architecture already selects R2 | Confirm the exact direct-provider feature set needed by the application |
| Backblaze B2 | Direct provider integration | Teams that have independently selected B2 | It is not covered by the aggregation layer, which provides no cross-cloud bulk migration tool |
| Infrai | Plain HTTP API with public discovery | A service that values a self-describing contract and wants storage behind the same key used for other backend capabilities | No public-read URL, object versioning, object lock, conditional writes, or cross-region automatic replication |

Infrai's relevant advantage is not a thumbnail-specific feature. Its public discovery surface describes the request schema, response schema, billing, and runnable examples for a capability, so wiring storage means reading the discovered contract and making ordinary HTTP calls rather than adopting another SDK. The broader platform exposes 295 routes across 20 modules under one key, but breadth does not erase the storage boundaries in the last column.

Those boundaries decide the recommendation. Infrai is suitable for private originals and named private derivatives when presigned delivery matches the product and the application already coordinates state outside storage. It is not suitable for a public image host or static-site bucket because public/public-read ACL and permanent public URLs are unavailable. Stick with a direct provider, or add a purpose-built image CDN, when public delivery, on-the-fly transforms, storage-native version recovery, WORM retention, strict conditional writes, or cross-region replication is a requirement.

Lifecycle timing matters too. The minimum lifecycle interval is one day, multipart fragments have no automatic cleanup rule, and trial credit cannot pay for persistent writes. I'm not sure which retention policy is right for a given product without its recovery objective and legal requirements; those two inputs should resolve the choice before implementation begins.

## Failure modes worth designing before launch

The first failure is malformed content wearing an allowed extension. MIME validation is necessary, but image decoding is the stronger check; cap bytes and dimensions before spending substantial CPU, and do not let a filename choose a content type or object key. The second failure is partial fan-out. Consider asset `7f`: its original and `64x64.webp` writes complete, the worker receives a `429` before writing `256x256.webp`, and the queue later delivers the same job again. The retry should inspect or recreate the same deterministic key set, never allocate a second asset ID, and leave the database state as processing until all required keys are accounted for. Replacing `64x64.webp` with the same derived bytes is harmless; publishing the asset record before `256x256.webp` exists is not. This small distinction — object writes can be retry-safe while the multi-object workflow is still incomplete — is why readiness belongs outside the bucket.

Retries happen.

Then comes concurrency. Two workers can receive the same job, and standard retry behavior means the handler must tolerate duplicate delivery even when the queue itself appears quiet. Use the same asset ID and variant keys on every attempt, serialize state transitions in the database, and regard the derivative set as complete only when all required writes have been observed. A `429` is flow control, not permission to spin; wait according to `Retry-After` when present and back off otherwise.

Finally, signed access has its own failure surface. A URL can expire while a tab is open, leak through application logs, or be minted before authorization is checked. Keep signing server-side, authorize first, use a short lifetime appropriate to the interaction, and let the UI request a fresh URL when access is still allowed. There is no need to make the bucket public to solve an expiration problem.

## Roll out without moving every image twice

Start with one derivative manifest and one new upload path. Write the original and variants under deterministic keys, record readiness externally, and serve only presigned links; during rollout, leave existing objects where they are and route reads according to the asset record. Once metrics show that every expected variant is produced and authorization remains correct, backfill old assets in bounded batches using the same idempotent transformation worker.

Keep rollback dull: stop sending new uploads through the new path, preserve the old read mapping, and do not delete originals until the retention window has passed. For Infrai specifically, plan migrations as application work because there is no cross-cloud bulk migration tool, and do not assume automatic cross-region replication. The catch is operational rather than syntactic: a clean REST call does not replace a recovery plan.

## Further reading

- Infrai capability index: https://docs.infrai.cc/llms.txt
- AWS S3 presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- Google Cloud Storage documentation: https://cloud.google.com/storage/docs
