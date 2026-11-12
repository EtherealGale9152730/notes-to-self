# Event Notifications — 3 API Evidence Tests for Reset Email Deliverability Troubleshooting

Start email deliverability troubleshooting for password-reset event notifications with a reproducible evidence review before retrying: for an e-commerce account, the deciding constraint is whether the team can explain a delivery decision without retaining the reset secret or implying that a transport acknowledgment proves account recovery.

TL;DR: Diagnose event notification email deliverability across three boundaries: account authorization, sender authentication, and transport feedback. Keep DKIM, SPF, domain verification, suppression, bounces, complaints, and API observations attached to the boundary they actually describe. Then test whether another reviewer can reconstruct the decision from retained evidence. A reassuring status label is insufficient.

This decision record selects a review method: reconstruct a reset attempt against the policy that applied when it occurred. It uses a proposed 300-second expiry as an example policy, not a standards requirement. The acceptance criterion is a reviewable explanation, including explicit gaps; it is not a claim that an evidence collection makes a system compliant with an unspecified regulation.

## What can evidence prove during email deliverability troubleshooting for event notifications?

The first invariant is separation of authority. The account service decides whether the reset remains unexpired and unused; sender authentication concerns mail identities; transport observations describe what a particular participant reported. OWASP's Forgot Password guidance calls for reset tokens to expire after an appropriate period and be invalidated after use. Neither an API acknowledgment nor a DKIM result substitutes for that authorization check.

Keep those boundaries.

Consider a proposed review fixture: the shopper requests a reset at 12:00:00 UTC, its expiry is 12:05:00, and a transport observation arrives locally at 12:06:00. That arrival alone cannot establish when the remote event occurred, whether the shopper saw the message, or whether the link was still usable. Retain the source time when available and the local observation time, with their meanings intact. At redemption, the account service must apply the reset policy independently of the email history. A worker's pre-submission validity check reduces avoidable stale submissions, but it cannot reserve validity throughout transit.

The second invariant is preservation of the decision's inputs. A reviewer needs the applicable suppression reason and policy version, rather than a lookup of the recipient's current suppression state. Otherwise a later complaint can make an earlier allowed attempt appear inexplicable, or a later policy change can erase the explanation for a withheld message. The proposed evidence record uses opaque reset and recipient references, an attempt reference, the decision time, expiry time, and the rule applied. It excludes the reset token, reset URL, password, and full message body.

Retention is a policy decision.

An object retained durably can still be misleading if it contains an undocumented transformation of a source event. Preserve provenance and the interpretation rule alongside any normalized status; define access and retention around the control being examined. This note specifies neither a legal retention period nor a certification. Those require requirements beyond a delivery troubleshooting workflow.

The third invariant is honest uncertainty. If the submission call times out after transmission, retain an unknown outcome; if complaint feedback is unavailable, retain that limitation. Turning either gap into a successful delivery count manufactures evidence the system does not possess.

## Compare review methods by the questions they leave unanswered

The selected method is a case reconstruction with explicit claims and supporting observations. That shifts the review away from the shape of a message dashboard: an investigator first states the claim, identifies the authority that could establish it, and checks whether the retained observation actually does so. A single successful reset redemption establishes something different from an authenticated sending domain, even when both appear under the same customer reference.

| Review method | Question it can answer | Evidence boundary | Decision for a short-lived reset |
| --- | --- | --- | --- |
| Inspect the latest delivery status | What status the integration currently exposes | May omit the original suppression decision and submission uncertainty | Useful as a triage pointer |
| Inspect current DNS and sender configuration | What configuration is observable now | Does not establish what a receiver observed for an earlier attempt | Useful for testing a current configuration hypothesis |
| Reconstruct the case from policy inputs and attributed observations | Why the attempt was allowed or withheld, and what later reports support | Requires retained provenance; missing observations remain missing | Selected as the release acceptance method |

The trade-off is extra storage and review work in exchange for reconstructing historical decisions: policy versioning, controlled evidence access, and a reviewer-facing explanation all need an owner. Storage volume also grows with retained observations. Choose retention and diagnostic detail deliberately, rather than collecting entire message bodies because they are convenient to search; message unit pricing isn't the deciding criterion for whether this review can answer its question.

For email deliverability troubleshooting, follow the first unsupported claim. If the claim is “we submitted this event notification,” inspect the application decision and submission observation before looking at DNS. If the attempt was suppressed, a changed signing selector cannot explain why no send was attempted. If the claim is “the receiver authenticated the sender,” look for evidence associated with the actual message identities and receiver observation, rather than treating administrative domain verification as its equivalent.

DKIM, specified in RFC 6376, verifies a signature associated with a signing domain. SPF, specified in RFC 7208, evaluates whether the connecting host is authorized for the checked SMTP identity. Preserve the relevant signing domain, selector, checked sender identity, and diagnostics when available. Administrative domain verification has a separate meaning: document the ownership check required by the submission system and the identity it enabled. A present-day DNS lookup cannot establish precisely what another system observed earlier.

Authentication has limits.

Neither DKIM nor SPF proves inbox placement, reset authorization, or recipient consent. A bounce and a complaint also answer different questions: a bounce reports delivery failure, while a complaint concerns recipient feedback. Keep those meanings separate when normalizing observations. RFC 3463 distinguishes success, persistent transient failure, and permanent failure classes; the application still needs a documented mapping from received diagnostics to its suppression or retry action. There is no universal retry rule supplied by the mere presence of an API.

## Make the critical path produce a reviewable claim

The following Python example generates the local decision record for the proposed policy. Its output can support the narrow claim that the application evaluated expiry, consumption, and suppression at a particular instant. It does not claim to send a message or observe a receiver, and its example identifiers are opaque references rather than credentials.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass(frozen=True)
class ResetState:
    reset_ref: str
    expires_at: datetime
    consumed: bool


def decide_attempt(reset, attempt_id, now, suppression_reason):
    if now.tzinfo is None or reset.expires_at.tzinfo is None:
        raise ValueError("Timezone-aware timestamps are required")

    if reset.consumed:
        action, reason = "withhold", "reset_consumed"
    elif now >= reset.expires_at:
        action, reason = "withhold", "reset_expired"
    elif suppression_reason is not None:
        action, reason = "withhold", suppression_reason
    else:
        action, reason = "queue", "eligible_at_decision_time"

    return {
        "attempt_id": attempt_id,
        "reset_ref": reset.reset_ref,
        "observed_at": now.isoformat(),
        "expires_at": reset.expires_at.isoformat(),
        "action": action,
        "reason": reason,
        "rule_version": "reset-send-v1",
    }


created_at = datetime(2026, 11, 9, 12, 0, tzinfo=timezone.utc)
reset = ResetState(
    reset_ref="reset-example-17",
    expires_at=created_at + timedelta(seconds=300),
    consumed=False,
)
decision = decide_attempt(
    reset=reset,
    attempt_id="attempt-example-42",
    now=created_at + timedelta(seconds=300),
    suppression_reason=None,
)
assert decision["action"] == "withhold"
assert decision["reason"] == "reset_expired"
```

Equality means expired.

In the surrounding implementation, commit the decision and queued work together before a worker performs an external submission, and recheck the relevant reset state before submitting. A local database transaction cannot atomically include a remote mail system. The reset endpoint must still enforce expiry and single use; secure token generation, token storage, concurrency control, and redemption are outside this focused decision function.

The troublesome boundary is a lost acknowledgment. A second local attempt identifier does not establish that the earlier request failed remotely. If the integration provides a documented idempotency mechanism, use it within its stated limits; otherwise the retry decision needs reconciliation and an explicit policy for the unresolved outcome. Preserve that uncertainty in the review record instead of overwriting it with the next attempt's result.

Incoming feedback should add attributed observations without rewriting the original decision. Define deduplication from authenticated source event identifiers where available, document handling where they are absent, and keep source time separate from arrival order. Confirm the integration's event authenticity checks and available diagnostics before relying on them. A delayed complaint can support a later suppression decision while leaving the earlier decision's historical inputs intact.

## Rejected shortcut and the acceptance review

The rejected option is declaring the case explained whenever the send API returns success. It has a valid use case: a narrow integration smoke test asking whether a caller can submit a request. For a reset investigation, it cannot explain a withheld attempt, establish validity when the shopper redeems the token, or account for later bounce and complaint reports.

Before release, use the example policy to exercise an attempt at exactly 300 seconds, an already consumed reset, a suppressed recipient, an uncertain submission outcome, and duplicated or delayed feedback. Give a second reviewer the retained records and applicable policy. Ask that reviewer to identify what authorized the attempt, what crossed the submission boundary, and which conclusions remain unsupported. If answering requires recovering the secret or consulting mutable current state to guess an old decision, the evidence design has failed its own acceptance criterion.

This is also a deployment check. Retain the interpretation version used during a rollout so that observations processed under different rules remain explainable. Operational reporting should distinguish withheld attempts, expired work, unresolved submissions, and reported bounces or complaints; record absent feedback as unavailable. That gives the on-call engineer a starting boundary for investigation and gives the reviewer a visible limitation rather than a misleading zero.

Accept the design when the retained evidence supports the claims it makes, at the time those claims apply. For a short-lived password reset, an explicit unknown is more defensible than a delivery assertion inferred from the wrong authority.

## References

- RFC 6376: DomainKeys Identified Mail (DKIM). https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208: Sender Policy Framework (SPF). https://datatracker.ietf.org/doc/html/rfc7208
- RFC 3463: Enhanced Mail System Status Codes. https://datatracker.ietf.org/doc/html/rfc3463
- OWASP Forgot Password Cheat Sheet. https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
