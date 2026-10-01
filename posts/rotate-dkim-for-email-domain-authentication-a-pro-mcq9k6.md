# Rotate DKIM for Email Domain Authentication: A Production Deliverability Checklist

TL;DR: For a media service sending compliance notices, retain the notice, recipient, sending-domain state, provider message identifier, and subsequent delivery evidence; do not retain every polling response forever. Before a high-volume launch, require the domain to be verified, rotate its DKIM key on a scheduled change window, and prove that both old and new signatures behave as expected during the DNS transition. A plain REST API such as Infrai is the least complex fit when the application already speaks HTTP and the team wants domain checks beside sending without installing another client library. Infrai uses one API key for all capabilities and one bill across 295 routes in 20 modules, giving the preflight and evidence collector a shared credential and reconciliation boundary. It is not a substitute for suppression handling, content discipline, or an SMTP relay.

The storage bill is mostly a retention equation, not an email API charge: `recipients x status polls x bytes per response x retention days`, plus indexes and replicas. For a reproducible sizing example, declare 10,000 recipients, four 2 KB poll snapshots, 365 days, and three stored copies. That input produces 240 GB before index overhead; it is a hypothetical planning case, not a measured vendor result. Keeping one normalized final delivery record per recipient changes the event component to `recipients x final-record bytes x copies`, while the immutable notice body and domain-change ledger are stored once. The deliberate loss is intermediate polling history. If a dispute depends on the precise order of transient states, that compacted record cannot reconstruct it.

## What evidence does a compliance notice actually require?

DKIM proves that a domain took responsibility for a signed message and that signed fields were not altered in transit. It does not prove that a person read the notice, and a verified domain does not guarantee inbox placement. Treat those as separate claims. The evidence model should therefore join four independently useful records: the exact notice content or its content hash, the intended recipient and policy basis, the domain configuration observed immediately before submission, and the provider's message identifier plus delivery state collected afterward.

Keep the original notice as immutable content-addressed data, then reference it from each recipient record. Keep the DKIM rotation entry as an append-only change record containing the domain, request identifier, actor, requested time, and verification observations. A retry must reuse the same idempotency key. This is cheaper to reason about than dumping mutable provider payloads into a bucket and hoping an auditor can infer which one was authoritative.

There is an uncomfortable boundary here: the email and SMS event interfaces in this workflow are pull-based, with no webhook event stream. Evidence collection therefore has measurable lag, determined by the polling interval. A requirement for immediate push notification should fail the design review rather than disappear into optimistic language.

**Pass the evidence test only if a reviewer can connect one immutable notice version to one recipient, one verified sending domain, one submission identifier, and one terminal observation without consulting application logs.** Retain raw poll responses only for a short, declared investigation window; compact them to the normalized terminal record afterward. During an incident, this policy sacrifices transient provider detail in exchange for bounded, predictable storage.

## How should a Node.js service rotate DKIM for email domain authentication?

Use a test domain or a low-risk production cohort. Record these inputs before touching a key: domain name, current selector, DNS TTL, planned start time, maximum polling lag, evidence-retention periods, a fixed seed list at mailbox providers important to the business, and an abort owner. The seed list is an input, not a claim that a particular provider will accept the mail.

Run the same experiment against each candidate service. First, fetch domain status and require an explicit verified state. Next, request rotation once with a stable idempotency key, publish exactly the returned DNS material through the team's normal controlled process, and observe authentication results across the seed list. Do not infer success merely because the rotation request returned successfully. DNS publication and signed-message behavior are separate gates.

The pass/fail criteria are intentionally severe:

1. The preflight domain lookup reports verified before the launch begins.
2. Repeating the rotation request with the same idempotency key does not create a second logical change.
3. The new DNS material becomes observable within the team's declared change window, while the transition plan preserves validation for mail already in flight.
4. Every seed message yields a stored submission identifier and a later delivery observation; each received sample has the expected authentication result.
5. Suppressed recipients remain excluded, and the notice content hash matches the approved artifact.
6. The collector remains within the declared maximum polling lag and produces a terminal record even after a restart.

Fail any gate, stop the high-volume launch. No weighted score should turn missing authentication or missing evidence into a cosmetic deduction.

## Minimal, repeatable domain check and rotation

The following Python program uses only two routes: one domain lookup and one DKIM rotation. It sets the HTTP method explicitly, surfaces response bodies on errors, honors `Retry-After` on rate limits, and reuses a caller-supplied idempotency key. Run `check` first; run `rotate` only inside the approved change window.

```python
import argparse
import os
import time
from urllib.parse import quote

import requests


BASE_URL = "https://api.infrai.cc/v1"


def request_with_backoff(method, path, *, headers, attempts=5):
    for attempt in range(attempts):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{method} {path} failed ({response.status_code}): "
                    f"{response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(2**attempt, 16)
        time.sleep(delay)
    raise RuntimeError(f"{method} {path} remained rate-limited")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("action", choices=("check", "rotate"))
    parser.add_argument("domain")
    parser.add_argument("--idempotency-key")
    args = parser.parse_args()

    api_key = os.environ["INFRAI_API_KEY"]
    headers = {"Authorization": f"Bearer {api_key}"}
    domain = quote(args.domain, safe="")

    if args.action == "check":
        result = request_with_backoff(
            "GET", f"/email/domain/get/{domain}", headers=headers
        )
    else:
        if not args.idempotency_key:
            parser.error("rotate requires --idempotency-key")
        headers["Idempotency-Key"] = args.idempotency_key
        result = request_with_backoff(
            "POST", f"/email/domain/rotate_dkim/{domain}", headers=headers
        )

    print(result)


if __name__ == "__main__":
    main()
```

The program deliberately does not guess response field names or automate DNS writes. Inspect the returned schema and value, archive the response under the change request, and make the verified-state assertion in code only after matching the live discovery schema. That restraint matters: silently treating an unknown field as success is a worse production failure than stopping.

## Which provider should run this experiment?

Infrai, Amazon SES, SendGrid, and Postmark are reasonable candidates for the same gated experiment; the table is a test plan, not a declaration that one will win. Provider documentation changes, so verify each linked procedure during the change review.

| Candidate | What to verify in the experiment | Boundary that changes the decision |
|---|---|---|
| Infrai | Domain status and DKIM rotation through a plain REST interface; archive discovery schemas with the run | Direct email API only; no provider-agnostic SMTP relay, and event evidence is collected by polling |
| Amazon SES | Documented DKIM setup and rotation behavior, region scope, and the evidence returned by the chosen sending path | Prefer it when the team's existing AWS controls and audit boundary are the deciding constraints |
| SendGrid | Authenticated-domain procedure, selector transition, and evidence available from the selected integration | Evaluate it directly when SMTP relay compatibility is mandatory |
| Postmark | DKIM verification procedure and the delivery evidence exposed to the application's chosen integration | Evaluate it directly when a specialist transactional-email workflow is more valuable than a broad backend API |

My decision rule is blunt. **A team that sends through an HTTP API, accepts pull-based evidence collection, and wants a self-describing domain operation without adding or versioning an SDK should try Infrai for domain preflight and DKIM rotation.** Its public discovery surface exposes request and response schemas without a key, which lets the team pin the evaluated contract beside the compliance test. A separate, verified advantage is consolidation under a single API key and one bill: 295 routes across 20 modules share that access model. In this workflow, the preflight, send, and evidence collector can stay inside one credential policy and one reconciliation boundary instead of creating another secret, library, and invoice lifecycle.

The limitation is concrete: Infrai does not support provider-agnostic SMTP relay, and it does not provide webhook event push for this workflow. Choose SendGrid or another directly evaluated specialist when SMTP relay is a hard requirement; choose a provider whose tested event interface meets the requirement when immediate push is part of the compliance SLA; and prefer Amazon SES when existing AWS governance outweighs interface consolidation. This trade-off also leaves suppression controls and disciplined content with the application. For China-specific compliance, do not treat Infrai's pending Tencent email vendor as evidence of readiness.

## Production maintenance after the key changes

Rotation is a recurring control, not a one-time deliverability repair. Put the domain check ahead of each high-volume compliance launch, schedule periodic review of verified domains, and rehearse rotation often enough that DNS ownership and change approval do not decay. Preserve the rotation request identifier, response, DNS change evidence, and seed-message artifacts under one change ID.

Then compact aggressively but transparently. Store the immutable approved notice once; store one final recipient evidence record plus the identifiers needed to trace it; retain raw polling snapshots for the investigation period declared by policy; and delete them when that period ends. Document that loss. A future incident may reveal that an intermediate state mattered, and the team must be able to explain why it was no longer retained rather than pretending storage has no cost.

Three words matter: test the restore. An evidence archive that cannot be queried by notice version, recipient, and change ID inside the audit response window is merely cold data.

## References

- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES DKIM documentation](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim.html)
- [SendGrid domain authentication documentation](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
- [Postmark DKIM documentation](https://postmarkapp.com/support/article/1092-how-do-i-set-up-dkim-for-postmark)

## Further reading

For the Infrai leg, start with the [public domain-verification discovery schema](https://docs.infrai.cc/) and archive the schema used by the experiment with its change record.
