# test_repo
Repository to test features
# DATA_ADDENDUM_V2.md — Vendor-Confirmed Data Plan

**Created:** 2026-09-18 · **Supersedes:** `DATA_FIRST.md` §3–§4 · **Patches:** `AGENTS.md` §6.1, §6.6, §7.1–§7.3, §13.1

Source: ThetaData support, written confirmation. Everything below is vendor-stated,
not inferred.

---

## 1. Subscriptions required

**Two, not one.**

| Product | Tier | Covers | First access |
|---|---|---|---|
| **Options** | Value (~$40/mo) | SPXW chain: EOD, OHLC, **Quote**, Open Interest | 2020-01-01 |
| **Index** | Value (price TBC) | **SPX 1-min spot, VIX 1-min, SPX EOD close** | 2023-01-01 |

Index is a **separate product** — an Options plan does not include SPX or VIX.
This is not optional: three hard requirements depend on it (spot series for every
trigger, VIX for the halt rules, and the settlement close).

Both first-access dates comfortably cover the 2023-09 lookback start.

**Confirm the Index Value monthly price on the pricing page before committing** —
support declined to quote it, correctly.

---

## 2. Settlement — vendor correction, adopt it

`AGENTS.md` §7.2 said "official SPXW PM settlement value." The precise, correct
definition per vendor:

> SPXW settles to the **official closing value of the SPX index**, from
> `index/history/eod` → `close` field. ThetaData publishes no separate settlement
> print on the option itself.

**The detail that matters:** the SPX index keeps printing until roughly
**16:04–16:05 ET** as closing prices arrive. The `index/history/eod` close is the
final settlement value; the 16:00:00 quote is **not**.

This confirms the §7.2 hazard and sharpens it — using the 16:00 print would bias
every held fly in the same direction. Also confirmed: **SET** is the AM-settled
standard SPX symbol, not SPXW, which validates the §2 guard.

**Patch `AGENTS.md` §3.8 and §7.2** to name `index/history/eod.close` explicitly.

---

## 3. Quote semantics — confirmed, with consequences

At Value, `interval=1m` returns the **last NBBO at that timestamp** — prevailing
bid, ask, sizes. A point-in-time **snapshot**, not an OHLC bar. No intraminute
high or low.

The option OHLC endpoint *is* included at Value, but its high/low come from
**trades**, not quotes. There is no NBBO high/low aggregate. True tick-level
quotes require Standard or above; Value granularity is capped at 1 minute.

### What this is fine for

Entry credit, marks, exit pricing, wing selection. All of these read the
prevailing NBBO, which is exactly what a snapshot gives. The `mid + 0.10` fill
model is unaffected.

### What it constrains

**Exit and entry pricing carries up to 60 seconds of latency.** A trigger firing
between snapshots is priced at the next snapshot, not at the trigger moment. This
is a real cost and must be modelled, not ignored — it is closer to live behaviour
than instantaneous fills would be, but it is a systematic drag.

**Patch `AGENTS.md` §6.5:** add `snapshot_latency` to the fill model. Default:
price at the next available snapshot after the trigger.

---

## 4. The open question — blocking

Everything in §3 concerns *option* data. But **every trigger in this strategy
reads SPX, not option prices**: adds at ±7/±10/±15 from the anchor center, drops
on a touch of a center.

So the question that decides the entire backtest is:

> **Does `index/history/price` with `interval=1m` return an OHLC bar (open, high,
> low, close) per minute, or a point-in-time snapshot like the option quote
> endpoint? If snapshot-only, is there an `index/history/ohlc` endpoint at
> 1-minute granularity available on Index Value?**

### Why it is blocking

| If SPX 1-min is… | Consequence |
|---|---|
| **OHLC (high/low available)** | Intrabar touch detection works. M5 runs as specified: conservative vs. optimistic ordering within the bar. |
| **Snapshot only** | You can only detect touches and add triggers **at minute boundaries**. Any touch that occurs and reverts inside a minute is invisible. |

The snapshot-only case is a **systematic bias, not noise**. It under-counts both
drops and adds — and the source author explicitly describes touches that revert
before a human can click, which is exactly the population that would vanish.

It does not necessarily kill the project. Minute-boundary detection is arguably
closer to what a human operator actually reacts to. But it changes M5 from
*"which order did events occur within the bar?"* to *"how much does the strategy
change when sub-minute touches are invisible?"* — a different test with a
different implementation.

**Ask this before launching D4.** Take their offer to walk through a specific
SPXW date and request an SPX index sample for the same date.

---

## 5. Confirmed request shape

```
GET /v3/option/history/quote
    root=SPXW
    expiration=*
    strike_range=30        # 30 above + 30 below + ATM = 61 strikes
    max_dte=0              # same-day expiries only
    right=both
    interval=1m
    date=YYYYMMDD
```

Vendor-confirmed: all parameters supported, endpoint minimum tier is Value, this
is "exactly the supported shape." One request per date, ~750 requests total.

`strike_range=30` ≈ ±150 points on a 5-point grid — matches `AGENTS.md` §6.2.

**Build on v3.** Vendor-recommended for new projects; all parameters above are
v3, and legacy v2 reader issues do not apply.

---

## 6. Bulk — resolved, no upgrade needed

Two meanings were conflated. The one that matters is confirmed available:

| Meaning | Tier | Relevant? |
|---|---|---|
| Full chain for one underlying, one request (`expiration=*`) | **Value** | **Yes — this is what we need** |
| Flat Files: whole options market, one date, one file | Professional, **7 most recent days only** | No |

Flat Files cover only the last 7 calendar days, so they are useless for a 2-year
backtest regardless of tier. **Do not upgrade to Professional for "bulk."**

---

## 7. Request types at Value

Four historical types, not three: **EOD** (free on all tiers), **OHLC**,
**Quote**, **Open Interest**. Quote is confirmed included.

Trade, Trade Quote, and Greeks require Standard+. Per `AGENTS.md` §6.3 the
strategy needs none of them — every trigger is price-distance based, and IV is
computed from quote midpoints.

---

## 8. Upgrade path if M5 fails

If the path-sensitivity gate fails at 1-minute, the remedy is **not** a vendor
change. Standard (~$80) unlocks tick-level data on the same API you will already
have built against. That is a $40/mo delta on a codebase that needs no
restructuring — only the loader changes.

This materially de-risks starting at Value.

---

## 9. Patch list

| File | Section | Change |
|---|---|---|
| `AGENTS.md` | §3.8, §7.2 | Settlement = `index/history/eod.close`; note the 16:04–16:05 print window; 16:00 quote is not settlement |
| `AGENTS.md` | §6.1 | Add Index Value as a required second subscription; SPX and VIX sourced there |
| `AGENTS.md` | §6.5 | Add `snapshot_latency` — trigger prices at next 1-min snapshot |
| `AGENTS.md` | §6.6 | Record confirmed request shape (§5); note Flat Files irrelevant; build on v3 |
| `AGENTS.md` | §7.1, §7.3 | Reframe intrabar hazard pending the §4 answer |
| `AGENTS.md` | §13.1 | M5 test definition depends on the §4 answer — do not implement until resolved |
| `DATA_FIRST.md` | §3 | Add Index Value to the pre-flight checklist |
| `DATA_FIRST.md` | §5 D1 | Probe must also pull an SPX index sample and report its shape |

---

## 10. Revised pre-flight checklist

- [ ] Subscribe **Options Value** (~$40/mo)
- [ ] Subscribe **Index Value** — confirm price on the pricing page
- [ ] Install and launch **Theta Terminal v3**
- [ ] Confirm the terminal's printed access levels match both subscriptions
- [ ] **Ask the §4 question** — SPX 1-min OHLC or snapshot?
- [ ] Accept the offer to review one SPXW date; request an SPX index sample too
- [ ] Verify Index Value reaches 2023-09 (stated: 2023-01-01)
- [ ] Free ~5 GB disk

**Do not launch D4 until the §4 answer is in.** It determines how M5 is built,
and M5 gates everything after it.

# USER_GUIDE.md — Operating Manual

**For:** you, the operator · **Not for:** opencode (that's `PROMPTS.md`)
**Companions:** `PROJECT.md` (scope) · `AGENTS.md` (rules) · `PROMPTS.md` (build prompts)
**Revision:** v1 · 2026-09-17

---

## 0. The 60-second version

You are building a backtest to answer one question: **does the add/drop cycle
actually earn anything, or does it just reshape a fair-value premium sale into
many small wins and rare large losses?**

Your job at each stage is **not** to find the best number. It is to decide
whether the number in front of you is real. This guide tells you what to look at
and what it means.

**Three rules that protect the whole project:**

| Rule | Why |
|---|---|
| Run the holdout **once** | There is no recovery. Burn it and you wait for new market data. |
| Adopt nothing under **1σ** | Below that, you are selecting noise and calling it a decision. |
| Never change a rule to fix a result | That converts a failed strategy into a fitted one. |

**A negative verdict is a successful project.** You will have spent ~$120 and a
few weeks to avoid risking $25,000 on a strategy that does not survive contact
with real fills.

---

## 1. Before you start

### What you need

| Item | Notes |
|---|---|
| Python 3.11+ | Standard scientific stack |
| opencode | Any capable coding agent works |
| ThetaData Options Value | ~$40/mo. **Do not buy until M5 passes** — see §4 |
| Theta Terminal | Local process; data will not flow without it |
| ~5 GB disk | Raw cache plus Parquet |
| Tradier account | Phase 2 only, not needed for the backtest |

### Files in the repo root

```
AGENTS.md      <- opencode reads this automatically
PROJECT.md     <- scope and MVP definition
PROMPTS.md     <- your copy-paste source
USER_GUIDE.md  <- this file, for human reference
```

### Time budget, realistically

| Phase | Effort |
|---|---|
| **D1** | Subscribe; verify bulk endpoints + history depth; probe one day | ~122 contracts × 390 min, continuous quotes |
| **D2** | Downloader: raw cache, checkpoint/resume, Parquet | 20 pilot days on disk |
| **D3** | Quality gates (`AGENTS.md` §6.4) + loader with holdout guard | All six gates pass on pilot |
| **D4** | **Full pull** — unattended, hours | Manifest complete, gates pass across range |
| M1 — rules engine | 1–2 sessions |
| M3–M4 — engine and scenarios | 2–3 sessions. **This is where bugs hide** |
| M5 — path gate | 1 session |
| M6 — staged runs | 3–5 sessions, one stage at a time |
| M7–M8 — holdout and verdict | 1 session |

Do not compress M3–M4. Everything downstream inherits those bugs.

---

## 2. How to run a session

1. Open opencode in the repo.
2. **Paste P0 from `PROMPTS.md`.** Every session, no exceptions — context resets
   and drift starts there.
3. Check the four comprehension answers. Wrong answers → re-paste, don't proceed.
4. Paste the milestone prompt.
5. Verify the acceptance check **yourself**. Do not accept "tests pass" as a
   report; run them.
6. Commit before moving on.

### When opencode pushes back

It will eventually suggest relaxing something — loosening a test tolerance,
skipping a quality gate, adjusting a threshold so a scenario matches.

**The answer is always no.** If you genuinely think a rule is wrong, stop, edit
`AGENTS.md` yourself, note why, then re-run. Rules change by your decision,
recorded in the spec — never inside a coding session to make something pass.

---
## 3.1 D0-D4 — data

Two things to check personally.

**The probe output**, before the full pull:
- ~122 contracts on a normal day
- 390 minutes of coverage
- continuous quotes, including on wings that never traded
- **bulk endpoints available at the Value tier**

That last one is a real risk. Without bulk access you are pulling ~122 contracts
× ~500 days one at a time, which may exceed rate limits and turn a few hours into
a few days.

**The holdout guard**, by trying to break it:
```python
loader.load("2026-03-15")             # must raise
loader.load("2026-03-15", stage=8)    # must succeed
```
This is the one guardrail with no recovery path. Test it before you need it.

## 3.2 M1–M4: building and verifying


### M1 — rules engine

The whole milestone runs with **no data and no network**. If opencode asks for a
data file here, purity has been violated somewhere.

**Verify yourself:**
```bash
pytest tests/ -v          # all green
grep -rn "now()\|open(\|requests" core/rules.py core/sizing.py   # expect nothing
```

Then open `config/default.yaml` and find `be_pct`, the three tier distances, the
add offset, `bp_utilization`, `target_flies`. If you cannot find them without
reading Python, the config extraction is incomplete.

**The test that matters most:** stop unreachability. It encodes the measured
finding that the −25% stop fires 13–15 points wider than the price exit at every
hour. If it fails, either the pricing model or the rule is wrong — do not let
anyone "fix" it by adjusting the test.



### M3–M4 — engine and scenarios

**Read one full event log by hand.** Not the summary — the log. You are checking
that the narrative makes sense: entry at 11:00, adds at plausible distances,
drops when price returns, the 14:00 switch, 2–3 flies into settlement.

**Scenario A and B must reproduce the fly sequence exactly.** P&L within a loose
band is fine — the source gives strike paths, not the vol surface. But a sequence
divergence is an engine bug, and everything downstream is meaningless until it's
fixed.

| Symptom | Likely cause |
|---|---|
| Sequence diverges at the first add | Tier selection or distance measurement |
| Diverges after a drop | Anchor not recentering (`AGENTS.md` §3.2) |
| Extra fly before 14:00 | Two-fly cap or evaluation order |
| Only ever 1 fly | Adds not firing — check the whole add path |
| P&L wildly off, sequence right | Settlement using 16:00 quote instead of PM value |

---

## 4. M5 — the path-sensitivity gate 🛑

**The single most important number in the project.** Do not buy the full
subscription before this.

### What it asks

1-minute bars give you the high and the low, but not the order they happened in.
When a bar both touches a fly center (drop) and extends past an add trigger, the
outcome depends on sequence — and you cannot know it. This runs the strategy
under both assumptions and measures how much that ambiguity is worth.

### What to do

```
spread = |pnl_conservative − pnl_optimistic| / mean(both)
```

| Spread | Meaning | Action |
|---|---|---|
| **< 25%** | Path ambiguity is minor | Proceed with confidence |
| **25–50%** | Material but tolerable | Proceed; treat later single-stage results with suspicion |
| **> 50%** | Ambiguity exceeds the effect you are measuring | 🛑 **STOP** |

Also look at the **same-bar collision count** — bars where both a drop and an add
triggered. A high count with a passing spread means you were lucky, not safe.

### If it fails

This is a data-resolution finding, not a bug. Two honest options:

1. **Buy tick or trade-level data** and re-run. More expensive, resolves the
   ambiguity.
2. **Stop.** The strategy may be fine; you just can't measure it with the data
   available at this price.

There is no third option. Proceeding anyway produces numbers with error bars
wider than the thing you're trying to detect — and you will believe them, because
they'll look like results.

---

## 5. M6 — reading staged results

### First, get σ

Before Stage 2, the bootstrap gives you the standard deviation of profit factor
across 10 resamples of the train window. **Write it on a sticky note.**

Everything after this is judged against it. Typical values land around 0.1–0.3 —
if σ comes back at 0.5+, your sample is too noisy for fine distinctions and you
should only trust large effects.

### The adoption test, every stage

```
adopt only if (variant_PF − base_PF) > 1σ
```

On ties, or anything close, **keep the simpler config**. Every variant adopted is
a degree of freedom spent, and you have a fixed budget of them.

### Stage 2 — the baseline and the defect checklist

Before reading any P&L, run the checklist. These are bug signals, not results:

| Check | Bad sign | What it means |
|---|---|---|
| `stop_25` count | **Any material count** | Bug. It's measurably unreachable (`AGENTS.md` §3.7) |
| Fly count | Routinely 1 | Adds not firing — the strategy isn't running |
| Scenario tests | Now failing | Engine regressed since M4 |
| Quality gates | Failures on test days | You're replaying bad data |
| Settlement | Not official PM value | Every held fly is biased the same direction |

**Only after a clean checklist does the P&L mean anything.**

Then read the headline at `realistic` × `conservative`:

| Profit factor | Read | Action |
|---|---|---|
| > 1.2 | Promising, pre-tuning | Proceed |
| 1.0–1.2 | Marginal | Proceed; Stage 3 matters a lot |
| 0.9–1.0 | Slightly negative | Proceed — a real defect plausibly eats it |
| 0.6–0.9 | Clearly negative | Proceed **only** if Stage 3 gives a mechanical explanation |
| < 0.6 | Premise is wrong | 🛑 Stop. No exit tweak recovers that |

**Also check concentration.** Three catastrophic days against 497 profitable ones
means tail management is broken — fixable. Steady daily erosion means you're
selling fair-value gamma and paying spread to do it — not fixable by parameters.

### Stage 3 — exits (the highest-value stage)

Look at **profitable-breakeven-exit share by hour** first. Under `static_85` you
should see it climb sharply in the last two hours. That confirms the documented
inversion: your "loss cutter" has become a profit-taker during the window where
the strategy claims most of its theta.

If `time_scaled` or `dual_mode` beats `static_85` by more than 1σ, that's a real
fix for an identified defect — the most trustworthy kind of improvement, because
the hypothesis was written down before the run.

Expect `stop_basis: none` to perform identically to `credit`. That's the
unreachability finding confirming itself. If they differ materially, something is
wrong.

### Stage 4 — calendar filters

**Judge on the tail, not the mean.** These remove a handful of days and should
barely move the average.

| Metric | Adopt if |
|---|---|
| Worst-5-day P&L | Materially improved |
| Max drawdown | Reduced |
| Mean P&L | Ignore — this isn't what the filter is for |

If skipping CPI cuts your worst day in half and costs 2% of mean return, take it.
That's the trade the source author recommends.

### Stage 5 — volatility halts

The key output is **slippage on halt exits vs. normal exits**. This converts a
doctrinal argument into a number: market-exiting 3–5 flies into a dislocation
with blown-out 0DTE spreads has a cost, and now you'll know it.

If `halt_adds_only` matches `flatten` on drawdown but avoids the slippage, it
wins — you keep the risk control and lose the forced liquidation.

`max_vix: none` tells you how censored your sample was. If the extra days are
catastrophic, the filter is earning its keep. If they're merely mediocre, you've
been excluding tradeable days for nothing.

### Stage 6 — structure

If Stage 1's spread was wide, **`touch_confirm` should have been dropped from the
matrix.** Confirm that happened. Testing it on data that can't resolve it
generates numbers that mean nothing, and you'd be adopting one.

`vol_capped` should show a tighter max-risk distribution than `premium_only`.
That's the point — it fixes wings widening exactly when vol rises.

### Stage 7 — sizing

**The sanity check comes first:** below ~$50k equity, `fixed_1` and
`weekly_scaling` must produce *identical* curves. If they differ in the train
window, that's a bug — stop and investigate before reading anything else.

Then look at drawdown **as a percentage of equity**, and at days spent within 10%
of the $25,000 PDT threshold. If the equity curve lives near that line, you have
no buffer: the drop rule round-trips constantly, and below $25k it can't.

---

## 6. M7 — the holdout 🛑

### Before you run it

- [ ] Ledger complete, every train run recorded
- [ ] Top 3 configs chosen and **written down**
- [ ] Baseline config identified
- [ ] You have accepted that this is the answer, whatever it says

### Running it

Six runs. `fill=realistic`, `intrabar=conservative`, `stage=8`. No tuning, no
re-runs, no "one more variant."

**If something looks wrong, report it — do not fix and re-run.** A second pass
burns the holdout permanently and there's no way to un-see the result.

### Reading it

| Outcome | Meaning | Action |
|---|---|---|
| Train winner also wins holdout, similar magnitude | Real | Adopt it |
| Train winner wins, much smaller margin | Partly real, partly fitted | Adopt cautiously; expect the smaller number |
| **Rankings invert** | Train result was noise | **Ship the baseline** |
| Everything negative on holdout | Regime-dependent or no edge | Do not trade |

Rank inversion is the outcome people rationalize. Don't. It's the holdout doing
exactly the job you built it for.

---

## 7. M8 — the verdict

Write it, then read it as someone who **wants the strategy to fail**. Would they
find it convincing?

Questions the verdict must answer plainly:

1. Did the add/drop cycle earn anything, or just reshape the distribution?
2. **What did whipsaw cost?** This is the most direct measure of whether the
   cycle earns its keep.
3. Where did the P&L actually come from — tail avoidance, exit discipline, or
   the add/drop mechanism?
4. How much of the result depends on assumptions you couldn't verify?

**If the answer is "this doesn't survive realistic fills," you are done and the
project succeeded.** Do not tune further. Do not add parameters. Do not re-run
the holdout. That is the outcome the whole structure was built to detect
honestly.

---

## 8. Troubleshooting

| Symptom | Likely cause | Where to look |
|---|---|---|
| `stop_25` firing | Price exit not evaluating, or wrong eval order | `AGENTS.md` §3.5, §3.7 |
| Only 1 fly ever | Adds not firing | Tier selection, anchor, distance calc |
| 3 flies before 14:00 | Cap or ordering broken | §3.4, §3.5 |
| P&L too good | Settlement or marking bug | §7.2 — official PM value? |
| Win rate > 85% | Almost certainly a bug on short gamma | Run the result-sanity audit |
| Scenario sequence drifts | Anchor not recentering | §3.2 |
| Fills better than model | Fill mode not applied | §6.5 |
| Drawdown < largest daily loss | Impossible; accounting bug | Equity curve construction |
| Downloader stalls | Theta Terminal down | Restart; resume from manifest |
| Quality gates fail on many days | Wrong strike range or missing expiries | §6.2, §6.4 |

**General principle:** on a short-gamma strategy, a surprisingly *good* number is
evidence of a bug until proven otherwise. Surprisingly bad numbers are usually
real.

---

## 9. Discipline checklist

Print this. Check it before every stage.

- [ ] P0 pasted this session
- [ ] σ written down and being applied
- [ ] Nothing adopted below 1σ
- [ ] No rule changed to make a test or result pass
- [ ] Holdout untouched (until M7)
- [ ] Ledger has `git_sha` and `config_hash` for every run
- [ ] Headline quoted at `realistic` × `conservative` only
- [ ] Run count ≤ 48 total
- [ ] Defect checklist clean before interpreting any P&L

**If you break one of these, write down which and when.** A documented deviation
is recoverable; a forgotten one silently invalidates the verdict.

---

## 10. Quick reference

| Milestone | You verify | Gate |
|---|---|---|
| M1 | Tests green with **no data files** | Purity holds |
| M2 | Six quality gates; holdout guard raises | Data trustworthy |
| M3 | Event log reads sensibly; ordering test passes | Engine sane |
| M4 | Scenario **sequences** reproduce | 🛑 Engine correct |
| M5 | Path spread **< 50%** | 🛑 Data sufficient |
| M6 | σ applied every stage; ≤42 runs | Discipline held |
| M7 | Six runs, one pass | 🛑 Holdout spent |
| M8 | Verdict a skeptic would accept | 🛑 **Sign-off** |
| M9 | ≥20 paper sessions | Live behaviour matches |
| M10 | Start at 1 contract | — |

### The four numbers that decide everything

| Number | Where | Decides |
|---|---|---|
| **Path spread** | M5 | Whether 1-min data can answer the question at all |
| **σ** | M6.0 | What counts as a real improvement |
| **Whipsaw cost** | Every run | Whether the add/drop cycle earns its keep |
| **Holdout profit factor** | M7 | Whether any of it was real |

### If you remember nothing else

Run the holdout once. Adopt nothing under 1σ. Never change a rule to fix a
result. A negative verdict is a win.
