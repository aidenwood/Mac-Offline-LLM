# Mac-Offline-LLM

Local, offline LLM inference on Apple Silicon. One model store for the whole machine,
llama.cpp as the engine, and an agentic loop that reuses the same Claude Code hooks that
govern Aiden's real sessions.

```bash
llm ls                        every model, with size
llm run qwen3-coder           interactive chat
llm serve qwen3-coder         OpenAI API on localhost:8080/v1
llm-agent --whip "build X"    agentic loop, refuses to finish early
test-whip                     prove the hook wiring (5 cases)
```

## One model store

`~/.lmstudio/models` and `~/.ollama/models` are **symlinks into this project**, so LM
Studio, Ollama and llama.cpp all read the same files. Exactly one copy of each model on
disk — about 75 GB that used to be heading for three copies.

```
Mac-Offline-LLM/
├── bin/           llm, llm-agent, sync-gguf, test-whip   ← tracked
├── halls/         models / serving / agentic             ← tracked
├── llama.cpp/     release binaries, Metal                ← ignored
├── gguf/          flat symlink farm — what llama.cpp loads ← ignored
├── models/        LM Studio store (~39G)                 ← ignored
└── ollama/        Ollama blob store (~35G)               ← ignored
```

`bin/sync-gguf` rebuilds `gguf/` from both stores, reading Ollama's manifests to turn
content-addressed blobs into readable names. Run it after any model download.

## Whipping a local model

`whip-it-good.sh` is a Claude Code **Stop hook** — it fires when an agent tries to end a
turn. `llama-server` has no turns and no events, so the hook cannot attach to the model.
It attaches to the loop instead.

`llm-agent` calls the model repeatedly, running tool calls. When a response comes back
with **no tool calls**, the model is trying to finish — and that moment is handed to the
real hook:

```
no tool_calls → ~/.claude/hooks/whip-it-good.sh stop
                   blocked? → inject the reason, keep going
                   allowed? → print the answer, exit
```

**The hook is invoked, never copied.** No threshold logic, no block message, no second
definition of the rule lives in this repo. `--whip` arms it by passing the literal string
"whip it good" to the hook, so the same match that arms a Claude Code session arms this
one.

Local floor defaults to **10 minutes**, not Claude Code's 45 — a local 30B does less per
minute, so a long floor mostly buys loops of restating. `--whip-minutes` to tune.

## Not every model can be agentic

Tool calling must be **structured**. Measured 2026-11-02:

| Model | Structured tool calls |
|---|---|
| `qwen3-coder:30b` | ✅ proper `tool_calls`, correct arguments |
| `deepseek-coder-v2:16b` | ❌ emits template text, then **hallucinates the tool output** |

That second one is the trap: it reports "file written successfully" and no file exists.
Probe a model before trusting it in the loop — see `halls/agentic.md`.

**And a `system` message breaks it too.** On ollama's qwen3-coder template, adding a
`system` role to a tools request displaces the injected tool definitions and the model
falls back to emitting `<function=…>` as text — so putting your rules in the system prompt
silently turns the agent into a liar. `llm-agent` folds the system prompt into the first
user turn on `:11434` for exactly this reason. Measured, not assumed.

## Verified end to end

```
unwhipped →  hello.py                     model stops when the task is done
whipped   →  hello.py + test_hello.py     tried to stop, hook blocked it,
                                          it wrote the missing test and continued
```

The whipped run did what the hook's own text asks for — "name the next concrete task the
work implies and do it: the missing test". A local 30B obeying a Claude Code rule.

## Tool safety

Writes confined to `--workdir`. `bash` restricted to a read-only allowlist unless
`--yolo`. Deliberately tighter than Claude Code, because a local model has no judgement
about destructive commands and no human approves each one. Default closed.

## Gotchas that cost real time

- **Wired memory is the ceiling, not total RAM.** Metal allocations cannot be paged out.
  An `llm run` chat left open in a forgotten tab holds its whole model wired at RSS
  0.0 GB — invisible, and enough to make an 18 GB model die with a bare `Compute error`.
  `/exit` your chats.
- **If this folder moves, four pointers break silently**: both store symlinks, `$LLM_STACK`
  in `~/.zshrc`, and every absolute link in `gguf/`. Re-point them and re-run `sync-gguf`
  in the same breath.
- **`llama-cli` is chat-only** in b12034; one-shot generation is `llama-completion`.
- **`-base` tags are not instruction-tuned** and will ramble at a chat prompt.

## Requirements

Apple Silicon, macOS. llama.cpp b12034+ (Metal). Python 3 stdlib only — no dependencies.
