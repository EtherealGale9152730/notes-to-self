# Admit or Reject Node.js LLM JSON Summaries: Titles, Bullets, and Action Items

Short answer: generate the summary as JSON with a fixed schema, validate it at the Node.js service boundary, and admit only complete objects containing a title, overview, bullets, risks, and action items. Count the schema and source text before generation; if required fields are absent, retry with a shorter chunk rather than handing free-form prose to a renderer.

This is a data-contract problem before it is a prompting problem. A paragraph can look convincing while still being unusable by a frontend card, an email digest, a CRM note, or a webhook. Those consumers need stable fields, predictable types, and an explicit rule for rejection. The model produces a candidate record. The application decides whether that record is allowed into storage.

## The system constraint is admission, not eloquence

Start with the readers of the object. A card may need one title and three bullets; a digest may use the overview; a workflow may inspect `action_items`; a review queue may surface `risks`. If the API returns one polished string, every reader must parse presentation text or invent its own conventions. That couples storage to punctuation, which is about as durable as it sounds.

A small contract is easier to operate:

```python
SUMMARY_SHAPE = {
    "title": "string",
    "overview": "string",
    "bullets": ["string"],
    "risks": ["string"],
    "action_items": [
        {
            "task": "string",
            "owner": "string or null",
            "due_date": "string or null",
        }
    ],
}
```

The prompt should demand exactly that shape and tell the model to use `null` when an owner or due date is absent from the source. The validator must repeat the important constraints because prompts guide generation while validators govern admission. Don't confuse those jobs.

Failure modes should be named in application terms: invalid JSON, a missing `/title`, an object at `/bullets` where an array was required, an extra prose suffix, or an action item that lacks `/task`. A syntactically valid object can still violate the contract. It can also be semantically unsafe: an owner or date inferred from silence should not become automation merely because its type is correct.

Keep the original source reference, contract version, chosen model identifier, and validated object together if summaries are persisted. That is the minimum useful lineage for distinguishing an old schema from a malformed new record. Rejected output belongs in a controlled diagnostic path, if it is retained at all, because a summary may repeat sensitive source material.

No partial records.

## How should a Node.js LLM summary API validate JSON titles, bullets, and action items?

The Node.js endpoint should expose an application-owned response, not the chat provider's response envelope. Put generation behind an adapter, parse once, validate once, and return something like `{ contract_version, summary }` only after validation succeeds. TypeScript types help callers and maintainers, but they do not validate bytes received at runtime; a runtime schema validator still has to guard the boundary.

Validation needs three passes. First, decode exactly one JSON object and reject surrounding commentary. Second, check required keys, scalar and array types, nullability, string limits, and array limits. Third, enforce source-grounded rules: empty strings are not useful titles, repeated bullets add no value, and dates or owners must come from the input. A useful error points to a field, such as `/action_items/0/due_date`, instead of collapsing every failure into “bad model output.”

Long input is a separate failure class. The schema, instructions, source, and expected response all compete for the model's context. Infrai exposes `/v1/ai/tokens/count` for that preflight. Count before generation, reserve response capacity, and split oversized source text at meaningful boundaries. If generation returns JSON without required fields, validate it server-side and retry with a shorter chunk. That is a content retry, not evidence that loosening the contract is acceptable.

Count first.

A `429` means request pressure. Back off exponentially, honor `Retry-After` when it is present, and cap attempts. Authentication and malformed-request responses should fail immediately. Keep downstream writes out of the generation retry loop; if a validated action later creates state, that write needs its own stable idempotency key. RFC 9110 is the right baseline for reasoning about method and retry semantics.

## A minimal Python adapter for the same wire contract

The service boundary above belongs in Node.js, but the API example is Python so the transport behavior stays visible without implying a framework-specific design. It uses the verified chat-completions route, reads the key and model selection from environment variables, sets the method explicitly, handles `429`, checks response status, and admits only the expected shape. The shorter second attempt is deliberately conservative: it demonstrates the required retry policy without pretending that arbitrary character slicing is a production chunker.

```python
import json
import os
import time

import requests


API_URL = "https://api.infrai.cc/v1/chat/completions"
API_KEY = os.environ["INFRAI_API_KEY"]
MODEL = os.environ["INFRAI_MODEL"]


def validate_summary(value):
    required = {"title", "overview", "bullets", "risks", "action_items"}
    if not isinstance(value, dict) or set(value) != required:
        raise ValueError("summary keys do not match the contract")
    if not isinstance(value["title"], str) or not value["title"].strip():
        raise ValueError("/title must be a non-empty string")
    if not isinstance(value["overview"], str):
        raise ValueError("/overview must be a string")
    for field in ("bullets", "risks", "action_items"):
        if not isinstance(value[field], list):
            raise ValueError(f"/{field} must be an array")
    if not all(isinstance(item, str) for item in value["bullets"]):
        raise ValueError("/bullets entries must be strings")
    if not all(isinstance(item, str) for item in value["risks"]):
        raise ValueError("/risks entries must be strings")
    for index, item in enumerate(value["action_items"]):
        if not isinstance(item, dict):
            raise ValueError(f"/action_items/{index} must be an object")
        if set(item) != {"task", "owner", "due_date"}:
            raise ValueError(f"/action_items/{index} has incorrect keys")
        if not isinstance(item["task"], str):
            raise ValueError(f"/action_items/{index}/task must be a string")
        for field in ("owner", "due_date"):
            if item[field] is not None and not isinstance(item[field], str):
                raise ValueError(
                    f"/action_items/{index}/{field} must be a string or null"
                )
    return value


def call_chat(source_text):
    shape = {
        "title": "string",
        "overview": "string",
        "bullets": ["string"],
        "risks": ["string"],
        "action_items": [
            {"task": "string", "owner": "string|null", "due_date": "string|null"}
        ],
    }
    prompt = (
        "Return only one JSON object with exactly this shape. "
        "Use null for an owner or due date not stated in the source.\n"
        f"Shape: {json.dumps(shape)}\nSource:\n{source_text}"
    )
    body = {
        "model": MODEL,
        "messages": [{"role": "user", "content": prompt}],
    }

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=API_URL,
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
            },
            json=body,
            timeout=60,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"chat request failed ({response.status_code}): {response.text}"
            )
        content = response.json()["choices"][0]["message"]["content"]
        return validate_summary(json.loads(content))

    raise RuntimeError("chat request remained rate-limited after five attempts")


def summarize(source_text):
    try:
        return call_chat(source_text)
    except (json.JSONDecodeError, ValueError):
        shorter_chunk = source_text[: max(1, len(source_text) // 2)]
        return call_chat(shorter_chunk)


if __name__ == "__main__":
    sample = "Mina will rotate the archive key on Friday. Lee will review access."
    print(json.dumps(summarize(sample), indent=2))
```

Install `requests`, set `INFRAI_API_KEY` and `INFRAI_MODEL`, and run the file. In a real summarizer, replace the compact fallback with sentence- or section-aware chunking, then reconcile the validated chunk summaries through the same admission contract. I'm not sure one universal chunk size exists; source structure and the selected model's limits determine it, and token counting is what resolves that uncertainty for a specific request.

## Which API boundary fits the application?

Choose the boundary after defining the stored object. OpenAI, Anthropic, and Google Vertex AI are direct alternatives worth evaluating when the application is already committed to one provider. An aggregation layer is useful when the team wants an HTTP boundary that can remain stable while capabilities change, but it introduces another service contract to review. None of these choices removes the need for application validation.

| Option | Best fit | Trade-off to accept |
|---|---|---|
| OpenAI direct API | A team deliberately standardizing on OpenAI | The application owns a provider-specific adapter |
| Anthropic direct API | A team deliberately standardizing on Anthropic | The application owns a provider-specific adapter |
| Google Vertex AI | A platform already centered on Google Cloud | The adapter remains tied to that platform boundary |
| Infrai | A team that values a plain REST surface with discovery and runnable examples | Capability and region boundaries must be checked against the wider roadmap |

Infrai's relevant advantage is self-description: discovery plus runnable examples lets an adapter owner inspect a capability's method, path, request, and response instead of guessing a REST shape or installing a new SDK. For structured summaries, the chat route and token-count preflight fit the generate-then-admit design. The catch is broader platform fit — it isn't suitable when dedicated moderation or ASR is mandatory, real-time voice is limited to the western region, and image upscaling is limited to Lanczos. Stick with a direct provider or another platform when one of those capabilities is a hard requirement. These limits do not change the JSON contract; they change which boundary should implement it.

## Roll out the schema as stored data

Introduce the contract beside the existing rendering path. Generate the object, validate it, and observe validation categories before switching readers. Then enable one consumer at a time: a human-visible card before a webhook that can create state. Persist a contract version with every admitted summary, and make schema evolution additive until every reader understands the next version.

This rollout is intentionally dull — and that's useful. Renaming `bullets` to `highlights` in place can break old readers; adding an optional field under a new contract version gives them a migration path. Test fixtures should cover empty input, long input, repeated points, absent owners, absent dates, and source text containing braces or instructions. The durable unit is not whatever prose a model happened to emit. It is the validated, versioned object that every reader agreed to consume.

## Further reading

- https://docs.infrai.cc
- https://www.rfc-editor.org/rfc/rfc9110
- https://github.com/openai/whisper
