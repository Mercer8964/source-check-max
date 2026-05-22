# source-checkm

> **A dual-verifier citation check for high-stakes AI output.**
> Spawns two independent verifiers per citation — one fetches the URL the agent gave, the other rediscovers the URL from scratch using only author/year/title. Cross-compared via a fixed 8-state table. Catches fabricated papers and wrong-URL citations that a single verifier misses.

Use it before you ship anything where a fabricated citation has material cost — **legal briefs, medical claims, financial reports, regulatory submissions, academic writing**.

For everyday work where a single web check is enough, use a regular source-check verifier instead.

---

## The problem

LLMs hallucinate citations at depressingly stable rates. Across 10 commercially deployed models over 69,557 citation instances, fabrication ran **11.4–56.8%** (arXiv:2603.03299). And it's worse than it looks — the fabricated citations *pattern* like real ones, so a single-verifier check often confirms them by fetching whatever URL the drafter handed over.

The failure modes that cost money:

1. **Wrong-URL citations.** Claim is real, but the agent paired it with the wrong URL. A single verifier fetches the URL, sees the content doesn't match, says "citation failed" — but doesn't tell you where the real source is.
2. **Fabricated papers.** Paper title sounds plausible, authors sound plausible, year is recent. A single verifier fetches whatever the agent gave (404 or unrelated page), says "citation failed" — but the user can't tell if the URL is broken or if the paper itself doesn't exist.

source-checkm separates these.

---

## How it works (30-second mental model)

For every factual citation, run **two verifiers in parallel**:

| | What it gets | What it does |
|---|---|---|
| **V1** | The full URL the agent cited | Fetches it. Substring-matches the claim against the page body. Standard URL-given verification. |
| **V2** | **Only** the author, year, and title — **no URL** | Has to find the source itself. Searches arXiv, Google Scholar, Semantic Scholar, CourtListener, CrossRef, the open web. Picks the most plausible candidate. Then runs a **mandatory metadata gate**: first author match? year ±0? title overlap? If any of the three fails, V2 stops and returns `URL_DISCOVERY_UNCERTAIN` — it refuses to "verify" content on a similar-but-wrong paper. |

Then the main agent applies a fixed table to combine V1 and V2:

```
✅ STRONG_VERIFIED       both pass, same paper      → confident green
⚠️ CONFLICT_REVIEW      one passes, one fails       → wrong URL, V2 has the real one
🚨 LIKELY_FABRICATED    V1 fails + V2 can't find it → paper doesn't exist; remove it
🚨 LIKELY_WRONG         both contradict             → claim is wrong even with real URL
⚠️ WEAK_VERIFIED        V1 ok but V2 metadata fails → metadata is iffy (year/author off)
🔵 UNVERIFIABLE         both insufficient evidence  → genuinely hard to check
```

The headline is **mechanical** — derived from the table, never overridden by the main agent. `LIKELY_FABRICATED` and `LIKELY_WRONG` force claim removal.

---

## Install (60 seconds)

Pick your platform — copy the folder, done.

**Claude Code**
```bash
git clone https://github.com/guoyurui138-hue/source-checkm.git
cp -r source-checkm/claude-code/source-checkm ~/.claude/skills/
# Restart Claude Code. Invoke with /source-checkm, or it auto-triggers on
# citation claims in high-stakes contexts.
```

**Codex**
```bash
cp -r source-checkm/codex/source-checkm ~/.agents/skills/
```

**OpenClaw**
```bash
cp -r source-checkm/openclaw/source-checkm ~/.openclaw/skills/
```

No config needed. The skill is self-contained — V1 and V2 prompts are inlined.

---

## When to use it (and when not to)

**Use it when:**
- You're shipping a legal brief, medical report, financial filing, regulatory submission, or academic paper
- A fabricated citation would cost money, reputation, or compliance
- You can spend ~2× the verification cost of a single-verifier check

**Don't bother when:**
- You're writing a blog post and a citation being slightly off doesn't matter — use a single verifier
- The citation is illustrative ("e.g., similar to how Vaswani et al. did attention") — single verifier is fine
- The error mode is *not* citation fabrication — e.g., the agent dropped a "but only in human trials" qualifier when summarizing a paper. That's a different failure mode (paraphrase distortion); use an audit-loop-style skill instead.

---

## What you get back

After source-checkm runs, every citation has:

- A single-word headline (the 8-state label above)
- The V1 verifier's evidence quote + URL
- The V2 verifier's evidence quote + URL (which may differ — that's the whole point)
- A combined recommendation: keep / fix-URL / remove

Example output for a wrong-URL case:

```
[B1] "The Kimi K2 paper reports pre-training on 15.5 trillion tokens."
⚠️ CONFLICT_REVIEW
V1: fetched arxiv:2508.06471 (GLM-4.5 paper) — no match for K2 claim
V2: independently found arxiv:2507.20534 — Kimi K2 Technical Report,
    quote: "We pre-train Kimi K2 on 15.5 trillion tokens of high-quality data"
→ The CLAIM is correct, but the URL was wrong. Use V2's URL.
```

Example output for a fabricated citation:

```
[C1] "Hashimoto, Patel, Choi (NeurIPS 2025) Latent Trajectory Distillation, FID 3.2"
🚨 LIKELY_FABRICATED
V1: arxiv:2511.04823 — unrelated paper on Steiner Triple Systems
V2: 5 independent search strategies, no candidate matches metadata
    (author/year/title all fail the gate)
→ REMOVE this claim. Paper does not exist.
```

---

## Does it actually work? (n=13, 7/13 advantage)

Validated against a 13-case test set with known ground truth (2 real-and-correct, 3 real-claim-wrong-URL, 6 fabricated, 2 real-paper-wrong-claim).

| Comparison | Actionable advantage of source-checkm over single verifier |
|---|---|
| v8 — vs older single-verifier source-check | **8/13** |
| v9 — vs current source-check (with negative-control + cross-corroboration already absorbed) | **7/13** |

The 7 cases where source-checkm wins:
- **3/3 wrong-URL cases** — V2 finds the correct URL and surfaces it (V1 alone only tells you "verification failed")
- **4/6 fully-fabricated citations** — metadata gate fails on author/year/title; user gets a clear `LIKELY_FABRICATED` signal instead of an ambiguous "verification failed"

The 6 cases where the single verifier is enough:
- 2/2 real-and-correct (both verifiers agree — `STRONG_VERIFIED` is just nicer wording for `VERIFIED`)
- 2/2 real-paper-wrong-claim (single verifier catches the wrong number on its own)
- 2/6 borderline fabricated cases (real paper with wrong authors, or wrong year — single verifier reports the specific mismatch already)

Full per-case results, methodology, and reasoning in **[EVIDENCE.md](./EVIDENCE.md)**.

---

## Limits (honest)

- **n=13 is a small test set.** The pattern is consistent (7/13 in v9, 8/13 in v8), but variance across larger samples is unknown. Treat the number as directional, not a confidence interval.
- **Same-model-family V1 and V2 share biases.** Cross-family pinning (V1 → Anthropic, V2 → OpenAI/DeepSeek) is recommended for highest-stakes use but not enforced. Preference Leakage (ICLR 2026) reports same-family 28–37% correlated false positives vs ±1.5% cross-family — meaningful but unmeasured in our test.
- **The `LIKELY_FABRICATED` label is loose when the paper is real but cited with wrong authors** (e.g., "Attention Is All You Need" attributed to Smith/Jones/Williams). The verdict — that the citation as written cannot be verified — is correct, but the label is more alarmist than it should be. A future version may add a `LIKELY_MISATTRIBUTED` state.
- **If the main agent didn't give a URL**, V1 must search the web itself, and its behavior converges toward V2. The advantage of having two verifiers shrinks.
- **source-checkm catches fabrication and wrong-URL only.** It does **not** catch subtle paraphrase distortions — cases where the source mostly matches the claim, but a key qualifier was silently dropped. That's a different failure mode and needs an audit-loop-style skill.

---

## Why the design works (one-paragraph version)

A single verifier that gets the URL from the drafter is *correlated* with the drafter: they share the same anchor. If the drafter fabricated the URL, the verifier just fetches the fabricated URL. The cure isn't more verifiers in series — it's a verifier that **never sees the drafter's URL**. V2 is forced to reconstruct the source path from scratch using only metadata. If the metadata is real, V2 converges to the same paper independently (`STRONG_VERIFIED`). If the metadata is fabricated, V2 fails the gate and signals that fact (`LIKELY_FABRICATED`). The combination is what produces the 7/13 advantage — neither verifier alone matches it.

The metadata gate (author/year/title) is what prevents the most dangerous failure mode: V2 finding a similar-but-different paper and "verifying" the claim against the wrong source. Without the gate, V2 would *create* the very fabrication-of-verification problem it exists to catch.

---

## Files in this repo

- `claude-code/source-checkm/SKILL.md` — Claude Code variant (uses `Agent` tool)
- `codex/source-checkm/SKILL.md` — Codex variant (uses custom agents in `~/.codex/agents/`)
- `openclaw/source-checkm/SKILL.md` — OpenClaw variant (uses `sessions_spawn` + `sessions_yield`)
- `EVIDENCE.md` — full v8 + v9 test methodology, per-case results, why-this-works analysis

---

## License

MIT. Take it, fork it, improve it. If you run a larger validation, please share the result — n=13 is a starting line, not a finish line.
