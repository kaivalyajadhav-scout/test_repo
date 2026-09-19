# USER_GUIDE.md — Operating Manual

**For:** you, the operator · **Not for:** opencode (that's `PROMPTS.md`)
**Companions:** `PROJECT.md` (scope) · `AGENTS.md` v5 (rules) · `PROMPTS.md` (build prompts)
**Revision:** v2 · 2026-09-18

---

## 0. The 60-second version

You are building a backtest to answer one question: **does the add/drop cycle
actually earn anything, or does it just reshape a fair-value premium sale into
many small wins and rare large losses?**

Your job at each stage is **not** to find the best number. It is to decide
whether the number in front of you is real.

**Three rules that protect the whole project:**

| Rule | Why |
|---|---|
| Run the holdout **once** | No recovery. Burn it and you wait for new market data. |
| Adopt nothing under **1σ** | Below that you are selecting noise and calling it a decision. |
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
| opencode | Any capable coding agent |
| ThetaData **Options Value** | ~$40/mo — SPXW chain quotes |
| ThetaData **Index Value** | **Mandatory, separate product.** SPX 1-min spot, VIX, settlement close. Price TBC on their pricing page |
| Theta Terminal **v3** | Local process; no data flows without it |
| ~5 GB disk | Raw cache plus Parquet |
| Tradier account | Phase 2 only |

Both subscriptions are required. An Options plan does **not** include SPX or VIX,
and three hard requirements depend on Index: the spot series driving every
trigger, VIX for the halt rules, and the settlement close.

### Files in the repo root

```
AGENTS.md      <- opencode reads this automatically
PROJECT.md     <- scope and MVP definition
PROMPTS.md     <- your copy-paste source
USER_GUIDE.md  <- this file, for you
```

### Time budget, realistically

| Phase | Effort |
|---|---|
| D0–D3 — setup, probe, pilot, gates | 1–2 sessions |
| D4 — full pull | **Hours, unattended** |
| M1 — rules engine | 1–2 sessions — **run concurrently with D4** |
| M3–M4 — engine and scenarios | 2–3 sessions. **This is where bugs hide** |
| M5 — path gate | 1 session |
| M6 — staged runs | 3–5 sessions, one stage at a time |
| M7–M8 — holdout and verdict | 1 session |

Do not compress M3–M4. Everything downstream inherits those bugs.

---

## 2. How to run a session

1. Open opencode in the repo.
2. **Paste P0 from `PROMPTS.md`.** Every session — context resets and drift
   starts there.
3. Check the six comprehension answers. Wrong answers → re-paste, don't proceed.
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

## 3. D0–D4: getting trustworthy data

### D0 — environment and the blocking question

Three things you verify personally:

1. **Terminal access levels at startup.** Both products visible. Options
   historical reaching 2020-01-01, index reaching 2023-01-01.
2. **The §6.7 answer.** Does `index/history/price` at `interval=1m` return an
   **OHLC bar** (open/high/low/close) or a **single point-in-time value** per
   minute? Make opencode print the raw response and quote the actual field
   names. Do not let it infer this from documentation.
3. **Both subscriptions billing.** Confirm Index Value is actually active, not
   just added to a cart.

**Why #2 matters more than anything else in the D phase:**

| If SPX 1-min is… | Consequence |
|---|---|
| **OHLC** | Intrabar touch detection works. M5 runs as originally specified. |
| **Snapshot only** | Touches detectable **only at minute boundaries**. Any touch that occurs and reverts inside a minute is **invisible** — a systematic bias that under-counts both drops and adds. |

The source author explicitly describes touches that revert before he can click.
Under snapshot-only, that entire population vanishes from the backtest.

It does not necessarily kill the project — minute-boundary detection is arguably
closer to what a human operator reacts to. But it changes what M5 is testing, so
settle it before opencode writes that code.

### D1–D2 — probe, then pilot

**Check in the probe output:**
- ~122 contracts on a normal day, 390 minutes
- continuous quotes, **including on wings that never traded**
- SPX index sample with field names visible
- SPX `eod.close` and the timestamp of the final print (expect ~16:04–16:05 ET)

Then pull **20 pilot days only**. Extrapolate elapsed time × 25 for the full
pull. A schema or coverage mistake replicated across 500 days is hours wasted.

### D3 — gates and the guard

All six quality gates pass on the pilot days. Then **break the holdout guard
yourself**:

```python
loader.load("2026-03-15")             # must raise
loader.load("2026-03-15", stage=8)    # must succeed
```

This is the one guardrail with no recovery path. Test it before you need it.

### D4 — full pull

Launch it and **start M1 immediately**. The rules layer is pure and needs no
data, so the download and the engine build overlap. That overlap is the whole
point of the data-first ordering.

---

## 4. M1–M4: building and verifying

### M1 — rules engine

Runs with **no data and no network**. If opencode asks for a data file here,
purity has been violated.

```bash
pytest tests/ -v
grep -rn "now()\|open(\|requests" core/rules.py core/sizing.py   # expect nothing
```

Open `config/default.yaml` and find `be_pct`, the three tier distances, the add
offset, `bp_utilization`, `target_flies`. If you can't find them without reading
Python, config extraction is incomplete.

**The test that matters most:** stop unreachability. It encodes the measured
finding that the −25% stop fires 13–15 points wider than the price exit at every
hour. If it fails, either the pricing model or the rule is wrong — do not let
anyone "fix" it by adjusting the test.

### M3–M4 — engine and scenarios

**Read one full event log by hand.** Not the summary — the log. Entry at 11:00,
adds at plausible distances, drops when price returns, the 14:00 switch, 2–3
flies into settlement.

**Scenarios A and B must reproduce the fly sequence exactly.** P&L within a loose
band is fine — the source gives strike paths, not the vol surface. A sequence
divergence is an engine bug, and everything downstream is meaningless until fixed.

| Symptom | Likely cause |
|---|---|
| Sequence diverges at first add | Tier selection or distance measurement |
| Diverges after a drop | Anchor not recentering (§3.2) |
| Extra fly before 14:00 | Two-fly cap or evaluation order |
| Only ever 1 fly | Adds not firing — check the whole add path |
| **P&L off, sequence right** | **Settlement using 16:00 quote, not `eod.close`** |
| Exits consistently worse than expected | `snapshot_latency` not modelled |
| Win rate > 85% | Almost certainly a bug on short gamma |

---

## 5. M5 — the path-sensitivity gate 🛑

**The single most important number in the project.**

### What it asks

**Its definition depends on the D0 §6.7 answer.** If SPX 1-min is OHLC, M5
measures intrabar event ordering — you know the high and the low but not which
came first. If snapshot-only, it measures sensitivity to *unobservable*
sub-minute excursions. Different test, same threshold.

### What to do

```
spread = |pnl_conservative − pnl_optimistic| / mean(both)
```

| Spread | Meaning | Action |
|---|---|---|
| **< 25%** | Ambiguity is minor | Proceed with confidence |
| **25–50%** | Material but tolerable | Proceed; treat single-stage results with suspicion |
| **> 50%** | Ambiguity exceeds the effect you're measuring | 🛑 **STOP** |

Also check the **collision count** — intervals where both a drop and an add
triggered. A high count with a passing spread means you were lucky, not safe.

### If it fails

1. **Upgrade to ThetaData Standard (~$80/mo).** Tick-level data on the same API
   you've already built against — only the loader changes. This is a $40/mo
   delta, not a vendor migration.
2. **Stop.**

Proceeding anyway produces numbers with error bars wider than the thing you're
trying to detect — and you will believe them, because they'll look like results.

---

## 6. M6 — reading staged results

### First, get σ

Before Stage 2, the bootstrap gives you the standard deviation of profit factor
across 10 resamples of train. **Write it down.** Typical values land around
0.1–0.3; if σ comes back at 0.5+, your sample is too noisy for fine distinctions
and you should only trust large effects.

```
adopt only if (variant_PF − base_PF) > 1σ
```

On ties or anything close, **keep the simpler config**.

### Stage 2 — the defect checklist comes first

Before reading any P&L:

| Check | Bad sign | Meaning |
|---|---|---|
| `stop_25` count | **Any material count** | Bug — it's measurably unreachable (§3.7) |
| Fly count | Routinely 1 | Adds not firing; the strategy isn't running |
| Scenario tests | Now failing | Engine regressed since M4 |
| Quality gates | Failures on test days | You're replaying bad data |
| Settlement | Not `eod.close` | Every held fly biased the same direction |

**Only after a clean checklist does P&L mean anything.**

| Profit factor | Action |
|---|---|
| > 1.2 | Promising pre-tuning — proceed |
| 1.0–1.2 | Marginal — Stage 3 matters a lot |
| 0.9–1.0 | Proceed; a real defect plausibly eats it |
| 0.6–0.9 | Proceed **only** with a mechanical explanation from Stage 3 |
| < 0.6 | 🛑 Stop. No exit tweak recovers that |

**Check concentration.** Three catastrophic days against 497 profitable ones means
tail management is broken — fixable. Steady daily erosion means you're selling
fair-value gamma and paying spread — not fixable by parameters.

### Stage 3 — exits (highest value)

Look at **profitable-breakeven-exit share by hour** first. Under `static_85` it
should climb sharply in the last two hours, confirming the documented inversion:
your loss-cutter has become a profit-taker during the window the strategy claims
most of its theta.

Expect `stop_basis: none` ≈ `credit`. That's the unreachability finding
confirming itself. If they differ materially, something is wrong.

### Stage 4 — calendar filters

**Judge on the tail, not the mean.** If skipping CPI halves your worst day and
costs 2% of mean return, take it.

### Stage 5 — volatility halts

Key output is **slippage on halt exits vs. normal exits**. If `halt_adds_only`
matches `flatten` on drawdown but avoids the slippage, it wins. `max_vix: none`
tells you how censored the sample was.

### Stage 6 — structure

If Stage 1's spread was wide, **confirm `touch_confirm` was dropped** from the
matrix. Testing it on data that can't resolve it generates numbers that mean
nothing — and you'd adopt one.

### Stage 7 — sizing

**Sanity check first:** below ~$50k equity, `fixed_1` and `weekly_scaling` must
be *identical*. A difference in train is a bug — investigate before reading
anything else.

Then drawdown as **% of equity**, and days spent within 10% of the $25,000 PDT
threshold. If the curve lives near that line you have no buffer.

---

## 7. M7 — the holdout 🛑

**Before:** ledger complete, top 3 configs written down, baseline identified, and
you have accepted that this is the answer whatever it says.

Six runs. `fill=realistic`, `intrabar=conservative`, `stage=8`. No tuning, no
re-runs. **If something looks wrong, report it — do not fix and re-run.**

| Outcome | Action |
|---|---|
| Train winner wins holdout, similar magnitude | Adopt |
| Wins with much smaller margin | Adopt cautiously; expect the smaller number |
| **Rankings invert** | **Ship the baseline** — train was noise |
| Everything negative | Do not trade |

Rank inversion is the outcome people rationalize. Don't. It's the holdout doing
exactly the job you built it for.

---

## 8. M8 — the verdict

Write it, then read it as someone who **wants the strategy to fail**.

1. Did the add/drop cycle earn anything, or just reshape the distribution?
2. **What did whipsaw cost?**
3. Where did P&L actually come from — tail avoidance, exit discipline, or the
   add/drop mechanism?
4. How much depends on assumptions you couldn't verify?

**If the answer is "this doesn't survive realistic fills," you are done and the
project succeeded.** Do not tune further, add parameters, or re-run the holdout.

---

## 9. Discipline checklist

- [ ] P0 pasted this session
- [ ] σ written down and being applied
- [ ] Nothing adopted below 1σ
- [ ] No rule changed to make a test or result pass
- [ ] Holdout untouched (until M7)
- [ ] Ledger has `git_sha` and `config_hash` for every run
- [ ] Headline quoted at `realistic` × `conservative` only
- [ ] Run count ≤ 48 total
- [ ] Defect checklist clean before interpreting any P&L

**If you break one, write down which and when.** A documented deviation is
recoverable; a forgotten one silently invalidates the verdict.

---

## 10. Quick reference

| Milestone | You verify | Gate |
|---|---|---|
| D0 | Both subscriptions live; **§6.7 answered** | Terminal reports both |
| D1 | Probe shape; SPX field names; `eod.close` timing | Coverage confirmed |
| D2 | 20 pilot days on disk | Extrapolate full-pull time |
| D3 | Six gates; holdout guard raises | Data trustworthy |
| D4 | Manifest complete | Blocked until §6.7 resolved |
| M1 | Tests green with **no data files** | Purity holds |
| M3 | Event log reads sensibly; ordering test passes | Engine sane |
| M4 | Scenario **sequences** reproduce | 🛑 Engine correct |
| M5 | Path spread **< 50%** | 🛑 Data sufficient |
| M6 | σ applied every stage; ≤42 runs | Discipline held |
| M7 | Six runs, one pass | 🛑 Holdout spent |
| M8 | Verdict a skeptic would accept | 🛑 **Sign-off** |
| M9 | ≥20 paper sessions | Live matches backtest |
| M10 | Start at 1 contract | — |

### The five numbers that decide everything

| Number | Where | Decides |
|---|---|---|
| **SPX 1-min shape** | **D0** | How M5 is even defined |
| **Path spread** | M5 | Whether the data can answer the question |
| **σ** | M6.0 | What counts as a real improvement |
| **Whipsaw cost** | Every run | Whether the add/drop cycle earns its keep |
| **Holdout profit factor** | M7 | Whether any of it was real |

### If you remember nothing else

Run the holdout once. Adopt nothing under 1σ. Never change a rule to fix a
result. On short gamma, a surprisingly **good** number is evidence of a bug until
proven otherwise. A negative verdict is a win.
