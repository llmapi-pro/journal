# Reasoning blocks at the relay: what changes when the upstream emits thinking

*Posted 2026-05-06 · ~10 min read*

> This is a working note from running [llmapi.pro](https://llmapi.pro), an Anthropic-protocol-compatible API relay. As open-weight reasoning models — DeepSeek's V3.x line, MiniMax's M-series with reasoning, Qwen3's reasoning variants — start to ship interleaved thinking blocks, the protocol-translation layer between Claude Code and "compatible-but-not-identical" backends gets a new wrinkle. This entry is what we have learned trying to make `extended thinking` round-trip cleanly.

## The shape of the feature

Anthropic added extended thinking to the Messages API in 2024. The shape, in summary:

- Requests opt-in with `thinking: { type: "enabled", budget_tokens: N }`.
- Responses then contain `content` blocks of type `thinking` (and sometimes `redacted_thinking`) interleaved with the normal `text` and `tool_use` blocks.
- Each `thinking` block carries a `signature` field that the client must echo back unchanged on the next turn.
- In streaming, the SSE timeline is: `content_block_start` (type=`thinking`) → many `content_block_delta` (type=`thinking_delta`) → `content_block_stop`, then the next block starts.

Several open-weight providers have shipped near-isomorphic features over the past year. Functionally similar. Wire format: not always identical.

For a relay that fronts Claude Code, the contract on the client side is fixed: Claude Code expects Anthropic's exact event ordering, exact field names, and a stable signature it can replay. The contract on the upstream side varies. The middle is where the work is.

## Mismatch 1 — Thinking emitted as content delta

The simplest and most painful version: an upstream that has reasoning enabled, but emits the reasoning tokens in the same `content_block_delta` events it uses for visible text. No `type: "thinking"` discriminator.

If you forward those bytes through, Claude Code renders the model's chain-of-thought directly into the assistant turn. Users see "Let me think about this. The user is asking… Actually, wait, I should check…" before the actual answer. It is the wrong UX, and depending on what is in the thinking, it can also be a privacy footgun.

The fix is structural, not regex: the relay needs an upstream-specific stream parser that knows where thinking starts and ends — usually a sentinel sequence in the raw stream, or a separate channel — and converts those byte ranges into Anthropic-shape `thinking` blocks before they reach the client.

Two implementation notes that bit us:

1. **Sentinel detection has to be byte-exact across UTF-8 boundaries.** A sentinel that lands across two TCP frames will not match if your matcher operates on per-frame buffers. Use a rolling window with a high-water mark, not a per-chunk regex.
2. **Thinking blocks that finish mid-stream need their own `content_block_stop` before the next block starts.** Forgetting this produces an SSE sequence Claude Code parses as one giant block, and the renderer never breaks it up.

## Mismatch 2 — Thinking ordering

Anthropic's spec emits `thinking` blocks **before** the final answer text. Some upstreams emit thinking **after** their answer, as a kind of trailing audit log. Others put thinking and text into completely separate streams that you have to multiplex.

For Claude Code, the order on the wire is what gets rendered. If thinking arrives after text, the user sees the answer first, then a confusing chain-of-thought block appearing under it. The session is functionally fine but visually wrong.

If your upstream gives you a structured response (non-streaming), reorder it server-side before re-emitting as SSE. If it streams in the wrong order, you have two choices: buffer the entire response and re-order (kills perceived latency on long responses), or emit thinking blocks as `redacted_thinking` placeholders during streaming and replace them in a follow-up turn (complex). We have not found a clean third option.

## Mismatch 3 — Thinking interleaved with tool_use

This is the case where things get interesting.

A reasoning model with tools enabled may produce: thinking → tool_use → (tool result on next turn, supplied by the client) → thinking → tool_use → thinking → text. Anthropic's protocol handles this by alternating block types within a single assistant turn or splitting across turns. Open-weight backends implement it three different ways we have seen so far:

- **One block of thinking, then a sequence of tool_uses, then text.** Closest to Anthropic. Easy.
- **Thinking emitted as a side-channel that runs concurrently with tool_use blocks.** Requires the relay to merge two streams into one ordered SSE timeline.
- **Thinking re-emitted after each tool_result, even if the result is identical.** The relay either dedupes or forwards verbose; both choices have failure modes.

We landed on "merge into the single SSE timeline, in source-emission order, and tag each block with its real type." Verbose is the right default — Claude Code is tolerant of extra blocks but not of missing ones.

## The signature field

Anthropic's `thinking` blocks carry a `signature` field. Claude Code echoes it back on the next turn so the server can verify that the client did not tamper with the chain-of-thought. Lose or rewrite the signature and the next turn 400s.

Open-weight backends do not have this signature. They do not need it — the trust model is different. But Claude Code does not know that, and will refuse to send the next turn if the signature is missing.

The pragmatic answer: the relay synthesizes a signature on emit, stores the mapping, and validates the echo on the next inbound request. This is a pure protocol-shape concern — no inspection of the thinking content itself, no logging beyond what is required to validate the signature on its return trip.

## Cost and budget accounting

The `budget_tokens` field on the request is supposed to bound the thinking. Most open-weight backends do not honor it directly; they have their own knob (or no knob at all). The relay either translates `budget_tokens` into the upstream's nearest equivalent or accepts that the upstream will run to its own limit.

We translate when the upstream has any equivalent, and pass-through with a logged warning otherwise. The user-visible token usage in the response should still account for thinking tokens — Claude Code's billing UI relies on `usage.cache_creation_input_tokens` and friends being accurate.

## Closing

Reasoning models are now table stakes for any backend that wants to compete with Sonnet 4.x for coding work. From the relay side, that means the protocol-translation layer can no longer treat the upstream response as "just content + tool_use." Thinking is a third kind of block, with its own ordering rules, its own streaming shape, its own signature semantics, and its own UX consequences if you get it wrong.

For anyone building or running a Claude-compatible relay against open-weight reasoning backends:

1. Detect thinking at the byte level on the upstream stream, not at the JSON level after parsing.
2. Re-emit thinking as Anthropic-shape blocks, in source order, with synthesized signatures the relay can later validate.
3. Buffer the minimum needed to fix ordering. Buffering the whole response defeats streaming.
4. Treat `redacted_thinking` as a real first-class block — it is the right tool when you cannot or should not pass the raw thinking through.
5. Make sure your token accounting includes thinking. Users notice when their billing UI disagrees with reality.

The protocol gap between "supports thinking" and "supports thinking the way Claude Code expects it" is wider than it looks. But it is closeable. Once it is closed, the open-weight reasoning models are very pleasant to drive Claude Code with.

---

*llmapi.pro is an independent, Claude-compatible API relay; we are not affiliated with Anthropic. The Claude Code CLI's `ANTHROPIC_BASE_URL` environment variable is documented and supported by Anthropic for pointing at compatible endpoints.*
