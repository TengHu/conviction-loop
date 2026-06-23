---
name: implement-bet
description: turn ONE bet into the trades(s)
---

# implement-bet — one bet → its trades(s)

Input: one bet `{thesis, budget, max_loss}` + its reconciled current holding(s) (`$`, actual basis) + the `deadband`.

1. `gap = budget − current_holding$`. If `|gap| ≤ deadband` → return **"no order"** (null).
2. **Decide how the bet is expressed.** Read the thesis README / its "Instructions for the next agent" for any vehicle preference (e.g. "common only", "prefer LEAPS"). A bet may need **one or more orders** — common + options to build the exposure, or legging a large gap across limit prices / days. Default to a single order; use multiple only when the bet's structure or the thesis instruction calls for it.
3. Pull live quotes for the instrument(s) (Alpaca, read-only).
4. Compute each order **in code**: `side = buy if gap > 0 else sell`; `qty = round(slice / price)` (per-contract × multiplier for options); `limit` near the quote. A budget of 0 → sell the whole position (exit). **The order(s) must sum to ≈ the gap.**
5. Anchor any basis / payoff math on the reconciled **actual** cost basis, never a thesis-assumed entry.

Return a **list** of specs executable trades

## Loop
- **Goal:** order spec(s) summing to ≈ the gap, or "no order".
- **Verify before returning:** the specs reconcile to the gap, sit within the deadband, and price off the *actual* basis.
- **Done when:** specs (or null) returned. **You** are the independent verifier — it places nothing.

## Tools
- **Alpaca MCP** (`get_stock_latest_quote`, `get_option_snapshot` / `get_option_chain`) — live quotes to size the order(s). Read-only: places nothing.
- **Plaid CLI** (`plaid investments holdings/transactions --all --json`) — current holdings
