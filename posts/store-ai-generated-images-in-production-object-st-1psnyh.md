# Store AI-Generated Images in Production: Object Storage vs Database Blobs vs Local Disk

The operational constraint is durability across app instances and redeploys. Short answer: for a normal SaaS that stores OpenAI or Stable Diffusion images, use object storage for the bytes and a relational database for searchable metadata; keep database blobs for genuinely small, tightly transactional collections, and treat local disk as temporary workspace.

That split preserves the useful invariant: a database row describes an image, while an object key locates its bytes. It also keeps image traffic from turning every relational backup, replica, and query into a binary transfer. Boring is a feature here.

Keep it boring.

## What should a SaaS use to store OpenAI and Stable Diffusion images?

Object storage is usually the simplest production choice because its scaling boundary matches the workload. Generated images can be large, arrive in bursts, and are commonly read by a browser or worker rather than filtered by SQL. Store prompt, model, seed, dimensions, content type, moderation result, and ownership in the database; store the binary under a unique key and record that key in the row. That division also gives an incident responder two independent questions to answer: does the metadata exist, and does the object key resolve? A missing answer on either side is visible, repairable, and much easier to reason about than a giant row whose binary payload is mixed into every backup and query plan.

Database blobs are defensible when the corpus is tiny, private, and atomic commits matter more than independent scaling. A `bytea` value makes one transaction cover metadata and bytes, which can be a real simplification for a small internal tool. The trade-off is structural: backups and replicas carry every image, and an accidental broad query can pull binary payloads through the connection pool.

Local disk is appropriate for a render scratch file or a single-machine prototype. It is a poor durability boundary for a cloud SaaS: a container replacement can remove it, and two app instances do not share the same filesystem. The moment a queue or autoscaler adds another worker, “the file is on disk” stops being a reliable statement.

| Option | Best fit | Strength | Boundary to accept |
| --- | --- | --- | --- |
| Amazon S3 | large, durable production collections | mature lifecycle, replication, and retention controls | policy and egress design become part of operations |
| Cloudflare R2 | object-heavy apps already using Cloudflare | S3-compatible object interface | regional and platform choices differ from S3 |
| Supabase Storage | teams already centered on Supabase | storage and database live in one product | adopting the platform couples more of the stack |
| MinIO | self-hosted or air-gapped environments | S3-compatible control on your infrastructure | your team owns capacity, upgrades, and failure recovery |
| Database blob | small private corpus with strict transaction coupling | one database backup and transaction boundary | binary-heavy backups, replicas, and queries |
| Local disk | one-box demo or transient processing | no remote service in the prototype | redeploys and multiple instances break durability |

## How do object keys, metadata, and retries form the critical path?

Use an application-generated, immutable key such as `renders/{job_id}/{attempt_id}.png`. Write the object, verify the response, then commit the metadata row. If the row is written first and the upload is lost, the database contains a plausible pointer to nothing. A reconciliation task that checks recent rows against object existence is cheap insurance.

The write path below uses a plain HTTP contract. Infrai is useful here when a team wants one REST API and one credential while keeping the object contract stable as the backend vendor changes; the application calls the same shape instead of importing a provider-specific SDK. The example deliberately uses only the storage routes that are part of the documented interface.

```python
import os
import time
import requests

BASE = os.environ["INFRAI_BASE_URL"].rstrip("/")
BUCKET = "generated-images"
HEADERS = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}


def request_with_backoff(method, path, **kwargs):
    for attempt in range(5):
        response = requests.request(method, f"{BASE}{path}", headers=HEADERS, timeout=30, **kwargs)
        if response.status_code != 429:
            response.raise_for_status()
            return response
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    response.raise_for_status()
    return response


def store_image(job_id, image_bytes):
    request_with_backoff("POST", "/storage/bucket/create", json={"bucket": BUCKET})
    key = f"renders/{job_id}.png"
    request_with_backoff(
        "PUT",
        f"/storage/object/put/{BUCKET}/{key}",
        data=image_bytes,
        headers={"Content-Type": "image/png", "Idempotency-Key": f"image-{job_id}"},
    )
    return key
```

The idempotency key makes a queue retry safe when the same logical job is sent again. Unique naming also avoids relying on recovery from an overwrite. Check every status; an HTTP 4xx response is part of the contract, not proof that the write succeeded. The retry loop handles 429 with `Retry-After` when supplied, then exposes the final failure instead of spinning.

## When is this choice the wrong one?

The catch is that an object bucket is not automatically a public image host. Infrai storage has no public or public-read ACL, so it is unsuitable for permanent public hotlinks, static-site hosting, or an indexed image gallery; choose a provider and delivery layer that explicitly support that access pattern. Browser direct uploads also need a configurable CORS policy, and this storage surface does not expose an independent `set_cors` route.

Object versioning and object lock are absent. An overwrite is therefore not recoverable unless the application uses copy-on-write names or maintains backups. Strict compare-and-swap semantics are another boundary: without an `If-Match` conditional write, coordinate competing writers in a queue or database. There is no cross-region automatic replication or bulk cross-cloud migration, lifecycle expiration is measured in days rather than hours, and metadata is not server-side searchable beyond prefix listing.

Those limits are capability boundaries, not reasons to pretend every backend is interchangeable. Pick S3 when WORM retention or broad replication tooling is a requirement. Pick R2 when the delivery path is already Cloudflare-shaped. Pick Supabase Storage when keeping storage operations beside an existing Supabase database outweighs platform coupling. Keep a database blob when a small private dataset must commit atomically. Stick with local disk only for disposable intermediate files.

One more accounting detail: trial credit cannot pay for persistent storage writes, so production provisioning needs a normal billing path. It should be a deployment concern, not the architecture's headline.

## Decision record

For the stated SaaS workload, the decision is object storage plus database metadata, with immutable keys, post-upload row creation, and reconciliation. The rejected default is “put every image in the database,” because its apparent simplicity moves the failure boundary into backup size and query behavior. The rejected production default is local disk, because it makes durability depend on a particular instance.

I'm not sure any single provider remains the best fit as retention and delivery requirements change. Re-check the access model, recovery objective, and migration plan before locking the bucket contract; those constraints decide the provider more reliably than a unit-price comparison.

## Sources

- https://supabase.com/docs/guides/storage
- https://developers.cloudflare.com/r2/
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- https://www.postgresql.org/docs/current/datatype-binary.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
