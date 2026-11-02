# Hall — Models

## Room: The quantisation rule

**Quantise deeper to fit a bigger model. Never quantise deeper on the same model.**

A 30B at Q4_K_M beats a 14B at Q8 comfortably. But Q4_K_M is the floor worth using: Q3 and
below fall off a cliff **specifically on code**, where one mangled bracket breaks the
entire output. Prose degrades gracefully; code does not.

## Room: Prefer MoE

A Mixture-of-Experts model activates a fraction of its weights per token. `qwen3-coder:30b`
is 30B total, **3B active** — 30B knowledge at 3B speed, for 18 GB of storage.

On a memory-constrained machine that is strictly better than a dense model of the same
size. Storage is the cost you pay; speed is the thing you get back.

## Room: What fits

Weight ceiling on a 36 GB machine is roughly **24 GB**, once the OS, KV cache, and
whatever else is running are accounted for. In practice it is lower — see the wired-memory
room in `serving.md`.

| Model | ~Q4 | Verdict |
|---|---|---|
| `qwen3-coder:30b` | 18 G | **the agentic default** — MoE, structured tool calls |
| `gpt-oss:20b` | 12 G | tools + thinking, comfortable headroom |
| `qwen3.6:27b` | 16 G | newer generation, tools + thinking, dense |
| `qwen2.5-coder:32b` | 19 G | dense coder, no thinking overhead |
| `nomic-embed-text` | 262 M | embeddings |

Out of reach: GLM-5.3 — 754B, and even the "Flash" variant is 313B. Checked on Hugging
Face 2026-11-02, not assumed.

## Room: `-base` is not `-instruct`

A `*-base` tag is a raw next-token predictor with no instruction tuning. Asked a question
it will continue the text rather than answer it — which is why `qwen2.5-coder:1.5b-base`
rambles and looks broken. It is not broken; it is the wrong tag.

Use default/instruct tags for anything conversational or agentic.

## Room: Adding a model

```bash
ollama pull <model>     # or download a GGUF into models/
sync-gguf               # rebuild the flat gguf/ farm
llm ls
```

`sync-gguf` reads Ollama's manifests to give each content-addressed blob a readable
`ollama-<model>-<tag>.gguf` name. Re-run it after **any** pull, and after moving this
folder.

## Room: Researching a model — go primary

A web search for "best local LLM" returns a wall of SEO farms reciting the same press
release. Twelve sites repeating one vendor benchmark is **one** data point.

Check the vendor's own model card, the Hugging Face repo, or the Ollama library listing
for real parameter counts and sizes. Both findings above — GLM-5.3's true size and the
DeepSeek tool-calling failure — came from primary sources and direct measurement, and both
contradicted the blog consensus.

See `Aidxn-Context/halls/research-standards.md`.
