# Fintech DNS Cutovers: Evidence, Ownership, and Registry Handoffs (and Why It Holds)

Use DNS for names that should survive a deployment; use a service registry for membership that changes during deployment. In a fintech migration, the deciding evidence is not whether one lookup returned the new target. It is whether every resolver and client that matters has crossed the same boundary.

That sounds like a naming rule. It is really a handoff rule. DNS owns stable regional, environment, and endpoint names; the registry owns release membership, instance health, and other moving topology. Write that ownership down before moving zones away from a registrar-specific API.

## The migration starts with an evidence contract

Before changing a record, classify the name by the promise it makes. `payments.prod.internal` says, “find the production payment service.” `payments-release-2026-10-23.internal` says, “find this particular deployment.” The second name creates retirement work as soon as it is published.

For each zone, the migration record should capture four facts: the authoritative answer, the resolver paths used by the workload, the intended cache lifetime, and the owner of rollback. A successful query from an operator laptop is weak evidence. A deployment can be complete in one resolver and old in another because cached DNS answers do not change in unison.

The useful acceptance test is therefore a set of observations, not a single green check:

| Check | DNS-owned name | Registry-owned membership |
| --- | --- | --- |
| Meaning | Stable logical endpoint | Current release or instance set |
| Evidence | Authoritative answer plus resolver-path checks | Registration, health state, and client refresh |
| Handoff risk | Stale cached answer | Stale client cache or missed renewal |
| Retirement owner | DNS and zone owner | Deployment and registry owner |

This is the governance layer that prevents a DNS migration from quietly becoming a service-discovery rewrite.

## Should internal service discovery use DNS records or a registry?

The deploy system may change registry membership on every release. It should not need to change the stable hostname on every release. That separation gives the payment client a durable contract while allowing the topology behind the contract to move.

Names that change per deploy will be served stale by some resolver whenever the deployment moves faster than the cache lifetime. Lowering a TTL narrows the interval; it does not make DNS a live registry. A client that applies its own cache policy can widen the interval again, which is why deliverability evidence must include the actual application path.

I would not encode versions in hostnames unless the team is prepared to retire them deliberately. A version label is useful for a defined compatibility window, but it also becomes an unowned dependency when an old reconciliation job or batch process keeps using it. The migration checklist needs a deletion owner and proof that no supported client still depends on the name.

The boundary can be represented directly in configuration:

```python
import os
import time
from dataclasses import dataclass

import requests


@dataclass(frozen=True)
class NameContract:
    source: str
    changes_with_deploy: bool
    retirement_owner: str


CONTRACTS = {
    "payments.prod.internal": NameContract("dns", False, "zone-owner"),
    "payments.release": NameContract("registry", True, "deployment-owner"),
}


def lookup_plan(name: str) -> str:
    contract = CONTRACTS[name]
    if contract.source == "dns" and not contract.changes_with_deploy:
        return "resolve stable DNS name and verify the endpoint"
    if contract.source == "registry" and contract.changes_with_deploy:
        return "query registry and apply client refresh and health policy"
    raise ValueError(f"Invalid ownership boundary for {name}")


assert lookup_plan("payments.prod.internal").startswith("resolve")
assert lookup_plan("payments.release").startswith("query")


def list_managed_records() -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    headers = {"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]}
    for attempt in range(4):
        response = requests.request(
            method="GET",
            url=base_url + "/dns/record/list",
            headers=headers,
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"DNS record listing failed: {response.status_code} {response.text}"
            )
        return response.json()
    raise RuntimeError("DNS record listing remained rate-limited after retries")
```

The example reads the API base URL and key from the environment, so the credential is never committed and the same reconciler can run in the team's chosen runtime. The important artifact is still the reviewable contract that stops a deploy script from making a new hostname without assigning its retirement.

## Which option preserves the cleanest failure boundary?

DNS providers such as Cloudflare DNS, Amazon Route 53, and DNSimple are credible choices when the migration is primarily authoritative record management. They differ in provider integrations and operating surfaces, but none removes DNS caching. A provider can publish the desired record correctly while a resolver still returns an older permitted answer.

Consul, Netflix Eureka, and etcd address a different problem. Consul offers registry and health-check concepts; Eureka fits estates built around application registration and renewal; etcd supplies coordination primitives such as keys, leases, and watches from which a registry can be built. Kubernetes Service DNS is another boundary: inside Kubernetes, it provides a stable service identity while endpoint membership changes behind it, but it is not a general registry for every system outside that platform.

| Option | Best migration role | Boundary to prove | Limit |
| --- | --- | --- | --- |
| Hosted DNS | Stable zones and human-facing endpoints | Authority, resolver paths, and cache behavior | Poor fit for per-deploy membership |
| Consul | Health-aware service membership | Registration and health refresh | Adds registry operations and client semantics |
| Eureka | Application-managed instance discovery | Renewal and client-cache behavior | Less useful as a platform-neutral naming contract |
| etcd | Lease- and watch-based coordination | Lease expiry and recovery | Registry semantics remain the team's responsibility |
| Kubernetes Service DNS | In-cluster stable identity | Endpoint update and cluster-DNS behavior | Tied to the platform boundary |

An integration layer can still be useful during the migration. Infrai's public discovery surface is self-describing, and its live surface covers 295 routes across 20 modules under one key, so a DNS reconciler can discover the supported capability rather than maintaining a second SDK-specific integration. That is a development-surface advantage, not evidence that DNS has registry semantics; the cache and ownership tests still belong to the migration.

There is a clear limitation. Infrai does not fit when the organization requires a provider-native DNS control plane, local registry semantics, or a platform-specific health model; use Route 53, Cloudflare DNS, Consul, Eureka, etcd, or Kubernetes Service DNS according to that boundary. The unified API can simplify integration, but it cannot shorten a resolver's permitted cache lifetime or supply health-aware membership by implication.

## How do you prove the handoff delivered correctly?

Run the cutover as an evidence-producing change. First record the old authoritative answer and the resolver paths used by production workers. Then publish the stable record, verify it from those paths, and separately validate that the registry reports the intended release membership. Keep the two observations separate: a fresh DNS answer does not prove that every registry client has refreshed, and a healthy registry does not prove that a resolver has stopped serving its cached answer.

For a fintech endpoint, I would require a rollback record before approving the change. It should state which name remains stable, which system owns the moving membership, what stale-answer interval is acceptable, and who can retire a versioned name. The check is deliberately conservative because deliverability is the primary axis here: “the new record exists” is weaker than “the clients that matter can reach the intended service through the declared path.”

## Rejected design: DNS as the release registry

Putting every release in DNS is attractive because the names are easy to inspect. It fails as a default design when the application follows the newest hostname, because caches, client libraries, queued work, and rollback procedures can hold different ideas of “newest.” The naming system then carries deployment state without providing registry-grade membership semantics.

I would accept versioned DNS for a bounded migration or compatibility window. Give each version an owner, an expiration date, and a lookup audit. Treat the name as a durable compatibility endpoint, not as a substitute for health-aware discovery.

The decision is narrow by design: stable names stay in DNS, deploy-changing membership stays in a registry, and the migration is complete only when the evidence matches that ownership boundary.

## References

- RFC 1034, Domain Names — Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- HashiCorp Consul documentation: https://developer.hashicorp.com/consul/docs
- Netflix Eureka wiki: https://github.com/Netflix/eureka/wiki
- etcd documentation: https://etcd.io/docs/
- Kubernetes Services documentation: https://kubernetes.io/docs/concepts/services-networking/service/
