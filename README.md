# thesis-loop

A lean, file-based **investing loop for Claude Code**. You curate noisy signal in; the framework atomizes it into falsifiable claims, verifies them daily into decaying conviction scores, turns conviction + your written goal into sized bets, and hands you broker-ready tickets. **The default output is nothing** — most things die at every gate. A trade is what survives.

It places **no orders**. Every output is a recommendation you enter manually.

---

## The idea

The loop is a **noise filter**, not a generator. Six small skills, four stages, two boundary objects:

```
Stage 0  INGEST     you paste a belief / WeChat / X dump (signal + noise)
                    → the framework atomizes it into a thesis + falsifiable claims     [create-thesis]

Stage 1  VERDICT    each claim → a decaying 0–100 score; per thesis → conviction (min)  [claim-verdict / daily-run]
                    ──────── boundary object: the VERDICT ────────

Stage 2  BET        conviction + your goal → sized BETS (a transparent formula)         [portfolio + sizing-policy]
                    ──────── boundary object: the BET ────────

Stage 3  EXECUTION  each bet vs what you hold → orders → you confirm and place them      [portfolio + implement-bet]

slow loop  REFLECT  distills the score history into one durable belief per thesis        [reflect]
```

**Your role is narrow on purpose** (think OODA): you **Observe** (curate + inject signal), you **set the objective** (the goal + constraints), and you **Act** (place the approved orders, with a veto). Everything in between — orient and decide — is the framework. You never author a position size, a per-claim action, or a fabricated number.

Time is baked in: the **same first-order (EWMA) filter** runs at two layers — the conviction **score** (belief) and the **target** (capital) — so neither a belief nor a position can whipsaw on one day's data, and capital phases in instead of dumping all at once.

See [`arch.md`](./arch.md) for the full design and rationale.

## The skills

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 0 Ingest | **create-thesis** | your signal | `book/theses/<slug>/{README.md, bet.yaml, claims.yaml}` |
| 1 Verdict (worker) | **claim-verdict** | one claim + its source + live data | that claim's `score` |
| 1 Verdict (orchestrator) | **daily-run** | every thesis's `claims.yaml` | scores + a run note; rolls conviction = `min` |
| 2+3 Bet + Execute | **portfolio** | `portfolio/README.md` + each `bet.yaml` + convictions + the broker | a dated delta note + broker-ready tickets |
| 3 Execute (worker) | **implement-bet** | one bet + its reconciled holding | order specs (places nothing) |
| slow loop | **reflect** | the score history (git) | one durable belief per thesis |

## Quickstart

1. Open this repo in **Claude Code** (the skills auto-load from `.claude/skills/`).
2. Seed your private book: `cp -r template_book book` (your `book/` is gitignored — it never leaves your machine).
3. Edit `book/portfolio/README.md` — your **goal + constraints** (the only thing you author by hand).
4. `/create-thesis` — paste a belief or a dump; the framework writes the thesis + claims.
5. `/daily-run` — score the claims into convictions.
6. `/portfolio` — turn convictions + your goal into bets, reconcile your book, and get broker-ready tickets.
7. You place the orders. The machine places nothing.

## Prerequisites

- **Claude Code.**
- **Alpaca MCP** — live prices/quotes for the verdict + sizing math (read-only).
- **Plaid CLI** *(optional)* — reconcile real holdings + NAV. Without it, enter holdings manually.

## What's tunable, and where

Every knob lives in **one** place — the file its agent reads (DRY). The full map is in `book/portfolio/README.md` under **Knobs — the complete map**. You author *policy* (goal, constraints); you never author a size or a score.

## Privacy

- Your real theses, positions, and policy live in `book/`, which is **gitignored** — this repo ships only the framework + mock demo theses.
- The framework is **read-only on execution**: it computes and recommends; it never places a trade.

## Prior art

Borrows the deterministic action-space pre-gate (ai-hedge-fund), typed structured verdicts (TradingAgents), config-as-data agents (FinRobot), and the decaying time-aware score (FinMem). Allocation is conviction × reward/risk + an EWMA target; Kelly and inverse-vol were considered and dropped. See `arch.md` → *Prior art*.

## Disclaimer

This is software for organizing your own research and discipline. It is **not financial advice**, makes no guarantees, and places no orders. You are responsible for every trade you enter. Use at your own risk.

## License

[MIT](./LICENSE)
