# Portfolio

> **This is the template policy — EXAMPLE numbers.** Copy `template/` to `book/`, then edit this file with your own goal + constraints. The numbers below are illustrative.

## Goal
Return target: **+30% in 12 months**.

The **12-month horizon** sets the deployment *pace*, not just the return. Capital phases in over the **`entry_half_life`** (its value lives in `sizing-policy.md`): a full-conviction bet reaches ~50% of its level in one half-life, ~75% in two — so cash is never dropped all at once, and a fresh thesis must hold up over time before it earns full size.


## Constraints
- **Loss limit (NAV floor): $700,000.** At or below, adds are vetoed; only de-risk.
- **Biggest bet: 25% of NAV** per thesis.

## Filter
A thesis survives if its **conviction > 0**. The rest get no bet.

## Knobs — the complete map

Every dial in the e2e flow and **where its value lives — one home each (DRY).** You author the `policy` ones; never the `machine` ones. **To see or change a value, open its home file** — this table is the map, not a second copy of the numbers.

| Knob | Home (value lives here) | Scope | What it does |
|---|---|---|---|
| **return target** | this README · Goal | global · policy | sets aggression `a = k × target` |
| **goal horizon** | this README · Goal | global · policy | anchors the deployment pace + thinking timeframe |
| **loss limit** | this README · Constraints | global · policy | NAV floor; at/below → adds vetoed, de-risk only |
| **biggest bet** | this README · Constraints | global · policy | per-thesis size cap |
| **filter** | this README · Filter | global · policy | which theses get a bet at all |
| `k` | `sizing-policy.md` · Knobs | global · knob | aggression multiplier (goal → `a`) |
| `deployable` | `sizing-policy.md` · Knobs | global · knob | max fraction of NAV invested; rest cash |
| `deadband` | `sizing-policy.md` · Knobs | global · knob | min gap to trade (the no-trade band) |
| `entry_half_life` | `sizing-policy.md` · Knobs | global · knob | default deployment pace (when a bet sets no `horizon`) |
| `horizon→pace divisor` | `sizing-policy.md` · Knobs | global · knob | `entry_half_life = horizon / 4` |
| `upside` / `downside` | each thesis's `bet.yaml` | **per-bet** · policy | reward/risk ratio → bet size + `max_loss` |
| **`horizon`** | each thesis's `bet.yaml` | **per-bet** · policy | the bet's pace: `1d` all-in today … `12mo` DCA glide |
| `half_life` | `claim-verdict` (SKILL) | verdict · knob | **score** decay — how fast *belief* goes stale |
| `evidence_delta` magnitudes | `claim-verdict` (SKILL) | verdict · knob | how hard each evidence grade moves a score |
| `conviction` | `claims.yaml` | per-bet · **machine** | the verdict score (0–100); do not author |

Two time knobs, two layers: `half_life` (in `claim-verdict`) smooths **belief**; `entry_half_life`/`horizon` (in `sizing-policy.md` / `bet.yaml`) smooths **capital**.

## Action
1. Apply the [sizing policy](./sizing-policy.md) to each survivor to produce a sized **bet** each.
2. Show the bets to the user and wait for approval.
3. On approval, spawn one general-purpose agent per bet to turn it into trades (gap vs current holdings, then order specs).
4. Collect all trades and report them back.
5. The user executes manually. The machine places nothing.
