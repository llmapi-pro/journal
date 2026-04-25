# Journal

> Build log and engineering notes from running [llmapi.pro](https://llmapi.pro), an Anthropic-protocol-compatible API relay.

What lives here:

- **Postmortems and war stories** from running an Anthropic-compatible relay in production.
- **Design notes** on protocol adaptation, streaming, tool-use interception, identity guards.
- **Decision logs** — why we picked specific tradeoffs, what we'd do differently.

These notes are written for engineers building or debugging Claude Code clients, agentic workflows, or LLM gateways. The goal is concrete and specific: real bug, real fix, real numbers.

---

## Articles

### Protocol adaptation

- [Twelve protocol patches behind one relay](./articles/2026-04-25-twelve-patches-behind-one-relay.md) — Six months of running Anthropic-protocol-compatible traffic against a non-Claude backend. What broke, what we fixed, what we'd ship differently on day one.

### Engineering culture

- [Engineering depth: a technical inventory](./articles/2026-04-25-engineering-depth-technical-inventory.md) — A deliberate inventory of what an Anthropic-protocol-compatible relay actually does at the engineering layer. Protocol adaptation, streaming, compliance, and business-side mechanics that live below the relay surface.
- [Why we say "Claude-compatible," not "Claude"](./articles/2026-04-25-claude-compatible-not-claude.md) — A short note on the legal and engineering value of holding a precise positioning vocabulary.
- [Drain deploy: how to swap a relay without tearing live SSE](./articles/2026-04-25-drain-deploy-no-tearing-live-sse.md) — A deploy incident cost us a support email with a timestamp. The fix was a twelve-line state machine and a two-nginx-reload deploy script. What we got wrong first.


### Strategy

- [The open-weight race and the protocol-layer relay](./articles/2026-04-25-open-weight-race-protocol-layer.md) — When base models commoditize, the wrapper layer becomes the moat. A note on what the DeepSeek V4 / open-weight acceleration means for relay operators.
- [The backend trap: when your customers don't pick you, your distributors do](./articles/2026-04-25-the-backend-trap.md) — Why operating an Anthropic-compatible relay puts you one layer behind your users, what we are and aren't doing about it.

---

## Adjacent projects

- [llmapi.pro](https://llmapi.pro) — The relay this journal documents.
- [llmapi-pro/claude-code-setup](https://github.com/llmapi-pro/claude-code-setup) — Cross-platform one-line Claude Code installer.
- [llmapi-pro/awesome-claude-code-china](https://github.com/llmapi-pro/awesome-claude-code-china) — Curated Claude Code resources for Chinese developers.
- [llmapi-pro/claude-md-templates](https://github.com/llmapi-pro/claude-md-templates) — `CLAUDE.md` templates for popular stacks.
- [llmapi-pro/claude-next](https://github.com/llmapi-pro/claude-next) — Session handoff for deep Claude Code conversations.

## License

[MIT](./LICENSE) — fork, quote, link freely. Attribution appreciated.

---

> *This journal is independent and not affiliated with Anthropic. Claude Code is Anthropic's official tool.*

---

## Sponsorship & Collaboration

This journal documents an independent team's production relay stack. We're open to:

- **Sponsored content / placement** in upcoming journal articles, with disclosure.
- **Tooling integration partnerships** — your router, IDE extension, or agentic framework, listed as a tested companion.
- **Backend / infrastructure partnerships** — model providers, search APIs, observability tools.
- **Strategic conversations** — acquisition, regional licensing, deeper integrations.

**Contact:** [partnerships@llmapi.pro](mailto:partnerships@llmapi.pro) · or open an issue here.
