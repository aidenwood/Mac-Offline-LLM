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

## Room: Measured, not assumed

Live on 2026-11-03, ~270ms per decision:

| command | regex | jev |
|---|---|---|
| `rm -rf "$BUILD_DIR"` | safe | **catastrophic** — unset variable makes it `rm -rf /` |
| `find / -name '*.gguf' -delete` | safe | **destructive** — would wipe the whole model store |
| `rm -rf node_modules` | safe | safe |
| `npm run build` | safe | safe |

Both escalations are cases a pattern genuinely cannot catch: one depends on a
variable's runtime value, the other on understanding what `find / -delete`
reaches. Neither is obscure, and both would have run.

finished / asked / partial / blocked, and small vs large model sizing, were
also all correct at confidence 1.00.

A known-bad command **short-circuits before the call** — the regex already
refuses it and nothing Jev returns can change that, so paying ~300ms and
tokens to discard the answer was waste. Pay for a decision only where it can
change the outcome.

## Room: Routing — one endpoint, three tiers

`bin/llm-router` is an OpenAI-compatible endpoint on :8099. Point anything at it
— Continue.dev, Aider, `llm-agent`, a cron job — and stop choosing a model by
hand.

```
  local-small  →  claude  →  local-large
  (ollama)        (claude -p)   (ollama, the floor)
```

**Classification routes UP; availability routes DOWN.** Those are different kinds
of question and must not share a mechanism. "Is this task big enough to need a
capable model?" is a judgement — jev answers it. "Is that tier reachable right
now?" is a fact about the world, and must never depend on a network call to
establish.

**Demote on failure, never on a guess.** Nothing in Claude Code exposes remaining
quota, so exhaustion is detected by a call failing and matching a marker in
`router-tiers.json`. One slow request is the honest price of having no signal to
read — predicting it would mean demoting while the good path still worked.

A demotion **expires** (30 min default). Permanent demotion would let one blip
cost the subscription path for the rest of the day.

### The billing guard

`claude -p` bills the **Claude Code subscription** — but only while
`ANTHROPIC_API_KEY` is absent from its environment. With the key set the call
still succeeds, so the mistake is invisible until the invoice. The router strips
`ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN` and `CLAUDE_API_KEY` before every
spawn, `bin/test-router` asserts the stripped environment explicitly, and CI
fails if a key is present at all. This is the economic premise of the whole
thing, so it is tested rather than commented.

### Local is the availability tier, not the speed tier

Measured 2026-11-03: `claude -p --model haiku` answered correctly in **3.6s**; a
local 1.5b took **13.8s** and got it wrong. Routing to local for speed makes
everything worse. Routing to local when the alternative is *nothing* is the point.
That gap closes as local models improve — nothing in the structure depends on it.

`--offline` forces local-only. It is a **switch, not an inference**: guessing that
you are offline is how a tool refuses to use a working connection.

### Preflight

Every tier is checked at startup and reported — a tier naming a model that is not
pulled is a config bug, and it should surface then rather than as a mysterious
demotion on the first real request. stdout is unbuffered so those warnings survive
being redirected to a log, which they did not at first.

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
jev status            # endpoint, model, key source, cache size
jev --selftest        # prove the fallbacks — forces the no-key path
```

The key lives at `~/.claude/jev/aidxn.token`, chmod 600, outside any repo —
the same shape as every other credential on this machine. `$JEV_API_KEY`
overrides it so CI can run without a file. Never commit it; `.gitignore`
blocks `*.token` and `apikey_*`.

**The endpoint is `POST https://api.typesafe.ai/v1/systemone`, model
`jev-latest`.** Third-party write-ups describe the OpenRouter route instead,
which wants an `sk-or-v1-` key and rejects a native `apikey_` one with 401.
The real paths came from `https://api.typesafe.ai/openapi.json` — it publishes
a full spec, so read that rather than trusting a blog.

Without a key, everything still runs. `jev status` says so plainly rather than
pretending, for the same reason `--whip` prints a banner instead of silently
doing nothing.
