# Mac-Offline-LLM — index

Local, offline LLM inference on Apple Silicon: one model store for the whole machine,
llama.cpp as the engine, and an agentic loop that can reuse Claude Code's real
Stop hook when one is installed.

## Halls — load on demand

| Hall | Load when |
|---|---|
| [halls/models.md](halls/models.md) | choosing a model, quantisation, what fits in RAM |
| [halls/serving.md](halls/serving.md) | running `llm`, ports, flags, llama.cpp behaviour |
| [halls/agentic.md](halls/agentic.md) | `llm-agent`, tool calling, Whip-It-Good integration |

## Hard rules for this project

### One model store. LM Studio and Ollama symlink into it.

`models/` (LM Studio) and `ollama/` (Ollama blobs) live here. If you symlink
`~/.lmstudio/models` and `~/.ollama/models` at them, there is exactly **one** copy of
every model on the machine instead of three. That symlinking is the operator's choice —
the installer does not reach into other apps' config.

**If this folder moves, re-point the symlinks and any PATH entry in the same breath**,
then run `bin/sync-gguf` — the `gguf/` farm holds absolute paths and every link in it
dangles after a move. All of them fail silently.

### Never fork the hook — and never let it fail silently.

`llm-agent` calls `whip-it-good.sh` directly (`$WHIP_HOOK`, default
`~/.claude/hooks/`). It does not reimplement the rule, copy the script, or keep its own
threshold logic — copying would create the exact two-definitions problem the hook exists
to prevent.

The hook is **optional**. When it is missing, `--whip` prints a boxed banner and the
final report says `WHIP WAS DISABLED`. An earlier version returned `None` quietly, so on
any machine without Claude Code the headline feature became decoration that warned
nobody. A disabled feature must never look identical to a working one.

### Don't rename anything inside `ollama/`.

Content-addressed blob store, Ollama owns it. `sync-gguf` reads the manifests to give each
blob a readable name in `gguf/` — that farm is the only place renaming happens.

### Check disk before pulling, not after.

Models are 1-65 GB and ollama assembles the download, so usage roughly doubles mid-pull.
`llm-recommend` reports free space on the volume ollama actually writes to, and
`llm install` refuses rather than failing halfway.

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
