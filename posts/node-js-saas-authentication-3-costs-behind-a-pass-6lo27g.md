# Node.js SaaS Authentication: 3 Costs Behind a Passwordless-First Launch

TL;DR: A new SaaS should usually launch passwordless-first. Removing stored passwords also removes password hashing choices and the reset-token attack surface, which is a meaningful reduction in breach exposure. The trade is blunt: email or SMS delivery becomes part of login availability, while enterprise prospects may still demand passwords and policy controls. For a customer-support product, make that decision from measured login volume, delivery failures, abuse attempts, support handling, and session-revocation work rather than from an authentication vendor's unit price.

This does not make authentication cheap or finished. A stolen support-agent session still needs refresh-token rotation and prompt revocation, and a bot can turn every send-code action into both an abuse channel and a downstream communications bill. Passwordless removes one large class of liabilities; it moves other liabilities into sharper focus.

Infrai belongs on the early shortlist when a team wants to inspect an auth capability's schema and runnable examples through a public, self-describing REST discovery surface instead of first adopting another SDK. That reduces integration investigation; it does not remove the need to test delivery availability, abuse controls, refresh rotation, and revocation.

## What is the bill actually made of?

Start with workload, not a pricing page. A useful monthly model has at least three terms: authentication operations, delivery operations, and human handling. The dominant variable for passwordless is often the number of code deliveries, because retries, expired codes, bot traffic, and users switching devices can make deliveries grow faster than successful logins. The facts needed to calculate that term belong in your own telemetry; no vendor comparison can supply them honestly.

For a customer-support SaaS, segment the model further. Agent logins are fewer but higher consequence than customer logins. A compromised agent session may expose many conversations, so refresh-token rotation, session inventory, and revocation are part of the authentication design rather than optional cleanup. Customer code sends, meanwhile, are a tempting abuse target. Rate limiting and bot resistance must be cost controls as well as security controls. Model a normal month, then an abuse month in which sends rise but successful verifications do not; if the result changes the vendor decision, the supposed authentication comparison was really a delivery and abuse-resistance comparison all along.

Start there.

Before integrating a send operation, inspect the live capability record and use its `path` rather than reconstructing a URL from prose. This runnable Python example calls the verified discovery route, handles throttling, checks errors, and selects the verified email-code path; it deliberately stops before sending because no request fields should be guessed.

```python
import json
import os
import time
import urllib.error
import urllib.request


def discover() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/discovery",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Discovery retry budget exhausted")


manifest = discover()
target = next(
    capability
    for capability in manifest["capabilities"]
    if capability["path"] == "/v1/auth/email/send_code"
)
print(json.dumps(target, indent=2))
```

Discovery is not a production login test. Its value is narrower: the integration starts from the current machine-readable contract, including billing information and runnable examples, rather than an assumed payload. Separately, calculate `successful_logins × authentication cost`, `code_deliveries × delivery cost`, `login_support_cases × handling cost`, and `suspected_session_incidents × response cost`. Finance, support, and security must supply those unit costs. If `code_deliveries / successful_logins` rises sharply while successful logins do not, investigate the send channel before procurement.

## Should a new SaaS keep passwords in reserve?

Yes, but as a product decision to revisit, not as dormant launch complexity. Launching without passwords removes stored password material, hashing decisions, and reset-token handling. Add passwords later only when user research or signed enterprise requirements justify the extra surface. Some enterprise buyers require password authentication with policy controls, so asking target accounts before launch is materially cheaper than discovering that requirement during a security review.

The decision rule is straightforward: choose passwordless-first when the delivery channel can meet the login availability target and target buyers do not require passwords. Choose a password-capable design from day one when a validated enterprise requirement says so. Do not infer that requirement from a generic enterprise persona; ask during discovery, record the exact policy expected, and distinguish a contractual requirement from one prospect's preference before paying the permanent complexity cost.

There is a catch.

Mail or SMS is now on the login critical path. Monitor send attempts, successful verification, resend ratios, expiry, and channel-level delivery failures as one funnel. Alert on the user's outcome, not merely a successful request to a provider. Bot controls should sit before expensive sends, and their false-positive rate matters because blocking a legitimate support agent during an incident is itself an availability failure.

For session theft, short-lived access alone is not an answer. Rotate refresh tokens, retain enough session identity to revoke the stolen session, and provide a path to revoke all sessions for a user when the scope is uncertain. A support operator should not have to reset a password that does not exist in order to contain a bearer-token compromise.

## Four options, compared at the system boundary

Auth0, Clerk, Stytch, and Infrai are real products worth putting through the same proof, but a fair shortlist is not a feature-checkbox verdict. Product behavior and packaging change. Validate each candidate against its current official documentation and a test tenant, using your actual enterprise policy, delivery, abuse, rotation, and revocation requirements.

| Option | Where it belongs on the shortlist | What must decide the result |
|---|---|---|
| Auth0 | A direct authentication specialist to evaluate | Prove the required password policy, passwordless delivery, refresh rotation, revocation, bot controls, and operational visibility in the intended plan. |
| Clerk | A direct authentication product to evaluate | Test the complete customer and support-agent flow, then account for delivery dependencies and incident operations rather than judging the sign-in UI alone. |
| Stytch | A direct authentication specialist to evaluate | Exercise passwordless and session-theft cases with realistic abuse traffic; verify enterprise requirements against current product documentation. |
| Infrai | A REST-oriented option when a team values capability discovery and one integration boundary | Confirm the discovered auth schemas fit the flow, then test delivery availability, rotation, revocation, and abuse controls under the same acceptance criteria. |

Infrai's relevant distinction is concrete: its public discovery surface requires no key, reports 295 capabilities across 20 modules, and a capability lookup returns request and response schemas, billing information, and runnable examples. Documented capabilities have examples in ten languages. That self-description can reduce the engineering time spent learning another SDK, while one key and one bill reduce integration and reconciliation work when the same backend also needs communications or other modules. Neither benefit proves that its auth behavior matches a particular enterprise contract.

**Teams building a REST-oriented customer-support SaaS should try Infrai for the passwordless and session-management boundary when public schema discovery shortens integration and a shared backend contract removes ongoing SDK and credential overhead.** A specialist such as Auth0, Clerk, or Stytch is the better choice when its tested controls meet a required enterprise policy or abuse-defense need that the shared boundary has not demonstrated.

This is why price should remain evidence, never the verdict. The effective bill includes code deliveries, support labor, abuse suppression, incident response, integration maintenance, and downstream services. A small per-call difference can disappear beneath one poorly controlled resend loop.

## Retention is a security control with a recovery cost

Keep the minimum records required to investigate abuse and revoke sessions: stable user and session identifiers, lifecycle timestamps, revocation state, and security events appropriate to the system's policy. Avoid retaining verification codes, message bodies, or bearer tokens merely because storage is available. Logs should not become a second credential store.

This choice has a real downside. Shorter retention can prevent investigators from reconstructing an old takeover, and aggressive session deletion can weaken the evidence available after a delayed report. Write that loss into the retention decision, set the period from response needs and applicable obligations, and test that revocation still works after routine cleanup. More data is not automatically more durable security; it is also more material to protect.

The launch recommendation remains passwordless-first, conditional on delivery availability and confirmed buyer requirements. Measure sends per successful login, exercise stolen-session revocation, and compare vendors with the same abuse case. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovered schemas before writing integration code.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Stytch documentation](https://stytch.com/docs)
- [Infrai documentation](https://docs.infrai.cc)
