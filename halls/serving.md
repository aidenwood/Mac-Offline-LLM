# Hall — Serving

## Room: Commands

```bash
llm ls                        every GGUF with size
llm run <match>               interactive chat
llm ask <match> "prompt"      one-shot completion
llm serve <match>             OpenAI API on :8080  (LLM_PORT to change)
llm bench <match>             tok/s
llm <anything-else> …         passes through to llama-<anything-else>
```

`<match>` is a case-insensitive substring of a filename in `gguf/`. It **errors rather
than guessing** if more than one matches — an ambiguous model choice is a bug, not a
coin toss.

## Room: Two servers at once

```bash
llm serve qwen3-coder                 # :8080
LLM_PORT=8081 llm serve gpt-oss       # :8081
```

Each holds its full model in memory. Two 8 GB models is fine on 36 GB; an 18 GB plus
anything substantial is not.

## Room: Wired memory is the real ceiling

Metal allocations are **wired** — they cannot be paged out or compressed. So the usual
"macOS will handle it" intuition does not apply.

The failure mode is opaque. You get:

```
E llama_init_from_model: ... Compute error.  ret = -3
E srv  update_slots: decode() failed: Compute error.
```

No mention of memory anywhere. `ret = -3` is an allocation failure.

**Diagnose it:**

```bash
vm_stat | awk '/Pages wired/{printf "wired %.1f GB\n", $4*16384/1073741824}'
ps -Ao pid,etime,rss,args | grep llama- | grep -v grep
```

*2026-11-02:* an `llm run` chat left open in a forgotten terminal tab for 1h46m was
holding its model wired at RSS 0.0 GB — invisible to a casual look at memory usage, and
enough to make an 18 GB model fail to load. **`/exit` your chats.**

Lowering `-c` does not rescue a genuinely over-committed machine. Free the memory.

## Room: llama.cpp b12034 behaviour

- `llama-cli` is **chat-only**. One-shot generation moved to `llama-completion`;
  `-no-cnv` no longer exists on `llama-cli`. `llm ask` calls the right binary.
- MLX `.safetensors` models are LM Studio-only. llama.cpp cannot load them, so they never
  appear in `gguf/`.
- `dspark-DeepSeek-V4-Flash` will not load: the `dflash` attention variant requires a
  `ctx_other` parameter this build does not expose. Run it in LM Studio.
- The server warns that no API key is set and CORS allows all origins. Fine on localhost;
  do not expose the port.

## Room: Pointing tools at it

`llm serve` speaks the OpenAI schema, so anything that takes a base URL works —
Continue.dev, Aider, any SDK — with `http://localhost:8080/v1` and any dummy API key.
