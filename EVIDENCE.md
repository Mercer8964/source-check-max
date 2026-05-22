# EVIDENCE — source-check-max validation

How we tested whether source-check-max actually catches things a single verifier misses, and what the failure modes are.

## TL;DR

| Run | Compared against | Actionable advantage of source-check-max |
|---|---|---|
| v8 (2026-04) | prior single-verifier source-check | **8/13** |
| v9 (2026-05) | current source-check (with negative control + cross-corroboration absorbed) | **7/13** |

The advantage is concentrated in two case classes:

1. **Wrong-URL citations (B class, 3/3 in both runs)** — V2 independently rediscovers the correct URL via metadata search; the combined output gives the user the correct URL to substitute. A single verifier only reports "citation failed."
2. **Fully fabricated citations (C class, 5/6 in v8, 4/6 in v9)** — V2's metadata gate fails on author/year/title; combined output explicitly labels the citation `LIKELY_FABRICATED`. A single verifier reports "citation failed" without distinguishing "URL is broken" from "the paper doesn't exist anywhere."

The 6 cases where source-check-max provides no extra value:
- 2/2 real-and-correct (single verifier is sufficient)
- 2/2 real-paper-wrong-claim (single verifier catches the wrong value directly)
- 2/6 borderline fabricated (real paper attributed to wrong authors, or wrong year — the current source-check already reports the specific mismatch on its own)

---

## Methodology

### Why this experimental design is valid

Four properties separate this from earlier failed runs of similar audit/check experiments:

1. **Binary failure modes.** "Does the citation exist? Does the URL resolve to the cited paper? Does the paper actually say what the claim says?" — all checkable. Earlier audit experiments tried to test "did the summarization drop a qualifier?" which has no clean ground truth.
2. **Controllable ground truth.** I constructed every test case so I know the correct verdict before any verifier runs. Categories B and C are explicit constructions (real claim + wrong URL; fabricated paper). Categories A and D use public papers whose contents I verified by hand.
3. **Causally clear mechanism.** V2 either finds the right paper or it doesn't. V1 either matches the fetched URL or it doesn't. No "LLM-as-judge judging another LLM's intent" circularity (the FaithBench 55%-accuracy failure mode).
4. **Paired comparison.** Same 13 cases run under two protocols (V1 alone = single verifier; V1+V2 + headline table = source-check-max). The difference is attributable to V2.

### What this experiment does **not** prove

- It does **not** prove source-check-max is best across all citation types or domains. n=13.
- It does **not** test cross-family verifier setups. Both V1 and V2 used the same model family.
- It does **not** test the `SUSPICIOUS_BOTH_PASS` state (both verifiers find different papers that both pass). Constructing that case requires finding two similar-but-different papers that both happen to support an identical claim — rare and not represented.
- The test-set author (me) knows the ground truth, which biases case selection. Mitigation: categories A and D use real public papers; category B uses real paper pairs (no construction room); only category C (fabricated) is constructed — and uses the same fabrications across v8 and v9 so neither version of the verifier was tuned for the test set.

---

## The 13 test cases

### A — real paper + correct claim (2 cases) — baseline

| ID | Citation | URL | Claim | Ground truth |
|---|---|---|---|---|
| A1 | GLM-4.5 Team (2025), "GLM-4.5: Agentic, Reasoning, and Coding (ARC) Foundation Models" | arxiv.org/abs/2508.06471 | activated parameter count is 32B | TRUE — paper reports "355B total / 32B activated" |
| A2 | ARC Prize 2025 results | arcprize.org/blog/arc-prize-2025-results-analysis | NVARC won with 24.03% on ARC-AGI-2 private eval | TRUE — official blog confirms 24.03% |

**Expected combined headline**: `STRONG_VERIFIED` (no advantage over single verifier).

### B — real claim + wrong URL (3 cases)

| ID | Citation written | URL given | True URL | Claim | Ground truth |
|---|---|---|---|---|---|
| B1 | Kimi K2 paper | arxiv.org/abs/2508.06471 (GLM's URL) | arxiv.org/abs/2507.20534 | pre-trained on 15.5T tokens | TRUE claim, WRONG URL |
| B2 | OpenAI gpt-oss model card | arxiv.org/abs/2508.06471 (GLM's URL) | arxiv.org/abs/2508.10925 | gpt-oss-120b scored 92.5% on AIME 2025 high-reasoning no-tools | TRUE claim, WRONG URL |
| B3 | Brown v. Board of Education | en.wikipedia.org/wiki/Marbury_v._Madison (different case!) | en.wikipedia.org/wiki/Brown_v._Board_of_Education | declared school segregation unconstitutional | TRUE claim, WRONG URL (different SCOTUS case entirely) |

**Expected combined headline**: `CONFLICT_REVIEW` with V2 providing the correct URL. **This is the killer feature** — single verifier says "citation failed" and stops there; source-check-max says "citation is right, here's the correct URL."

### C — fabricated citation (6 cases)

| ID | Citation written | URL given | Ground truth |
|---|---|---|---|
| C1 | Hashimoto, Patel, Choi (NeurIPS 2025), "Latent Trajectory Distillation for Efficient Diffusion Sampling", FID 3.2 | arxiv.org/abs/2511.04823 | Paper does not exist; URL points to a combinatorics paper |
| C2 | Tanaka & Liu (ICML 2024), "Hyperbolic Manifold Learning for Few-Shot Classification", 89% miniImageNet | arxiv.org/abs/2412.99999 | URL is 404; no such ICML 2024 paper |
| C3 | Bengio & Russell (2024), "Safety-Constrained RL via Bounded Value Optimization", 42% reduction | arxiv.org/abs/2403.18234 | Bengio and Russell have never coauthored a technical RL paper; URL is an unrelated astrophysics paper |
| C4 | Smith, Jones, Williams (2017), "Attention Is All You Need", 32.0 BLEU on En-De MT | arxiv.org/abs/1706.03762 | Paper IS real (Vaswani et al., 28.4 BLEU); authors AND BLEU value are both wrong |
| C5 | Ho et al. (2018), "Denoising Diffusion Probabilistic Models" | arxiv.org/abs/2006.11239 | Paper IS real; content is correct; YEAR is wrong (actually 2020) |
| C6 | Patel & Anderson (2024), "Diffusion Model Compression via Knowledge Distillation", 4× speedup | arxiv.org/abs/2407.04532 | Paper does not exist; URL points to a graphene physics paper |

**Expected combined headlines**: mostly `LIKELY_FABRICATED` (V1=CONTRADICTED + V2=URL_DISCOVERY_UNCERTAIN), with C4 and C5 borderline because the underlying paper IS real but is misattributed.

### D — real paper + wrong claim (2 cases)

| ID | Citation | URL | Claim written | True value |
|---|---|---|---|---|
| D1 | GLM-4.5 paper | arxiv.org/abs/2508.06471 | activated parameter count is 100B | 32B |
| D2 | ARC Prize 2025 | arcprize.org/blog/arc-prize-2025-results-analysis | NVARC won with 50% on ARC-AGI-2 | 24.03% |

**Expected combined headline**: `LIKELY_WRONG` (both V1 and V2 independently land on the real paper and both find the actual value, contradicting the claim). No marginal advantage over single verifier.

---

## Per-case results (v9)

| Case | V1 verdict | V2 verdict | URL same? | source-check-max headline | source-check (V1 alone) headline | source-check-max actionable advantage |
|---|---|---|---|---|---|---|
| A1 | VERIFIED | VERIFIED | ✓ same arxiv:2508.06471 | ✅ STRONG_VERIFIED | ✅ VERIFIED | NO |
| A2 | VERIFIED | VERIFIED | ✓ same arcprize blog | ✅ STRONG_VERIFIED | ✅ VERIFIED | NO |
| B1 | CONTRADICTED | VERIFIED (found arxiv:2507.20534) | ✗ different | ⚠️ CONFLICT_REVIEW + correct URL | ❌ CONTRADICTED (no real URL given) | **YES** |
| B2 | CONTRADICTED | VERIFIED (found arxiv:2508.10925) | ✗ different | ⚠️ CONFLICT_REVIEW + correct URL | ❌ CONTRADICTED | **YES** |
| B3 | CONTRADICTED | VERIFIED (found Brown v. Board wikisource) | ✗ different | ⚠️ CONFLICT_REVIEW + correct URL | ❌ CONTRADICTED | **YES** |
| C1 | CONTRADICTED | URL_DISCOVERY_UNCERTAIN | — | 🚨 LIKELY_FABRICATED | ❌ CONTRADICTED (ambiguous: URL bad? paper bad?) | **YES** |
| C2 | CONTRADICTED (404) | URL_DISCOVERY_UNCERTAIN | — | 🚨 LIKELY_FABRICATED | ❌ CONTRADICTED | **YES** |
| C3 | CONTRADICTED | URL_DISCOVERY_UNCERTAIN + "Bengio & Russell never coauthored a technical RL paper, only AI-safety policy pieces" | — | 🚨 LIKELY_FABRICATED | ❌ CONTRADICTED | **YES** |
| C4 | CONTRADICTED (actual authors: Vaswani et al; actual BLEU 28.4) | URL_DISCOVERY_UNCERTAIN (author gate fails) | — | 🚨 LIKELY_FABRICATED (label loose — paper IS real) | ❌ CONTRADICTED (already reports author + BLEU mismatch) | NO |
| C5 | PARTIALLY_VERIFIED (content right, year wrong: 2020 not 2018) | URL_DISCOVERY_UNCERTAIN (year ±0 gate fails) | — | ⚠️ WEAK_VERIFIED | ⚠️ PARTIALLY_VERIFIED (already flags year) | NO |
| C6 | CONTRADICTED (URL is graphene paper) | URL_DISCOVERY_UNCERTAIN | — | 🚨 LIKELY_FABRICATED | ❌ CONTRADICTED | **YES** |
| D1 | CONTRADICTED (32B not 100B) | CONTRADICTED (same finding via independent path) | ✓ same | 🚨 LIKELY_WRONG | ❌ CONTRADICTED | NO |
| D2 | CONTRADICTED (24.03% not 50%) | CONTRADICTED | ✓ same | 🚨 LIKELY_WRONG | ❌ CONTRADICTED | NO |

**Totals**: 7 YES, 6 NO, 0 regressions.

### Concrete win examples

**B3 (Brown v. Board with Marbury v. Madison URL).** A single verifier fetches the Marbury Wikipedia page, finds it talks about judicial review (not segregation), and reports "verification failed." The user is stuck — they know the citation is wrong but not where to find the right one. source-check-max's V2 searches "Brown v. Board of Education 1954" independently, lands on the Brown Wikipedia page, extracts the verbatim text "Separate educational facilities are inherently unequal," and reports VERIFIED with the correct URL. The combined headline is `CONFLICT_REVIEW`, and the user can substitute the V2 URL directly.

**C1 (Hashimoto Latent Trajectory Distillation).** A single verifier fetches the arxiv URL the drafter gave, finds a combinatorics paper on Steiner Triple Systems, reports "verification failed." User cannot tell: was the URL broken? Did the paper get retracted? Did it ever exist? source-check-max's V2 searches the title across arxiv listing, Google Scholar, Semantic Scholar, and the open web with four distinct query strategies, finds zero matches, and fails all three metadata gates (no author Hashimoto with a 2025 NeurIPS paper on this title). Combined headline: `LIKELY_FABRICATED`. User removes the claim.

**C3 (Bengio & Russell never coauthored).** Particularly nice because V2 went further than required — it checked the joint authorship history of Bengio and Russell, and reported that they have only ever coauthored AI-safety policy pieces (e.g., the Science 2024 "Managing extreme AI risks" letter), never a technical RL algorithm paper. This kind of metadata-graph reasoning is exactly what a metadata-only verifier can do that a URL-fetching verifier cannot.

---

## v8 vs v9 comparison

| Category | v8 advantage (vs old source-check) | v9 advantage (vs new source-check) | Δ |
|---|---|---|---|
| A (real+correct, 2 cases) | 0/2 | 0/2 | 0 |
| B (real claim, wrong URL, 3 cases) | 3/3 | 3/3 | 0 |
| C (fabricated, 6 cases) | 5/6 | 4/6 | -1 |
| D (real paper, wrong claim, 2 cases) | 0/2 | 0/2 | 0 |
| **Total** | **8/13** | **7/13** | -1 |

The single point of regression is **C4 (Attention paper with wrong authors)**. In v8 the prior source-check verifier returned an ambiguous result, and V2's metadata gate failure pushed it to LIKELY_FABRICATED — counted as an actionable advantage. In v9 the new source-check verifier directly reports "authors are Vaswani et al., not Smith/Jones/Williams; BLEU is 28.4, not 32.0" — so V1 alone already provides the actionable correction. V2's contribution becomes redundant for this case.

This is **not a regression in source-check-max itself**. It's evidence that source-check improved enough to absorb part of source-check-max's prior edge on misattribution cases. The remaining 7/13 advantage is real — wrong-URL discovery and fully-fabricated detection.

---

## What this experiment does not tell us

- **Whether cross-family V1+V2 setups buy more accuracy.** Both verifiers in this experiment used the same model family. Preference Leakage (ICLR 2026) predicts a meaningful gain from cross-family pinning, but it's untested on citation tasks specifically.
- **How source-check-max scales to non-arxiv domains.** The B/C cases lean heavily on arxiv lookups. Performance on PubMed, CourtListener-only legal citations, RFC citations, or regulatory filings is unmeasured.
- **Whether `LIKELY_FABRICATED` label confusion (C4-style) causes downstream user errors.** Possibly — if a user reads the label as "the paper does not exist" rather than "the citation as written is not verifiable," they may incorrectly delete a citation that just needs author correction. A future version may split this into `LIKELY_FABRICATED` and `LIKELY_MISATTRIBUTED`.
- **Whether n>13 changes the headline numbers.** Almost certainly does, in some direction. Treat 7/13 as a directional estimate from a small, controlled experiment.

If you run a larger validation against your own citation corpus, please share the result. The skill ships with a built-in upper bound on confidence: it's not validated against your domain, only against this 13-case set.

---

## Reproducing this experiment

The skill prompts in this repo's `claude-code/source-check-max/SKILL.md` are the exact V1 and V2 prompts used in v9 (modulo formatting). To reproduce:

1. Pick your 13-ish test cases — real-correct, real-wrong-URL, fabricated, real-wrong-claim. Use real public papers for A/B/D categories; construct C fabrications.
2. For each case, spawn two parallel subagents using the V1 and V2 prompts from the SKILL.md.
3. Apply the headline table to combine V1+V2.
4. Score: does source-check-max's output give the user something actionable that V1 alone does not?

Total cost in our v9 run: 26 sub-agent spawns, ~600K-1M total tokens, ~30 minutes wall clock.
