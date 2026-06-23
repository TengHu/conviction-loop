---
name: create-thesis
description: Atomize a raw signal (your belief, a WeChat/X dump, a KOL post) into ONE thesis + as many granular, falsifiable, verifiable claims as possible. Writes book/theses/<slug>/{README.md, bet.yaml, claims.yaml}. Stage 0 ingest.
---

# create-thesis — signal → thesis + falsifiable claims (Stage 0)

Input: raw signal (your belief, a KOL WeChat dump, an X post). Output: a thesis folder the loop can verify. The whole job is **atomization** — break the signal into the most granular, machine-checkable claims you can.

1. **Slug.** Pick a short kebab slug (e.g. `crwv-big-bet`). AUQ if unsure.

2. **Forcing questions** (AUQ, capture verbatim — don't editorialize). These elicit the **signal**, not numbers — the user is a sensor, not an author:
   - The bet: what is true that the market is missing?
   - Why now / the catalyst.
   - What would break it — your *rough sense* of what would prove you wrong (the framework turns this into the formal predicate + load-bearing flags; you don't author those).
   - Vehicle / hold preference, only if they have one (common, LEAPS, multi-year) — used to *derive* downside, not to size.

3. **Write two sidecars.** `bet.yaml` = the bet's **shape** — **extracted from the signal, never invented** (the user is a sensor); the sizer reads it, the verifier NEVER does. `claims.yaml` = **pure truth** (the verifier reads; the sizer never does). Keeping them apart is what makes verdict-isolation structural, not a promise.

   `book/theses/<slug>/bet.yaml` — the shape, **extracted from the signal or defaulted** (never ask the user for a number):
   - `upside`: a stated target multiple from the source ("20-30x" → 24.0; "翻倍/double" → 2.0). **Silent → 1.0** (no multiple assumed; sizes on conviction alone).
   - `downside`: a stated loss-if-broken; **silent → derive from the vehicle** (OTM/LEAPS that can expire worthless → 1.0; otherwise the conservative 1.0).
   - `horizon`: only if the source stated a *deployment pace*; otherwise **omit** → global default.
   Tag each value's basis, then AUQ to confirm the extraction — the user *approves* it, they don't originate it.
   ```
   thesis: <slug>
   upside: 1.0      # from source: "<quote>"   |  or: default — signal silent (symmetric)
   downside: 1.0    # from source: "<quote>"   |  or: default — derived from vehicle (LEAPS → 1.0)
   # horizon: 6mo   # ONLY if a deployment pace was stated; else omit → global default
   ```

   `book/theses/<slug>/claims.yaml` — atomize the bet into as many **falsifiable, verifiable** claims as the signal supports, each a single checkable predicate with a named source. **No cap: break every compound claim into its atomic facts** (prefer ten one-fact claims over three bundled ones).
   - Worth a claim: a quantitative target with a date, a catalyst-bound event, a verifiable counterparty / industry / macro fact, a contrarian "market thinks X, I think Y."
   - Not a claim: vibes, background, price levels (those are risk rules in the portfolio README, not claims).
   - **You (the framework) propose `load_bearing` — the user never marks it.** A claim is load-bearing if the bull case collapses without it (a hard catalyst or a quantitative predicate the thesis rests on); supporting color is not. Conviction = `min` over the load-bearing set, so this flag drives sizing — derive it from the bet's logic.
   - **You also draft each falsification `predicate`** — turn the user's rough "what breaks it" into the exact observable, tied to a named `fetch` source. The user supplies the signal; the framework writes the test.

   ```
   thesis: <slug>
   claims:
     - id: <kebab>
       claim: "<one falsifiable sentence>"
       load_bearing: true        # framework-proposed (bull case rests on it); user confirms
       source: self              # | kol:<handle> | filing  (attribution)
       falsification:
         mode: any               # any leg true → broken; or all
         legs:
           - class: quantitative # | textual | user-attested
             predicate: "<the exact thing that, if true, breaks it>"
       fetch: "<source to check: EDGAR / transcript / IR / Alpaca / web>"
       score: 50                 # neutral seed; first /daily-run sets it from evidence
       score_updated: <today>
   ```
   AUQ to **confirm** the drafted claims — the `load_bearing` flags and predicates included. The user approves or corrects an extraction; they don't originate it (sensor, not author).

4. **Write `book/theses/<slug>/README.md`** — the narrative (the bet + the bull/bear case behind `upside`/`downside`) and a `## Instructions for the next agent` (vehicle / hold prefs the executor honors).

## Loop
- **Goal:** a thesis + as many atomic, falsifiable claims as the signal supports, each with a predicate + a `fetch` source.
- **Verify before writing:** each claim is atomic (split compounds), falsifiable, and sourced; the shape, `load_bearing` flags, and falsification predicates are all **framework-drafted from the signal, never authored by the user** — the user only confirms; then AUQ.
- **Done when:** `bet.yaml` + `claims.yaml` + README written and you approved them.

## Tools
- **AskUserQuestion** — the forcing questions + the claims.yaml review.
- **WebSearch / WebFetch** — ground tickers, dates, and sources while atomizing (optional).
