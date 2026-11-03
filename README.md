# Mac-Offline-LLM

[![ci](https://github.com/aidenwood/Mac-Offline-LLM/actions/workflows/ci.yml/badge.svg)](https://github.com/aidenwood/Mac-Offline-LLM/actions/workflows/ci.yml)

Run a capable coding LLM entirely offline on an Apple-Silicon Mac. One command
detects your hardware, picks a model that will actually fit, installs what's
missing, and gives you an agentic coding loop with real tool use.

No API keys. No network after setup. No per-token cost.

```bash
git clone https://github.com/aidenwood/Mac-Offline-LLM.git
cd Mac-Offline-LLM
bin/llm-install
```

Then:

```bash
llm chat                      long-running session, full shell, writes anywhere
llm agent "build X"           one-shot agentic run
llm status                    what's running and holding memory
llm down                      free the RAM when you're done
```

**Requires** an Apple-Silicon Mac (M1 or later). Intel is detected and declined —
no unified memory, no usable Metal backend. Homebrew is the only prerequisite.

---

## It tells you what fits, and admits what it hasn't tested

```
$ bin/llm-recommend

  machine   Apple M4 Max  ·  36.0 GB RAM  ·  macOS 26.5
  memory    18.9 GB reclaimable right now  (IDE/browser open)
  disk      14.9 GB free where ollama writes
  budget    19.2 GB max model size for this box

  → qwen3-coder:30b
    18.0 GB on disk, ~21.0 GB resident   [measured on this hardware]
    fallback: gpt-oss:20b
```

RAM is the gate, not GPU cores. The budget comes from real arithmetic rather than
a vibe:

```
usable        = total_ram - 4 (OS) - (4 if IDE/browser open else 2)
safety_margin = 0.15 × total_ram        # wired memory, Metal fragmentation
kv_cache      ≈ 0.5 GB per 8K context
max_on_disk   = (usable - safety_margin - kv_cache) / 1.15
```

Every recommendation in `models.json` is tagged `measured` or `extrapolated`, and
the output says which. Only the 36 GB tier has been tested on real hardware so
far — the rest are interpolated and labelled as such. A tool that asserts
untested recommendations as fact isn't worth trusting.

**Quantise deeper to fit a bigger model; never deeper on the same model.** A 30B
at Q4_K_M beats a 14B at Q8. Q4_K_M is the floor — below it, code specifically
falls apart, because one mangled bracket breaks an entire file where prose would
just read slightly worse.

**Prefer MoE.** `qwen3-coder:30b` activates 3B parameters per token: 30B knowledge
at 3B speed, for 18 GB of storage.

---

## Not every model can really call tools

This is the single most important check, and it's why `llm-install` runs a probe
before declaring success.

A model that *looks* like it's calling tools but is actually emitting text will
tell you **"I've created the file as requested"** when no file exists. Confident
fiction is worse than an error, because you only find out later.

```
$ bin/llm-probe-tools qwen3-coder:30b
    ✓ plain                  structured tool_calls
    ✗ with system message    emitted a tool call as TEXT (harness cannot see it)
    ✓ after tool history     structured tool_calls
```

Three variants, because the naive single-turn probe passes models that fail in a
real loop. `deepseek-coder-v2:16b` is rejected outright by ollama —
`does not support tools` — and through llama.cpp it hallucinates tool output.

The system-message failure is a genuine trap: **putting your rules in the system
prompt is what turns the agent into a liar**, because ollama's template injects
the tool definitions into that same slot and your message displaces them.
`llm-agent` folds the system prompt into the first user turn to avoid it, and
parses text-format tool calls back out when a template misbehaves anyway.

---

## The agentic loop

`llm-agent` keeps calling the model and running its tool calls until the work is
done. Tools: `list_dir`, `read_file`, `write_file`, `edit_file`, `bash`.

```bash
llm agent "add a README documenting each script"
llm-agent --verify "fix the failing test"      # runs the tests before finishing
llm-agent --chat --yolo --unrestricted         # what `llm chat` expands to
```

**`--verify`** runs the project's tests (`pytest` / `npm test`) before a run is
allowed to finish, and sends failures back as the next task. It exists because a
local model will cheerfully write a test, then change the code under it, then
report that everything is verified without having run anything.

**Safety.** Writes are confined to `--workdir` unless `--unrestricted`. `bash` is
limited to a read-only allowlist unless `--yolo`. A short list of
machine-ending commands — disk wipes, fork bombs, `shutdown`, force-pushing to
main — is refused even under `--yolo`. That's not a security boundary (a model
with your shell can route around a regex) — it stops the handful of commands a
local model has no judgement about.

### Optional: Whip-It-Good

If you have Claude Code installed, `--whip` hands the "model is trying to finish"
moment to its real `whip-it-good.sh` Stop hook, which refuses to let a work run
end early. The hook is **invoked, never copied**, so one file governs both.

Without it, `--whip` prints a boxed warning and the final report says
`WHIP WAS DISABLED`. It is never silently skipped — an earlier version returned
quietly and the feature became decoration that nobody noticed.

Set `$WHIP_HOOK` if yours lives elsewhere. Everything else works without it.

### Optional: your own rules

`--rules code,comms` loads markdown from `$LLM_AGENT_CONTEXT/halls/<name>.md`
into the system prompt. Unset by default; `--rules` without it is a hard error
rather than a silent no-op.

---

## Faster decisions, not another chat model

Several things here were decided by a regex standing in for judgement. A regex
cannot tell `rm -rf ./build` from `rm -rf "$BUILD_DIR"` when the variable is
unset. A chat model can — at the cost of seconds, at a point in the loop that
has to be fast.

[Jev](https://jevai.dev) sits in between: you enumerate the answers, it returns
a typed one in ~150ms. `bin/jev` wires it into the three decisions that needed
judgement — shell safety, did-it-finish-or-give-up, and which model size a task
actually needs.

```bash
export OPENROUTER_API_KEY=...
llm jev status
llm jev --selftest       # proves the fallbacks, no key needed
```

**It is optional and the fallbacks are the point.** This is an offline tool, so
every decision keeps the deterministic rule that was there before; it runs when
there is no key, no network, or a timeout, and the result always reports which
one answered. A Jev outage can make the judgement coarser — it can never change
what the tool is willing to do. The regex also runs first on shell commands and
can only be escalated, never overruled.

Detail, including where NOT to use it: [halls/decisions.md](halls/decisions.md).

## One model store

`bin/sync-gguf` builds a flat `gguf/` symlink farm from both the LM Studio and
ollama stores, reading ollama's manifests to give content-addressed blobs
readable names. If you point `~/.lmstudio/models` and `~/.ollama/models` at this
project's `models/` and `ollama/` directories, every tool reads the same files
and you stop keeping three copies of a 20 GB model.

```bash
llm link --status     # what's linked now
llm link --dry-run    # the plan, no changes
llm link              # do it
llm link --undo       # reverse it exactly
```

Kept out of `llm install` on purpose, because it touches other applications' data
directories. Nothing is deleted — an existing store is **moved** into this project
and symlinked back, so both apps keep working, and a cross-volume move is refused
rather than silently copying 75 GB.

---

## Gotchas that cost real time

- **Wired memory is the ceiling, not total RAM.** Metal allocations can't be
  paged out. A forgotten `llm run` chat in another tab holds its whole model
  wired at RSS 0.0 GB — invisible to a casual look — and is enough to make an
  18 GB model die with a bare `Compute error, ret = -3` that mentions memory
  nowhere. `/exit` your chats, or `llm down`.
- **ollama beats a pinned llama-server on a tight machine**, because it unloads
  idle models.
- **`llama-cli` is chat-only** in recent builds; one-shot generation moved to
  `llama-completion`. `llm ask` calls the right one.
- **A `-base` tag is not instruction-tuned** and will ramble at a chat prompt.
- **Don't rename anything inside `ollama/`** — content-addressed, ollama owns it.

---

## Commands

```
llm install [--dry-run] [--model TAG] [--yes] [--uninstall]
llm link [--status] [--dry-run] [--undo]
llm recommend / llm probe <model>
llm up [tag] / llm down / llm status
llm chat / llm agent "task"
llm ls / llm run / llm ask / llm serve / llm bench
llm-recommend [--json] [--quiet]
llm-probe-tools <model> [--json]
sync-gguf
test-whip
```

`llm install --dry-run` decides everything and changes nothing. `--uninstall`
removes exactly the PATH wiring it added and leaves your models alone.

---

## Tests

```
bin/test-whip        13 cases: whip, verify gate, context trimming
bin/test-recommend   30 cases: budget formula, tier selection, honesty
bin/jev --selftest    6 cases: every decision falls back with no key
```

`test-recommend` exercises RAM tiers this machine is not — 8 GB through 128 GB —
because the sizing logic decides what a stranger downloads, and getting it wrong
is invisible from the author's own box. It caught exactly that: an 8 GB Mac was
being recommended a model that does not fit, with no warning.

Whip cases SKIP (not FAIL) when the optional hook isn't installed. All Python
targets **3.9.6** — stock macOS ships that, and PATH order decides whether you
get it or a newer brew build.

---

## Licence

MIT. Built on [ollama](https://ollama.com) and
[llama.cpp](https://github.com/ggml-org/llama.cpp), which do the actual hard part.
