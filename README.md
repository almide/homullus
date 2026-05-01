# homullus

**Modular AI agent CLI runtime, written in [Almide](https://github.com/almide/almide).**

> *homullus* — Latin diminutive of *homo*: a small, made human. Almide is the language LLMs write; *homullus* is what the language writes into existence: a precise, autonomous agent that reads, writes, and runs your code.

```bash
# Pick any tool-calling provider (groq is free-tier, OpenAI-compatible)
export GROQ_API_KEY=gsk_...
MODEL=groq/llama-3.3-70b-versatile almide run src/main.almd
```

```
homullus v0.0.1
model: groq/llama-3.3-70b-versatile    trust: default    /help for commands

> list the .almd files in src/ using a tool
[Bash] {"command":"ls /abs/src/*.almd"}
[b2vcvddjg] /abs/src/agent_check.almd
                  /abs/src/agent.almd
                  /abs/src/main.almd
                  ...

> /model anthropic/claude-sonnet-4-6
Model set to: anthropic/claude-sonnet-4-6
```

End-to-end run captured: [`docs/dogfooding-2026-05-01.md`](docs/dogfooding-2026-05-01.md) — a real LLM driving Read + Edit through homullus's tool runtime, no human in the loop.

## Why homullus

The agent loop is small but it's the densest IO surface in any almide program: HTTP, fs, process, json, REPL, structured tool dispatch, recursive state. Building it in Almide does two things at once:

1. **Real, daily-driver agent** built on [`almai`](https://github.com/almide/almai) — multi-provider, no Anthropic-only lock-in.
2. **Pressure test** for the language. Wherever the implementation strains, the friction goes back upstream as a stdlib gap, a diagnostic improvement, or a new idiom for the cheatsheet.

It is also a deliberate alternative to [Aid-On/famulus](https://github.com/Aid-On/famulus) (TypeScript). Same shape, different substrate.

## Features (v0.0.1)

- **REPL** with slash commands (`/model`, `/tools`, `/trust`, `/clear`, `/help`, `/exit`)
- **6 built-in tools** — Bash, Read, Write, Edit, Glob, Grep
- **Multi-provider** via almai — Anthropic, OpenAI, OpenRouter, Groq, Cloudflare, Azure, Google, Bedrock, Claude CLI
- **3 permission modes** — `default` (read auto, others ask) · `accept-edits` (read+write auto) · `bypass` (everything auto)
- **Dangerous-pattern detection** for Bash (`rm -rf`, `curl | sh`, `mkfs`, fork bombs) — confirmed even in `bypass`
- **Retry with exponential backoff** on 429/5xx (via `almai.call_retry`)
- **Native tool roundtrip** — `assistant.tool_calls` ↔ `role:"tool"` with `tool_call_id`, working across every almai-supported provider that exposes tool calls
- **Provider injection** for testing — `agent.run_turn` accepts any `(model, msgs, opts) -> Result[LLMResponse, String]` callable, so tests use scripted responses without API keys
- **Smoke test** — `almide run src/smoke.almd` round-trips a real LLM call without a REPL
- **Integration check** — `almide run src/agent_check.almd` exercises 24 assertions across tool roundtrip / multi-round loop / history threading / provider error / no-tool path

## Verified end-to-end

| Provider             | Tool calls | Verified |
|----------------------|:---:|:---:|
| `groq/`              | ✅  | dogfooding 2026-05-01 (Read + Edit, [log](docs/dogfooding-2026-05-01.md)) |
| `cf/`                | ❌  | text-only (provider doesn't expose tool calls) |
| `cli/claude`         | ❌  | turnkey-agent (Claude Code dispatches its own tools) |
| `anthropic/`, `openai/`, `openrouter/`, `azure/`, `bedrock/`, `google/` | ✅ wire | covered by `almai`'s wire-format tests, no API keys at hand to run end-to-end |

## Layout

```
homullus/
├── almide.toml          package = homullus, deps: almai
├── src/
│   ├── main.almd        REPL, slash commands, state threading
│   ├── agent.almd       single-turn query + tool loop
│   ├── tools.almd       Bash/Read/Write/Edit/Glob/Grep dispatch
│   ├── permission.almd  3-mode resolver + dangerous patterns
│   ├── smoke.almd       non-interactive end-to-end check (1 round-trip)
│   └── agent_check.almd 24 integration assertions w/ scripted providers
└── README.md
```

`main.almd` is a conductor. State (model / mode / history / system prompt) is threaded through a recursive REPL — no globals, no mutability outside a single `run_turn`.

## Install

Requires [Almide](https://github.com/almide/almide) at the develop tip (PRs #231–#235 are required for cross-package codegen and JSON unicode handling). The next tagged release will include all of these.

```bash
git clone https://github.com/almide/homullus.git
cd homullus
almide test                       # 5 unit tests pass
almide run src/agent_check.almd   # 24 integration assertions pass
almide run src/main.almd          # interactive REPL
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

### v0.1.0
- Streaming text deltas (HTTP streaming intrinsic in almide stdlib + SSE parsers in almai providers)
- Token usage display in REPL prompt
- CLAUDE.md / git status auto-injection into the system prompt

### v0.2.0
- Auto-compact (token estimation + LLM summary at threshold)
- Output filtering (rtk-style) for `git status` / `npm install` / test output
- Session persistence (serialize/restore JSON, `/resume`)
- Memory integration (MEMORY.md + relevance search)

## License

MIT / Apache-2.0
