# The backend trap: when your customers don't pick you, your distributors do

*Posted 2026-04-25 · ~6 min read*

> A reflection on the structural difference between operating an Anthropic-protocol-compatible relay and operating a tool that *uses* such relays. Written from the relay side.

## The shape of the market

When developers in restricted regions want to use Claude Code, the path looks roughly like this:

```
[Claude Code CLI]  →  [some configuration glue]  →  [some relay endpoint]  →  [some model]
                            ↑                            ↑
                       client tool                     backend (us)
```

Both layers are commercial:

- **Client tools** (CC Switch, claude-code-router, claude-code-hub, dozens of forks) help users manage which relay they use, with one-click setup and provider switching.
- **Backend relays** (PackyCode, AICodeMirror, llmapi.pro, and ten more) provide the actual API endpoint and model routing.

Both layers are valuable. But they have asymmetric leverage.

## The leverage gap

If you operate a backend relay, your customers experience your product through someone else's interface. The defaults that ship in the client tool determine which relays get tried first. The sponsor blocks in client READMEs determine which names users see. The provider preset list inside the app determines which relays even appear as one-click options.

You can build the world's most reliable, lowest-latency, most-honest Anthropic-compatible endpoint. If the dominant client tool's preset list doesn't include you, most of your potential customers never know you exist.

This is the **backend trap**: your distribution is mediated by a layer you do not own.

## Why this is structural, not solvable by being better

It's tempting to think "we just need to be the obvious best choice and the client tools will list us." This is wrong, and the wrongness has three sources:

1. **Client tools have their own economics.** Sponsorship slots, partner deals, default-preset placement — these are revenue for the client tool. The decision is "who pays best," not "who routes traffic best."

2. **Switching cost runs the wrong way.** A user who installed CC Switch and configured PackyCode there has zero reason to evaluate llmapi.pro on technical merit. They evaluate based on what's already in the dropdown.

3. **Data asymmetry compounds.** The client tool sees user-level signals — switches, retentions, configuration patterns. The backend sees only request-level signals — token counts, error rates, latency distributions. Over time, the client tool can tell which backends users actually prefer; the backend can only tell who's currently sending traffic.

## What we're not doing

The reflexive response is "build our own client tool." We're not doing that, for now, for three reasons.

**It's a worse use of capital.** A 50k-star desktop app is two-plus years of UX iteration with a community that already trusts the maintainer. Forking is a commodity move. Building from scratch in the same vertical means competing on UX in a market where switching cost works against new entrants.

**The protocol-layer moat we're already building is real.** Twelve protocol patches (postmortems coming on this journal); native Anthropic-protocol fidelity across edge cases; commercial backend partnerships; identity guard chain; subscription cycle stacking; all hard to replicate quickly. None of this is visible to the user but every piece reduces the substitutability that the backend trap relies on.

**Vertical specialization is more efficient than horizontal.** The market we serve — Chinese-region Claude Code users — needs payment rails, support workflow, and tooling defaults that no global client app will ship. We can build vertical features that no general client tool will ever match the depth of, without trying to win on UX surface.

## What we are doing

Three concurrent moves:

1. **Be in every client tool's surface.** Sponsorship inquiries, GitHub issues with concrete UX feedback, awesome-list submissions, README cross-links. The goal is "show up in every door the user might walk through."

2. **Maintain the data flywheel at the backend.** The backend sees less, but what it sees is harder for any client tool to replicate — full request shapes, tool-call patterns, model-behavior signals. We invest there. Forty-plus rules in our internal flywheel pipeline live in this layer.

3. **Document the engineering work in public.** This journal exists because the asymmetry between "what a client tool advertises" and "what a backend actually does" is most visible in the engineering details. If a future user is evaluating which relay to point their Claude Code at, our hope is that a single google search lands them on a postmortem that demonstrates the work.

## The triggers we watch

We're not building a client tool today. The conditions under which we would are concrete:

- **A dominant client tool signs an exclusive backend partnership.** The current market is multi-vendor on both sides; if that breaks, we move.
- **Two consecutive months of declining new-subscription growth that we can attribute to default-preset migration.** The loss of inbound from a category is a signal.
- **A peer backend (PackyCode, AICodeMirror, etc.) launches their own client tool.** The market dynamic shifts; we follow.

Until then, we're a backend that's pleasant to use, hard to replace, and visible everywhere users might be looking. That's a tenable position. It's just not a permanent one.

## Closing

The backend trap is real. Being aware of it is most of the work. The rest is showing up.

— [llmapi.pro](https://llmapi.pro)

---

*[← Back to journal index](../README.md)*
