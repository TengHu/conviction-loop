---
name: portfolio
description: Reads the conviction list + your goal/constraints (book/portfolio/README.md) and reconciles the book, places nothing.
---

# portfolio

The manager. You author the goal + constraints; the formula does the rest. 

## Step 1 — Read policy (`{book}/portfolio/README.md`)
- **Goal:** return target + horizon → aggression `a = k × return_target`.
- **Constraints:** loss limit (NAV floor), biggest bet (per-bet cap %), factor caps.
- **Policy:** the filter rule of thesis convictions (e.g. "conviction > 0").
- **Knobs:** `deadband` (trade threshold), `half_life` (score decay), `k` (goal→aggression), `entry_half_life` (deployment pace; the horizon sets it).


## Step 2 — Conviction list
**in code** Read every thesis's conviction score.

## Step 3 — Reconcile the book (broker, once)
**in code** Pull live holdings once → `do-hold $` per thesis + NAV. The money-saving traps (join on security_id, decode OCC, mark options/leveraged ETFs from Alpaca not Plaid. The broker is the truth: never store positions, re-read every run.

## Step 4 — FILTER
Keep the subset passing your criteria; read the portfolio README for the actual filter policy. AUQ if not sure. A held thesis that filters out stays in as a forced **exit** (target 0), never silently dropped.

## Step 5 — ALLOCATE (the math, in code)
Given the policy + survivors, come up with bets — each `{thesis, budget, conviction, max_loss}` + enough for a person to implement it with trades.
1. **raw_target** per survivor — `NAV·k·conviction·(upside/downside)`, reading `upside`/`downside` from the thesis's **`bet.yaml`** (conviction is the daily-run rollup) — capped + scaled to `deployable`.
2. **Smooth it (continuity), pace per bet.** Read each thesis's previous smoothed `target` from the **last dated note** (seed 0 if new), then `target = α·raw_target + (1−α)·prev_target`. **α is per bet:** `α = min(1, ln2 / entry_half_life)`, where `entry_half_life = horizon/4` from the thesis's **`bet.yaml`** if it sets `horizon`, else the global default. A `horizon` of `1d` → α=1 → deploy in full today (same-day bet); a long horizon → crawl in (DCA). This smoothed `target` is the bet's **budget** — it phases in geometrically from 0, so cash is never dropped at once and a one-day conviction swing barely moves it.
3. **Fast-exit exception.** A thesis at conviction 0 from a *fired falsification* (structural break, not a fade) → `target = 0`, **bypass smoothing**, exit now. Same fast-path the score uses when a leg slams to 0.

## Step 6 — Delta note
Write `book/portfolio/notes/<DATE>.md`: one table `thesis | conviction | raw_target | target | do-hold | gap | action` (`target` = smoothed; `gap = target − do-hold`). The `target` column is the **carry** — next run reads it back as `prev_target`. This dated note **is** the delta of current holdings, the run's decision record, AND the smoothing state (git-diffable run-over-run).

## Step 7 — Execute (fan out)
Spawn one `/implement-bet` general-purpose subagent per bet, in parallel (safe — budgets sum within `deployed`), passing each bet's reconciled holding + the `deadband`. Collect the order specs.

## Step 8 — You confirm (DUMB-PROOF broker tickets)
Present the orders as a **broker-ready ticket list a 12-year-old could enter** — no jargon, no math left to do. SELLS first (they free the cash), then BUYS, numbered in execution order. One row per order with exactly these columns:

`# | thesis | 标的 (exactly what to trade) | Buy/Sell | how many (shares or contracts) | limit price | ≈ $`

Rules for the rows:
- **标的 must be self-contained.** Stock → just the ticker. Option → ticker + **Call/Put + strike + plain expiration date** (e.g. "CRWV Call, strike $210, expires Jan 21 2028"), and quantity in **contracts**.
- Every order is **Limit / Day**. Say "SELL ALL (~N sh)" for full exits; "buy N sh/contracts" otherwise; for partial trims say "keep M".
- Include a one-line legend (how to place a stock vs an option order) and a "if it won't fill, nudge the limit" note.
- This ticket list **is** the deliverable — put it at the top of the dated note and in the chat reply.

Place nothing. Every order is a recommendation the user enters manually.

At last, use git to for state tracking.

## Loop
- **Goal:** a bet per survivor, a dated delta note, orders within `deployable` + constraints.
- **Verify (pre-gate, in code):** Σtargets ≤ deployable, every gap = target − hold, no order breaches `loss_limit` / `biggest_bet` — before presenting. You approve before anything is placed (the human is the final checker).
- **Done when:** delta note written + orders presented; machine placed nothing.

## Tools
- **Plaid CLI** (`plaid investments holdings/transactions --all --json`) — reconcile current holdings + NAV.
- **Alpaca MCP** — live marks for options / leveraged ETFs (Plaid marks are stale).
- **General-purpose subagents** (Agent) — spawn one `/implement-bet` per bet.
- **AskUserQuestion** — present the bets and get approval.