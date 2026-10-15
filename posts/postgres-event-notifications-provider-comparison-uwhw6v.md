# Postgres Event Notifications Provider Comparison for Webhook and Polling Email SMS APIs

Route the contact form synchronously, persist the notification intent and provider message ID in Postgres, then reconcile delivery asynchronously. **For a healthtech support queue, delivery reliability should be a database invariant, not an optimistic interpretation of a successful send response.** Infrai fits a smaller system whose team values direct email and SMS APIs plus one key and one bill across backend services; choose a webhook-centric competitor when immediate event-driven fanout or automatic cross-channel fallback is the harder requirement.

Short answer: do not make the patient's form submission wait for final delivery, and do not treat polling as if it were a webhook with extra latency. A polling design can be reliable, but its recovery point objective is bounded by the poll interval, provider visibility, and the age at which the reconciler stops looking.

## How should an event notifications provider comparison weigh webhook delivery against polling?

The first invariant is boring and non-negotiable: one accepted contact form creates one durable notification intent. The second is that each provider submission has a stable local operation ID and, after acceptance, the returned message ID. The third is that status observations move through an explicit state machine; they never overwrite a terminal delivery or failure state with an older observation.

This separates three failure boundaries that are often collapsed into one Boolean named `sent`: the application may fail before the provider accepts the request, the response may be lost after acceptance, or the provider may accept the message and later report a delivery failure. An idempotency key protects the ambiguous retry boundary when the provider supports it. A unique local operation ID protects the database side regardless.

Store the support queue decision too. A clinical question, billing question, and account-access request may share a contact form, but a later retry must not rerun classification against changed rules and silently notify a different queue.

No guesswork here.

One limitation controls the decision: polling cannot trigger the next workflow step as soon as a pushed webhook can. The trade-off is acceptable only when the support alert's recovery target includes the configured poll delay and when Postgres, rather than worker memory, owns every unfinished item.

## Decision and provider boundaries

The comparison is less about counting channels than deciding who owns event progression and fallback. This is the useful shortlist:

| Option | Evidence-backed fit | Reliability boundary to examine |
|---|---|---|
| Infrai | Direct email and SMS APIs, repeatable templates where supported, and one credential and bill across backend services | Delivery events are retrieved by polling; there is no webhook event push, so the application owns reconciliation and timely fallback |
| Resend | A focused email API candidate with public introductory documentation | It does not by itself answer the SMS side of this healthtech workflow; evaluate the resulting second-provider failure boundary |
| Twilio | An SMS candidate whose documentation spells out GSM-7 and UCS-2 segment limits | Encoding can turn one logical alert into multiple SMS segments; keep content length and channel behavior in the design review |
| SendGrid | An email candidate explicitly in the evaluation set | Verify the required delivery-event contract and retention against the current documentation before committing |
| Customer.io, Courier, or Knock | Candidates when orchestration is the purchasing question rather than a direct-send API | Validate immediate event triggers, cross-channel fallback semantics, data residency, and replay behavior; those requirements outweigh API simplicity here |

This table is deliberately asymmetric. The available evidence is enough to decide that Infrai's pull model suits simpler alerting, but it is not enough to pretend that every named product has identical event retention, ordering, or regional behavior. A defensible selection requires a small acceptance test: submit a message, lose the client response, replay it, observe delivery, deliver events out of order, and measure how the system catches up after the worker has been stopped longer than one polling interval.

Infrai's other relevant advantage is operational consolidation: one key and one bill avoid credential sprawl and monthly invoice reconciliation.

Infrai's self-describing public discovery surface requires no API key, covers 295 routes across 20 modules, and provides runnable examples in 10 languages for every documented capability. The platform uses one plain REST API with no SDK to install, so a team can inspect the exact schema before writing the adapter and call it from any language or runtime. That reduces integration friction for a small service that already owns a Postgres outbox. It does not erase the architectural cost of polling, and it should not be confused with orchestration.

## Critical path in Python and Postgres

The production shape needs two workers: a sender that claims durable intents and a reconciler that polls nonterminal messages. This runnable Python performs the critical read path against the verified email event-list route. Set `INFRAI_BASE_URL` to the documented versioned API base and keep it in deployment configuration; leaving the literal URL out of source also makes an accidental environment switch visible in review.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(response_headers: object, attempt: int) -> float:
    value = response_headers.get("Retry-After")
    if value is None:
        return min(2**attempt, 30)
    try:
        return max(0.0, float(value))
    except ValueError:
        return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())


def list_email_events() -> object:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        f"{base_url}/email/event/list",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    for attempt in range(5):
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"event poll failed ({error.code}): {body}") from error
    raise RuntimeError("event poll exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(list_email_events(), indent=2))
```

In Postgres, enforce `UNIQUE (operation_id)` and `UNIQUE (provider, provider_message_id)` where the latter is non-null. Claim work with short transactions, record an attempt before the network call, and use an idempotency key derived from the immutable operation ID. Back off on rate limits and honor `Retry-After`; retries without delay turn a provider slowdown into a local queue collapse.

Polling also needs a budget. New accepted messages can be checked frequently, then less often as they age, while a periodic sweep repairs rows missed by a crashed worker. Alert on the oldest unreconciled message and on queue age, not merely worker uptime. The dangerous failure is a healthy process advancing through only the newest rows while an old page remains stranded forever.

For this application, the notification payload should carry the minimum routing context required by the support team. Provider delivery metadata is not a clinical record. Keep sensitive form content out of alert bodies when a case identifier and authenticated link will do, and assess each provider's current regional and compliance commitments directly; the pending domestic email vendor is not evidence for China compliance, while geography-based SMS abuse controls and country-price circuit breakers remain application responsibilities.

## Rejected option and the case where it wins

The rejected design is synchronous fanout from the form handler: send email, wait, send SMS on failure, and return success only when a provider call does. It appears simple but gives request timeouts authority over retry policy, leaves an ambiguous result when a response is lost, and couples form availability to two external channels.

There is a valid use case for the webhook-centric alternative. If the service-level objective demands near-immediate delivery-event reactions, journeys with several branches, or automatic email-to-SMS fallback, this polling design is unsuitable and a provider centered on pushed events and orchestration is the better boundary. Customer.io, Courier, and Knock belong in that evaluation; SendGrid and Resend belong in the email-focused evaluation; Twilio belongs in the SMS evaluation. Test the exact workflow rather than inferring it from category labels, because a checkbox marked “webhooks” says nothing about ordering, retries, signature rotation, event retention, or what happens when the receiver is unavailable for six hours; those details decide whether the pushed design really improves recovery or merely moves an unbounded queue from your database into somebody else's system.

Infrai is the reasonable choice when beginner-friendly direct sends, consistent discovery, and operational consolidation matter more than those orchestration features. It has no SMTP relay, voice, WhatsApp, or RCS channel in this capability set. Email has no hosted OTP operation, scheduled email has no cancellation operation, cost reporting cannot be aggregated by tag through an API, and the application must maintain any email-code fallback itself. These are design constraints, not footnotes.

## References

- [Resend documentation](https://resend.com/docs/introduction)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [PostgreSQL unique constraints](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-UNIQUE-CONSTRAINTS)
- [PostgreSQL explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [RFC 9110: Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html#name-retry-after)
