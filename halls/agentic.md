# Hall — Agentic

`llm-agent` is the loop that turns a stateless completion endpoint into something that
can be governed.

## Room: Why the hook can't be applied directly

`whip-it-good.sh` is a Claude Code **Stop hook**. It fires on an event the harness emits
when the agent tries to end a turn.

`llama-server` has no hooks, no events, and no concept of a turn. It is a stateless
request/response endpoint: you post messages, you get a completion. There is no moment
for a hook to intercept, because nothing ever "decides to stop" — the request simply
returns.

So the hook cannot be attached to the model. **It has to be attached to a loop that has
turns.** That loop is `llm-agent`.

## Room: How the integration works

`llm-agent` keeps calling the model, executing tool calls and feeding results back. When a
response comes back with **no tool calls**, the model is trying to finish. That is the
Stop moment, and that is where the hook is invoked:

```
model returns no tool_calls
        ↓
bash ~/.claude/hooks/whip-it-good.sh stop   ← the real hook, not a copy
   stdin: {"session_id": …, "transcript_path": …}
        ↓
   {"decision":"block","reason":"…"}        → inject reason as a user turn, loop again
   (no output)                              → print the answer, exit
```

Two details make it work against the unmodified hook:

- **Arming.** The hook arms itself by string-matching the prompt for "whip it good". So
  `--whip` literally passes that string to the `start` event. Same trigger, same code
  path, no special case.
- **The transcript.** The hook decides "was this a work turn?" by counting `Edit`/`Write`
  `tool_use` blocks in a JSONL transcript newer than the turn's start stamp. `llm-agent`
  writes that exact shape to `/tmp/llm-agent-transcripts/<session>.jsonl` whenever a write
  tool runs. A run that only read files is a chat turn and is never blocked — same
  semantics as Claude Code.

## Room: Never fork the hook

`llm-agent` contains **no** copy of the rule, no threshold logic, no block message. All of
that lives in the one hook file. If the heuristics get sharpened there, the local agent
inherits it for free.

A reimplementation would be a second definition that drifts — see
`Aidxn-Context/halls/code-standards.md`. The whole reason the hook exists is that a rule
stated in two places stops being a rule.

## Room: Usage

```bash
llm serve qwen3-coder                      # terminal 1
llm-agent "refactor utils.py" --rules code # terminal 2

llm-agent --whip "build the thing"                   # 10 min floor (local default)
llm-agent --whip --whip-minutes 45 "build the thing" # Claude Code's floor
llm-agent --rules code,comms "…"                     # load halls as system prompt
llm-agent --workdir ~/project --port 8081 "…"
```

The local floor defaults to **10 minutes, not 45**. A local 30B does far less per minute
than Claude Code with a full toolchain, so a 45-minute floor on a local run mostly buys
loops of restating. Tune with `--whip-minutes`.

Disable for one run even when armed: `touch /tmp/.whip-it-good-off` — same escape hatch as
the real hook.

## Room: Tool safety

Writes are confined to `--workdir`. `bash` is restricted to a read-only allowlist (`ls`,
`cat`, `grep`, `find`, `git status|log|diff`, `npm run build|test|lint`, `pytest`, …)
unless `--yolo` is passed.

This is deliberately tighter than Claude Code's permission system, because a local model
has no judgement about destructive commands and there is no human in the loop approving
each one. **Default closed.**

## Room: Not every model can do this

Tool calling must be **structured** — the model has to emit OpenAI-format `tool_calls`
that the server parses. Many models instead emit tool calls as *text* in their own
template syntax, which the harness cannot see.

Measured 2026-11-02:

| Model | Structured tool calls |
|---|---|
| `qwen3-coder:30b` | ✅ emits proper `tool_calls` with correct arguments |
| `deepseek-coder-v2:16b` | ❌ emits `<｜tool▁calls▁begin｜>` text, then **hallucinates the tool output** and claims success |

That second failure mode is the dangerous one: the model reports "file written
successfully" and no file exists. An agentic harness over a model that can't really call
tools produces confident fiction.

**Check before trusting a model in the loop:**

```bash
curl -s localhost:8080/v1/chat/completions -H 'Content-Type: application/json' -d '{
 "messages":[{"role":"user","content":"list /tmp"}],
 "tools":[{"type":"function","function":{"name":"list_dir","description":"List a directory",
   "parameters":{"type":"object","properties":{"path":{"type":"string"}},"required":["path"]}}}],
 "max_tokens":100}' | python3 -m json.tool
```

A real `tool_calls` array in the response means the model is usable here. Prose means it
is not.

## Room: Testing

`bin/test-whip` drives the real hook through `llm-agent`'s own code path with a synthetic
transcript, so the integration is testable without loading a model. Five cases: unarmed,
armed-under-floor, armed-over-floor, armed-but-no-edits, and the one-strike guarantee.

```
test-whip     # 5/5 expected
```

Run it after touching either `llm-agent` or the hook.
