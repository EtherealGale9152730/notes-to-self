# Suppressed Recipient Password Reset Email Checks (With Operator-Owned Compliance Templates)

TL;DR: For a property manager whose resident cannot open a compliance notice because account-recovery mail never arrives, check the delivery event and the recipient's suppression state before sending again. Keep the notice and recovery templates under application control, record exactly which immutable template revision was rendered, and remove a suppression only after the address has been corrected or the cause has been resolved. A blind retry preserves neither deliverability nor a defensible audit trail.

The least complex design is one application-owned send ledger joined to normalized transport events. Its largest long-lived storage term is usually repeated event payload and rendered-message retention, not the one-row decision that says why a send was allowed or blocked. Model that term before adding another webhook archive:

`retained bytes = sends x events per send x stored event bytes x retention factor`

This is a capacity formula, not a benchmark. Measure all four inputs from the actual workload. If a reset attempt produces several transport callbacks and every callback stores the full rendered body, recipient metadata, and original payload, duplication grows with every event. Store one immutable render record, small event projections that refer to it, and an append-only suppression decision. That change attacks the repeated term while preserving the evidence an operator needs.

## What should the audit retain, and what should it discard?

A compliance workflow creates two related records. The compliance notice proves what the property operator intended to communicate. The password-reset message gives a resident temporary access to the account where that notice can be read. Combining them into one mutable "latest email" row makes both records weaker: a later retry can overwrite the content revision, delivery outcome, or destination used by the earlier attempt.

Keep four compact objects instead. The notice record identifies the lease or case, approved content revision, audience, and creation time. The recovery-attempt record identifies the account, a server-generated attempt ID, the application-owned template revision, and the transport message correlation ID. Event rows store only the normalized outcome, event time, source event ID, and a pointer to the attempt. A suppression decision stores the address fingerprint, scope, reason category, evidence event, decision time, and actor or policy that changed it.

The address fingerprint should be produced by a keyed construction so an offline reader of the database cannot cheaply test a list of known addresses against plain hashes. Key rotation and access to the key are separate operational concerns. If support staff must see the address, keep that separately encrypted and access-controlled rather than copying it into every event.

| Data | Retention purpose | Avoid duplicating |
|---|---|---|
| Approved notice and template revision | Prove which content was selected | Rendered body in every callback |
| Recovery attempt | Correlate request, render, and send | Reset token or secret |
| Normalized delivery event | Diagnose accepted, bounced, or blocked processing | Entire webhook payload by default |
| Suppression decision | Explain why later sends were stopped or permitted | Mutable reason text with no history |

Passwords and reset tokens do not belong in this ledger. OWASP recommends random, sufficiently long, securely stored, single-use, expiring reset tokens, consistent responses for existing and nonexistent accounts, and rate limiting. Those controls also mean the support view must not reveal whether an email address has an account merely because an unauthenticated caller asks why a message did not arrive.

I would deliberately stop retaining routine raw callbacks after their short investigation window, provided the normalized projection, source event ID, integrity metadata, and immutable render reference have been committed. The loss is concrete: an unusual parser dispute discovered after expiration cannot be reconstructed from the full original payload. Keeping every raw body forever merely moves the risk into a larger, more sensitive archive. Set the window from legal, security, and incident-response requirements, then test that deletion actually occurs.

## Why was the recipient's password reset email bounced or suppressed?

Treat "not received" as a state-reconstruction problem. Start with the recovery attempt ID generated before transport submission. If no attempt exists, the failure is before delivery: request throttling, account-state policy, or job creation deserves investigation. Do not infer a transport problem from an empty inbox.

Start there.

If the attempt exists but has no transport correlation ID, inspect the enqueue and submission boundary. If it has an ID, reconcile the latest normalized transport event. A bounce-related suppression should point back to the event that created it. A manually created suppression should identify the authorized actor and reason. An imported suppression needs its import batch and source. Without provenance, an operator cannot distinguish a deliberate safety control from stale state.

No event is also evidence, but weak evidence. It can mean the callback has not arrived, correlation failed, or the event consumer is delayed. Compare event lag with the service's measured operating window before declaring a message lost. This is why callback receipt time and provider event time should be separate fields.

Do not remove a suppression merely because the resident clicked "forgot password" again. First classify it. If the destination was mistyped, update the verified account address and create a new recovery attempt. If the address is correct but the evidence indicates a persistent rejection, repeated submission is unlikely to establish anything except that the system ignored its own control. If the cause is transient, use an explicit retry policy with a cap and record the policy decision. If the cause cannot be established, escalate for controlled review rather than converting uncertainty into an automatic unsuppression.

The user-facing response remains neutral across these branches. That prevents account discovery, while the authenticated support workflow can use the attempt ID and authorized resident-verification process to inspect the audit record. The distinction matters. Public ambiguity and internal traceability can coexist.

SPF is one input to domain authorization, not proof that a particular mailbox received a particular message. RFC 7208 defines SPF as a way for a receiving system to check whether a host is authorized to use a domain in SMTP identity terms, and it explicitly notes that authorization does not validate message content. Record authentication evidence when available, but do not turn an SPF result into a delivery receipt or a reason to erase a recipient-level suppression. This distinction is easy to lose during support triage: a domain-level authorization result, a transport event, a suppression decision, and a resident's report that nothing arrived are four observations with different scopes, so the investigation should preserve each one rather than collapsing them into a single delivered-or-failed flag.

## Template custody determines what the record can prove

For this workflow, the property operator should own the semantic templates: the compliance notice, the recovery instructions, required legal language, and the variables permitted in each revision. The transport boundary may accept already rendered subject and body content, but it should not silently choose a different mutable template with the same friendly name. Otherwise the audit can prove that a message ID existed while failing to prove what the resident was sent.

Use immutable revision identifiers and a digest of the canonical rendered content. Record the locale and the non-secret variable schema. Review changes as code or controlled content artifacts, and promote a revision rather than editing it in place. The send ledger then answers a narrow question: which approved revision and input set produced this render?

The trade-off is operational weight. An immutable revision catalog, render digests, keyed recipient fingerprints, and event normalization require migrations, key custody, deletion jobs, and reconciliation tests. This design is a poor fit for a low-risk message stream with no audit obligation and no staff able to operate that machinery; a smaller append-only attempt log with short retention may be the more defensible choice there. It is also insufficient where the governing process requires proof of receipt rather than evidence of content selection and mail handling. In that case, choose an approved acknowledgment or legally recognized delivery workflow instead of asking email telemetry to prove something it cannot prove.

That limit is deliberate.

The following Python sketch shows the boundary. It omits persistence and key management, but the fields expose the important ownership decision. The transport receives content; it does not own the template revision.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
import hashlib
import hmac
import json


@dataclass(frozen=True)
class RecoveryRender:
    attempt_id: str
    template_revision: str
    locale: str
    subject: str
    body: str


def canonical_digest(render: RecoveryRender) -> str:
    document = {
        "body": render.body,
        "locale": render.locale,
        "subject": render.subject,
        "template_revision": render.template_revision,
    }
    encoded = json.dumps(
        document, sort_keys=True, separators=(",", ":")
    ).encode("utf-8")
    return hashlib.sha256(encoded).hexdigest()


def recipient_fingerprint(address: str, audit_key: bytes) -> str:
    normalized = address.strip().lower().encode("utf-8")
    return hmac.new(audit_key, normalized, hashlib.sha256).hexdigest()


def build_send_record(
    render: RecoveryRender, address: str, audit_key: bytes
) -> dict[str, str]:
    return {
        "attempt_id": render.attempt_id,
        "template_revision": render.template_revision,
        "render_digest": canonical_digest(render),
        "recipient_fingerprint": recipient_fingerprint(address, audit_key),
        "recorded_at": datetime.now(timezone.utc).isoformat(),
    }
```

A digest detects later content mismatch; it does not prove delivery and is not a substitute for retaining legally required notice content. HMAC-based fingerprinting reduces casual disclosure; it does not make address data anonymous while the organization retains the key and can relink the value. Name those limits in the data classification.

Deployment should preserve the same boundary. Validate template variables before enqueueing, canary a new immutable revision, and keep the previous revision available for rollback without rewriting old records. Test a suppressed address, a malformed callback, duplicate events, callbacks arriving out of order, and deletion at the retention boundary. Event ingestion should be idempotent on the source event ID, while state derivation should use explicit precedence rules rather than arrival order alone.

## The operating rule

A useful dashboard counts recovery attempts that stopped before submission, submitted attempts with no event beyond the measured lag window, newly created suppressions by reason category, and authorized suppression reversals. Alert on changes from the system's own baseline. An absolute threshold copied from another mail stream says little about a property portfolio's volume or resident population.

Cost belongs beside those signals, but it is not the decision maker. Track render-object bytes, normalized-event bytes, raw-callback bytes, index bytes, and archive retrieval work separately. Sample actual serialized records; object counts alone hide variable payload size, while logical row size hides indexes and replicas. A storage estimate that omits deletion lag is optimistic.

The decision rule is short: retry only when evidence and policy classify the previous failure as retryable; reverse suppression only after the destination or cause has been resolved and the change is attributable; keep the operator-owned template revision and delivery evidence immutable. For the resident, issue a fresh reset attempt after resolution because recovery tokens are single-use and expire under the security policy. Never recover an old token from the audit store.

This design does not promise that email is a delivery receipt. It gives the property operator a bounded, reviewable record of content selection, submission, transport evidence, suppression, and recovery action. That is enough to troubleshoot the missing reset without weakening account enumeration defenses or confusing access recovery with proof that a compliance notice was read.

## Further reading

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
