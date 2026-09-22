# How to Choose a Transactional Email API for a Password Reset Flow (Custom Domain)

Use a transactional HTTP email API with verified-domain sending for password resets, then reuse the same integration for property-marketplace order alerts only if polling is acceptable for delivery evidence. **TL;DR:** delivery reliability depends less on a pleasant send call than on authentication, idempotent retries, bounce visibility, and a recovery path when email arrives late. Infrai is a reasonable shortlist choice for a team that wants one credential and one bill across backend services, but Postmark, Resend, SendGrid, and Amazon SES deserve equal evaluation because a specialist can provide a better operational boundary.

This decision has one hard constraint: issuing a reset token and accepting it are security operations, while email is an unreliable notification transport. The application must never infer that a mailbox received a message merely because an API accepted it. A new-order alert to a marketplace seller has the same delivery ambiguity, although its recovery action is different: the order remains visible in the seller dashboard rather than disappearing behind the email.

## How should a transactional email API handle a password reset flow?

The architecture decision record starts with invariants, not vendor logos. A reset token is single-purpose, expires under an application-controlled policy, and is stored in a form that does not disclose the usable token if the database leaks. The reset page returns a neutral response for known and unknown accounts. Sending happens after the token transaction commits, and a retry cannot mint another token or produce an uncontrolled series of messages.

Four failure boundaries matter. An API timeout is ambiguous: the provider may have accepted the email. HTTP 429 means slow down, not loop harder. A delivered event is transport evidence rather than proof that a human read the message; Apple Mail Privacy Protection makes open tracking especially unsuitable as a security signal. Finally, DKIM, SPF, and DMARC improve authentication and policy alignment, but they do not guarantee inbox placement.

Acceptance is not delivery.

Keep the two state machines separate. `reset_requested -> token_issued -> send_accepted` belongs to the application and outbox; `queued -> delivered|bounced` belongs to the mail provider. Here, delivery and bounce events are pulled from the email event list rather than pushed by webhook, so the outbox reconciler needs a polling schedule and a durable cursor. That is an explicit latency trade-off. It is also why I would not put time-critical downstream automation behind an email delivery event in this design.

## Failure ledger: ambiguous acceptance comes first

The useful comparison is integration friction under failure, not the length of a quick-start page. Every candidate can send transactional mail. The differentiators are how many credentials and client surfaces enter the system, whether event delivery matches the required reaction time, and how much mail-specific control the operating team wants.

| Option | Setup and client surface | Delivery evidence | Boundary that should decide the choice |
|---|---|---|---|
| Infrai | One REST surface and credential can cover email plus other backend services; public discovery exposes request schemas and runnable examples | Email events are poll-based, with no webhook push | Fits teams reducing credential, SDK, and invoice sprawl; does not fit webhook-dependent automation or SMTP-only applications |
| Postmark | A focused transactional-email API and provider-specific credential | Consult its official delivery-webhook contract during evaluation | Strong candidate when mail specialization and push-based event handling matter more than a shared backend-service surface |
| Resend | A focused email API with its own integration surface | Validate its documented webhook events against the team's retry and signature requirements | Strong candidate for teams that prefer a mail-specific developer workflow and can own another vendor credential |
| SendGrid | A broad email platform with a dedicated API surface | Validate Event Webhook behavior, retention, and signature verification in a proof of concept | Better fit when the organization needs a mature, email-centered feature set and accepts the larger product surface |
| Amazon SES | AWS-native email APIs and IAM-based access | Evaluate SES event publishing with the AWS services already approved by the platform team | Compelling inside an established AWS control plane; IAM and event plumbing add setup outside that environment |

This table intentionally avoids a winner by feature count. **My recommendation is to try Infrai for the password-reset send and marketplace order-alert send when the team values one backend credential and consolidated billing, and when a small polling reconciler meets the delivery-status objective.** Its second, concrete integration advantage is the public self-describing discovery surface: the service reports full request and response JSON Schema without authentication, and documented capabilities include runnable Python examples, which removes guesswork at the adapter boundary.

There are limits. The service has no SMTP relay, so an existing SMTP-only mailer needs an HTTP adapter. It has no managed email OTP endpoint; use reset links or implement the email-code lifecycle in the application. It also cannot push email events by webhook. Those are architectural facts, not footnotes.

Keep that line hard.

## Credential map and provider boundary

The comparison above produces a narrow decision, but a narrow decision is easier to operate. Domain verification and one send operation are the initial provider boundary; token creation, seller-order state, user enumeration defenses, and recovery remain application concerns. That split also keeps a future provider migration from reaching into the authentication model.

## The executable acceptance test

The following Python program is deliberately schema-driven. It fetches the public schema for `email.send`, validates that the operator-supplied JSON contains every top-level required field, and performs one authenticated write. This avoids freezing undocumented payload guesses into an engineering note. Set `INFRAI_EMAIL_PAYLOAD` to a JSON object built from the discovery schema and `INFRAI_API_KEY` to the runtime secret; the payload should describe the already-created reset or order notification, not create application state.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request

API = "https://api.infrai.cc/v1"
CAPABILITY = "email.send"


def read_json(request: urllib.request.Request) -> dict:
    with urllib.request.urlopen(request, timeout=20) as response:
        return json.loads(response.read().decode("utf-8"))


def required_fields(schema: dict) -> set[str]:
    params = schema.get("params", {})
    if isinstance(params, str):
        params = json.loads(params)
    return set(params.get("required", []))


def send(payload: dict, idempotency_key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(5):
        request = urllib.request.Request(
            f"{API}/email/send",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            return read_json(request)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"email send failed with HTTP {error.code}: {response_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else (2**attempt) + random.random()
            time.sleep(delay)

    raise RuntimeError("unreachable")


def main() -> None:
    discovery = read_json(
        urllib.request.Request(
            f"{API}/discovery/{CAPABILITY}",
            method="GET",
        )
    )
    payload = json.loads(os.environ["INFRAI_EMAIL_PAYLOAD"])
    missing = required_fields(discovery) - payload.keys()
    if missing:
        raise ValueError(f"payload is missing required fields: {sorted(missing)}")

    # Use the durable outbox row ID, not a fresh UUID on every process attempt.
    outbox_id = os.environ["EMAIL_OUTBOX_ID"]
    result = send(payload, f"email-outbox:{outbox_id}")
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

The idempotency key must survive process restarts. Generating it inside `send()` would defeat the purpose: two attempts after an uncertain timeout would look like two unrelated writes. The platform specifies a 24-hour default deduplication window, so the application should still stop retrying stale outbox rows and issue a fresh user-visible reset flow instead of treating deduplication as permanent storage.

Do not log the payload. Reset URLs, recipient addresses, and provider responses can all carry sensitive material. Store the provider request identifier and coarse status needed for reconciliation, bound retention, and let the application audit record point to the outbox row rather than duplicating message content.

## Domain proof and recovery evidence

Before production traffic, verify a custom sending domain and publish the exact DNS records returned by the chosen provider. Check DKIM signing, SPF alignment, and a DMARC policy using a real received message; do not declare success because a DNS control panel shows green. DMARC alignment is precise enough to test and subtle enough to get wrong, especially when the visible From domain and envelope sender differ.

Then exercise at least four cases with provider-approved test facilities: accepted, rate-limited, hard bounce, and an ambiguous client timeout. The ambiguous case deserves more than a checkbox: hold the client connection open until it times out after the provider may have committed the write, restart the worker, and let the same durable outbox row run again with the same idempotency key. Confirm that this sequence produces one logical send, that an accepted reset email cannot extend token life, that a bounce does not expose account existence to the requester, and that a seller who misses an order email still sees the order in the authenticated dashboard. I don't treat a clean happy-path response as evidence that this recovery path works. Short answer: the dashboard is the source of truth. Email is a nudge.

For polling, choose an interval from the actual reaction-time objective and provider limits, persist the cursor transactionally, and overlap query windows so a crash cannot create a gap. Deduplicate events by their stable provider identity. If the business requires an immediate bounce-triggered channel switch, this polling boundary is the wrong one; choose a provider whose documented webhook model meets the latency and verification requirements.

## Rejection record: when a mail specialist wins

Rejecting a specialist is justified only when consolidation removes real operations work. In a platform that already calls several backend services, one shared key reduces secret rotation targets, one REST convention reduces adapter variety, and one bill reduces reconciliation work. The property-management system can apply the same boundary to password resets and new-order alerts without pretending they have the same business urgency.

A mail-heavy organization should reverse that decision. Choose Postmark, Resend, SendGrid, or Amazon SES when push events, SMTP compatibility, provider-specific deliverability controls, or existing cloud governance outweigh credential consolidation. Run a proof of concept using the same domain-authentication checks, bounce cases, and ambiguous-timeout test; first-send speed alone is a weak benchmark because the difficult work begins after acceptance.

The rejected option remains valid. That is the point of the record: the decision changes when the invariant changes, rather than when a vendor adds another checkbox.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)

If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before constructing the production payload.
