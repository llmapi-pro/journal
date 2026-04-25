# Engineering depth: a technical inventory

*Posted 2026-04-25 · ~7 min read · revised*

> A short, deliberate inventory of what an Anthropic-protocol-compatible relay actually does at the engineering layer. Written because the public framing of relay services as "API resellers" or "people taking a margin" understates the work involved — and because being precise about your own work is a prerequisite for anyone else to take it seriously.

## Why this article exists

If you operate an Anthropic-compatible relay, the public framing is unkind. The casual reading from users is *"it's a middleman that takes a cut."* The casual reading from sponsoring tools is *"another sponsor candidate, behind PackyCode."* The casual reading from competitors is *"a commodity backend among many."*

Each of those readings is wrong, but you don't get to argue with them — you have to demonstrate the work.

This article is that demonstration. It is the sum of what one team built across six months running [llmapi.pro](https://llmapi.pro). It is not a marketing page. There are no claims of being "the best" or "industry-leading." Every item below is a real piece of code that solves a real problem, and most of them only exist because we tripped over the problem the hard way.

## The work that lives at the protocol layer

The Anthropic Messages API is documented, but documentation is not the same as compatibility. Twelve concrete protocol patches sit between "we forward HTTP requests" and "Claude Code works without users noticing the relay is there":

- **Wire format normalization** for the long tail of `tool_use_id`, `content_block`, and `message` shapes that Claude Code emits which strict-mode backends reject.
- **Streaming event shape and ordering** corrections so `content_block_start`, `content_block_delta`, and `message_stop` arrive in the order client parsers expect.
- **Tool-call ID normalization** between `toolu_01...` (Anthropic-style) and `call_...` (OpenAI-style) so IDs round-trip through the client without breaking mid-conversation.
- **Message-shape guards** for the empty-content / adjacent-role / orphan-tool-pair edge cases that trigger upstream errors.
- **Long-context tool-use loop detection** for when the client and model fall into byte-level identical retry loops past 80K tokens.
- **Per-key quota gating** to separate the large user bucket from per-key sub-buckets so distribution semantics work correctly.
- **Session hash collision fixes** so users running CC skill-hooks don't share session state across separate conversations.
- **Image, tool-result, and search content normalization** so heterogeneous backend responses match what the client expects to render.

Each of these started as a production incident and ended as a small piece of code with tests. Together they're the difference between a relay that works in a demo and one that survives sustained real traffic.

## The work that lives in the streaming layer

SSE looks simple from the outside; it is not. The streaming layer is where the most user-visible failures happen, and where the most engineering goes per visible feature:

- **Connect-time keep-alive heartbeats**, not data-time. The textbook keep-alive only fires after the first content frame; if the upstream takes 30 seconds to start emitting (long context, slow model), the connection dies before content arrives. Three lines of code, several thousand `socket closed` errors before the fix.
- **Parse-error retry friendliness** for backends that intermittently return non-HTTP responses on edge requests. The fix is internal: a single attempt-once retry, plus an SSE-friendly text fallback so partial responses don't corrupt the user's terminal.
- **Drain-aware deploys**. `docker stop` cuts active SSE mid-stream; we use a 180-second drain that closes upstreams on new connections but lets in-flight streams finish. Long sessions don't tear during deploys.
- **In-stream advisory hints**, opt-in by design, that surface long-running session warnings as plain text inside the SSE stream — never persisted server-side, never affecting the model output, just human-readable hints the user can ignore.

## The work that lives in the tool delegation layer

When the model invokes a tool, the relay can delegate that tool's execution to a faster backend. Concretely:

- **Server-side WebSearch tool delegation**: a faster search integration directly inside the relay. Chinese-language search queries return in 1–2 seconds instead of timing out at 8–25 seconds with default tool integrations. The delegation is fully scoped — only `WebSearch` calls are matched, every other tool flows through unchanged.
- **Strict tool-matcher precision** so the relay never delegates the wrong tool. Earlier work over-matched; the current matcher requires both URL field shape and tool-name regex to match exactly.

## The work that lives in the compliance layer

Operating a relay where users routinely ask the underlying model "are you Claude?" is a regulated grey zone, and the cost of getting it wrong is a trademark complaint:

- **Five-layer identity guard chain**: L1 system-prompt prefix, L3 web-grounded factual response routing, L5 streaming substring filter on identity-related content, plus a non-streaming variant and a recovery layer. Real production traffic shows clean smoke results across all paths.
- **Positioning vocabulary discipline**: "Claude-compatible relay," not "Claude." Every public surface, every API response, every README. The model field never spoofs.
- **Disclosure pattern**: every public artifact ends with explicit non-affiliation language, repeated across the org's GitHub repos, the journal, and the awesome list.

## The work that lives in the business layer

The non-technical work matters too:

- **Subscription cycle stacking**: monthly / quarterly / annual purchases plus a year-pays-for-ten-months promotion plus pro-to-max5x upgrades, all coexisting in a single user record without invariant violations.
- **Subscription lineage tracking** across four lifecycle stages, with the schema and admin paths for both backend and dashboard.
- **CNY rails**: Alipay, WeChat Pay, USDT for users in regions where official Anthropic billing is impossible.
- **Public changelog** at llmapi.pro/changelog, updated continuously. Users who can see what we're fixing tolerate things that briefly break.

## What this list doesn't include

A list of the things we did *not* do, deliberately:

- **No "smart" multi-model routing.** Single-base routing with one upstream gives consistency users actually feel. We tried the alternative; it was worse.
- **No general-purpose CLI fork.** Claude Code is openly configurable via documented environment variables; building a competing CLI doesn't move the needle and cedes Anthropic's continuous improvements to a fork that will lag.
- **No claim to being Claude.** The trademark line is held religiously. Every public-facing artifact uses "Claude-compatible." Every disclaimer repeats it.

## A note on visibility

The hardest thing about engineering work at the protocol and streaming layers is that none of it is visible to the user. The fixes that matter most are the ones the user never had to file a bug for. The scaffolding that protects against trademark exposure looks, from the outside, like nothing happening. The drain-deploy script that prevents broken streams during a release is invisible by design.

This article exists because the gap between "what was done" and "what the public sees" is the gap that lets people frame a relay operator as a middleman taking a margin. The framing is wrong, but it doesn't correct itself.

The correction is concrete demonstration. We're going to publish more articles like this one — not all at once, not as a marketing campaign, but as a steady accumulation of postmortems and decision logs. Six months in, this is what the public-facing slice of the engineering work looks like.

## Closing

If you're considering working with [our endpoint at llmapi.pro](https://llmapi.pro), or building something similar, or evaluating whether the work is real — this is what the work looks like. Open questions, war stories, and "you're wrong about #N" replies welcome at the [journal repo](https://github.com/llmapi-pro/journal/issues).

---

*[← Back to journal index](../README.md)*
