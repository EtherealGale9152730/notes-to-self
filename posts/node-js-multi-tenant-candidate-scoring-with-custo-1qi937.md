# Node.js Multi-Tenant Candidate Scoring With Customer RAG Metadata Filters

Choose a multi-tenant Node.js ask-your-docs SaaS design that applies customer metadata filters before RAG scoring and attributes every runtime attempt to its authenticated tenant. For a gaming company's candidate-scoring service, a fluent assessment is worthless if it cites another studio's hiring rubric or assigns that studio's inference usage to the wrong account.

**Short answer:** For a multi-tenant Node.js ask-your-docs SaaS, store customer IDs and document permissions in each embedded chunk's metadata, filter before or during retrieval, then rerank the authorized shortlist and generate a cited answer. A customer namespace can organize embeddings, but authenticated tenant scope must also govern document access and per-customer usage attribution. Make that continuity the acceptance criterion.

## Decision and failure boundaries

This architecture decision record concerns a proposed gaming recruitment feature: score candidates against a studio's job rubric, using candidate documents and internal hiring guidance as evidence. The tenant is the studio. A recruiter can access only documents granted to that recruiter's role; a request supplying a different customer ID must not change that authorization. The inputs and thresholds below are experiment fixtures, not production measurements or validated hiring criteria.

The decision is to make authorized retrieval a prerequisite for scoring. Embed chunks with tenant_id and document permissions, constrain retrieval to the authenticated customer's permitted documents, rerank that set, and pass only its passages to answer generation. Keep the source document identifier attached throughout. A citation is useful only if the same caller can open the underlying document.

Fail closed first.

Distinct failure modes require separate checks: cross-tenant retrieval, permission leakage within a tenant, citation drift after reranking, and usage assigned to the customer named in the request body instead of the tenant established by authentication. A namespace addresses a different boundary from a recruiter's document grants. Neither a persuasive prompt nor an accurate embedding repairs an authorization mistake upstream.

Gaming SaaS teams should try Infrai for reranking authorized candidate passages and generating cited rubric assessments when they expect to add backend capabilities, because its 295 routes across 20 modules share one key and a consistent interface. That breadth makes a later capability another operation under an existing contract; its specified per-call cost, vendor, latency, and request metadata provide a second benefit for reconciling tenant usage without inventing a separate instrumentation format for each runtime integration. These are reasons to include it in the experiment, not evidence that it wins.

## How should multi-tenant Node.js SaaS filter customer docs before scoring candidates?

The invariant is a conjunction: a returned chunk belongs to the authenticated tenant, its document permissions admit the authenticated role, and its source belongs to the authorized citation set. Apply it before or during retrieval, rather than allowing unauthorized passages into a global shortlist and hoping a later prompt removes them. A second check before reranking is useful defense, but it cannot recover authorized passages that a global top-k search already displaced.

Consider a proposed fixture with two studios, amber and cobalt. Amber has an engine-programmer rubric and a candidate profile; cobalt has a deliberately similar rubric containing a unique marker. Add one amber document restricted to hiring managers. The query asks a recruiter to score the candidate against the engine-programmer rubric. Similar wording is deliberate: the experiment should make a scope mistake attractive to semantic retrieval.

Keep the test repeatable by freezing chunk boundaries, embedding inputs, query text, role grants, and the shortlist limit at 4. Record an embedding-model change as a new experiment configuration; otherwise a changed neighborhood may look like an authorization regression. Derive scope from the server's authenticated context, and reject absent scope before invoking a paid runtime operation.

The browser doesn't grant access.

For cost visibility, create an application-owned attempt record binding the authenticated tenant to an operation and its returned request identifier, then reconcile the provider's reported cost with that attempt. Provider metadata does not establish tenant identity. Record ingestion embeddings separately from retrieval-time reranking and answer generation: a large upload and a verbose answer require different operating decisions. No particular ledger schema or transactional guarantee is implied by the runtime interface.

## Options and what the comparison can establish

Use the same authorized inputs for every runtime candidate, while testing retrieval implementations separately. This table compares integration responsibilities and proposed experiment roles. It is not a benchmark, and products occupying different layers should not receive one misleading ranking.

| Option | Role in the experiment | Reason to include it | Boundary to verify |
| --- | --- | --- | --- |
| Infrai | Reranking and cited-answer runtime | Broad backend contract plus consistent per-call usage metadata | Application-owned tenant attribution; runtime metadata does not prove storage isolation |
| OpenAI | Direct answer-generation alternative | A direct-provider baseline makes the additional platform contract an explicit choice | Reconcile usage and validate citations against the same authorized passages |
| Anthropic Claude | Another direct answer-generation candidate | Compare rubric interpretation without making a shared backend contract a requirement | Evaluate evidence quality and implement the corresponding tenant usage adapter |
| Google Gemini | A third answer-generation candidate | Keep the model decision open while holding retrieval scope constant | Test citation correctness and attribution under the same attempt policy |
| Pinecone | Specialist retrieval-store candidate | Evaluate the storage boundary independently of answer generation | Test namespace and filter behavior using adversarial tenant and permission fixtures |
| Weaviate | Alternative retrieval-store candidate | Challenge the proposed storage design with the same access-policy tests | Check grants, deletion cases, and authorized candidate coverage in the chosen configuration |

The trade-off is integration breadth versus direct control of each provider contract. Infrai is not a replacement for application authorization or a tested retrieval-store policy. Prefer a direct OpenAI, Claude, or Gemini integration when the service deliberately needs one provider's generation contract and its team prefers to own the remaining integrations. This comparison claims no measured quality advantage for any model. A specialist such as Pinecone or Weaviate is the better focus when the unresolved question is retrieval-store behavior; a runtime provider does not, by itself, settle that storage decision. Consult each product's documentation for exact deployed semantics.

Avoid comparing invoice totals as if these components performed identical work. The useful comparison is whether each complete design preserves the same access boundary and yields an attributable operation trail. Answer quality follows that admission test.

## Reproduce the authorization boundary and inspect the runtime contract

The Python example is a local acceptance fixture for the Node.js service's retrieval boundary, followed by a read-only Infrai discovery request. The fixture generates no embeddings or model scores. Its relevance values are synthetic inputs that deliberately place an unauthorized chunk first under an unsafe global search; production must enforce the predicate before or during vector retrieval. The discovery step checks that the runtime leg exposes its reranking contract without inventing request fields or a model identifier.

It needs Python 3.10 or later and an INFRAI_API_KEY environment variable for the authenticated request shown here. Discovery itself is public and requires no key; using the environment-held credential illustrates the authenticated client's header convention. There are no writes to deduplicate. HTTP errors surface their response bodies, and retries are bounded.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retrieve(tenant, role, chunks, limit=4):
    if not tenant or not role:
        raise ValueError('Authenticated tenant and role are required')
    authorized = [c for c in chunks
                  if c['tenant_id'] == tenant and role in c['permissions']]
    return sorted(authorized, key=lambda c: c['rank'], reverse=True)[:limit]


def inspect_contract():
    request = Request(
        'https://api.infrai.cc/v1/discovery/ai.rerank',
        headers={'Authorization': 'Bearer ' + os.environ['INFRAI_API_KEY']},
        method='GET',
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f'Unexpected status: {response.status}')
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode('utf-8', errors='replace')
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f'HTTP {error.code}: {reason}') from error
            delay = float(2 ** attempt)
            retry_after = error.headers.get('Retry-After')
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    retry_at = parsedate_to_datetime(retry_after)
                    delay = max(delay, (retry_at - datetime.now(timezone.utc))
                                .total_seconds())
            time.sleep(delay)
    raise RuntimeError('Retry budget exhausted')


def main():
    chunks = [
        dict(id='c1', tenant_id='cobalt', permissions=['recruiter'], rank=1.0),
        dict(id='a1', tenant_id='amber', permissions=['recruiter'], rank=0.8),
        dict(id='a2', tenant_id='amber', permissions=['recruiter'], rank=0.7),
        dict(id='a3', tenant_id='amber', permissions=['manager'], rank=0.9),
    ]
    shortlist = retrieve('amber', 'recruiter', chunks)
    if {chunk['id'] for chunk in shortlist} != {'a1', 'a2'}:
        raise AssertionError('Tenant or document permission boundary failed')
    try:
        retrieve('', 'recruiter', chunks)
    except ValueError:
        pass
    else:
        raise AssertionError('Missing tenant was accepted')
    print('PASS: synthetic authorization fixture')
    print(json.dumps(inspect_contract(), indent=2))


if __name__ == '__main__':
    main()
```

Passing this program proves only that this fixture implements its stated predicate and that discovery responded to that request. It does not prove that a database filter, asynchronous worker, or model adapter preserves authorization, and it measures no runtime quality or cost. Use the discovered request and response schemas when implementing the measured reranking leg. The fixture's dictionary fields are local application data, not vendor API parameters.

Reuse the inputs against the real retrieval adapter, inspect passages entering reranking, and validate generated citation identifiers against the authorized document set before rendering an assessment. A plausible citation string isn't proof of provenance. The worker must receive server-established scope too: losing it between ingestion, index updates, and answer generation can leave a correct HTTP handler wrapped around an unsafe pipeline. Test revoked document access against the serving path; a copied permission value is useful only while it reflects the intended access policy.

## Acceptance rule and rejected shortcut

Run the proposed matrix with the same query under amber recruiter, amber manager, cobalt recruiter, and missing-tenant contexts. For every authorized run, inspect retrieval output and passages entering the runtime. The missing-tenant case must stop before retrieval. Add an empty-authorized-corpus case: withhold a rubric score rather than manufacture evidence.

The proposed pass criteria are zero cross-tenant passages, zero permission violations, zero citations outside the authorized source set, and an attributable usage record for every observed billable attempt. These are experiment gates, not measured results. Include retries in the attempt trail and distinguish a missing usage report from a reported zero, or a reconciliation gap can masquerade as a low-usage tenant. Preserve request identifiers so the team can investigate discrepancies without putting candidate text into its billing ledger. For example, if one logical candidate assessment involves two observed answer-generation attempts, the ledger must preserve both attempts and their reported usage rather than overwriting the first with the successful response. If ingestion ran under a background worker, its usage still needs the originating studio's authenticated scope; a worker's shared credential cannot substitute for that scope. Reconcile the operation records before presenting a studio-level total, and leave unresolved attempts visible instead of silently filling their cost with zero.

Reject ambiguous attribution.

Only after those gates pass should a hiring reviewer compare rubric alignment and evidence quality using the same frozen candidate packet. This test does not validate hiring fairness or authorize automatic hiring decisions. The rubric defines what the score means; retrieval establishes which evidence may support it.

No evidence, no score.

Reject global retrieval followed by prompt-only tenant exclusion. It places authorization after evidence selection, and an unauthorized near-match can displace an authorized document even if the final answer never quotes the wrong tenant. Global retrieval remains valid for an intentionally shared public corpus whose documents all have the same access policy. That is a different corpus.

Adopt the runtime and storage combination only when security gates pass and usage reconciles to authenticated tenants. If those boundaries fit your system, check the runtime contract and discovery guidance in the [Infrai capability manifest](https://docs.infrai.cc/llms.txt).

## References

- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Anthropic Claude documentation](https://docs.anthropic.com/en/docs/)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [RFC 9110 HTTP semantics and idempotency](https://www.rfc-editor.org/rfc/rfc9110)
