# Twelve protocol patches behind one relay

*Posted 2026-04-25 · ~10 min read*

> This is a postmortem-style retrospective from running [llmapi.pro](https://llmapi.pro), an Anthropic-protocol-compatible API relay, in production for six months. The relay terminates Claude Code's HTTP and forwards translated requests to a different backend. We label the patches H1 through H12 internally; this is what each one taught us.

## Why the relay exists

Direct access to `anthropic.com` is restricted in mainland China. Local developers either run their own reverse proxy (logistically painful), buy access from third-party relay providers (a market with rapidly varying quality), or don't use Claude at all.

We chose to build a relay because Anthropic's official documentation supports the use case: Claude Code's `ANTHROPIC_BASE_URL` environment variable is documented and explicit. Point it at any Anthropic-API-compatible endpoint and Claude Code routes there.

We took it literally. Six months later, we've shipped twelve protocol-level patches we did not see coming.

## Surprise 1 — "Compatible" is a spectrum

The first version of the relay assumed protocol translation would be a thin layer. The MiniMax HTTP surface looked Anthropic-compatible: same Messages API shape, same tool-use schema, same SSE event names. We assumed `ANTHROPIC_BASE_URL=https://our-relay/` would route Claude Code's traffic with minimal transformation.

It did not.

Three categories of mismatch surfaced under real workloads:

1. **Wire-format edge cases.** Anthropic accepts both `tool_use_id` and certain legacy field shapes; the backend was strict-mode and 400'd on shapes Claude Code regularly emits.
2. **Streaming semantics.** Both servers emit SSE, but the shape and ordering of `content_block_start` / `content_block_delta` / `message_stop` events differ subtly.
3. **Tool ID conventions.** Claude Code emits `toolu_01...`-prefixed IDs. Some backends emit `call_...`-prefixed (OpenAI-style). The IDs round-trip back from the client and need to match exactly, or you get `400: tool_use_id mismatch` mid-conversation.

Each became a patch. Lesson: capture real client traffic from day one. A folder of raw HTTP captures from the actual Claude Code binary is worth a hundred protocol-spec readings.

## Surprise 2 — SSE breakage doesn't look like SSE breakage

We knew the textbook fix for long-running SSE: send periodic comments during silent windows or intermediate proxies (Cloudflare, nginx, browsers) close the connection. We had keep-alive heartbeats from day one.

What we missed: a long *first chunk* gap is just as fatal. Our keep-alive logic only kicked in *after* the first content frame. If the upstream model needed thirty seconds to start emitting (large context, complex tool resolution), the connection died before we got a chance.

The fix was three lines:

```typescript
// before
upstream.on('data', startHeartbeat);  // too late

// after
upstream.on('connect', startHeartbeat);  // catches first-chunk gap
```

Several thousand `socket closed unexpectedly` errors before we figured this out.

## Surprise 3 — Tier mismatch is an injection surface

Our development pool shared keys with a few free LLM providers. One day a developer hardcoded a paid-tier model into the development branch. Two weeks later we found a small invoice for inference we never intended to bill.

Lessons:

- API key tier and model tier must be jointly validated at the request level, not at config-load time.
- Provider abstractions hide tier mismatches if you're not careful.
- Free-tier providers should explicitly refuse to route to paid models, even if a key would technically authorize it.

We added a tier-aware router gate after that. Cheap insurance.

## Surprise 4 — `tool_result` rebroadcast can hijack live conversations

This one was nasty.

Claude Code emits `tool_result` blocks back to the model on every turn. Our relay intercepts certain tool calls (notably `WebSearch` — we route those through MiniMax's built-in search rather than relying on the model's own tool execution).

For a brief window, our `isSearchTool()` check matched too broadly — it treated `WebFetch` calls (a different tool entirely) as search calls and overwrote their `tool_result` with a search summary. The user saw "I'm fetching X" → and then a totally unrelated answer about something the relay searched for.

Patch: tighten the matcher to require both a URL field shape *and* the actual tool-name regex. Ninety-one sessions affected; zero hijacks since the patch.

The takeaway: when intercepting protocol-level events, the matcher must be precise to a level that feels almost paranoid. "Probably this kind of tool" is a bug.

## Surprise 5 — Identity questions bypass the safety net

Users started asking the model "are you actually Claude?" and getting a variety of inconsistent answers depending on which backend the routing landed on. Some returned "Yes, I'm Claude." Some "I'm Qwen." Some refused to answer.

This is a compliance issue, not just a UX one. We don't claim to *be* Claude — we claim to be *Claude-compatible* (different word, different legal posture). The relay had to enforce that posture even when the underlying model would happily roleplay otherwise.

The fix is a five-layer guard chain (L1 prompt prefix, L3 web-grounded fact response, L5 streaming filter on identity-related substrings). It's invasive — touching system prompts is dangerous — but the alternative is being one bad screenshot away from a trademark complaint.

If you're building any kind of model-routing layer, plan for the identity layer up front, not after a user posts a confusing screenshot somewhere.

## What we'd ship on day one

In rough priority order:

1. **Capture all real client traffic from day one.**
2. **Validate at the boundary, twice.** Once when the request arrives (translate + reject if shape is wrong), once before forwarding (compose + assert your translation is well-formed).
3. **Ship keep-alives before you ship streaming.**
4. **Tool-call interception is a privileged action.** One well-tested place; don't sprinkle "if it looks like a search" checks across the codebase.
5. **Identity guards are a Day-1 feature, not a Day-90 patch.**

## What worked

- **Single-base routing** instead of "smart" multi-model routing. We tried mixing providers and the consistency cost was worse than the latency win.
- **Public changelog.** Users tolerate broken things if they can see what we're fixing. They do not tolerate silent regressions.
- **Drain-deploy script** (`docker compose down`-driven 180-second drain instead of `docker stop` immediate kill). Long SSE sessions don't get torn during deploy.

## Closing

If you're considering building a Claude-compatible relay, you can do it. The protocol is documented and the official `ANTHROPIC_BASE_URL` knob is real. But "compatible" is a spectrum, not a checkbox, and the gap between 95% and 99% is most of the work.

Our relay runs at [llmapi.pro](https://llmapi.pro). Protocol surprises welcome — open an issue at [llmapi-pro/journal](https://github.com/llmapi-pro/journal/issues) if you've hit the same kind of wall.

---

*[← Back to journal index](../README.md)*
