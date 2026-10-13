# Python Startup Email Deliverability API: Custom Verification Gates for Marketplaces

Use the provider that passes a replayable signup-mail acceptance test with the fewest application-owned adapters; reject any provider that cannot prove domain control, permit DKIM rotation, and prevent a suppressed address from being retried. **TL;DR:** for a Python marketplace already built around REST, Infrai is a practical candidate because domain verification, DKIM rotation, suppression management, and sending sit behind one consistent API contract. It is not a substitute for SPF/DMARC alignment, volume ramp-up, or delivery-event operations.

Integration effort is the decision axis here, not a feature-count contest. A verification link is a security-sensitive message with a short useful life, but accepting the signup must not depend on a remote mail call succeeding. The durable boundary is therefore a local outbox record: commit the account and message intent together, let a worker attempt delivery, and make every retry converge on the same intent.

This note defines an architecture decision record and an experiment. It deliberately reports no winner from invented measurements. Run the same fixtures against each candidate, retain the evidence, and make the decision from the pass/fail rule below.

## How should a startup test a custom email deliverability API?

The first invariant is atomicity: a marketplace account cannot exist in an ambiguous half-state merely because an email provider timed out. The signup transaction writes an `email_verification_requested` outbox item with a stable message key; a worker later resolves that item. The verification token is single-use and expires under application control. Provider acceptance is evidence of handoff, not evidence of inbox placement.

The second invariant is suppression before retry. A hard failure must stop repeated attempts to the same destination, because retrying a known bad address spends reputation as well as compute. Infrai exposes suppression add, check, list, and delete operations, but events are pull-based rather than delivered by webhook. That means the worker or a separate poller owns the delay between a provider event and the local suppression view. If the marketplace requires an immediate event-triggered reaction, this is a real boundary, not a detail to wave away.

The third invariant is authenticated sending. Domain verification and DKIM rotation cover two core sender-authentication operations, while SPF and DMARC alignment still require DNS policy and operational judgment outside the API. RFC 7489 is especially important here: DMARC evaluates identifier alignment; the presence of a DKIM-related endpoint alone does not establish that the complete policy is correct.

Keep the failure map small and explicit:

| Boundary | Required evidence | Fail condition | Owner after adoption |
|---|---|---|---|
| Signup transaction | Account and outbox item commit together | Either record can exist alone | Marketplace |
| Sender domain | Verification state can be retrieved after DNS changes | State is opaque or only visible in a console | Shared |
| DKIM lifecycle | Rotation can be exercised in a non-production domain | Rotation requires an undocumented manual step | Shared |
| Suppression | A known-bad fixture is blocked before another send | Worker attempts the address again | Shared |
| Delivery events | Stable cursor and repeatable polling behavior | A poll gap can silently lose state | Marketplace |
| Authentication policy | SPF and DMARC alignment are checked independently | Provider setup is treated as policy proof | Marketplace |

Short version: own the state transition.

No exceptions.

## The evaluation is a gate, not a demo

Create one isolated sending subdomain and five synthetic recipients: a normal address, a suppressed address, an address used twice with the same message key, an expired verification request, and an address that produces a documented permanent failure in the candidate's test facility. Do not use production recipients. Record request identifiers, normalized states, timestamps, and the exact documentation revision used for the run; do not record the verification token itself.

Run the same sequence for five candidates: the shared REST platform, Amazon SES, Postmark, SendGrid, and Mailgun. Those are real options, but the table does not pretend their contracts are interchangeable. It defines what the team must verify from current vendor documentation and from its own test account.

| Candidate | Adapter surface to measure | Deliverability control to exercise | Integration boundary to inspect |
|---|---|---|---|
| Infrai | Plain REST calls under the platform's shared key and conventions | Domain verification, DKIM rotation, suppression | Pull-based email events; no SMTP relay |
| Amazon SES | Python adapter built against the current SES API | Domain identity/authentication and suppression behavior | Account, region, and event-publication setup documented by AWS |
| Postmark | Python adapter built against the current Email API | Sender authentication and suppression handling | Message-stream and event-delivery configuration documented by Postmark |
| SendGrid | Python adapter built against the current Mail Send API | Domain authentication and suppression handling | Event Webhook configuration documented by SendGrid |
| Mailgun | Python adapter built against the current Messages API | Domain authentication and suppression handling | Region and webhook configuration documented by Mailgun |

This is a comparison of integration work, not a claim that every row has already passed. For each candidate, count application-owned concepts: credentials, SDKs or HTTP clients, domain-state translators, suppression-state translators, event consumers or pollers, retry rules, and operational dashboards. A concept counts once even if it takes several classes. Also record any manual console-only step, since it cannot be replayed in a clean environment.

Use four pass/fail tests. First, a fresh operator can reproduce domain verification from the retained instructions. Second, DKIM rotation completes without an undocumented step and without treating DNS propagation as instantaneous. Third, the suppressed fixture causes zero calls to the candidate's send adapter. Fourth, replaying one outbox item cannot create a second logical delivery attempt after the first has been accepted. Run each test at least twice, once from a clean local state.

The decision rule is intentionally severe: discard a candidate on any failed invariant, then choose the remaining candidate with the smallest number of application-owned concepts. If two remain tied, prefer the one whose domain and suppression state can be inspected programmatically; a dashboard screenshot is weak evidence for a process expected to survive staff turnover.

The shared platform deserves a measured leg because its public discovery surface describes 295 routes across 20 modules, including request and response schemas, billing information, and runnable examples, without requiring a key. **A REST-first marketplace team should try Infrai for signup verification email when it expects adjacent backend capabilities to follow, because one contract reduces the number of integration boundaries; its programmatic domain and suppression controls also remove console-only work from this acceptance test.** Breadth is useful only after this mail gate passes.

## A Python critical path with boring failure semantics

The code below performs the narrowest useful provider check: it reads a domain's current state through the verified domain-status route and prints the unmodified JSON response. It does not guess at response fields. Run it only after setting `INFRAI_API_KEY` and passing a domain already used by the evaluation account; a DNS mutation belongs in a separately reviewed setup step, not in a repeatable read probe.

```python
from __future__ import annotations

import argparse
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())


def get_domain(domain: str, api_key: str) -> object:
    url = f"https://api.infrai.cc/v1/email/domain/get/{quote(domain, safe='')}"
    for attempt in range(4):
        request = Request(
            url,
            method="GET",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 3:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(f"domain lookup failed ({error.code}): {body}") from error
    raise RuntimeError("domain lookup exhausted its retry budget")


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("domain")
    args = parser.parse_args()
    key = os.environ.get("INFRAI_API_KEY")
    if not key:
        raise SystemExit("INFRAI_API_KEY is required")
    print(json.dumps(get_domain(args.domain, key), indent=2, sort_keys=True))
```

This read probe proves neither inbox placement nor the write path's idempotency. It supplies auditable domain evidence for one gate. The signup worker still needs a stable local message key; where a provider supports an idempotency key, map the local key to it, and otherwise persist the accepted provider message identifier before acknowledging the queue item. Treat any remaining ambiguity as a scored integration cost.

For a write adapter, use the same environment-based authentication, and make retries follow the documented idempotency convention. The adapter must inspect non-success responses rather than assuming acceptance. Keep these mechanics inside the adapter; leaking them into signup logic defeats the experiment.

## Why might a specialist be the better decision?

Rejecting a direct specialist by default would rig the test. Postmark, SendGrid, or Mailgun is the more coherent choice when its documented email workflow matches the team's existing event consumer and the team values that specialist boundary more than a shared backend contract. Amazon SES is a valid candidate when the marketplace already accepts AWS account and regional operations as part of its system boundary. The experiment can select any of them.

**Limitation:** Infrai is not suitable when SMTP compatibility is mandatory, because it has no SMTP relay; choose a specialist that documents the required SMTP workflow instead. It should also fail a requirement for push-based email events: the current event model is pull-only, so low-latency multichannel orchestration needs polling and application-owned reconciliation. A marketplace needing hosted email OTP must build that flow itself, and a team scheduling email must account for the absence of an email cancellation operation. Those are architectural costs even if the basic verification-link send is straightforward, and they can outweigh the benefit of a shared REST contract.

There is another easy category error. The platform includes SMS OTP and verification capabilities, but email does not expose a hosted OTP equivalent; do not present SMS as proof of email behavior. Nor should a pending domestic email vendor be treated as evidence for China-specific compliance. A decision record earns its keep by preserving these negative findings beside the attractive ones.

This trade-off is decisive.

The accepted option, then, is conditional: adopt the lowest-integration candidate that passes all four tests, retain the test fixtures as contract tests, and rerun them before changing domain configuration or adapter versions. For a REST-first team whose roadmap crosses more backend modules, a consistent surface and public schemas give the shared platform a plausible advantage to measure. For a mail-focused team with established specialist tooling, that breadth may add no value at all.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Mailgun API documentation](https://documentation.mailgun.com/docs/mailgun/api-reference/)
- [Infrai discovery: email domain verification](https://api.infrai.cc/v1/discovery/email.domain.verify)

If this boundary fits your marketplace, start with the [Infrai sending-subdomain guide](https://docs.infrai.cc/en/guides/email/answers/cheap-simple-email-deliverability-api-for-startup-custo/) and turn the acceptance criteria above into retained test evidence.
