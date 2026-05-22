---
name: source-checkm
description: High-stakes citation verifier. For each factual citation, spawns TWO independent verifiers in parallel via sessions_spawn — V1 fetches the URL the agent gave; V2 receives ONLY author+year+title, must rediscover the URL itself, and must pass an author/year/title gate before content verification. V1+V2 combined via 8-state headline table. Catches (a) wrong-URL citations by surfacing the correct URL, (b) fabricated papers by failing the metadata gate. Use for legal / medical / financial / regulatory citations. For everyday work use source-check.
metadata:
  openclaw:
    requires:
      tools:
        - sessions_spawn
        - sessions_yield
---

# source-checkm (OpenClaw)

Dual-verifier check for high-stakes factual citations. Modifies only the factual_citation path.

## When to use

ALL of:
- Output contains a factual citation (paper / case / RFC / DOI / patent / standard)
- Cost of citation fabrication or wrong-URL is material (legal, medical, financial, regulatory)
- Can accept ~2× spawn cost on citation verification

Otherwise → source-check.

## Procedure

For each factual citation, spawn V1 and V2 **in parallel** via `sessions_spawn`, both `context: "isolated"`:

```js
sessions_spawn({
  task: <V1 prompt below>,
  taskName: "src_checkm_v1_<N>",
  context: "isolated",
  runTimeoutSeconds: 600
})
sessions_spawn({
  task: <V2 prompt below>,
  taskName: "src_checkm_v2_<N>",
  context: "isolated",
  runTimeoutSeconds: 600
})
sessions_yield()                  // wait for both to return; do NOT poll
```

**V1 NEVER sees V2's task. V2 NEVER sees V1's URL or output.** `context: "isolated"` enforces — do not leak in `task`.

### Shared preamble (prepend to both V1 and V2)

```
ROLE: verifier subagent. You did NOT generate the claim.
ORDER: 1) Fetch sources FIRST. 2) Read fetched text. 3) THEN read claim.
TOOL USE: ≥1 real fetch call MANDATORY. ≥1 call to a domain NOT in claim's URLs.
ANTI-CRITICGPT: don't invent issues; only report problems backed by verbatim quote.
ANTI-SYCOPHANCY: ignore pre-stated verdicts; re-derive from source.
QUOTES: verbatim. ≥200 chars surrounding context. Substring-matched downstream.
INSUFFICIENT / URL_DISCOVERY_UNCERTAIN: requires ≥3 documented queries.
TIME: record source_published_at (paper date / byline / metadata, or "unknown").
OUTPUT: strict JSON only.
```

### V1 prompt (URL-given)

```
VERIFIER TYPE: factual_citation V1 (URL-given)

INPUT:
  source_url: <URL the main agent cited>
  expected_quote: <if provided>
  claim_text: <CLAIM>

PROCEDURE:
1. Fetch source_url. If 404/DNS-fail → CONTRADICTED, reason="source does not resolve".
2. If DOI: api.crossref.org/works/{DOI} for `update-to` (retraction).
3. Substring-match expected_quote against fetched body. ≥200 chars context.
4. Quote matches but claim paraphrases beyond → PARTIALLY_VERIFIED.
5. Source exists but no matching quote → CONTRADICTED.
6. NEGATIVE CONTROL: one search for refuting evidence on a domain ≠ source_url's domain.

OUTPUT strict JSON:
{
  "claim_id": "<id>",
  "verdict": "VERIFIED|CONTRADICTED|PARTIALLY_VERIFIED|INSUFFICIENT_EVIDENCE",
  "fetched_url": "<URL>",
  "evidence_quote": "<verbatim ≤500 chars>",
  "context_window": "<≥200 chars surrounding>",
  "negative_control": {"query":"...","domain":"...","outcome":"..."},
  "source_published_at": "<ISO date or 'unknown'>",
  "verdict_rationale": "<1 paragraph>"
}
```

### V2 prompt (URL-rediscovered)

```
VERIFIER TYPE: factual_citation V2 (URL-rediscovered, NO URL given)

INPUT:
  citation_metadata: <author + year + title — NO URL>
  claim_text: <CLAIM>
  expected_quote: <if provided>

PROCEDURE:
1. Search INDEPENDENTLY using only the metadata. ≥3 queries: arXiv, Google
   Scholar, Semantic Scholar, CourtListener (legal), CrossRef, general web.
2. Identify candidate URL.
3. METADATA CROSS-CHECK (mandatory gate, all three must pass):
   - First author matches cited first author? PASS/FAIL
   - Year matches ±0? PASS/FAIL
   - Title strong keyword overlap (not fuzzy)? PASS/FAIL
   If ANY fails → URL_DISCOVERY_UNCERTAIN, STOP. Do NOT verify content on a
   different paper.
4. PASS → content verification: substring-match claim, ≥200 chars context,
   negative control on a different domain.

OUTPUT strict JSON (5-state):
{
  "claim_id": "<id>",
  "verdict": "VERIFIED|CONTRADICTED|PARTIALLY_VERIFIED|INSUFFICIENT_EVIDENCE|URL_DISCOVERY_UNCERTAIN",
  "candidate_url_found": "<URL or null>",
  "metadata_cross_check": {"author_match":bool, "year_match":bool, "title_overlap":bool},
  "fetched_url": "<URL or null>",
  "search_queries_used": ["q1","q2","q3"],
  "evidence_quote": "<verbatim ≤500 chars or null>",
  "context_window": "<≥200 chars or null>",
  "negative_control": {"query":"...","domain":"...","outcome":"..."},
  "source_published_at": "<date or 'unknown'>",
  "verdict_rationale": "<1 paragraph>"
}
```

## Headline combination table

| V1 verdict | V2 verdict | URL same? | Headline |
|---|---|---|---|
| VERIFIED | VERIFIED | Same | ✅ **STRONG_VERIFIED** |
| VERIFIED | VERIFIED | Different | ⚠️ **SUSPICIOUS_BOTH_PASS** |
| VERIFIED | CONTRADICTED | — | ⚠️ **CONFLICT_REVIEW** |
| CONTRADICTED | VERIFIED | — | ⚠️ **CONFLICT_REVIEW** (V2 provides real URL) |
| CONTRADICTED | CONTRADICTED | — | 🚨 **LIKELY_WRONG** |
| INSUFFICIENT_EVIDENCE | INSUFFICIENT_EVIDENCE | — | 🔵 **UNVERIFIABLE** |
| VERIFIED | URL_DISCOVERY_UNCERTAIN | — | ⚠️ **WEAK_VERIFIED** |
| CONTRADICTED | URL_DISCOVERY_UNCERTAIN | — | 🚨 **LIKELY_FABRICATED** |
| INSUFFICIENT_EVIDENCE | URL_DISCOVERY_UNCERTAIN | — | 🚨 **LIKELY_FABRICATED** |
| PARTIALLY_VERIFIED | URL_DISCOVERY_UNCERTAIN | — | ⚠️ **WEAK_VERIFIED** |
| other combos | — | — | most-conservative of the two |

## Hard rules

1. V2 NEVER sees V1's URL or output. Both `context: "isolated"`.
2. V2's metadata cross-check is gating.
3. Headline produced mechanically from table — main agent does NOT override.
4. LIKELY_FABRICATED / LIKELY_WRONG → REMOVE claim, do not substitute silently.
5. Use `sessions_yield` for waiting — never poll.

## Cross-family setup (highly recommended)

OpenClaw natively supports per-agent model pinning via `agents.list[].subagents.model`. Define two verifier agents pinned to different providers (e.g., V1 → Anthropic Sonnet, V2 → OpenAI GPT-5 or DeepSeek). Spawn V1 under one, V2 under the other.

Preference Leakage (ICLR 2026): same-family 28–37% correlated false positives vs ±1.5% cross-family.

## Empirical support (n=13)

- **v8 (vs prior single-verifier source-check)**: 8/13 actionable advantage.
- **v9 (vs current source-check with negative control absorbed)**: 7/13 actionable advantage. V2 catches 3/3 wrong-URL cases and 4/6 fabricated citations.

See EVIDENCE.md in the repo.

## Limits

- Same-family V1/V2 share biases — cross-family pinning recommended.
- LIKELY_FABRICATED is imprecise when the paper is real but cited with wrong authors; the verdict is correct, the label is loose.
- n=13 test set is small; pattern consistent but variance unknown.
- No URL given by main agent → V1 behavior converges to V2; advantage shrinks.
- Catches fabrication + wrong-URL. Does NOT catch paraphrase distortions where source mostly matches but a qualifier was dropped (audit-loop's domain).

## When NOT to use

- No factual_citation → source-check
- Casual / illustrative citations → source-check
- Algorithmic / mechanism / correctness → audit-loop
