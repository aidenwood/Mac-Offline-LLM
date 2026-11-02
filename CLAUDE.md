# Mac-Offline-LLM — index

Local, offline LLM inference on Apple Silicon: one model store for the whole machine,
llama.cpp as the engine, and an agentic loop that reuses Aiden's real Claude Code hooks.

## Halls — load on demand

| Hall | Load when |
|---|---|
| [halls/models.md](halls/models.md) | choosing a model, quantisation, what fits in RAM |
| [halls/serving.md](halls/serving.md) | running `llm`, ports, flags, llama.cpp behaviour |
| [halls/agentic.md](halls/agentic.md) | `llm-agent`, tool calling, Whip-It-Good integration |

## Hard rules for this project

### One model store. LM Studio and Ollama symlink into it.

`models/` (LM Studio) and `ollama/` (Ollama blobs) live here; `~/.lmstudio/models` and
`~/.ollama/models` are symlinks pointing at them. There is exactly **one** copy of every
model on the machine.

**If this folder moves, re-point both symlinks and `$LLM_STACK` in `~/.zshrc` in the same
breath**, then run `bin/sync-gguf` — the `gguf/` farm holds absolute paths and every link
in it dangles after a move. Four pointers, all of them silent when broken.

### Never fork the hook.

`llm-agent` calls `~/.claude/hooks/whip-it-good.sh` directly. It does not reimplement the
rule, does not copy the script, does not keep its own threshold logic. Sharpening the hook
sharpens both Claude Code and the local agent. Copying it would create the exact
two-definitions problem the hook exists to prevent.

### Don't rename anything inside `ollama/`.

Content-addressed blob store, Ollama owns it. `sync-gguf` reads the manifests to give each
blob a readable name in `gguf/` — that farm is the only place renaming happens.

### Disk, not RAM, is the live constraint.

The volume runs near full. A model pull competes with render trees elsewhere on the disk.
Check free space **before** pulling, not after.

### Close your sessions.

An `llm run` chat left open in a forgotten terminal tab holds its whole model in **wired**
Metal memory, which cannot be paged out. One idle 8GB chat is enough to make an 18GB model
fail to load with a bare `Compute error`. `/exit` when done — same reasoning as closing
browser tabs.

## Layout

```
Mac-Offline-LLM/
├── bin/           llm, llm-agent, sync-gguf, test-whip   ← tracked
├── halls/         the rules                              ← tracked
├── llama.cpp/     release binaries                       ← ignored, re-downloadable
├── gguf/          flat symlink farm                      ← ignored, regenerated
├── models/        LM Studio store                        ← ignored, ~39G
└── ollama/        Ollama blob store                      ← ignored, ~35G
```

Only `bin/` and the docs are tracked. Everything else is re-downloadable or regenerated,
and amounts to ~75G.
