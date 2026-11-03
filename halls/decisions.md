# Hall — Decisions

Where a typed decision model earns its place, and where it does not.

## Room: What Jev is for here

Several decisions in this repo were made by a regex standing in for judgement.
A regex cannot tell `rm -rf ./build` from `rm -rf "$BUILD_DIR"` when the
variable is unset. A full LLM call can, but costs seconds at a point in the
loop that must be fast, and a round-trip to a chat model to answer a yes/no is
the wrong shape of tool.

[Jev](https://jevai.dev) is TypeSafe's "System One" model: you enumerate the
possible answers, send it state, and get a typed answer back in roughly 150ms.
Input is priced per million tokens and output is free. It is not a chat model
and must never be used as one — ask it closed questions with named options.

`bin/jev` wraps it. Three primitives:

| | Returns | Use for |
|---|---|---|
| `choice` | one key from your enum, plus probabilities and confidence | classification |
| `noul` | 0–1 probability a statement is true | yes/no with a confidence dial |
| `score` | a number | ranking, severity |

## Room: Offline is the point, so every decision has a fallback

This project exists so you can work with no network. A hard dependency on a
hosted model would betray that, so:

- every call has a deterministic fallback — the rule that was there before
- the fallback runs when there is no key, no network, or a timeout
- the result always reports `used`: `jev`, `cache`, `no-key`, or the failure
- **a Jev outage never changes what the tool is willing to do.** It may only
  make the judgement coarser.

`jev --selftest` proves the fallbacks with the key deliberately unset. Run it
after touching any decision.

## Room: The decisions wired up

**Shell safety** (`llm-agent`, `judge_command`). The regex runs first and can
only ESCALATE. Jev may catch something the pattern missed; it can never
de-escalate a pattern match, because a known-bad command must not be argued out
of a refusal by a probabilistic model. Only consulted under `--yolo`, since
that is the only mode where arbitrary shell is permitted at all.

**Did it finish or give up** (`judge_finished`). The whip infers this from
"edited files AND under the time floor", which genuinely cannot distinguish
*finished early because the job was small* from *gave up and asked a question*.
That is why it misfires. Four options — finished / asked / partial / blocked —
is the decision the heuristic was always trying to approximate.

**Model sizing** (`judge_model_size`). Loading an 18GB MoE to rename a variable
costs ~20 seconds and most of the RAM on a 36GB machine. When unsure it answers
`large`: a wrong-small wastes an entire run, a wrong-large only wastes memory.

## Room: Where NOT to use it

Do not route a decision through a model when a deterministic answer exists.
These stay exactly as they are:

- **Tool-call recovery** — parsing `<function=…>` out of text is a parser, not
  a judgement. It has one right answer.
- **The SEO integrity checks** — "does this schema match the rendered copy" is
  string equality. A probabilistic answer would be strictly worse.
- **The sizing arithmetic** — RAM, disk and KV cache are measured, not judged.
  Jev picks *which model class*; the budget formula decides *what fits*.
- **`CATASTROPHIC` itself** — keep the regex. It is the floor that holds when
  the network is down, and it is what makes consulting Jev safe at all.

The test: *does this question have a correct answer I can compute?* If yes, it
is not a decision, it is a calculation. Jev is for the grey.

## Room: Preloading

Decisions are deterministic for a given (state, question), so `bin/jev` caches
them on disk by hash. That turns a repeated decision into a disk read, and it
means a queue of work can be judged **before** any of it runs — warm the cache
while the model is still loading, and the decisions are free at the point they
are needed.

`JEV_CACHE` moves the directory. Cache writes never raise: a caching failure
must not break a decision.

## Room: Setup

```bash
export OPENROUTER_API_KEY=...     # Jev is served via OpenRouter
jev status                        # key present? cache size?
jev --selftest                    # prove the fallbacks, no key needed
```

Without a key, everything still runs. `jev status` says so plainly rather than
pretending, for the same reason `--whip` prints a banner instead of silently
doing nothing.
