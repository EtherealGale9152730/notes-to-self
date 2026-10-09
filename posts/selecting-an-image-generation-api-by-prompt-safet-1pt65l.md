# Selecting an Image Generation API by Prompt Safety Retention (Without Dedicated Moderation)

TL;DR: Select the image API only after defining the safety record you can afford to retain. For a B2B SaaS workflow that turns sales calls into CRM actions and then generates account-summary artwork, the least complex defensible design stores the transcript segment identifiers, normalized prompt, schema-validated safety decision, request fingerprint, and output reference; it does not keep duplicate image bytes in the moderation ledger. The dominant storage term is usually the generated media, so separating evidence from payload changes the bill far more than shaving fields from JSON.

The arithmetic should come first. Let `N` be generation attempts per day, `B` the average retained image bytes per attempt, `E` the evidence bytes per attempt, `R` the retention days, and `C` the number of full media copies. Steady-state storage is approximately `N * R * (C * B + E)`. This is a capacity equation, not a vendor price claim. In an illustrative workload of 50,000 attempts per day, a 2 MB image, 8 KB of evidence, 30 days of evidence, and 7 days of media, one media copy occupies about 700,000 GB-days across that seven-day window, while thirty days of evidence occupies about 12,000 GB-days. The units deliberately expose the comparison without pretending that every provider bills storage the same way.

**My selection rule is therefore simple:** reject any API integration that cannot be wrapped in a deterministic, schema-validated gate and an independently controlled retention policy. A dedicated moderation endpoint is optional. A reviewable decision record is not.

## What are we actually paying to remember?

The workflow has three different data classes, and treating them as one blob is the expensive mistake. The source class contains a call transcript or a narrow set of transcript spans. The decision class contains the proposed CRM actions, the derived image prompt, a policy version, a typed safety verdict, and stable hashes. The media class contains the generated image plus its delivery metadata. Their retention requirements should not be coupled merely because one request happened to produce all three.

For example, a sales call might yield the CRM action `schedule_security_review`, while a separate presentation step asks for an image representing a secure deployment workshop. Structured output correctness matters twice: the CRM action must match its schema, and the prompt gate must return one allowed decision shape. Free-form prose such as "probably okay" has no useful operational meaning.

| Record | Why retain it | Dominant failure mode | Sensible boundary |
|---|---|---|---|
| Transcript span IDs and hashes | Reconstruct why an action or prompt existed | Full transcripts leak unrelated customer speech | Keep references and the minimum necessary excerpt |
| Safety decision JSON | Prove which policy evaluated which normalized prompt | Schema drift silently changes verdict meaning | Version the schema and policy together |
| Request fingerprint | Detect duplicate submissions and retry ambiguity | A retry creates multiple images with one business intent | Bind it to tenant and workflow action |
| Generated media | Deliver and review the result | Large objects dominate retained bytes | Expire independently from evidence |
| Provider response metadata | Diagnose latency and request outcomes | Unbounded raw payloads become accidental archives | Allowlist fields; hash the rest when possible |

The transcript itself may originate from an open speech-recognition system such as Whisper, but that choice does not change the boundary: transcription confidence, CRM extraction, prompt safety, generation, and retention are separate stages. Collapsing them makes deletion hard to reason about and makes a provider swap much riskier than it needs to be.

Short records win.

## How should an image generation API handle prompt safety without moderation?

Without a specialized moderation endpoint, use a chat-capable policy evaluator that can return JSON matching a strict schema. The contract should be small: allow or deny, enumerated reason codes, a policy version, and an optional sanitized prompt. Do not ask the evaluator for an essay and then parse its mood.

The following Python example validates shape, rejects unknown fields, canonicalizes the decision before hashing, and keeps the image generator behind a generic interface. It intentionally omits a model identifier and network route because those are deployment choices, not properties of the safety contract.

```python
import hashlib
import json
from dataclasses import dataclass
from typing import Any, Protocol


ALLOWED_REASONS = {"allowed", "sexual", "violence", "hate", "personal_data", "unknown"}


class PolicyEvaluator(Protocol):
    def decide(self, prompt: str, policy_version: str) -> dict[str, Any]: ...


class ImageGenerator(Protocol):
    def generate(self, prompt: str, idempotency_key: str) -> str: ...


@dataclass(frozen=True)
class SafetyDecision:
    allowed: bool
    reason: str
    policy_version: str
    sanitized_prompt: str | None


def validate_decision(raw: dict[str, Any], expected_version: str) -> SafetyDecision:
    expected = {"allowed", "reason", "policy_version", "sanitized_prompt"}
    if set(raw) != expected:
        raise ValueError("decision fields do not match the contract")
    if type(raw["allowed"]) is not bool:
        raise ValueError("allowed must be a boolean")
    if raw["reason"] not in ALLOWED_REASONS:
        raise ValueError("unrecognized reason code")
    if raw["policy_version"] != expected_version:
        raise ValueError("policy version mismatch")
    sanitized = raw["sanitized_prompt"]
    if sanitized is not None and not isinstance(sanitized, str):
        raise ValueError("sanitized_prompt must be a string or null")
    if raw["allowed"] and raw["reason"] != "allowed":
        raise ValueError("allowed decisions require the allowed reason")
    return SafetyDecision(**raw)


def canonical_hash(value: dict[str, Any]) -> str:
    encoded = json.dumps(value, sort_keys=True, separators=(",", ":")).encode()
    return hashlib.sha256(encoded).hexdigest()


def create_account_visual(
    tenant_id: str,
    action_id: str,
    proposed_prompt: str,
    evaluator: PolicyEvaluator,
    generator: ImageGenerator,
) -> dict[str, str]:
    policy_version = "sales-visual-v3"
    raw = evaluator.decide(proposed_prompt, policy_version)
    decision = validate_decision(raw, policy_version)
    evidence_hash = canonical_hash(raw)
    if not decision.allowed:
        return {"status": "denied", "evidence_hash": evidence_hash}

    final_prompt = decision.sanitized_prompt or proposed_prompt
    request_key = hashlib.sha256(
        f"{tenant_id}:{action_id}:{policy_version}:{final_prompt}".encode()
    ).hexdigest()
    object_ref = generator.generate(final_prompt, request_key)
    return {
        "status": "generated",
        "evidence_hash": evidence_hash,
        "object_ref": object_ref,
    }
```

This gate fails closed on malformed output, a stale policy version, an unknown reason, or a contradictory verdict. It also scopes the generation key to tenant, action, policy, and final prompt. That key supports duplicate detection, but it does not magically make a generation request idempotent: HTTP defines idempotent methods by their intended effect, and a non-idempotent operation should not be retried automatically unless the client knows the request was not applied or has application-specific protection. RFC 9110 is explicit about that retry boundary.

There is an uncomfortable limit here. Schema validity proves structure, not policy quality. A perfectly typed `allowed` decision can still be wrong, so a candidate evaluator needs a fixed, versioned test corpus containing allowed prompts, prohibited prompts, ambiguous prompts, transcript injection attempts, and prompts with customer identifiers. Measure false allows, false denies, invalid JSON, timeouts, and disagreement after policy revisions separately; one blended score hides the failure that matters. This approach also adds a second inference step, another timeout surface, and another policy artifact to deploy. It is not suitable when policy requires a specialized classifier, a human decision before generation, or analysis of the finished image rather than the text prompt. In those cases, use the required classifier or review queue and treat prompt gating as an early filter, not as the final control.

No schema fixes that.

## Retention changes the selection matrix

API selection often starts with image quality and latency. Those matter, but the integration becomes operationally brittle when output storage, request logs, and policy evidence cannot be separated. The comparison below stays at the capability level because product terms change and a durable engineering note should survive those changes.

| Criterion | Evidence to collect in a trial | Disqualifying boundary |
|---|---|---|
| Structured gate correctness | Schema-valid rate and labeled-corpus confusion matrix | Safety decisions require parsing unconstrained prose |
| Retry control | Duplicate results under forced disconnects and timeouts | No way to correlate one business intent across attempts |
| Data lifecycle | Documented deletion behavior for prompts, logs, and images | Retention cannot meet the application's policy |
| Observability | Correlation IDs, status classes, and latency by stage | Failures cannot be assigned to gate or generator |
| Portability | Exportable image objects and provider-neutral evidence | Audit history exists only inside one provider console |
| Operational limits | Published request, payload, and concurrency limits | Required limits are undocumented or untestable |

Run the trial with the same corpus, dimensions, retry schedule, and acceptance rubric for every candidate. Three outcomes deserve separate counts: policy denial, technical failure, and successful generation. Treating a denial as an API error encourages retries that cannot improve the result; treating a timeout as a denial corrupts safety metrics.

This is a trade-off, not a workaround that erases moderation risk.

Correctness also outranks convenience in the CRM half of the system. If `next_step_date` is required, the extractor must either return a valid value or an explicit incomplete state. It must never invent a date so the image stage can proceed. An attractive account brief attached to the wrong follow-up action is a data-integrity failure with a polished surface.

## Failure modes worth testing before launch

Start with the retry path. Force a connection loss after request transmission but before the response arrives, then verify that the workflow records an indeterminate attempt rather than blindly producing another object. Reconcile by application key where the integration supports it; otherwise, require an operator or a bounded job to decide whether another generation is acceptable.

Then test policy drift. Replay the labeled corpus whenever the evaluator, prompt template, schema, or policy version changes. Store both old and new decisions long enough to explain the delta, but do not extend raw transcript retention solely for convenient regression testing. A de-identified, reviewed corpus is a different asset and should have its own ownership and deletion rules.

The less obvious failures are cross-tenant cache keys, Unicode normalization differences, truncated structured output, a sanitized prompt that becomes empty, and logs that capture the full transcript despite the evidence record storing only hashes. Consider one concrete retry sequence: the gate allows a normalized prompt, generation finishes remotely, the client loses the response, and an automatic retry creates a second image before either object reference reaches the ledger. If the request fingerprint exists only in process memory, the two media objects look unrelated after a restart. Persisting intent before submission makes reconciliation possible, although it cannot guarantee that an external service will deduplicate the operation. The log pipeline deserves the same redaction tests as the database. Otherwise the official retention table is fiction.

Finally, put stage-specific counters and latency histograms around extraction, policy evaluation, image generation, object persistence, and deletion. Alert on invalid decision shape and deletion backlog directly. An overall success rate cannot tell an on-call engineer whether the CRM contract failed, the safety gate denied correctly, or the media service timed out.

## Keep less, and accept the consequence

The material change to the dominant storage term is expiring generated media sooner than safety evidence, while retaining a stable object hash and the compact decision record. Deduplicating JSON or shortening reason names barely matters beside image payloads. Compression and lifecycle tiers may alter physical bytes or billing, but they do not repair an undefined retention policy.

**Deliberately stop keeping duplicate media, full provider payloads, and entire call transcripts in the generation ledger.** Keep the minimum transcript evidence permitted by the business and legal policy, a versioned structured decision, request correlation, hashes, and deletion outcomes. This narrows exposure and makes capacity predictable.

The cost appears during an investigation: after media expiry, an operator may prove which prompt and policy decision produced an object hash but may be unable to inspect the original pixels. After transcript expiry, the team may be unable to rerun a newer extractor against the original call. Those losses are real. Set the media and transcript windows from the actual dispute, support, and regulatory requirements, record the decision, and test deletion as carefully as creation. The best API for this workflow is the one that fits that boundary without forcing the evidence system to become a permanent archive.

## Further reading

- RFC 9110: HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- Whisper open-source speech recognition repository: https://github.com/openai/whisper
