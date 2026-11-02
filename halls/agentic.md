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

## Room: Long-running chat

```bash
llm-agent --chat --yolo --unrestricted --port 11434 --model qwen3-coder:30b
```

A persistent REPL: history carries across turns, tools stay live, and the model
remembers what it built three turns ago.

```
/exit /quit        leave
/reset             clear history (keeps the rules)
/whip [mins]       toggle the whip, optionally set the floor
/verify            toggle running tests before each answer
/cd <path>         change the working directory mid-session
/yolo              toggle arbitrary bash
/tokens            context usage
/save <file>       write the conversation out
/help
```

Every turn is logged to `/tmp/llm-agent-transcripts/<session>.chat.md`.

**Each chat turn is its own whip turn.** The start stamp is reset per turn, so the floor
measures *this* turn rather than the whole session — otherwise an hour-long chat trivially
satisfies a 10-minute floor and the whip stops meaning anything.

**Context is trimmed, not dropped.** Tool output is bulky and a long session will overflow
32k. `trim()` keeps the first message (which carries the rules) and discards the oldest
middle turns, never orphaning a `tool` result from its call. Without it the backend
truncates silently from the front — which removes the rules first and leaves you wondering
why the model stopped behaving. `/tokens` shows where you are.

## Room: Unlimited permissions, and the one thing still refused

`--yolo` lifts the bash allowlist. `--unrestricted` lifts the `--workdir` write
confinement. Together they give a local model your shell.

**That is a real risk and it is Aiden's to take** — it is his machine. But one list is
refused regardless:

```
rm -rf / | rm -rf ~ | sudo rm        mkfs | diskutil erase/reformat/partition
dd of=/dev/disk*                     > /dev/disk*
fork bombs                           shutdown | halt | reboot
killall -9 WindowServer/launchd      force-push to main/master
```

This is **not** a security boundary — a model with shell access can do anything, and a
determined one could trivially route around a regex. It stops the handful of commands
that end the *machine* rather than the session, which a 30B has no judgement about and
will emit if a task description nudges it that way.

Ordinary work is untouched: `rm -rf node_modules`, `rm -rf dist`, `dd if=a.img of=b.img`,
`git push origin main` all pass. 18/18 on the guard cases in `test-whip`.

If you genuinely need one of those commands, run it yourself.

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

## Room: A `system` message silently breaks tool calling on ollama

**Measured 2026-11-02. This one is nasty because it fails quietly.**

With ollama's `qwen3-coder` template, adding a `system` role to a tools request makes the
model stop emitting structured `tool_calls` and start emitting `<function=write_file>…`
as plain **text**. The template injects the tool definitions into the system slot, so a
user-supplied system message displaces them.

Identical request, system message the only difference:

| Request | `tool_calls` |
|---|---|
| user turn only | ✅ structured |
| **system + user** | ❌ text, no tool calls |
| system prompt folded into the user turn | ✅ structured |

The consequence is the dangerous part. The loop sees no `tool_calls`, treats the response
as a final answer, and the model reports *"I've created the hello.py file as requested"* —
with no file on disk. **Rules in the system prompt turn the agent into a liar.**

`llm-agent` therefore folds the system prompt into the first user turn automatically on
`:11434`, and `--no-system` forces the same on any backend. Both the rules and the tools
survive.

If you add a backend, test it both ways before trusting it.

## Room: Text-format tool calls are recovered, not trusted

The system-message fix above is necessary but **not sufficient**. Ollama's qwen3-coder
template also drops the tool definitions once there is `tool` role history — so a run
goes: step 1 structured `tool_calls`, step 2 `<function=write_file>` as plain text.

Same deadly consequence: the loop sees no tool calls, calls it a final answer, and the
model reports *"All files have been verified and are in place"* with an empty directory.

`llm-agent` therefore **parses the text form too** (`recover_text_tool_calls`). A tool
call the harness can see is a tool call that actually runs; anything the harness cannot
see is fiction. Recovered calls are rewritten into the message as structured
`tool_calls`, so the next prompt render carries a clean history.

The parser is narrow — it requires a literal `<function=NAME>` block, so prose describing
a function never trips it. Verified against the exact text qwen3-coder emitted, plus a
no-false-positive case.

**If you add a model, check this both ways.** Any model whose tool calls the harness
can't parse will confidently lie about work it never did.

## Room: Never ask — build

A local model handed a one-line brief will often come back with a questionnaire. Two
defences, both on by default:

1. **The system prompt forbids it.** "NEVER ask a clarifying question… choose the most
   reasonable interpretation, BUILD IT, and state the assumption in one line."
2. **`--whip` catches it anyway.** The hook's work-turn test is "did it edit files",
   which is correct for Claude Code but leaves a hole here: a run that asks a question
   and edits nothing sails through as conversational. So `llm-agent` adds one forced
   retry when a whipped run produced **no writes** *and* ended in a question. One strike,
   then it goes through — same shape as the hook, cannot deadlock.

The detector is a narrow phrase list ("could you provide", "what kind of", "let me know",
"do you want", "shall I"…) so a normal report containing a question mark does not trip
it. 7/7 on real phrasings.

Measured: `"build the thing"` went from a 5-bullet questionnaire and zero files to a
Flask app with tests, README, requirements and .gitignore.

## Room: What it still gets wrong

The whip makes a local model keep going. It does **not** make it correct.

In that same run it wrote `test_app.py` asserting the home page returns `'Hello, World!'`,
then — prompted by the whip to keep improving — changed the route to return
`'Hello, World! Welcome to the Flask web application.'` and never updated the test. It
also wrote `requirements.txt` without installing anything, so the test cannot even
import.

It then reported *"All files have been verified and are in place."* They were not
verified. Nothing was run.

**`--verify` closes the worst of this.** Before a run is allowed to finish, the project's
tests are detected (`pytest` if `test_*.py` / `tests/` exists, else `npm test` if
package.json defines it) and **actually run**. A failure goes back to the model as the
next task, with the real output pasted in and an explicit instruction: if you changed
behaviour after writing a test, the test is now wrong — update it; do not claim
verification you have not performed. `--verify-rounds` caps the retries (default 2).

```bash
llm-agent --whip --verify --port 11434 --model qwen3-coder:30b "…"
```

The gate runs **before** the whip, so a red test is always the next task whether or not
the run is whipped. A run that ends with failing tests is not done.

**What `--verify` still does not fix:**

- **No tests, no gate.** If the model writes no tests, there is nothing to run and the
  run passes through. `--verify` reports "no tests found" rather than pretending.
- **The model writes its own marking.** It can write a weak test that passes and call it
  verified. The gate proves the test runs green, not that the test is worth anything.
- **Clutter.** The whip rewards doing more, and "more" includes junk. A `wc2.py` run that
  passed produced `demo.py`, `verify_wc2.py` AND `final_verification.py` alongside the
  real files. Expect to delete things.

**Treat local agent output as a draft.** `--verify` means it compiles and its own tests
are green — not that it is correct.

## Room: Backends

| Port | Backend | Notes |
|---|---|---|
| 8080 | `llm serve` (llama-server) | pins the model in wired memory until killed |
| 11434 | `ollama serve` | **unloads idle models** — survives memory pressure far better; needs `--model` |

On a machine that is tight on RAM, prefer `:11434`. `llama-server` holding an 18 GB model
wired will lose to anything else that wants memory, and the only symptom is
`Compute error`.

## Room: Testing

`bin/test-whip` drives the real hook through `llm-agent`'s own code path with a synthetic
transcript, so the integration is testable without loading a model. Five cases: unarmed,
armed-under-floor, armed-over-floor, armed-but-no-edits, and the one-strike guarantee.

```
test-whip     # 5/5 expected
```

Run it after touching either `llm-agent` or the hook.
