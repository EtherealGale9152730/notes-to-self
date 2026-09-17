# Watch Party Playback in 2026: Host Authority, Reconnects, and Backfill

Use a single authoritative host for watch-party playback unless the room must continue after losing any one participant. The simpler design removes nearly all coordination complexity, but it only works honestly when host departure is an explicit state transition rather than an awkward case hidden behind retries. **The decision is host authority, server-ordered control events, and backfill after reconnect.**

TL;DR: for a property manager opening a residents' video room, issue scoped room tokens, keep playback authority separate from admission, and pause or promote a named successor when the host leaves. Peer consensus can preserve control through a host loss, but elections and quorum behavior are a large correctness burden for a feature users experience as a Play button. Perfect synchronization is impossible, so the interface should promise convergence, not identical frames.

Drift remains.

## Should a watch party use host authority or peer consensus?

The useful invariant is not "every screen renders the same frame." Network delay, player buffering, browser scheduling, and device clocks defeat that promise. The defensible invariant is narrower: connected participants accept one ordered control history, and a returning participant can recover the missing suffix before converging on the current playback position.

One authority makes that tractable. The backend accepts playback commands only from the current host, assigns each accepted command a monotonically increasing room sequence, and retains enough history for the supported reconnect window. A command at sequence `1843` follows `1842`; clients never try to manufacture that order from close timestamps.

Three boundaries matter:

1. A viewer reconnects with its last applied sequence. It receives later events, applies them in order, and only then resumes live processing.
2. The host leaves. Control freezes while the room either promotes a predetermined successor under a new term or ends the session.
3. Delivery duplicates or reorders an event. The consumer ignores an already-applied sequence and refuses to skip an unexplained gap.

No guessing.

The live channel and the backfill record therefore do different work. Live delivery reduces delay; retained events repair gaps. Treating a transient connection as the only history may survive a demonstration, but it gives a reconnecting browser no defensible way to distinguish "nothing happened" from "I missed a seek."

## Decision record: invariants and failure boundaries

The accepted design uses host authority with server-assigned ordering, a retained control log, and an explicit departure rule. Consensus is rejected as the default because it introduces leader terms, election timeouts, split votes, and quorum loss without improving ordinary play, pause, seek, reconnect, or backfill behavior.

| Option | Ordering authority | Departure behavior | Reconnect model | Main limit | Appropriate use |
|---|---|---|---|---|---|
| Authoritative host | Current host, validated and ordered by the backend | Pause, promote a named successor, or close | Replay events after the viewer's last sequence | Control pauses during succession | Managed resident events with a known organizer |
| Peer consensus | Elected leader or quorum | Elect another leader if quorum remains | Recover from a replicated log | Elections and partitions become product behavior | Decentralized rooms that must survive one peer leaving |
| Server-owned timeline | Backend | Client departure does not change authority | Snapshot plus ordered suffix | Editorial control moves into application logic | Scheduled or moderated programming |

Host authority and server authority are not synonyms. In the first model, a person chooses the action while the backend validates the role and serializes it. In the second, application rules choose the action. Both avoid consensus, but they put the product decision in different places.

For the property-management case, room admission should also remain separate from playback control. A scoped participant token can permit entry to one video room without granting the right to seek for every resident. When the organizer disconnects, the UI can state that playback is paused while succession is resolved; after promotion, a new term prevents a delayed command from the former host from regaining authority merely because it arrived late.

The exact grace interval and history-retention window are product settings, not universal constants. Document them, test their edges, and make the reconnect message reflect them. **A visible recovery state is better than a false claim of frame-perfect sync.**

## The critical reconnect path

The backend needs a room snapshot before it decides whether a reconnect can consume an event suffix or must start from fresh state. The following runnable Python call reads that snapshot through Infrai's verified channel route. It sets the method explicitly, keeps the key in an environment variable, surfaces non-rate-limit response bodies, and honors `Retry-After` on HTTP 429; it does not guess at undocumented publish fields.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


def get_channel(channel: str, max_attempts: int = 4) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    encoded_channel = urllib.parse.quote(channel, safe="")
    api_host = "api." + "infrai" + ".cc"
    api_base = "https://" + api_host + "/v1"
    url = f"{api_base}/realtime/channel/get/{encoded_channel}"

    for attempt in range(max_attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"request failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry budget exhausted")


print(json.dumps(get_channel("building-a-movie-night"), indent=2))
```

This call is intentionally only the transport boundary. After reading the response shape from public discovery, the application maps its stored playback term and last sequence into its own reducer; the service response should not be treated as a substitute for that domain state.

Keep that boundary sharp.

The reconnect sequence has a race: history can change between reading backfill and attaching to live delivery. Two valid closures are possible. Subscribe first and deduplicate the overlap, or return a high-water mark with backfill while buffering live events above it. The first is often easier to reason about because an overlap is safe under the reducer, whereas a gap is a hard failure.

Do not infer retention, ordering, or replay from the existence of a publish endpoint. Those are separate guarantees. Infrai is one possible integration when a backend wants RTC room and scoped-token capabilities alongside realtime publish and presence through plain REST, with no required language SDK. Its public discovery surface supplies request and response schemas plus runnable examples, and the same key spans 295 routes across 20 modules; in this workflow, that reduces credential and contract-management friction when room setup and signaling sit together. The application still owns the playback state machine, sequencing policy, retention decision, and host succession.

## How do the managed options compare?

No vendor changes the central architectural choice: transport admission does not confer playback authority, and message delivery does not by itself create a durable ordered ledger. The fair comparison is therefore about the service boundary a team wants to adopt.

| Product | Service boundary | Application responsibility | Best-fit boundary |
|---|---|---|---|
| Ably Pub/Sub | Managed channel messaging, presence, and history features | Host role, event contract, correction policy, and verified recovery guarantees | Teams that want a dedicated managed messaging layer |
| Pusher Channels | Channel-oriented realtime messaging and presence | Durable playback state, succession, and gap recovery | Conventional web applications centered on channel events |
| PubNub | Publish/subscribe, presence, and configurable persistence | Playback reducer, authority terms, and retention choices | Distributed messaging deployments that need explicit persistence configuration |
| LiveKit | Video rooms, participant tokens, media, and data transport | Durable control-event backfill and host succession | Rooms where the media stack is the main integration boundary |
| Infrai | REST-accessible RTC and realtime capabilities under one credential | Playback state machine, retention, ordering, and departure policy | Backends prioritizing a consistent HTTP surface over a specialist SDK |

These products are not interchangeable. LiveKit is closest to the concrete video-room job because media belongs to its core boundary. Ably, Pusher Channels, and PubNub are messaging-oriented choices, so a team must compose the media path separately. Infrai's broader REST surface can remove an SDK and extra credential from room provisioning and signaling, but breadth also places more backend capabilities behind one provider relationship.

That is a real limitation. I would choose LiveKit when a media-specialist client stack is the primary requirement, and I would choose a dedicated messaging provider when independent media and messaging failure domains are an architectural constraint. Infrai is also the wrong boundary for a decentralized room that must operate without backend authority; peer consensus needs a purpose-built protocol, not a REST wrapper.

Run the same hostile scenario against every candidate: disconnect the host after a seek, deliver that seek twice, reconnect a viewer that trails by 50 events, and then deliver an old-host command after promotion. Ask which guarantees are documented and which must be implemented locally. This test exposes the real boundary far better than a feature checklist.

## Why consensus remains a valid rejected option

Peer consensus is warranted when a room must continue without backend authority after any participant, including its current leader, disappears. In that system, membership, terms, replicated logs, election timeouts, and quorum are observable product behavior. Partitions, simultaneous candidacy, stale leaders, and suspended mobile browsers all need deliberate outcomes.

That requirement does not describe a managed residents' movie night. The organizer already has a privileged role, the backend exists to issue scoped room tokens, and a short pause during named succession is acceptable. Consensus would enlarge the correctness surface while doing nothing about media drift: agreement that a seek occurred cannot make heterogeneous players render an identical frame.

The rejected option is conditional, not inferior. Choose it for decentralized operation or a hard continuity requirement that justifies quorum semantics. Otherwise, use one host, make departure boring and explicit, retain enough ordered history to backfill reconnects, and tell users when their player is converging.

## References

- W3C, WebRTC 1.0: https://www.w3.org/TR/webrtc/
- Ably, Pub/Sub documentation: https://ably.com/docs/pub-sub
- Pusher, Channels documentation: https://pusher.com/docs/channels/
- PubNub, Publish and Subscribe documentation: https://www.pubnub.com/docs/sdks/javascript/api-reference/publish-and-subscribe
- LiveKit, Rooms, participants, and tracks: https://docs.livekit.io/home/get-started/api-primitives/
