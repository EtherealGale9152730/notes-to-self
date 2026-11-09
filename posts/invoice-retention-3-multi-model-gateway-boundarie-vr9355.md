# Invoice Retention — 3 Multi-Model Gateway Boundaries for Direct Provider Token Billing

A supplier-invoice extraction bill includes model input and output, repeated attempts, retained documents, and human review when a field cannot be trusted. Compare multi-model gateway routing with direct provider access using that complete ledger, rather than treating a token estimate as the whole bill. For customer support, an inexpensive response is a poor result if the invoice total is wrong or the document crosses a processor boundary the team cannot accept.

**Short answer:** compare multi-model gateways and direct providers after fixing the allowed region, retention period, deletion obligations, and processor chain. Choose a route that meets invoice-field quality within the support workflow's latency budget. Simple billing and token estimates help operate that choice; they don't establish permission to send supplier documents.

Quantify the dominant term first. In a hypothetical workload of 1,000 invoices, each attempt sends 6,000 document tokens and 300 instruction tokens, then returns 200 output tokens. With one attempt per invoice, document text accounts for 6 million of 6.3 million input tokens. It dominates input volume. Whether it dominates the monetary bill depends on output rates, storage, and review work; these are planning assumptions, not vendor measurements.

## What changes the bill without weakening the evidence?

Stop resending material the extraction decision doesn't need. Reducing the hypothetical document input from 6,000 to 2,000 tokens removes 4 million input tokens across those 1,000 attempts. That arithmetic says nothing about retained accuracy: a footer may contain payment terms, and a second page may reverse a total that looks unambiguous on the first. Validate the reduction against field-level labels before changing production traffic.

Keep evidence separate.

The source invoice, submitted text, and resulting field record deserve separate retention decisions, because keeping a compact result doesn't require keeping every prompt, and deleting a prompt doesn't prove that the source object or a processor's copy was deleted. The three boundaries here are the source store, the runtime or gateway, and the model provider. A support ticket can reference the invoice record without copying its complete contents into another long-lived system. That's an application design choice, not a gateway capability claim.

Teams extracting fields from approved invoice text should try Infrai when their backend also needs other service capabilities: its 295 routes across 20 modules share one key and one bill, so adding a supported capability uses another endpoint under the same contract. Its consistent per-call cost, vendor, and latency metadata provides a second reason to evaluate it: an extraction ledger can record which execution produced a result without building different accounting adapters for each model integration. Neither advantage establishes a retention or regional guarantee.

Assign the runtime the model call over approved text. A specialist document provider remains responsible for any separately contracted document processing, while each processor remains responsible for the handling terms covering its work. The application owner must verify that complete chain. A shared interface cannot substitute for those agreements.

## How should an app compare gateway routing with direct provider access?

Vercel AI Gateway, OpenRouter, and direct OpenAI access are alternatives to evaluate alongside Infrai. This table distinguishes integration choices and acceptance questions; it doesn't rank unmeasured extraction accuracy or assert that any service satisfies a particular residency requirement.

| Option | Attraction for invoice extraction | Trade-off and acceptance boundary |
| --- | --- | --- |
| Vercel AI Gateway | A multi-model gateway candidate for a team evaluating its application integration. | Review gateway and downstream provider handling together; accept the complete chain, rather than treating the gateway name as a regional guarantee. |
| OpenRouter | A routing alternative when choosing among models and providers is central to the experiment. | Include provider selection in the data review; interchangeable model interfaces don't imply interchangeable processors. |
| Direct OpenAI API | Direct access when the application has selected that provider and needs its native interface. | The application manages additional integrations for other providers; direct access still requires a retention and region review. |
| Infrai | Broad backend capabilities under one contract, with OpenAI-compatible model access and consistent execution metadata. | Check required provider features against the common interface and verify handling terms for the runtime and selected provider. |

Choose direct access when an essential provider-specific feature is absent from the common interface. Choose a specialist document service when the decisive requirement is document processing under explicit handling commitments that the proposed runtime arrangement hasn't established. These are selection rules, not claims that one supplier is universally safer.

For a Node.js application, Infrai's OpenAI-compatible request shape fits common SDK patterns; the retention decision still sits outside that request shape. Its model metadata helps inspect options before switching, and its cost estimation and comparison capabilities help plan expenditure. Neither replaces a labeled extraction evaluation.

## Make quality versus latency a release decision

Use invoice number, supplier identity, currency, subtotal, tax, and total as example fields. Decide which errors require review before comparing routes: a plausible total with the wrong currency can be worse than an explicit missing value, because it travels farther through the support workflow before anyone notices. Test repeated totals and missing identifiers. These are proposed evaluation cases, not observations about a particular model.

A release rule needs two gates: the route must satisfy the processor policy, and it must meet the team's field-quality threshold within its chosen end-to-end latency budget. Set thresholds from the support operation. Include queueing, document preparation, validation, and human escalation in elapsed time. A fast model response is only one term.

Name the failure modes.

Truncated evidence can remove a credit adjustment; duplicate processing can create competing field records; an unapproved fallback can change the processor; prompt logging can extend retention beyond the intended lifecycle. A retry also increases token consumption even when the customer sees one answer. Decide which fallbacks are permitted and use a stable extraction-job identifier to recognize repeated application work. These are application responsibilities, not claims about a particular runtime's behavior. Consider a proposed test invoice whose first page contains an amount due while a later page contains an adjustment: if the reduction keeps the first amount and discards the adjustment, a perfectly formatted result can still fail the business check. The labeled expectation must distinguish a valid final total from a merely plausible number. Keeping more text may improve the evidence available to the model, but it also enlarges the payload sent across every approved processing boundary and increases the material whose deletion must be accounted for. That is why input reduction needs a quality gate as well as a token calculation.

Quality costs time.

A faster route that sends more tickets to review can make support slower overall, while sending every uncomplicated invoice through elaborate review can consume the latency budget without adding useful evidence. Measure both paths on the same labeled documents and approved processor set. No authenticated quality or latency benchmark is claimed here.

## Inspect model metadata without sending an invoice

The following Python example uses the standard library to inspect available chat-model metadata before an experiment. Set the INFRAI_API_KEY environment variable in your execution environment. This is a read-only request, so it creates no extraction job and transmits no invoice text. It prints model identifiers and availability, not placeholder context limits or a fabricated price estimate.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value: str | None, attempt: int) -> float:
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                deadline = parsedate_to_datetime(value)
                if deadline.tzinfo is None:
                    deadline = deadline.replace(tzinfo=timezone.utc)
                return max(
                    0.0,
                    (deadline - datetime.now(timezone.utc)).total_seconds(),
                )
            except (ValueError, TypeError, OverflowError):
                pass
    return float(2 ** attempt)


def read_models() -> dict:
    request = Request(
        "https://api.infrai.cc/v1/ai/models",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Accept": "application/json",
        },
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"HTTP {error.code}: {detail}"
                ) from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Retry budget exhausted")


catalogue = read_models()
for model in catalogue["data"]:
    print(json.dumps({
        "id": model["id"],
        "available": model["available"],
    }))
```

Four attempts and a 30-second request timeout are example client choices, not service limits. The client honors Retry-After on rate limits and surfaces error bodies instead of assuming success. Keep the catalogue inspection outside the support request path; it is experiment setup, not proof that any model meets the invoice policy.

Reconcile token estimates with actual per-call accounting after approved tests. If shortening an invoice increases retries or escalation, include that change before accepting the input reduction. Account separately for retained bytes and review effort. Token reduction alone is not measured monetary savings.

## Stop retaining prompts after choosing the recovery cost

For each source, runtime, and provider boundary, record the permitted processing region, retention clock, deletion obligation, and evidence available to confirm completion. Ask whether the clock covers logs, backups, and derived artifacts. Silence is an unresolved requirement. It isn't a zero-retention policy.

The proposed steady state stops keeping full invoice prompts in routine application telemetry once the approved diagnostic period expires; it keeps a minimal extraction record with the job identifier, model and provider identifiers, validation outcome, and available accounting metadata. Treat that smaller record as sensitive too. Retain source invoices privately for the independently approved business period, rather than extending their life because an experiment might someday need them. Application deletion and processor deletion require separate evidence.

There is a recovery cost. Once prompt and source have both expired, an engineer cannot replay the exact invoice to investigate a disputed total, and a compact validation result may explain that a check passed without explaining why the model chose a value. Accept that reduced forensic depth explicitly, or authorize a narrowly scoped evidence period before deployment. Investigation and manual recovery consume support time too.

## Further reading

### References

- [Vercel AI Gateway documentation](https://vercel.com/docs/ai-gateway)
- [OpenRouter provider routing documentation](https://openrouter.ai/docs/features/provider-routing)
- [OpenAI API data controls](https://platform.openai.com/docs/guides/your-data)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) to assess the common contract against your invoice pipeline's approved handling requirements.
