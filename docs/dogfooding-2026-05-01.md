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

## Round 3: streaming + tool calls in one session

Added live token streaming for OpenAI-shaped providers (PRs #236 in almide, #5 in almai). homullus's `streaming_provider` routes openai / openrouter / groq through `almai.call_streaming`, which delegates to the new `http.openai_streaming_call` intrinsic — Rust owns the SSE parsing + JSON accumulation + delta dispatch.

### Live counting (streaming-only, no tool dispatch)

```
$ MODEL=groq/llama-3.3-70b-versatile homullus

> count from 1 to 8 one number per line

1
2
3
4
5
6
7
8
```

Each digit and newline arrived as a separate SSE `data:` event. The numbers materialize one-by-one in the terminal rather than appearing as a block, exactly the way Claude Code feels.

### Streaming + multi-round tool dispatch + error recovery

```
> ls .almd files in src using Bash, just the names

[Bash] {"command":"ls /abs/src/*.almd | sed 's/.*///'"}
[ERROR: sed: 1: "s/.*///": bad flag in substitute command: '/']
[Bash] {"command":"find /abs/src/ -name '*.almd' -print | sed 's/.*///'"}
[ERROR: sed: 1: "s/.*///": bad flag in substitute command: '/']
[Bash] {"command":"find /abs/src/ -name '*.almd' | sed 's/.*///'"}
[ERROR: sed: 1: "s/.*///": bad flag in substitute command: '/']
[Bash] {"command":"ls /abs/src/*.almd | awk -F/ '{print $NF}'"}
[agent_check.almd
agent.almd
main.almd
permission.almd
smoke.almd
tools.almd]
The .almd files in the src directory are: agent_check.almd, agent.almd, main.almd, permission.almd, smoke.almd, tools.almd
```

Four tool rounds, three failures (sed delimiter conflict because the substitution pattern itself contained `/`), then recovery via `awk -F/` — driven entirely by the LLM with each Bash result fed back as `role:"tool"` and the agent loop continuing. The final summary streamed token-by-token. All in one REPL turn.

That's structurally the same shape as a Claude Code session: streaming text, persistent multi-round tool dispatch, error-aware recovery. The only thing left to make the UX visually identical is text-rendering polish (the ANSI escape sequences in our log are unrendered because we piped to a non-TTY).

## Round 4: streaming + recursive dogfood — homullus updates `/clear` to also reset usage

After token-usage display landed in v0.1.0+1, `/clear` only wiped history and left the running token counter alone. Asked homullus (via groq + streaming) to fix it.

### Input

```
> Read /Users/o6lvl4/workspace/github.com/almide/homullus/src/main.almd. Find the
> line that says exactly `      some({ ...state, history: [] })` (with 6 spaces of
> indent). Use the Edit tool to replace it with
> `      some({ ...state, history: [], usage: agent.empty_usage() })` so that
> /clear also resets the token usage counter. Use absolute path. Keep old_string
> and new_string both as a single line each.
```

### Tool calls homullus dispatched

```
[Edit] {
  "old_string":"      some({ ...state, history: [] })",
  "new_string":"      some({ ...state, history: [], usage: agent.empty_usage() })",
  "path":".../src/main.almd"
}
[edited]

The line that says exactly `...history: [] })` has been replaced with `...usage: agent.empty_usage() })`.

[in:1866 out:129 total:1995]
```

Final assistant text streamed live (the `[in:... out:... total:...]` line below the prompt is the running token counter — the 1.9k tokens are reasonable for the file payload + tool result). The Edit was first-try correct.

### Verification

```
$ almide check src/main.almd
No errors found

$ printf "say hi\n/clear\nsay bye\n/exit\n" | homullus
> Hi
[in:791 out:2 total:793]
> Conversation cleared.
> bye
[in:791 out:2 total:793]   ← still 793, not ~1586 — usage WAS reset
> Bye!
```

Cumulative tokens after the second turn equal exactly the second-turn cost (793), proving `/clear` now zeros usage. **Streaming + recursive self-modification + immediate verification, all in one REPL session.** The same shape Claude Code achieves; demonstrated entirely on Almide.
