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

- [Why we say "Claude-compatible," not "Claude"](./articles/2026-04-25-claude-compatible-not-claude.md) — A short note on the legal and engineering value of holding a precise positioning vocabulary.

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
