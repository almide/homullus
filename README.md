# homullus

**Modular AI agent CLI runtime, written in [Almide](https://github.com/almide/almide).**

> *homullus* — Latin diminutive of *homo*: a small, made human. Almide is the language LLMs write; *homullus* is what the language writes into existence: a precise, autonomous agent that reads, writes, and runs your code.

```bash
export OPENAI_API_KEY=sk-...      # or ANTHROPIC_API_KEY / OPENROUTER_API_KEY / ...
almide run src/main.almd
```

```
homullus v0.0.1
model: openai/gpt-4o-mini    trust: default    /help for commands

> show me almide.toml
[Read] {"path":"/abs/almide.toml"}
[call_abc] [package]
name = "homullus"
...

> /model anthropic/claude-sonnet-4-6
Model set to: anthropic/claude-sonnet-4-6
```

## Why homullus

The agent loop is small but it's the densest IO surface in any almide program: HTTP, fs, process, json, REPL, structured tool dispatch, recursive state. Building it in Almide does two things at once:

1. **Real, daily-driver agent** built on [`almai`](https://github.com/almide/almai) — multi-provider, no Anthropic-only lock-in.
2. **Pressure test** for the language. Wherever the implementation strains, the friction goes back upstream as a stdlib gap, a diagnostic improvement, or a new idiom for the cheatsheet.

It is also a deliberate alternative to [Aid-On/famulus](https://github.com/Aid-On/famulus) (TypeScript). Same shape, different substrate.

## Features (v0.0.1)

- **REPL** with slash commands (`/model`, `/tools`, `/trust`, `/clear`, `/help`, `/exit`)
- **6 built-in tools** — Bash, Read, Write, Edit, Glob, Grep
- **Multi-provider** via almai — Anthropic, OpenAI, OpenRouter, Cloudflare, Azure, Google, Bedrock, Claude CLI
- **3 permission modes** — `default` (read auto, others ask) · `accept-edits` (read+write auto) · `bypass` (everything auto)
- **Dangerous-pattern detection** for Bash (`rm -rf`, `curl | sh`, `mkfs`, fork bombs) — confirmed even in `bypass`
- **Retry with exponential backoff** on 429/5xx (via `almai.call_retry`)
- **Native tool roundtrip** — `assistant.tool_calls` ↔ `role:"tool"` with `tool_call_id`, working across every almai-supported provider that exposes tool calls
- **Smoke test** — `almide run src/smoke.almd` round-trips a real LLM call without a REPL

## Layout

```
homullus/
├── almide.toml          package = homullus, deps: almai
├── src/
│   ├── main.almd        REPL, slash commands, state threading
│   ├── agent.almd       single-turn query + tool loop
│   ├── tools.almd       Bash/Read/Write/Edit/Glob/Grep dispatch
│   ├── permission.almd  3-mode resolver + dangerous patterns
│   └── smoke.almd       non-interactive end-to-end check
└── README.md
```

`main.almd` is a conductor. State (model / mode / history / system prompt) is threaded through a recursive REPL — no globals, no mutability outside a single `run_turn`.

## Install

Requires [Almide](https://github.com/almide/almide) ≥ 0.15.

```bash
git clone https://github.com/almide/homullus.git
cd homullus
almide test            # 5 tests pass
almide run src/main.almd
```

For a binary install (planned, once `[[bin]]` lands in `almide.toml`):

```bash
almide build src/main.almd -o ~/.local/bin/homu
homu
```

## Slash commands

| Command            | Description |
|--------------------|-------------|
| `/model [spec]`    | Show or set model. Format: `provider/model` (e.g. `anthropic/claude-sonnet-4-6`, `openai/gpt-4o-mini`, `openrouter/meta-llama/llama-3.3-70b-instruct`) |
| `/tools`           | List available tools |
| `/trust [mode]`    | Permission mode: `default` / `accept-edits` / `bypass` |
| `/clear`           | Clear conversation history |
| `/help`            | Show commands |
| `/exit`            | Exit |

Environment overrides at startup: `MODEL`, `HOMU_TRUST`.

## Roadmap

- Streaming text deltas (waiting on almai)
- Auto-compact (token estimation + LLM summary)
- Output filtering (rtk-style) for `git status` / `npm install` / test output
- Session persistence (serialize/restore JSON)
- Memory integration (MEMORY.md + relevance search)

## License

MIT / Apache-2.0
