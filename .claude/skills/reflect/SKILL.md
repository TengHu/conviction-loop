---
name: reflect
description: slow loop (weekly/monthly). Reads each thesis's verdict history (the score trail in claims.yaml git history + the daily notes) and writes ONE durable belief per thesis — has it held up over time? — to book/theses/reflections/<DATE>.md. Advisory only; feeds your next authoring session, never the sizing formula.
---

# reflect — distill the verdict history into durable beliefs (slow loop)

The fast loop (`daily-run`) reacts and decays day by day. This is the slow loop: step back over weeks and ask, per thesis, **has it earned trust over time?** One durable belief each — the worldview the daily snapshot can't see. This is the only stage that reads *history* instead of today.

1. **Scope.** Every thesis in `{book}/theses/`. Default cadence weekly (or when asked).

2. **Read the trail, in code.** For each thesis, reconstruct its conviction *path*: `git log -p` the `score` fields in `claims.yaml` + the per-day lines in `book/theses/notes/<DATE>.md`. You want the shape over time, not the latest number.

3. **Judge durability** — the one thing only time reveals:
   - How long has conviction held in its band? (sustained vs fresh)
   - How many *independent* catalysts confirmed it? (proven vs one data point)
   - Has it wobbled — recovered its dips, or repeated breaks? (resilient vs fragile)

   One sentence per thesis, citing the path:
   ```
   demo-cloud: conviction 88–92 held 90d across 3 catalysts → durable.
   demo-chip:  bounced 3 of 4 weeks, never above 65 → fragile, discount.
   ```

4. **Write `book/theses/reflections/<DATE>.md`** — one durable belief per thesis, plus what the daily loop is missing (a thesis to re-author, a fading 认知, a claim that should be split). Read by your next calm authoring session — **never by the sizing math.**

## Loop
- **Goal:** one time-grounded belief per thesis — has it held up?
- **Verify before writing:** each belief cites the actual score path (dates, catalysts), not vibes; advisory only — touch no `claims.yaml` score, no portfolio file, no formula knob.
- **Done when:** the dated reflection note covers every thesis.

## Tools
- **git** (`git log -p` on `claims.yaml`) — the score history; the slow memory.
- **Read** — the daily notes for the per-day verdict trail.
