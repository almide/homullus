# Dogfooding session — 2026-05-01

First end-to-end run of homullus driving a real LLM through its own tool runtime, with the LLM modifying a file based on its own tool-call plan. The file shown here is the actual transcript, not a contrived demo.

## Setup

- **Compiler**: almide @ develop (post-PRs #231 #232 #233 #234 #235)
- **Library**: almai @ main (post-PRs #2 #3 #4)
- **Provider**: `groq/llama-3.3-70b-versatile` (free tier, OpenAI-compatible function calling)
- **Permission mode**: `bypass` (auto-accept all tool calls — no human in the loop)

## Task

Add `fn greet(name: String) -> String = "hi, " + name` to `/tmp/dogfood2.almd`, which contains a single function `fn current_year() -> Int = 2026`.

## Input

```bash
echo 'fn current_year() -> Int = 2026' > /tmp/dogfood2.almd

set -a; . ./.env; set +a   # exposes GROQ_API_KEY
printf "Read /tmp/dogfood2.almd. Then use Edit to add a single new line at the end. The old_string should be 'fn current_year() -> Int = 2026' and the new_string should be the same line followed by a newline and then 'fn greet(name: String) -> String = \\\"hi, \\\" + name'.\n/exit\n" \
  | MODEL=groq/llama-3.3-70b-versatile HOMU_TRUST=bypass almide run src/main.almd
```

## Transcript

```
homullus v0.0.1
model: groq/llama-3.3-70b-versatile    trust: bypass    /help for commands

> Read /tmp/dogfood2.almd. Then use Edit to add a single new line at the end. ...

[Read] {"path":"/tmp/dogfood2.almd"}
[tcnkz0ywn] fn current_year() -> Int = 2026

[Edit] {
  "old_string":"fn current_year() -> Int = 2026",
  "new_string":"fn current_year() -> Int = 2026\nfn greet(name: String) -> String = \"hi, \" + name",
  "path":"/tmp/dogfood2.almd"
}
[sjja1ke30] edited /tmp/dogfood2.almd

The file /tmp/dogfood2.almd has been read and edited to add a new line at the end.

> /exit
Bye!
```

## Result

```almide
fn current_year() -> Int = 2026
fn greet(name: String) -> String = "hi, " + name
```

The file ends with the new function exactly as instructed, with proper `->` arrows (no `?` corruption from the JSON unicode escape stub that PR #235 fixed).

## What this proves

1. **REPL** accepts piped stdin, parses the user's request, dispatches to the LLM
2. **almai groq dispatcher** correctly routes `groq/...` model spec, populates `GROQ_API_KEY` via `require_env`, makes an OpenAI-compatible request
3. **Tool definitions** advertised to the LLM (Bash/Read/Write/Edit/Glob/Grep) match what the model emits in `tool_calls`
4. **Native tool roundtrip** — the assistant message (with `tool_calls`) and the subsequent `role:"tool"` message with `tool_call_id` correctly correlate across turns
5. **Permission resolver** runs (`bypass` short-circuits, but `default` would have prompted)
6. **Edit tool** verifies uniqueness of `old_string` and performs the in-place replacement
7. **Multi-round agent loop** — LLM said "first Read, then Edit", and homullus executed both rounds with the file content threaded through history

That's the same shape Claude Code uses; it's now demonstrated running entirely on Almide.

## What this does *not* yet prove

- Streaming UI (responses arrive as a single block — addressed in upcoming streaming PR)
- Long-horizon sessions (a multi-hour coding task with auto-compact)
- Confirmation flow (`default`/`accept-edits` permission UX)
- Provider with tool-calling other than groq (anthropic / openai unverified end-to-end here, only verified at the wire-format level)

## Compiler / library bugs surfaced and fixed during this run

| PR | repo | fix |
|---|---|---|
| #231 | almide | external dep self-imports + lambda inference |
| #232 | almide | cross-module default_fields |
| #233 | almide | module-level effect fn auto-Try |
| #234 | almide | top-level auto-Try + match subjects + declared-Result preservation |
| #235 | almide | json `\uXXXX` unicode escape decoding (was a stub pushing `?`) |
| #2   | almai  | tool_call_id / tool_calls native roundtrip |
| #3   | almai  | cli.almd multi-turn + `--output-format json` |
| #4   | almai  | groq provider |

Each was hit by writing real code, not synthetic tests. That's the dojo working as intended.

## Round 2: recursive dogfood — homullus improving homullus

After the basic dogfooding session above, the next test: **have homullus modify its own source**. The task was to add a `/version` slash command that prints `homullus v${VERSION}`.

### Input

```
> Read /Users/o6lvl4/workspace/github.com/almide/homullus/src/main.almd. Then use Edit
> to add a new slash command "/version" that prints the VERSION constant. Specifically,
> find the line `    "exit" => none` and replace it with two lines: the same `"exit" => none`
> followed by `    "version" => { io.print("homullus v${VERSION}\n") ; some(state) }`. Use a single Edit call.
```

### Tool calls homullus dispatched (via groq/llama-3.3-70b-versatile)

```
[Read] {"path":".../src/main.almd"}
[1jpbtfpa6] // homullus — Modular AI agent CLI runtime, written in Almide ...

[Edit] {
  "old_string":"    \"exit\" => none",
  "new_string":"    \"exit\" => none\n    \"version\" => { io.print(\"homullus v${VERSION}\\n\") ; some(state) }",
  "path":".../src/main.almd"
}
[508yw729d] edited .../src/main.almd
```

### Verification

```bash
$ almide check src/main.almd
No errors found

$ printf "/version\n/exit\n" | MODEL=cli/claude almide run src/main.almd
homullus v0.0.1
model: cli/claude    trust: bypass    /help for commands
> homullus v0.0.1
> Bye!
```

The new slash command compiled and worked on first try. homullus successfully edited its own source via its own tool runtime, driven by an open-weights LLM with no human typing the diff.

That's the milestone: **the agent built in Almide is good enough to extend itself, with feedback flowing back to the language and stdlib.** Each round of dogfood produces compiler PRs (8 so far) and homullus features (provider injection, scripted integration check, /version) — both improving in lockstep.
