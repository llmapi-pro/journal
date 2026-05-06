# Tool-use IDs in the wild: when the same id appears twice

*Posted 2026-05-06 · ~8 min read*

> Postmortem from running [llmapi.pro](https://llmapi.pro), an Anthropic-protocol-compatible relay. Anthropic's Messages API uses `tool_use_id` strings to correlate tool calls with their results across turns. They are supposed to be unique within a conversation. We discovered, the hard way, that they are not always — and shipping a fix that does not break round-trip correlation is more delicate than it sounds.

## The protocol contract

In the Anthropic Messages API:

- An assistant turn can contain `tool_use` blocks, each with a `tool_use_id` (canonical shape `toolu_01...`).
- The next user turn supplies a `tool_result` block referencing the same `tool_use_id`.
- The server uses the id to associate the result with the originating call.

The contract is: ids must be unique within a conversation. Claude Code, when generating ids itself, draws from an entropy pool large enough that practical collisions are zero. When the upstream model is what generates ids, the entropy depends on the model.

## The symptom

A handful of long sessions started failing mid-turn with `400: duplicate tool_call id`. The error came from the upstream backend, not from the relay.

The cases all shared a pattern. Long conversations — twenty or more tool-use turns — eventually produced a request the upstream rejected. Restarting the session worked. Resuming from the last good message worked. The middle of the conversation was the problem.

When we dumped the rejected request, the same `tool_use_id` value appeared in two different `tool_use` blocks, several turns apart. The model had reused an id.

## Why the model reuses ids

We do not have a clean answer; we have a hypothesis. The upstream we route to in this case generates tool_use ids itself rather than letting the relay synthesize them. Its id space is finite. In long contexts, the model — having seen many earlier `toolu_...` strings in its own past output — sometimes produces a new tool_use whose id collides with an old one. Whether this is a tokenizer artifact, a sampling artifact, or something deeper in the model's id-generation behavior, we do not know.

What we know is that we cannot rely on the upstream's id stream being collision-free in practice. A relay that wants to keep long sessions alive has to handle collisions.

## What does not work

A few approaches we considered and rejected:

**Regenerate every id at the relay edge.** Tempting: replace every upstream-generated `tool_use_id` with a fresh `toolu_relay_<uuid>`. Problem: when Claude Code emits the matching `tool_result` on the next turn, it carries the relay's regenerated id, which the upstream does not recognize. Now we need a translation table that round-trips both directions, and getting the table right turned out to be harder than the dedup we ended up doing.

**Reject conversations once a duplicate is observed.** This is the "tell the user" path. It works as a fallback but gives up on recovery. Long sessions are exactly the ones a user is most invested in not losing.

**Ask the model to retry.** Sometimes works, often doesn't, and adds a round-trip every time we hit a collision.

## What we shipped — occurrence-counter dedup

The shape that worked is small.

For each `tool_use_id` value seen in the conversation, the relay maintains a per-conversation counter of how many times it has been emitted. On the second occurrence, the relay rewrites the id by appending a counter suffix:

```
First occurrence:  toolu_abc123
Second occurrence: toolu_abc123-2
Third occurrence:  toolu_abc123-3
```

The translation table maps `(rewritten_id) → (original_id, occurrence)` and is keyed on the conversation. When Claude Code emits a `tool_result` for `toolu_abc123-2`, the relay translates back to `toolu_abc123` before forwarding to the upstream — and the upstream finds the matching call in its own state by occurrence ordering.

The pieces in detail:

- **Counter is per-id, per-conversation.** Resets on a new conversation.
- **Suffix shape is regex-safe.** We use `-N` not `(N)` because parentheses appear elsewhere in tool-use payloads and the dash is unambiguous in the canonical id charset.
- **Translation runs both directions.** Outbound: rewrite on emit if collision. Inbound: rewrite back on the matching `tool_result`.
- **Translation table is in-memory, scoped to the request handler chain.** Conversations are stateless from the relay's perspective; the table reconstructs from the request history each turn.

## What broke when we first shipped it

Two things, both predictable in hindsight.

**Block ordering.** The dedup pass ran on `tool_use` blocks but not on the `tool_use_id` field that appeared inside `tool_result` blocks earlier in the same request. Claude Code sometimes constructs requests where `tool_result` references and `tool_use` declarations of the same id appear in different turns of the supplied history. We had to walk the entire `messages` array, not just the most recent turn.

**Incomplete tool_use blocks during streaming.** When the upstream emits a `tool_use_id` mid-stream (it shows up in `content_block_start`), the dedup decision has to fire before any subsequent `content_block_delta` references the id. We gated SSE forwarding behind the dedup check. This added a small per-block latency — single-digit milliseconds — which we accepted.

## The diagnostic dump

Even with dedup in place, we still occasionally see translation errors that suggest a case the dedup logic does not handle. To debug those, the relay now dumps any 400-class response from the upstream — full message timeline, the rejected payload, the relay-side translation table — into a write-once file under a date-stamped path. We grep these files when a user reports a confusing failure. Most of the time they are routine; occasionally they surface a new edge case.

This is the kind of diagnostic that pays for itself the first time a user reports "my session broke at turn 47," and you can replay the exact translation state from cold.

## Closing

A relay's mental model of "the protocol" is the textbook spec. The protocol you actually run is the textbook spec plus every quirk of every backend you route to. Tool-use id collisions are the kind of quirk you only find in production, on long sessions, on the third or fourth time a particular shape of conversation appears.

The fix is small. The thing that makes the fix work — diligent two-direction translation, full-history dedup, replay-able diagnostic dumps — is not. If you are running a Claude-compatible relay against any backend that generates its own ids, plan for this from day one. Not because it is hard to fix afterward; because the user-visible failure during the period before the fix lands is exactly the kind that erodes trust the most: long, productive sessions that suddenly break in the middle.

---

*llmapi.pro is an independent, Claude-compatible API relay; we are not affiliated with Anthropic. The Claude Code CLI's `ANTHROPIC_BASE_URL` environment variable is documented and supported by Anthropic for pointing at compatible endpoints.*
