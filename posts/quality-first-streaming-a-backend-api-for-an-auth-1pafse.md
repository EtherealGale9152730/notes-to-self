# Quality-First Streaming: A Backend API for an Authenticated Web App Chatbot

In a game knowledge chatbot, the operational constraint is not whether the backend can stream tokens. Most HTTP stacks can do that. The difficult choice is deciding how much retrieval and validation the request can afford before a player stops waiting, while ensuring that an authenticated player never sees an answer assembled from the wrong private documents. Short answer: put retrieval, authorization, and model selection behind the application backend, then stream only the answer that has passed the checks your quality target requires.

That sounds less exciting than connecting a browser directly to an AI endpoint. Good. A private knowledge base needs a trust boundary, and a streaming connection needs an explicit answer state. If either is vague, latency numbers become decoration around a data leak or a transcript that cannot be reconciled.

This is an architecture decision record for an in-app chatbot that answers questions about game mechanics, live-service rules, and internal support material. The axis is quality versus latency. The choice is a bounded retrieval pipeline with a normal authenticated request and a browser-facing event stream, measured against a deliberately small set of quality checks.

## Start with the questions the game knowledge base must answer

Before comparing an API, build a small evaluation set from the questions players and support staff actually ask. Include exact rule lookups, questions whose answer changed after a patch, account-specific questions, ambiguous wording, and questions for which the private corpus has no supported answer. A latency chart without these cases rewards a system for answering quickly even when it should have declined.

For each example, record the expected evidence, the acceptable answer, and the conditions that make an answer invalid. This is not a demand for a perfect benchmark. It is a way to catch a very ordinary regression: a new index returns a general gameplay guide above a current entitlement policy because the guide shares more words with the player's question.

Wrong corpus, wrong answer.

I would compare retrieval recall, grounded-answer judgments, unsupported-answer rate, and time to first useful token. “Useful” matters: an empty heartbeat or a generic preamble is not a player-visible answer. A single aggregate score can hide a bad result for one game mode, so split the set by patch version, question type, and authorization scope.

I would write these invariants before choosing an API shape:

1. The browser presents the application session; it never receives a provider credential or a broad knowledge-base credential.
2. Every retrieved chunk is authorized for the player, the game, and the current tenant before it reaches the prompt.
3. A streamed answer is not considered complete until the server receives the terminal upstream event and persists the same turn identity exactly once.
4. A citation or source identifier belongs to the retrieved document actually used for that answer, not to a search result that was discarded later.

The third invariant is where many “fast” implementations become expensive. A player can close a tab after seeing half a sentence. A proxy can time out while the model is still producing text. A retry can then create a second assistant message unless `(conversation_id, turn_id)` is unique in the application datastore. The UI may show a partial answer, while the database records success; those are different states and should remain different states.

Three words matter: authorized before generation.

For this scenario, quality is also more than a plausible sentence. A useful answer should retrieve the current rule, preserve important qualifiers such as “only during ranked play,” and decline when the private corpus does not support the claim. Latency is measured from the user action to the first useful event, not merely to the first empty heartbeat. Streaming can improve perceived responsiveness, but it cannot repair irrelevant retrieval.

## Make latency a budget with explicit quality gates

There is no universal quality setting. A question such as “How does armor interact with frost damage?” may need one tightly scoped rules document. “Why was my tournament reward removed?” may require an entitlement policy, an event configuration, and a player-specific record. Treating both requests as the same retrieval job creates either needless waiting or confident incompleteness.

Set a separate budget for authentication, retrieval, authorization filtering, prompt construction, and generation. When one stage consumes its budget, the system should return a known state: narrow the search, ask a clarifying question, or say that the corpus does not support the answer. It should not manufacture certainty to preserve a first-token target.

| Architecture choice | Quality behavior | Latency behavior | Failure boundary | Appropriate use |
| --- | --- | --- | --- | --- |
| Retrieve once, then stream | Fastest path when search is precise | First useful event waits for one search and prompt build | Wrong or stale chunk poisons the answer | Stable, narrow rulebooks |
| Retrieve, rerank, then stream | Better evidence ordering and fewer distractors | Extra work before generation | Reranker can favor fluent but less authoritative text | Conflicting or large corpora |
| Stream a draft, verify after | Very responsive appearance | Verification arrives after the claim | A wrong draft can already influence the player | Low-risk exploratory help |
| Retrieve in stages | Can combine public rules, private policy, and player state | Slowest cold path | Missing one stage must be visible to the answer | Entitlements and sensitive support cases |

For a private gaming corpus, I would use the second choice as the default and the fourth choice for account-specific questions. The distinction is not a product preference; it follows from the evidence needed. A compact rule lookup should not wait for a large fan-wiki search, while a refund or reward answer should not be generated from generic gameplay text.

The catch is that reranking and staged retrieval are not free. If the product promise is sub-second first useful output, a deep pipeline may be unsuitable when the corpus is remote, the authorization check is per-document, or the model must see a long context. In that case, keep the fast path narrow, return a visible “needs more evidence” state, and route the question to a slower review flow. Do not silently trade evidence for speed.

## A small critical path in Python

The endpoint below is an application contract, not a vendor-specific SDK. It keeps identity and storage decisions local, and it treats the upstream stream as an input that must be parsed and persisted. The `retrieve_authorized` and `upstream_stream` functions are boundaries to the search and model services selected by the team.

```python
import json
import time
from typing import Callable, Iterable


def answer_question(
    session: dict,
    conversation_id: str,
    turn_id: str,
    question: str,
    emit: Callable[[dict], None],
) -> None:
    if not session.get("user_id"):
        raise PermissionError("authenticated session required")

    evidence = retrieve_authorized(
        user_id=session["user_id"],
        game_id=session["game_id"],
        query=question,
        limit=8,
    )
    if not evidence:
        persist_turn(conversation_id, turn_id, status="unsupported", text="")
        emit({"type": "final", "status": "unsupported"})
        return

    prompt = build_prompt(question=question, evidence=evidence)
    pieces = []
    for event in upstream_stream(prompt, timeout_seconds=45):
        if event["type"] == "delta":
            pieces.append(event["text"])
            emit({"type": "delta", "text": event["text"]})
        elif event["type"] == "done":
            text = "".join(pieces)
            persist_turn(
                conversation_id,
                turn_id,
                status="complete",
                text=text,
                sources=[item["source_id"] for item in evidence],
            )
            emit({"type": "final", "status": "complete"})
            return

    persist_turn(conversation_id, turn_id, status="interrupted", text="".join(pieces))
    emit({"type": "final", "status": "interrupted"})
```

There are two intentional omissions. First, the datastore operation must enforce uniqueness on the turn identity; a comment saying “make it idempotent” is not enforcement. Second, the event transport can be Server-Sent Events or another application-supported stream, but its reconnect behavior must be tested independently from the upstream model stream. A browser reconnect is not permission to regenerate an answer.

The request should also carry a bounded deadline. Retrying a whole generation after a client disconnect can increase load and duplicate side effects. Retry retrieval when the search layer has a documented transient policy; retry generation only when the turn contract makes the operation safe. A `429` or timeout is an operational signal, not a reason to tell the player that a rule exists.

## After the first token, inspect the evidence and the state

The first failure mode is stale authority. A vector index can return an older patch note because its text is semantically close to the question. Store an effective date and authority class with each chunk, filter by game version where possible, and test questions whose answer changed between patches. Embeddings are useful for semantic retrieval, but they do not decide which document is authoritative; the retrieval guide itself should not become a substitute for version policy.

Consider a player asking why a tournament reward disappeared. The search may find an event FAQ, a current entitlement policy, and the player's support record. The answer is only good if the authorization layer permits that support record, the event document matches the relevant season, and the generated explanation keeps the exception that applies to that player's account. Sending the first semantically similar paragraph to the model is fast, but it turns a three-document reasoning problem into a polished guess. A quality gate can require the policy and event evidence to agree before streaming; if they do not, the response should explain that the case needs review. That pause is an intentional product state, not a transport failure, and it is cheaper than teaching players to trust an answer that contradicts their account history.

The second is tenant bleed. A result can be relevant and still forbidden. Authorization belongs in retrieval, not in a late prompt instruction. Log the document IDs selected for a turn, the authorization decision, and the corpus version, but avoid logging private player text unless the retention policy explicitly allows it.

The third is a stream that looks healthy while the answer is poor. Count time to first useful token, time to final event, retrieval hit rate, unsupported-answer rate, citation coverage, and disconnects by route. A green connection metric cannot tell you that the model answered a seasonal rule from last month.

I would add a small evaluation set to every deployment: canonical questions, versioned answers, deliberately ambiguous questions, and questions with no supported answer. Compare retrieval recall and grounded-answer judgments before changing the latency budget. I'm not sure a single aggregate score can represent a live-service game corpus; your mileage may vary with how often rules and entitlements change, so keep those slices visible rather than hiding them in one average.

## Which backend API shape should a web app chatbot use?

The browser should call an application route that already knows the session and game context. The route can expose a narrow streaming contract, while the server chooses the search and generation implementation behind it. This keeps the browser protocol stable when the corpus, reranker, or model changes.

| Boundary | Required decision | What to measure |
| --- | --- | --- |
| Browser to application | Session identity and reconnect behavior | Unauthorized reads, duplicate turns, disconnect rate |
| Application to retrieval | Filters for tenant, game, version, and authority | Recall, stale-document rate, filter latency |
| Application to generation | Evidence format and bounded deadline | Grounded-answer rate, first useful token, completion time |
| Application to storage | Unique turn identity and final state | Partial-turn recovery, replay safety, source coverage |

The interface is intentionally small. A backend API that hides these boundaries behind a large client SDK may feel productive on day one, but it can make authorization and state transitions harder to inspect. A plain HTTP contract is a reasonable implementation choice when the team needs language independence; it is not a quality guarantee by itself.

## Rejected option, and when it is still valid

I would reject direct browser-to-model access for this application. It makes the credential boundary harder to control, gives the model service too much authority over tenant selection, and leaves transcript durability coupled to the player's connection. The browser should speak to an application route that already knows the session and can enforce the private corpus policy.

That rejected shape can be valid for a disposable, non-private prototype using synthetic documents and no durable player data. It is not suitable when answers depend on entitlements, internal support material, moderation policy, or patch-specific authority. For those cases, keep the application backend in the middle even if the initial version has only one retrieval index and one upstream model.

The practical rule is narrow: spend latency on evidence when an incorrect answer changes player action or support liability; spend it on streaming when the answer is already well bounded and the user mainly benefits from earlier rendering. Measure both at the same time. A fast wrong answer is still a storage and trust problem.

## References

- OpenAI Embeddings guide: https://platform.openai.com/docs/guides/embeddings
- Prompt Engineering Guide: https://www.promptingguide.ai
- WHATWG HTML Living Standard, Server-Sent Events: https://html.spec.whatwg.org/multipage/server-sent-events.html
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
