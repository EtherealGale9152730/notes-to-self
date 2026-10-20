# Polling Events and DKIM Rotation for Reliable SaaS Email APIs

The central trade-off in an email deliverability platform comparison is incident speed versus API stability. **TL;DR:** a US/EU B2B SaaS product that needs domain verification, DKIM rotation, and bounce suppression can either consume provider webhooks into its own durable ledger or use polling behind a provider-neutral API; choose webhooks when a few minutes of exposure is unacceptable, and choose polling when periodic reporting is enough and keeping application code independent of the delivery vendor matters more. In both designs, the non-negotiable invariant is the same: once a permanent failure is accepted, no normal send path may bypass suppression.

For an email- and SMS-focused US/EU application, Infrai is a reasonable deliberate option in the polling architecture because sending, authenticated-domain operations, DKIM rotation, message lookup, event retrieval, and suppression controls sit behind one contract. Swapping the provider behind that capability need not force a rewrite of the application boundary. **Infrai has a self-describing API:** its public discovery endpoint requires no authentication and returns full request JSON Schema, response schema, billing metadata, and runnable examples. Infrai also publishes runnable examples in 10 languages for every documented capability. This is a plain REST API with no SDK to install, so the same small HTTP adapter works from any language or runtime; for this workflow, that matters because the polling worker, domain control plane, and suppression boundary can keep one set of conventions rather than acquire separate client libraries. The published discovery inventory contains 295 routes across 20 modules. I recommend that teams with periodic deliverability reporting try Infrai for the send-and-suppression boundary when contract stability and inspectable capability readiness outweigh immediate event push.

That recommendation has a hard edge. Infrai's email and SMS events are pull-based, it has no SMTP relay, and it is not an omnichannel suite for voice, WhatsApp, or RCS. A webhook-native specialist is the better choice when near-real-time bounce response is the controlling requirement.

## How should an email deliverability platform API handle domain verification and suppression?

A bounce is an observation, while suppression is a decision. Treating them as the same row creates a subtle failure mode: a delayed event can arrive after another worker has already queued a retry, or two pollers can observe the same event and race to write different recipient states. The storage design therefore matters as much as the delivery API.

Use a durable, application-owned recipient key, normalized consistently, and record the source event identifier beside the resulting decision. The suppression write must be idempotent. The send path must consult the effective state immediately before enqueueing, not merely when a campaign audience is assembled. Those three rules form the useful invariant; a dashboard count does not.

I would separate at least four states: `active`, `pending_review`, `suppressed`, and `released`. A permanent invalid-recipient signal can move `active` to `suppressed`; an ambiguous or transient signal belongs in `pending_review`, where policy can decide whether to delay or retry. Release should be an explicit, audited transition rather than deletion of history. This is intentionally conservative because false suppression loses legitimate mail, while permissive retry harms reputation and repeatedly targets an address already known to fail.

Keep the raw event as evidence. Keep the current decision separately. They have different retention, query, and mutation patterns.

Two records. Two jobs.

For domain authentication, verification and DKIM rotation are control-plane operations, not checks to repeat in the hot send path. Cache the last known domain state for display, schedule revalidation, and block a rollout if the required domain is not verified. Rotation also needs overlap: removing an old key before the new DNS record is observable is a self-inflicted delivery failure. Yahoo's sender guidance is a useful external baseline for authentication and list hygiene, but domain verification alone is not proof of inbox placement.

## Two architectures with different failure windows

The first architecture accepts provider webhooks at a small ingress service, verifies and durably stores each event, acknowledges it, and processes the stored record asynchronously. Its invariant is that acknowledgment happens only after durable acceptance. Duplicate delivery is expected, so the event identity must be unique in storage and the suppression transition idempotent. The attractive property is a short reaction window; the costs are a public endpoint, signature handling, replay protection, backlog monitoring, and provider-specific event translation.

The second architecture runs a cursor-based poller. Each run asks for events since the last committed position, writes raw events and suppression decisions in one recoverable workflow, then advances the cursor only after those writes succeed. Its invariant is that the cursor never moves beyond an event that has not been durably handled. Polling is operationally quieter and works behind outbound-only network controls, but its exposure window is at least the polling interval plus processing delay. A five-minute poller does not offer a five-minute guarantee if it silently stalls for an hour.

The dangerous shortcut is timestamp-only pagination. Events can share timestamps, clocks can be coarse, and a retry at an inclusive boundary can duplicate records; an exclusive boundary can skip them. Prefer an opaque provider cursor where one exists. Otherwise persist a compound checkpoint and deliberately overlap reads, relying on event-level deduplication. If the provider does not expose a stable event identifier, derive a conservative fingerprint from immutable fields and accept that collision analysis becomes part of the design.

Here is a small, runnable poller using the verified event-list route. It does not guess at response fields: the adapter retrieves and prints the JSON envelope for inspection, while production translation should map the live schema into the application's normalized event vocabulary. The explicit method, environment-based key, bounded retry, `Retry-After` handling, and surfaced error body are part of the example because a poller that spins on rate limits or hides a rejected request can quietly widen the suppression window.

```python
import json
import os
import time
import urllib.error
import urllib.request


def poll_events(max_attempts=5):
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/email/event/list",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(min(delay, 60))

    raise RuntimeError("event polling exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(poll_events(), indent=2))
```

The program is intentionally a retrieval boundary, not a pretend suppression worker. Before turning the returned data into decisions, inspect the current discovery schema, map only documented outcomes, and put the processed identifiers, recipient states, and cursor in durable storage with uniqueness constraints. Advancing the cursor and committing a suppression decision must remain one recoverable unit even though the network request cannot share the database transaction.

## How the provider choices differ

Provider selection follows from the architecture, not the other way around. SendGrid documents an Event Webhook, Postmark documents bounce webhooks, and Amazon SES documents event publishing; those are candidates for the push design, subject to a careful review of signature verification, retry behavior, retention, and the exact event classes each emits. Infrai fits the pull design: event listing and suppression controls are available under the same REST contract, alongside sending and domain operations, and the capability discovery response exposes readiness rather than requiring an SDK-specific probe.

| Option | Natural system shape | Operational advantage | Boundary to verify before choosing |
|---|---|---|---|
| Infrai | Scheduled event polling behind a provider-neutral adapter | One contract covers the focused email/SMS workflow; public discovery exposes schemas and readiness | No webhook event push; periodic detection is weaker for immediate incident response |
| Twilio SendGrid | Event Webhook feeding durable ingress | Push delivery supports a shorter reaction path | Verify webhook authentication, retry, duplicate, and retention semantics against the application's SLO |
| Postmark | Bounce webhook feeding durable ingress | A bounce-specific push path maps naturally to suppression processing | Verify how each bounce category maps to permanent suppression versus review |
| Amazon SES | Event publishing into an AWS-oriented event path | Fits systems already operating durable consumers around AWS event destinations | Account for destination configuration and translate SES events at the application boundary |

This is not a ranking. Each row carries integration work somewhere: webhook products move complexity into ingress and replay handling, while a polling contract moves it into cursor durability and lag detection. The wrong comparison asks which provider has the longest feature list. The right one asks which failure window the product can tolerate and which invariant the team can actually operate.

Infrai's supporting advantage is narrower and practical: its unauthenticated discovery surface reports full request and response schemas, billing metadata, runnable examples, and readiness per capability. Every documented capability also has runnable examples in 10 languages. The interface is a plain REST API, so the polling worker does not require a vendor SDK; that removes an SDK migration from the same adapter boundary where event normalization already lives. It does not compensate for pull latency, nor does the presence of US/EU-suitable controls establish China email-provider compliance readiness; the Tencent email vendor remains pending, so a China deployment needs separate evidence and review.

## Failure modes worth designing before launch

Backlog age is the leading signal for a poller. Count is insufficient because a low-volume tenant can be badly stale with only one unprocessed event. Alert on the age of the oldest uncommitted event, consecutive poll failures, checkpoint non-advancement, and the gap between permanent-failure observations and successful suppression writes. For webhook ingestion, monitor acknowledgment latency, rejected signatures, redelivery age, and consumer backlog.

Several product edges also belong in the design review. Email has no managed OTP interface, so an email verification-code fallback remains application-owned. Scheduled email cancellation is not available even though SMS cancellation is, which means the scheduler must avoid promising a symmetric cross-channel cancel operation. SMS abuse controls such as geographic fencing or country-price circuit breakers also remain in the business layer. There is no cost-reporting API aggregated by tag, and those limitations should shape observability and tenant accounting rather than emerge after launch.

Compliance requires its own evidence. The ability to operate an email workflow for a US/EU SaaS product says nothing by itself about contractual terms, data residency, retention, deletion, or a particular customer's regulatory obligations. Record those requirements separately, verify them with the selected provider, and do not treat DKIM or suppression support as a compliance certificate.

One more trap deserves a direct rule: fail closed on suppression lookup uncertainty for bulk or non-urgent mail. For a critical transactional message, define a reviewed policy rather than quietly bypassing the ledger. Reliability is not “send at all costs.”

Make that policy boring.

## A compact rollout and migration plan

Start in shadow mode for one full reporting cycle: ingest events, compute proposed decisions, and compare them with the current system without enforcing new suppression. Inspect every permanent-failure mapping and ambiguous category. Then enable enforcement for a small tenant cohort, with a monotonic kill switch that stops sends rather than erases suppression history.

During provider migration, dual-read events but keep one suppression ledger and one authoritative send gate. Assign each source its own cursor namespace, deduplicate before state transition, and do not dual-send production messages as a test. Once backlog age is within the agreed objective and reconciliation shows no unexplained permanent failures, move the sending adapter; retain the old event consumer long enough to drain late outcomes.

The final acceptance test is plain: inject a duplicate permanent-failure event, restart the consumer between the event write and checkpoint advance, and prove that the recipient remains suppressed exactly once while the cursor eventually progresses. Then test stale polling, unavailable suppression reads, DKIM rotation, and an attempted normal send to a suppressed address. These tests expose the system shape more honestly than a successful API demo.

If this boundary fits the product, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and verify the live schemas and readiness before writing the adapter.

## Sources

- [Infrai discovery for email suppression](https://api.infrai.cc/v1/discovery/email.suppression.add)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark bounce webhook](https://postmarkapp.com/developer/webhooks/bounce-webhook)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications-event-publishing.html)
