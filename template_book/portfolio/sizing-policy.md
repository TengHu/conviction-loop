# Sizing policy: conviction to bet

Convert the surviving theses into bets. Reads goal and constraints from `README.md`; owns the knobs below.

## Formula (symmetric, absolute)

Each bet is computed on its own, in two steps — the **level** today's conviction wants, then the **smoothed target** continuity actually trades toward.

**Step 1 — raw target (the level):**
```
raw_target_i = NAV x k x conviction_i x (upside_i / downside_i)
cap each at biggest_bet x NAV
if sum(raw_target) > deployable x NAV: scale all down in proportion
```

**Step 2 — smoothed target (what we trade toward), pace is per bet:**
```
target_i,t        = α_i x raw_target_i + (1 − α_i) x target_i,(t−1)
α_i               = min(1, ln2 / entry_half_life_i)              # PER BET, clamped
entry_half_life_i = horizon_i / 4    if the thesis sets `horizon`,  else the global entry_half_life default
```
`α` is **per bet**: each thesis carries its own `horizon` in `bet.yaml` — the time-scale of the bet, how you think about it — and the deployment pace is derived from it. `target_i,(t−1)` is the smoothed target from the **last run** (the `target` column in the previous dated note); seed 0 for a new thesis. From 0 this is geometric phase-in — `target = raw x (1 − (1−α)^t)` — ~50% of the level after one half-life, ~75% after two. The smoothed target is always ≤ raw, so it never breaks the cap or `deployable`.

**One authored number spans both extremes** — the patience of reaching 100% is the bet's `horizon`:

| Bet | `horizon` | entry_half_life | α | behavior |
|---|---|---|---|---|
| same-day expiry option | `1d` | 0.25d | **1.0** (clamped) | smoothing **off** — full size today, all-in now |
| swing trade | `2w` | ~2.5d | ~0.27 | ~full in days |
| core position *(default)* | *unset → global* | ~21d | ~0.033 | full in ~2–3 months |
| 12-month accumulation | `12mo` | ~63d | ~0.011 | DCA-style glide across the year |

As `horizon → 0`, `α → 1`: the recursion collapses to `target = raw_target` and you deploy the whole gap **today** (the same-day bet). As `horizon` grows, the bet crawls in. Same formula, no special cases — the bet's own time-scale picks where it sits between "bet it this morning" and "accumulate for a year."

**Fast-exit exception (symmetric with the score layer):** a thesis whose conviction is 0 from a *fired falsification* (a structural break, not a fade) sets `target = 0` and **bypasses smoothing** — exit immediately. This is the exact mirror of the score slamming to 0 (sticky) when a leg fires: smoothing filters noise, and a fired falsification is by definition not noise.

- conviction_i (0 to 1, = score / 100): machine belief.
- upside_i / downside_i: bull and bear case, as a ratio (reward over risk). Both enter symmetrically.
- k: aggression, the one dial (the README return goal sets its level).

No exp, no inverse-vol, no upside-over-downside favoritism, no normalizing across names. Same inputs always give the same bet. The survival filter lives in the README; this sizes the survivors (a broken thesis, conviction 0, also self-exits at target 0).

Asymmetry is still captured: a 10x with bounded downside has a high reward/risk ratio, so it sizes large. The capture is in the ratio, not in a thumb on the scale.

## The bet (output)

```
Bet { thesis, budget $, conviction, upside / downside, vehicle, note }
max_loss = budget x downside
```

## Over time (continuity)

```
gap_i = target_i - hold_i      trade if |gap_i| > deadband
```

`target_i` is the **smoothed** target (Step 2); `hold_i` is reconciled live from the broker each run (Plaid positions, Alpaca marks), **never stored or smoothed** — it is reality. The smoothing *is* the continuity: the target carries from the last run and moves only a fraction toward the new level, so a one-day swing in conviction barely moves the target and the deadband eats the residual — no whipsaw, no day-one dump. Buy and sell are the same rule; because each target is independent, a bet moves only when its own thesis moves; a neighbor changing never touches it. NAV at or below loss_limit: adds vetoed, de-risk only. A price move never moves a target; only evidence (via conviction) does.

## Knobs (owned here — the one home for these values; DRY)

- `k = 10` — aggression (scaled by the README return goal)
- `deployable = 0.90` — max fraction of NAV invested; the rest is cash
- `deadband = $20k` — trade threshold (the no-trade band)
- `entry_half_life = 21d` — the **global default** deployment pace, used only when a bet sets no `horizon`; sets `α = ln2 / entry_half_life`
- `horizon_to_halflife_divisor = 4` — a per-bet `horizon` sets that bet's pace as `entry_half_life = horizon / 4`

These are the live values; edit them here. The README **Knobs map** only *indexes* them (it restates no numbers).

From `README.md` (policy, its own home): return goal + horizon, loss_limit, biggest_bet, filter.

## Per thesis

Authored in **`bet.yaml`** (the sizer reads; the verifier never does):
- upside / downside — reward/risk ratio → bet size and `max_loss`.
- **horizon** — the bet's time-scale → its own deployment pace (`entry_half_life = horizon / 4`). Same-day option `1d` … 12-month accumulation `12mo`. Omit to use the global default. The only per-bet pace knob.

From **`claims.yaml`** (the verdict layer writes; `daily-run` rolls up):
- conviction — machine, **do not author** (the `min` of load-bearing scores).

## Open

- upside and downside as point estimates vs ranges.
- k fixed vs auto-scaled so the top bet just reaches its cap.
- optional per-bet loss cap (a symmetric constraint, not a bias) if you want a hard loss bound per bet.
- LEAPS imply downside near 1.0 (can go to zero).
