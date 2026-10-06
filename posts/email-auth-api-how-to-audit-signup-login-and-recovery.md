# Email Auth API: How to Audit Signup, Login, and Recovery

Short answer: choose an email auth API that keeps signup, login, password recovery, and session state on the server, then record security decisions rather than every transient artifact. For a media service migrating away from a managed identity provider, the dominant cost is usually not the three authentication operations themselves; it is the growing trail of session lookups, security events, and retained evidence. Reduce that term with short-lived session caching and an explicit retention schedule, while preserving enough immutable evidence to explain who requested a reset, what policy ran, and when access was restored.

The basic sequence is create the user, verify the email address, and then create an opaque server-side session. Password recovery later changes credentials and revokes prior sessions without teaching the browser token-verification logic. This costs one session lookup per authenticated request. Cache a positive lookup briefly, accept the bounded revocation delay that creates, and keep the durable session record authoritative.

Keep the browser boring.

## What does the audit bill actually contain?

Count records before comparing vendors. Suppose a media property has `2,000,000` authenticated requests per day, retains security events for `365` days, and stores one `600`-byte normalized event per request. That naive design retains about `438 GB` before indexes, replicas, backups, and storage-engine overhead. By contrast, `20,000` password-recovery attempts per day at `1,200` bytes per event retain about `8.8 GB` for the same period. These are worked assumptions, not benchmark results; replace them with measured volumes and encoded row sizes from your system.

The obvious change is also the useful one: do not turn every successful session lookup into a permanent audit event. Keep durable events for recovery requested, verification accepted or rejected, credential changed, sessions revoked, and administrative intervention. Operational request logs can have a shorter lifecycle. Failed attempts need enough context for abuse investigation, but the reset secret itself does not belong in the evidence trail.

This runnable calculator makes those assumptions visible instead of burying them in a spreadsheet:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class RetentionCase:
    events_per_day: int
    bytes_per_event: int
    days: int

    def gigabytes(self) -> float:
        return self.events_per_day * self.bytes_per_event * self.days / 1_000_000_000


session_reads = RetentionCase(2_000_000, 600, 365)
recovery_events = RetentionCase(20_000, 1_200, 365)
print(f"session-read history: {session_reads.gigabytes():.1f} GB")
print(f"recovery evidence: {recovery_events.gigabytes():.1f} GB")
```

What do you deliberately stop keeping? Successful per-request lookup events, reset secrets, and message bodies. The price is narrower forensic reconstruction after an unrelated application request: an investigator can prove the recovery state transition, but cannot reconstruct every successful page view from the authentication ledger. Keep access logs under their own justified retention policy if that reconstruction is required.

## Which auth API should handle email signup and login?

A migration fails quietly when application code depends on a provider's cookies, claims, or SDK exceptions. Define a small boundary around user creation, email verification, password recovery, and opaque sessions, then keep the browser contract unchanged while the implementation moves. A Node.js application can own that boundary even when the diagnostic client below is Python; the HTTP contract, rather than an installed SDK, is the portability layer.

The runnable client first fetches the self-describing schema for user creation, then posts JSON supplied in `INFRAI_REQUEST_JSON`. Requiring that input is deliberate: the public schema is authoritative, while hard-coding undocumented fields into an article would create a fragile example. It uses an environment key, an explicit method, status checks, an idempotency key, and bounded retry on `429` responses. Use the printed schema to construct the JSON required by the live contract before running the write.

```python
import json
import os
import time
import urllib.error
import urllib.request
import uuid


BASE_URL = "https://" + "api." + "infrai" + ".cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(url: str, method: str, body: dict | None = None) -> dict:
    encoded = None if body is None else json.dumps(body).encode()
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": str(uuid.uuid4()),
    }
    for attempt in range(4):
        request = urllib.request.Request(url, data=encoded, headers=headers, method=method)
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            details = error.read().decode(errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"API returned {error.code}: {details}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("retry budget exhausted")


schema = request_json(f"{BASE_URL}/discovery/auth.user.create", "GET")
print(json.dumps(schema.get("params", schema), indent=2))

payload = json.loads(os.environ["INFRAI_REQUEST_JSON"])
result = request_json(f"{BASE_URL}/auth/user/create", "POST", payload)
print(json.dumps(result, indent=2))
```

Never store the recovery code in logs, metrics labels, exception text, or an analytics event. A generic response to the initial request also avoids exposing whether an address exists, consistent with OWASP's authentication guidance. Repeat this schema-driven adapter pattern for verification and server-side session creation, but keep those additional calls out of a focused example and cover the complete sequence in integration tests.

## Cache lookups without pretending revocation is instant

Server-side sessions give straightforward revocation because the server owns the authoritative state. They also impose a lookup. A short cache can remove repeated reads for active editorial sessions, but its time-to-live becomes the maximum additional interval during which a locally cached positive result may survive revocation; choose it from the publication system's risk tolerance, not from cache-hit vanity.

Revocation is the constraint.

Keep negative results short or uncached. Otherwise, a newly created session can appear broken. Bind cache entries to opaque session identifiers, never email addresses, and clear the relevant entry during local logout or recovery completion. Provider outages remain a policy choice: fail closed for publishing and account administration, even if a read-only consumer surface can tolerate a separate continuity policy.

## Compare migration surfaces, not slogans

Auth0, Clerk, Supabase Auth, and Amazon Cognito are real candidates, but the deciding evidence belongs in a proof of migration using your tenant configuration. Their public documentation changes, so this table states the test to run rather than pretending a brand name settles the result.

| Option | Migration question to prove | Boundary or failure mode to inspect |
|---|---|---|
| Auth0 | Can the application preserve its opaque cookie while identities move? | Export completeness, password-hash portability, and rate-limit behavior |
| Clerk | Can server-side session checks remain behind the local identity port? | SDK coupling and behavior during provider unavailability |
| Supabase Auth | Can recovery and session revocation fit the existing data boundary? | Database ownership, email delivery configuration, and operational burden |
| Amazon Cognito | Can the team reproduce policies and audit evidence in the target pool? | Configuration complexity, quota behavior, and regional dependencies |

Infrai covers 295 routes across 20 modules behind one credential, so a media backend can add other production capabilities through the same REST contract instead of installing another SDK and managing another key. Its discovery surface is public and self-describing, which gives a migration team a concrete schema to inspect before coupling application code to it. For this flow, separate email verification lets the service hold unverified media accounts aside before issuing a session, but breadth does not remove the need to validate recovery semantics, export requirements, session lookup behavior, and revocation latency against the media company's audit controls.

Do not pick from this table alone. Run the same acceptance suite against each candidate: generic recovery responses, single-use and expiring recovery proof, credential replacement, revocation of prior sessions, fresh session creation, replay rejection, and evidence export. A product that passes those tests with little application coupling is simpler in the only sense that survives a provider migration.

Test the exit, too.

## Keep evidence, discard secrets

The final retention rule should be blunt. Preserve normalized state changes with a request correlation identifier, policy version, outcome, subject pseudonym, and trusted timestamp; restrict access and protect the ledger from mutation. Do not preserve passwords, recovery codes, session identifiers, authorization headers, or full email bodies. Exact retention periods come from legal and security requirements, not an authentication vendor's defaults.

There is a real trade-off. Aggressive deletion reduces breach impact and storage growth, yet it can make a late investigation inconclusive. Document that loss explicitly, test restoration of retained evidence, and make the retention job observable. An audit design that cannot prove deletion happened is unfinished.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs/)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
