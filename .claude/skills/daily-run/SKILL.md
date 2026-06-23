---
name: daily-run
description:  Spawns one general-purpose agent per thesis; each thesis agent spawns one general-purpose agent per claim to invoke /claim-verdict; the thesis agent rolls its claim scores into a conviction  and returns it. daily-run assembles the conviction list
---

# daily-run — score all theses (Stage 1)

Produce one thing: the **conviction list**

1. **Scope.** Default: every thesis in the user designated book, `{book}/theses/`.

2. **Fan out — one general-purpose agent per thesis.** For each in-scope thesis, spawn a general-purpose subagent whose job is:
   - Read `book/theses/<slug>/README.md` first and `book/theses/<slug>/claims.yaml`.
   - **One claim, one isolated agent — non-negotiable.** For each load-bearing claim, spawn a SEPARATE general-purpose subagent that runs `/claim-verdict <slug> <claim-id>` on that ONE claim. **Never batch claims into a shared context, even when they share a source.** Isolation IS the verification guarantee: a shared context lets the agent anchor across claims (nudge a weak claim up because the thesis "looks strong"), and one misread of a shared source silently taints every claim that read it. Re-fetching a shared source per claim is the price of an independent verdict, not waste.
   - Each claim agent **returns** its `{id, score, Δ, why}`; it does **not** write the file. The thesis agent writes the returned scores back to `claims.yaml` in ONE pass (concurrent writes to one file race and corrupt it).
   - **Conviction = `min` of the load-bearing claim scores** (the weakest pillar).
   - Return `{ thesis, conviction, claims: [{id, score, Δ}], pending_attestations? }`.

3. **Report — one file.** Write `book/theses/notes/<DATE>.md`, the only comprehensive output. Concise, nothing else (scores already live in each `claims.yaml`). Per thesis: a header `## <slug> — <conviction = min(load-bearing)>`, then one bullet per validated claim:
   ```
   - <claim-id>: <previous_score> -> <current_score>
   ```
   If the claim had no previous score, just `<current_score>`.

## Loop
- **Goal:** every load-bearing claim has a fresh score (`score_updated = today`); conviction = min per thesis.
- **Verify (separate pass):** after the fan-out, confirm each load-bearing claim actually got a fresh score — don't trust the workers' self-report; re-run the misses, cap at N, then report gaps. Anti-spin: never re-spawn a claim already scored today.
- **Done when:** the conviction list covers every in-scope thesis with no stale load-bearing claim.

## Tools
- **General-purpose subagents** (Agent) — fan out one per thesis, then one per claim.
- **/claim-verdict** — the per-claim scorer each leaf agent invokes.
