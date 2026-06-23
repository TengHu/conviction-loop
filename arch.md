# Loop architecture: verdict → bet → execution

Status: **canonical — the single source of truth.** This is the only doc you need to understand the system; time-smoothing (the EWMA target + per-bet `horizon`) and the slow reflection loop are folded in below.

This is the only doc you need to run the system. It covers how the loop works and why, every skill, and every artifact the loop reads and writes.

---

## One line

The loop is a **noise filter**. You curate noisy signal in; the machine atomizes it into falsifiable claims, verifies them into decaying conviction scores, and turns conviction + your written goal into sized bets, then orders. **The default output is nothing** — most things die at every gate. A trade is what survives.

---

## In my words (the flow, 2026-06-20)

> Concretely, the flow is from one end: I paste info (WeChat, X) with both noise and signal, then break down into as many theses as possible, break down into as many claims as possible, every day verdict scores for each claim, and the survivors injected into the portfolio manager — which is the other end, where I give my grand goal and constraint. Then the portfolio manager turns the crank based on surviving claims and the grand goal, comes up with actions.
>
> The current issue is:
> - a simple, unbiased mechanism that puts survivors into the portfolio manager's eye
> - a mechanism for how goal + survivor verdicts → trades

The two open issues map directly: issue 1 is **Stage 1 → the verdict boundary** (the decaying score is the filter; survivors are claims whose score is material), and issue 2 is **Stage 2 → Bet** (`conviction × reward/risk` turns goal + conviction into sized bets). Both are resolved below.

---

## Design principles (the invariants)

1. **Noise filter, not a generator.** Every stage has a null path and the null path is the default. The loop *removes* signal; it never manufactures output. A trade (buy or sell) is a rare survivor, never scheduled work.
2. **Verification-based (generate → verify).** Every seam is a generate-then-verify pair: you generate claims, the machine verifies them; the machine generates bets, you verify them. No stage acts on its own unchecked output.
3. **One stage per agent.** Each agent = **(goal, constraints, reasoning unit)**. Verdict agents decompose recursively (claims are independent). The allocator is indivisible (capital, factors, and the loss limit couple every position into one problem). The executor implements bets.
4. **Simple + unbiased mechanism.** The math is a transparent formula — reproducible, symmetric, no judgment thumb on the scale, no price input in a verdict. Same inputs always produce the same bets.
5. **You author only the goal + constraints.** Turning verdicts into positions is the framework's job. You never write a per-name size or a per-claim action.
6. **Discretion at authoring time, never at 9:30am.** You set the policy (goal + limits) in a calm session; the daily machine just executes the formula against it.
7. **Minimal inductive bias from the framework.** The framework supplies only *mechanism* — the score, the allocation math, the gap. Every *opinion* (which theses matter, the filter, how aggressive, the caps, the biggest bet) is user **policy**, authored in the goal + constraints. The framework holds no view on your book; it is a transparent pipe plus dumb math. Anywhere the framework would inject a default judgment is a smell to remove.
8. **Time is baked into the architecture — the same filter at every layer.** A verdict is a *trend over time*, not a point: scores decay by recency, so stale evidence fades and only sustained signal survives. The *same* first-order filter (`x_t = α·new + (1−α)·x_{t−1}`) runs at **two** state layers — the **score** (belief, Stage 1) and the **smoothed target** (capital, Stage 2) — so neither a belief nor a position can jump on one day's data; that is what "time-smoothing in the whole thing" means. Time is first-class everywhere: recency-decay on scores, catalyst/cadence timing on claims, the goal's *horizon* feeding both aggression and the **deployment pace**, a **per-bet `horizon`** that lets each bet pick its own pace (same-day … 12-month), and a **slow reflection loop** that distills the score *history* into durable beliefs. The system never treats today as a fresh, memoryless snapshot.

---

## The pipeline — four stages, two boundary objects

```
Stage 0  INGEST      you paste WeChat/X (noise+signal) → atomize into as many theses,
                     then as many claims, as possible                     [existing ingest skills]

Stage 1  VERDICT     each claim → a decaying score (0–100); per thesis → conviction
                     ──────── boundary object: the VERDICT ────────       [claim-verdict / daily-run]

Stage 2  BET         verdicts + your goal → a set of sized BETS
                     ──────── boundary object: the BET ────────           [allocator = portfolio]

Stage 3  EXECUTION   each bet vs what you hold → orders → you confirm      [executor = portfolio]
```

The **verdict** is what crosses from verify to allocate. The **bet** is what crosses from allocate to execute. Each agent owns one stage; the boundary object is all that passes between.

---

## Stage 1 — Verdict (claims → conviction)

**A claim's score is 0–100**, produced by verification:
- rises on confirming evidence (weighted by grade: primary > issuer-attested > shadow),
- **decays over time** as evidence goes stale (`score ×= exp(−days/half_life)`) — this decay is the low-pass filter: a one-day blip decays away before it matters; sustained evidence accumulates,
- a fired falsification leg slams it toward 0 (a structural break — sticky until the claim is rewritten).

A price move **never** changes a score. Only evidence does.

**The verdict case space** (what happens each check):

| Score behavior | Verdict | Downstream |
|---|---|---|
| nudged but no material move (blip decayed / nothing new) | unchanged | **null** |
| sustained confirming evidence | rises | feeds bigger conviction |
| sustained weakening evidence | falls | feeds smaller conviction |
| falsification leg fires | → ~0, sticky | exit that thesis |
| data missing / claim edited | frozen / reset to unknown | flagged, no trade |

**Conviction (per thesis) = the weakest load-bearing claim (`min`), 0–100.** Not a sum (a sum is biased by claim count). A thesis is only as strong as its shakiest pillar; any pillar breaking drops conviction to ~0, which is exactly how a thesis actually fails. Example: CRWV pillars `{95, 92, 90, 95}` → conviction `90`.

**Agents:** verdict agents are recursive — a thesis decomposes into claims, a compound claim into falsification legs; each is `(goal: conclude this, unit: the thing below it)`, and conclusions combine upward. `claim-verdict` is the per-thesis worker; `daily-run` is the orchestrator that fans it out.

---

## Stage 2 — Bet (verdicts + goal → bets)

**A bet is a sized intention for one thesis:**

```
Bet { thesis, budget $, conviction (0–100), max_loss $ }
```

**The manager's desk — the conviction score is the handover artifact; everything but the math is policy:**

```
all thesis scores (0–100)        # full list, nothing hidden upstream — the manager sees everything
  → FILTER + post-process        # POLICY (yours): keep a subset (e.g. conviction > floor, or top-N),
                                  #   optionally adjust scores (factor-group, normalize)
  → subset of {thesis, score}    # the survivors on the desk
  → ALLOCATE (the math)          # the formula below → raw budgets
  → CLAMP                        # POLICY (yours): biggest bet, factor caps, loss-limit veto
  → bets                         # → one executor subagent per bet (Stage 3)
```

Only **ALLOCATE** is framework-supplied; FILTER, post-process, and CLAMP are your policy (minimal inductive bias). A position you **hold** whose thesis is filtered out passes through as **bet = 0 (exit)** — never silently dropped.

**Your goal IS the policy — three numbers you author:**
- **return target** (e.g. +30%/12mo) → the **aggression** dial,
- **loss limit** (the NAV floor) → the **risk budget** (caps total + each bet's `max_loss`),
- **biggest bet** (e.g. ≤25% of NAV) → the **per-bet cap**.

**The allocation mechanism** — conviction × reward/risk, absolute and per-bet (each bet sized on its own, no cross-name normalization). All arithmetic, no judgment:

```
c_i          = conviction_i / 100                          # weakest pillar, 0..1
raw_target_i = NAV × k × c_i × (upside_i / downside_i)      # the level today's conviction wants
cap each at biggest_bet × NAV
if Σ raw_target > deployable × NAV: scale all down in proportion
```

- **Aggression is the dial.** `k` (set by the return goal) scales every bet; a higher goal → larger raw targets → more deployed, up to `deployable`.
- **Asymmetry is the risk lens, not volatility.** A bet rides its `upside/downside` ratio: a 10x with bounded downside sizes large, a coin-flip sizes small. The capture is in the ratio — no thumb on the scale, no σ term. (Inverse-vol sizing was considered and dropped; see `sizing-policy.md`.)
- **Kelly was rejected:** it needs an edge/odds we don't have; a confirmation score isn't a calibrated probability. The reward/risk ratio encodes conviction honestly without faking an edge.

**Smoothing — the same filter, one layer down, paced per bet.** The formula above gives a *raw* target (the level today's conviction wants). What the book actually trades toward is the **EWMA-smoothed** carry:

```
target_t          = α_i × raw_target_i + (1 − α_i) × target_(t−1)
α_i               = min(1, ln2 / entry_half_life_i)           # PER BET, clamped
entry_half_life_i = horizon_i / 4   (the bet's own `horizon` in bet.yaml), else the global default
```

This is the identical first-order filter the score uses to decay — applied a second time, at the capital layer ("time-smoothing in the whole thing" = the same filter at every state layer). From flat it is geometric phase-in (`raw × (1 − (1−α)^t)`): ~50% deployed at one half-life, ~75% at two — **day-one full deployment is structurally impossible unless the bet itself is same-day.** The pace is **per bet**: each thesis carries a `horizon` (its time-scale), so a `1d` option deploys in full today (α→1) while a `12mo` accumulation crawls in like DCA — one authored number, the whole range, no special cases. A *fired falsification* bypasses smoothing and exits at full speed, mirroring the score's sticky-0. Grounded in Gârleanu–Pedersen (optimal target = EWMA of the aim portfolio under trading costs), not a heuristic.

**Worked example** (book $1M, +30% goal → high `k`, biggest bet 25%; budgets illustrative under `NAV·k·c·(upside/downside)`):

| Thesis | conviction | budget (bet) |
|---|---|---|
| CRWV | 90 | $250k (hit 25% cap) |
| NOK | 60 | $150k |
| BE | 60 | $120k |
| GLD | 30 | $80k |
| cash | — | $400k |

**The allocator is indivisible** — it must see all verdicts + the whole book at once (shared capital, shared factors, one loss limit). It cannot be split into per-thesis deciders. Only its legwork (reconcile, the formula in code) is mechanical.

---

## Stage 3 — Execution (bet → orders)

**The gap is the filter:** `gap_i = budget_i − actual_holding_i` (the budget is the **smoothed** target, so the gap closes geometrically at the bet's own pace). Trade only when `|gap| > deadband`; otherwise null.

| Bet | budget | hold now | gap | order |
|---|---|---|---|---|
| CRWV | $250k | $100k | +$150k | buy toward $250k |
| NOK | $150k | $150k | 0 | none |
| BE | $120k | $200k | −$80k | sell to $120k |

- **Deterministic hard-limit pre-gate:** in a drawdown / at the loss line, adds are vetoed and only de-risk sells emit. The agent physically cannot breach a limit (ai-hedge-fund's pattern).
- **Reconcile first** (the money-saving traps: join on security_id, decode OCC options, mark options/leveraged ETFs from Alpaca not Plaid, NAV from per-holding sums, cross-check transactions).
- **You verify; the machine places nothing.** Read-only on execution, always.

**One executor subagent per bet.** Each bet spawns a subagent that computes its own `gap = budget − current holding` → an order past the deadband (a bet of 0 → the exit). Safe to parallelize: the manager already set budgets that sum within deployable capital, so no single bet can overspend globally. The **orchestrator** then collects the orders, **sequences sells before buys** (settlement), applies the loss-limit veto, and hands the set to you. The subagent owns "my bet → my order"; the orchestrator owns "order the orders."

---

## Agent map

| Stage | Agent | Decomposable? | In → Out |
|---|---|---|---|
| 1 Verdict | verdict agents | **yes** — per thesis, recursive (thesis → claims → legs) | claims → conviction (0–100) |
| 2 Bet | portfolio manager | **no — indivisible** (shared capital, factors, one loss limit) | all scores + goal → bets |
| 3 Execution | one subagent per bet | **yes** — legwork only | bet → order (orchestrator sequences) |

The pattern is **fan out → converge → fan out**: verdicts fan out per thesis, converge into one indivisible manager, then fan out per bet. The indivisible step is the only place the whole-book decision is made; everything around it parallelizes.

---

## The null paths (where things die)

| Gate | Dies (the common case) | Survives |
|---|---|---|
| ingest → claim | noise with nothing falsifiable | a testable claim |
| claim → verdict | score didn't move materially (decay ate the blip) | sustained evidence moved the score |
| verdict → bet | conviction change too small to move the budget past the deadband | a budget that pulls away from what you hold |
| bet → order | gap < deadband; or vetoed by the loss limit | a gap worth trading, within the limits |
| order → trade | you decline | you confirm |

Null is the default at every gate. The whole machine is subtractive.

---

## What you author (the only inputs)

You author **policy** in `book/portfolio/README.md`:
- **Goal:** return target + **horizon**.
- **Constraints:** loss limit (NAV floor), biggest bet (per-bet cap), factor caps, filter.

**Knobs live in the file their agent executes — one home each, DRY.** Sizing knobs (`k`, `deployable`, `deadband`, `entry_half_life`, the `horizon/4` divisor) → `sizing-policy.md`. Verdict knobs (`half_life`, `evidence_delta` magnitudes) → `claim-verdict`. The **Knobs map** in `portfolio/README.md` indexes every dial → its home; it restates no values.

Per thesis: **`bet.yaml`** holds the shape you author (`upside`, `downside`, `horizon`); **`claims.yaml`** holds the claims + falsification legs (the machine fills the scores). You never write a size or an action.

> The complete, current knob list — every dial, its home, scope, and default — lives in `portfolio/README.md` under **Knobs — the complete set**. That table is the reference; this section is the why.

---

## Memory & continuity

The loop has memory, and the memory is **git + the broker** — no database, no separate store. Three stores, each already present:

| Store | What it holds | Where | Speed |
|---|---|---|---|
| **Belief** | claim scores (0–100) | `book/theses/<slug>/claims.yaml`, git history | slow (decays over weeks) |
| **Position** | what you actually hold | the broker (Plaid/Alpaca), **reconstructed each run** | live |
| **Decision** | the delta you acted on **+ the smoothed-target carry** | `book/portfolio/notes/<DATE>.md`, git-diffable | per run |

Position is never stored: the broker is the truth, so reconcile re-reads it every run. Belief and decision are git, because they are structured numbers that must be exactly reproducible, auditable, and diffable. **gbrain is rejected as the substrate** — it is a semantic-recall store (embeddings, fuzzy search) built for unstructured knowledge, not for exact numeric state; at most it could later *index* the git book for search, never own the numbers.

**A bet is reproducible, not stored.** `bet = f(git scores, git goal, broker holdings)` — recomputed each run, no ledger to drift. The only persisted bet artifact is the dated delta note.

**Continuity is the stored decaying score *and* the stored smoothed target** (principle 8 made concrete). The same first-order filter runs at both layers — the score carries the *belief*, the smoothed target carries the *capital*. Because both *persist* in git rather than recompute from zero, neither a belief nor a position can whipsaw on one day's data, and capital phases in instead of dumping on day one:

- *Break Monday, recover Tuesday (non-structural):* Monday's weakening evidence nudges the score down by a small `evidence_delta`; Tuesday's confirming evidence adds it back. A no-evidence day only decays ~3% (`90 × exp(−1/30) ≈ 87`). The execution **deadband** absorbs the residual — no trade fires on the round trip.
- *Structural break:* a fired falsification leg slams the score to `0`, **sticky** until the claim is rewritten. This is intentional — a structural break must NOT auto-recover on a one-day bounce.

This is FinMem's two-speed model (`memory_functions/decay.py`): a slow importance score plus a fast recency score, checkpointed. Our analog — the git-stored score is the slow base, `evidence_delta` the fast nudge, git the checkpoint.

---

## Worked journey — an alpha signal over months

One signal end to end, on the mock thesis **demo-cloud** (MockCloud, `MCLD`; numbers illustrative). Book $1M, goal +30%/12mo (high `a`), biggest bet 25% (illustrative — the live knob values live in their home files, see the Knobs map). Watch one set of scores persist and decay across four months: most days are null, and the whole year holds just two trades.

| Date | What happens | Skill | Artifact written | score → conviction → gap → action |
|---|---|---|---|---|
| Mar 2 | you paste a WeChat/X thread (noise+signal) | `/pod-create-investment-thesis` | `theses/demo-cloud/{README.md, claims.yaml}` | 3 claims seeded ~70; conviction 70; no position |
| Mar 3–13 | quiet — no new evidence | `/daily-run` daily | scores in `claims.yaml` decay 70 → ~68 | conviction ~68; gap < deadband → **null** |
| Mar 14 | earnings transcript confirms "sold out" + margin inflection | `/daily-run` → `claim-verdict` | scores → 95 / 90 / 92, `score_updated: Mar 14` | conviction = min = **90**; target $250k vs hold $0; gap +$250k → **BUY** |
| Mar 14 | size the bet → order | `/portfolio` → `implement-bet` | `portfolio/notes/2026-03-14.md` (delta + order) | you confirm; fill toward $250k |
| Mar 15 | reconcile the fill | `/portfolio` | `notes/2026-03-15.md` | do-hold $250k; gap ≈ 0 → **null** |
| Mar 16 – Jun 17 | carry — slow decay, periodic IR re-ups recency | `/daily-run` daily | scores drift ~95 → ~88 | conviction ~88–90; gap < deadband → **null** (≈60 quiet days) |
| Apr 6 (Mon) | bearish blog (shadow grade) | `/daily-run` → `claim-verdict` | one score −3 | conviction dips ~2; gap < deadband → **null** |
| Apr 7 (Tue) | IR statement restores it | `/daily-run` → `claim-verdict` | that score +3 back | net ≈ 0 over two days → **no whipsaw** |
| Jun 18 | MCLD withdraws "sold out" → a falsification leg **fires** | `/daily-run` → `claim-verdict` | `cloud-2026-sold-out` score → **0 (sticky)** | conviction = min = **0**; thesis filters out → target $0 vs hold $250k; gap −$250k → **SELL-TO-CLOSE** |
| Jun 18 | size the exit → order | `/portfolio` → `implement-bet` | `notes/2026-06-18.md` (exit delta) | you confirm; flat. Score stays 0 until you rewrite the claim. |

What the timeline shows:
- **Memory is the stored score.** The same `claims.yaml` numbers carry from March to June; quiet days only decay them, nothing is recomputed from scratch.
- **Continuity over a blip (Apr 6 → 7).** A Monday dip and Tuesday recovery net to ~zero on a stored score, and the deadband eats the residual — no trade. A *structural* break (Jun 18) is sticky at 0 and does **not** auto-recover.
- **Time is the axis.** Decay (`half_life`), `score_updated` stamps, and dated `notes/<DATE>.md` make the whole journey a git-diffable trail; "today" is never a memoryless snapshot.
- **Null is the default.** ~250 trading days, two trades: one entry on a confirmed catalyst, one exit on a structural break.

> **Phase-in note.** The Mar-14 entry above is shown as a single fill for clarity. With per-bet smoothing the entry is *not* one trade — it phases in geometrically over the bet's `entry_half_life` (a tranche each time the gap re-crosses the deadband), unless the bet's `horizon` is same-day. The exit on a fired falsification still happens at full speed (smoothing bypassed).

---

## Two loops — fast reaction, slow reflection

Long-horizon coherence in an agent loop comes from *structure*, not a smarter daily step (Generative Agents; Anthropic's long-running-agent harness). Three primitives produce long-term thinking, and the system has all three:

- **Fast loop (daily):** `daily-run` verifies claims → scores decay → `portfolio` smooths the target (per-bet pace) → maybe an order. Reactive, mechanical; state carries in git so it is never a memoryless snapshot.
- **Slow loop (weekly/monthly):** `reflect` reads the *history* — the score path over weeks — and distills one **durable belief** per thesis (*held 90d across 3 catalysts → durable*; *wobbled 3 of 4 weeks → fragile*). It abstracts the pile of daily verdicts into a worldview the daily snapshot can't see.
- **The anchor:** both loops run against `portfolio/README.md`'s goal, which the daily step reads but can never rewrite — discretion at authoring time, never at 9:30am (principle 6). This read-only north-star stops local optimization from drifting.

The three primitives: **decaying carried state** (scores + smoothed targets), a **stable out-of-band objective** (the goal), and **periodic reflection** (`reflect`). Reflection is **advisory** — it feeds your next authoring session, never the sizing formula, preserving the minimal-inductive-bias invariant. Patience is already enforced mechanically by the smoothing (a fresh thesis ramps slowly); reflection adds the *vision*, not a thumb on the scale.

---

## Skills (the whole system)

All live in `.claude/skills/` as plain SKILL.md (no build). Ingest reuses existing skills; the loop is four lean ones. Verbose `pod-*` twins exist but the lean skills are canonical.

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 0 Ingest | **create-thesis** / **pod-wechat-kol** | your belief / a KOL's WeChat (noise+signal) | creates `README.md` + `bet.yaml` (shape) + `claims.yaml` (claims) |
| 1 Verdict — worker | **claim-verdict** `<slug> <claim-id>` | that claim's row in `claims.yaml` + its `fetch` source + live data | that claim's `score` + `score_updated` |
| 1 Verdict — orchestrator | **daily-run** [`verify <slug>`] | every thesis's `claims.yaml` | scores (via the worker) + a run note; rolls conviction = `min` per thesis |
| 2+3 Bet + Execute — manager | **portfolio** | `portfolio/README.md` (policy) + each thesis's `bet.yaml` (shape) + all convictions + the broker | `book/portfolio/notes/<DATE>.md` (the delta) |
| 3 Execute — worker | **implement-bet** | one bet + its reconciled holding + `deadband` + the thesis's Instructions | nothing — returns order specs; you place them |
| slow loop | **reflect** (weekly/monthly) | the score *history* (git) + the daily notes | one durable belief per thesis → `book/theses/reflections/<DATE>.md` (advisory) |

Fan-out shape: `daily-run` → agent per thesis → agent per claim → `claim-verdict`; then `portfolio` → agent per bet → `implement-bet`. The loop order is **claims-verdict first, then portfolio**.

---

## Artifacts in book/

Two homes hold the state; git is the history. Nothing else is needed to run the loop.

```
book/
├── theses/<slug>/
│   ├── README.md      narrative + "## Instructions for the next agent"
│   │                  (vehicle/hold prefs the executor honors). Ticker(s) live here.
│   ├── bet.yaml       the bet SHAPE (you author): upside, downside, horizon.
│   │                  The thesis→sizer touchpoint; the verifier never reads it.
│   └── claims.yaml    pure TRUTH (verifier reads, sizer never does):
│                      the claims — each id, claim, load_bearing,
│                      falsification {mode, legs}, fetch, score, score_updated.
├── theses/reflections/<DATE>.md  SLOW-LOOP OUTPUT — one durable belief per thesis
│                      (has it held over time?). Advisory; feeds authoring, not the math.
└── portfolio/
    ├── README.md      your policy: Goal (return+horizon), Constraints (loss limit,
    │                  biggest bet, factor caps), Filter (conviction rule),
    │                  the Knobs map (each dial → its home file), Reconcile source.
    │                  + your strategy.
    └── notes/<DATE>.md  OUTPUT — the dated delta:
                         thesis | conviction | raw_target | target | do-hold | gap | action.
                         target = smoothed (carries run-over-run; next run's prev_target).
                         The decision record AND the smoothing state; git-diffable.
```

**You author** `claims.yaml` (claims + falsification legs) and `portfolio/README.md` (policy). The machine fills the `score` fields and writes `notes/<DATE>.md`. **Positions are never stored** — reconciled live from the broker (Plaid + Alpaca) every run.

---

## Prior art (what this borrows, and what it deliberately avoids)

- **virattt/ai-hedge-fund** — steal: the deterministic action-space pre-gate (hard limits the agent can't cross). Avoid: flat label signals fed to a book-blind manager.
- **TauricResearch/TradingAgents** — steal: typed structured verdicts that combine upward. Avoid: flat single-ticker, book-blind portfolio manager.
- **AI4Finance/FinRobot** — steal: typed thesis verdict + config-as-data agents. Avoid: no analyze/decide split.
- **pipiku915/FinMem** — steal: the decaying, time-aware score + checkpoint continuity. Avoid: its crude 3-day-sign trend detector.
- **Allocation method** — conviction × reward/risk ratio (absolute, per-bet, no cross-name normalization); inverse-vol and Kelly both rejected for now. Prior art is portfolio construction, not the LLM agent repos.

---

## Open questions (deferred to tuning)

- `evidence_delta` magnitudes and `half_life` (now homed in `claim-verdict`) — tune the starting values against real claim histories.
- `k` (return target → aggression) and the per-name volatility/correlation caps — start round, calibrate.
- `min`-pillar conviction vs a softer rollup — start with `min` (matches how a thesis breaks).
- Whether `deployed`/aggression also carries a cash floor — single dial first.
