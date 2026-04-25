# Why we say "Claude-compatible," not "Claude"

*Posted 2026-04-25 · ~4 min read*

> A short note on the value of holding a precise positioning vocabulary when you operate in a regulated grey zone.

## The temptation

When you build a relay that lets Chinese-region developers run Claude Code, the easiest English copywriting in the world reads:

> *"Use Claude in China."*

It's short, it's clear, it ranks well, it converts. It is also wrong.

What we operate is an **Anthropic-protocol-compatible API relay**. The traffic protocol matches Anthropic's published Messages API. The Claude Code CLI is Anthropic's official tool, which Anthropic itself documents as configurable via `ANTHROPIC_BASE_URL`. Our relay routes Claude-Code-shaped requests to a backend that *might or might not be the same model that Anthropic ships*.

The difference between *"Claude"* and *"Claude-compatible"* is the difference between a trademark complaint and a defensible product description.

## What our copy actually does

Concretely:

- Internal English: "Claude-compatible relay" / "Anthropic-protocol-compatible endpoint"
- External marketing English: same vocabulary, no exceptions
- Internal Chinese: we allow more direct phrasing because the audience expects it
- API responses: `model` field reflects the actual underlying model, never spoofed

The first time a user asks the model `"are you Claude?"` and gets `"I am Claude"`, you've crossed a line. We've spent meaningful engineering effort on the identity guard chain that prevents this.

## Three rules we hold

1. **Claude Code is openly configurable.** Anthropic's own documentation says so. Pointing the CLI at a custom backend is not a hack; it's a documented use case. We can describe this freely.
2. **Claude (the model) is proprietary.** We never claim to *be* Claude. We claim to be *compatible with the same protocol shape*.
3. **Disclosure is a feature.** Every public README and project page ends with: *"This project is independent and not affiliated with Anthropic."* Repeating it is not redundant — it's load-bearing.

## Why it's worth it

In the short term, the precise vocabulary loses you marketing punch. "Use Claude in China" beats "Anthropic-protocol-compatible relay" on every conversion metric.

In the long term:

- **Trademark exposure.** A single user-facing claim that the product *is* Claude is enough to attract a cease-and-desist. We've seen it happen to others.
- **Engineering posture.** When the team's vocabulary is precise, the engineering decisions follow. The identity guard chain exists because we wrote down the line; we'd never have built it from a "use Claude" framing.
- **User trust.** Power users notice. The fraction of customers who care about positioning is small but disproportionately influential — they write the LinkedIn posts, the Reddit comments, the technical blog reviews.

## Closing

If your product wraps a famous model, the words you choose matter as much as the protocol you implement. Hold the line on positioning vocabulary. The rest follows.

— [llmapi.pro](https://llmapi.pro)

---

*[← Back to journal index](../README.md)*
