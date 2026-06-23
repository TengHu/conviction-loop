---
name: claim-verdict
description: verify ONE claim and give its 0–100 score. Reads that claim's row in book/theses/<slug>/claims.yaml + the source named in its `fetch` + live data for its tickers; writes score + score_updated back.
---

# claim-verdict — score ONE claim 

Input: a thesis slug + a claim id. Update that one claim's **score (0–100)** from evidence. A price move never moves a score — only evidence does. You are the only writer of this claim's score field.

1. Read the claim's row in `book/theses/<slug>/claims.yaml` — its falsification legs, `fetch` source, and current `score` + `score_updated`.

2. Check each falsification leg and turn the result into an **evidence_delta** (one number):
   - **quantitative** → compute from live data (last completed close; honor `consecutive_closes`; an intraday touch never counts).
   - **textual** → a verbatim quote from the source named in `fetch` (EDGAR / transcript / IR).
   - **user-attested** → ask via AskUserQuestion (standalone) or return `pending-user-attestation` (subagent).
   `evidence_delta` is **+** if the leg confirms, **−** if it weakens; magnitude by evidence grade — **primary ±15 · issuer-attested ±8 · shadow ±3** (owned here; the verdict stage's only such knob — tune against real histories). No fresh evidence → 0.

3. Update the score, **in code**:
   ```
   score = clamp(0, score × exp(−days_since(score_updated) / half_life) + evidence_delta, 100)
   ```
   `half_life = 30 days` — owned here (the verdict stage's decay constant; tune it here — the verdict layer reads no portfolio file). A fired or structural falsification (claim now flatly false / position closed) → `score = 0` (sticky until rewritten). Write back `score` and `score_updated = today`.

Return: `{ claim_id, score, Δ, pending? }`. Touch nothing but this claim's score fields — no other claim, no rollup, no portfolio file. (The thesis agent in /daily-run takes the `min` across claims for conviction.)

## Loop
- **Goal:** this claim's `score` reflects today's evidence.
- **Verify before writing:** the Δ traces to a cited `evidence_delta`, never to price; cap evidence fetches (don't hunt endlessly). Its independent check is /daily-run's verify pass.
- **Done when:** `score` + `score_updated` written for this one claim.

## Tools
- **Alpaca MCP** (`get_stock_bars`, `get_stock_snapshot`) — closes / prices for quantitative legs (read-only).
- **WebSearch / WebFetch** — textual evidence: EDGAR filings, transcripts, IR.
