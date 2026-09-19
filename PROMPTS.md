# PROMPTS.md — Build Prompts for opencode

**Companion to:** `AGENTS.md` (authoritative rules) · `PROJECT.md` (scope) 
**Revision:** v1 · 2026-09-18
---

## How to use this file

Each prompt below is **copy-paste ready**. Run them **in order**. Do not start a
prompt until the previous one's **Acceptance check** passes.

| Convention | Meaning |
|---|---|
| `▶ PROMPT` | Paste this verbatim into opencode |
| `✅ ACCEPTANCE` | You verify this before moving on |
| `🛑 STOP` | A hard gate — do not proceed past it without the stated condition |

**Before the first prompt**, place `AGENTS.md` and `PROJECT.md` in the repo root.
opencode reads `AGENTS.md` automatically; the prompts reference it by section
rather than restating rules, which keeps one source of truth.

**Golden rule for every prompt:** if opencode proposes changing a rule in
`AGENTS.md` to make something easier, the answer is no. Rules change only by
your explicit decision, and then the file is edited first.

---

## P0 — Session bootstrap

Paste at the start of **every new opencode session**. Context resets between
sessions and this is what prevents drift.

```
▶ PROMPT

Read AGENTS.md and PROJECT.md in full before doing anything.

Working agreement for this project:

1. AGENTS.md is authoritative on all rules, thresholds and parameters. If
   something in my instructions conflicts with AGENTS.md, stop and ask — do not
   silently pick one.
2. Never change a rule, threshold or default to make a test pass or a result
   look better. If you believe a rule is wrong, say so and wait.
3. core/rules.py and core/sizing.py must stay pure: state snapshot in, decision
   out. No I/O, no clock access, no network, no broker calls.
4. Every number from AGENTS.md sections 3, 4 and 5 lives in config/default.yaml.
   Nothing hard-coded in logic.
5. The engine must not know whether it is backtesting or live. One code path.
6. Python 3.11+, standard scientific stack. No new dependency without telling me
   what it is and why.
7. Work in small commits with clear messages. Run the test suite before each.

Confirm you have read both files by answering these four questions, then stop:
- What is the anchor fly, and what does it determine?
- Why is the -25% stop loss described as a bug signal rather than a risk control?
- What happens on the holdout window before Stage 8?
- Name the five build-only items that are never test axes.
- What is the settlement source, and why is the 16:00:00 quote wrong?
```

✅ **ACCEPTANCE** — all four answers correct. Wrong answers mean it did not read
the files; re-paste rather than continuing.


## D1 — 

▶ PROMPT — D0

Read AGENTS.md sections 6.1 through 6.8 before starting, especially 6.7.

Do not write any downloader code yet. Verify the environment only.

1. Confirm Theta Terminal v3 is running. Print the access levels it reports at
   startup for each data type.
2. Confirm BOTH subscriptions are active and visible: Options Value and Index
   Value. AGENTS.md 6.1 requires both — an Options plan does not include SPX or
   VIX.
3. Confirm option historical access reaches 2020-01-01 and index historical
   access reaches 2023-01-01.
4. Make one minimal call to each of these and print the raw response shape
   before any parsing:
     - option/history/quote   (one SPXW date, narrow strike_range)
     - index/history/price    (SPX, one date, interval=1m)
     - index/history/eod      (SPX, one date)
5. CRITICAL, per AGENTS.md 6.7: report whether index/history/price at
   interval=1m returns an OHLC bar with open/high/low/close, or a single
   point-in-time value per minute. Quote the actual field names returned. Do not
   infer — show me the raw response.

Stop after printing. Do not build the downloader.


## D1 — Probe

```
▶ PROMPT

Read AGENTS.md sections 6.1 through 6.6 before starting.

Do NOT build the full downloader yet. Write a probe script only.

The probe should pull ONE trading day of SPXW 0DTE option data from ThetaData
and print:
- total rows returned
- distinct strikes, and the min/max strike relative to that day's SPX open
- first and last timestamp, and the count of distinct minutes
- how many strikes have continuous quotes across the session vs. gaps
- whether bulk endpoints (all contracts for one symbol+expiration in a single
  request) are available on this subscription tier
- the earliest date the account can access, to confirm history depth
- an SPX index sample for the same date via index/history/price at interval=1m,
  with its exact field names
- the SPX index close from index/history/eod for that date, and the timestamp of
  the final print

Check the Theta Terminal is running first and fail loudly with a clear message
if it is not. Do not retry silently.

Print the raw response shape before any parsing so I can see what we actually
receive. Stop after printing — do not proceed to building the downloader.
```

**Check:** ~122 contracts, 390 minutes, continuous quotes on wings that never
traded, bulk endpoints confirmed.

## D2 — Downloader, pilot only

```
▶ PROMPT

Build data/thetadata.py per AGENTS.md section 6.6.

Requirements, in priority order:
1. Cache every raw response to disk BEFORE any parsing or transformation. This
   is the most important requirement in the module — a parsing bug must never
   force a re-download.
2. Checkpoint and resume via a manifest of completed date/expiry pairs. Skip
   completed work on restart.
3. Check Theta Terminal is running before any request; fail loudly if not.
4. Chunk multi-day requests to roughly one month per call.
5. Write partitioned Parquet, one partition per date.

Strike bands differ by window, per DATA_FIRST.md section 4:
- 2023-09 to 2024-09: +/-25 points around the day's open (IV lookback only)
- 2024-09 to 2026-09: +/-150 points around the day's open (full replay)

Run it for 20 pilot trading days from the TRAIN window only. Report rows
written, disk used, elapsed time, and any dates that failed.

Do not launch the full multi-year pull yet.
```

**Check:** 20 days on disk, extrapolate elapsed time × 25 for the full pull.

## D3 — Quality gates and loader

```
▶ PROMPT

Build data/quality.py implementing all six gates from AGENTS.md section 6.4:
missing strikes, crossed/locked quotes, zero-bid wings, stale quotes, half days,
and 0DTE availability per date. Each returns pass/fail plus affected timestamps
and counts.

Then build data/loader.py:
- Loads partitioned Parquet into ChainSnapshot objects, memory-mapped per day.
- CRITICAL per AGENTS.md 6.1: takes an explicit window parameter and REFUSES to
  load the holdout window (2026-01-01 to 2026-09-30) unless stage == 8 is passed
  explicitly. Raise a loud exception otherwise. This guard cannot be disabled by
  config.
- Refuses any day that failed the quality gates unless explicitly overridden
  with a logged reason.

Run the gates across the 20 pilot days and show me the report.

Write tests/test_loader.py including: loading a 2026 date without stage=8
raises; with stage=8 it succeeds.
```

**Check:** all six gates pass. Then break it yourself:

```python
loader.load("2026-03-15")           # must raise
loader.load("2026-03-15", stage=8)  # must succeed
```

## D4 — Full pull

```
▶ PROMPT

Launch the full historical pull using the validated downloader.

Windows:
- 2023-09-01 to 2024-08-31, +/-25 strikes (IV Rank lookback)
- 2024-09-01 to 2026-09-30, +/-150 strikes (full replay, train + holdout)
- VIX 1-min and daily open across the whole range
- SPXW PM settlement values, daily
- Economic calendar flags per AGENTS.md section 6.1 item 6

Run unattended with checkpointing. Log progress so I can check status without
interrupting.

When complete, run the quality gates across the ENTIRE range and produce a
summary: days passed, days failed and why, total rows, disk used, and any date
gaps.

While this runs, we will build the rules engine in parallel — it needs no data.
```

**Then start M1 immediately.** Don't wait for the download.

---

## M1 — Core rules engine

No data, no network. Everything here is testable in isolation.

### M1.1 — Domain objects

```
▶ PROMPT

Build the domain layer, per AGENTS.md section 2 and section 12.

Create core/fly.py:
- A Fly dataclass: four legs (short put, short call, long put, long call),
  center strike, credit (points, positive), wing width, open timestamp,
  contracts.
- Properties: max_risk in dollars, breakeven_low, breakeven_high.
- A mark(chain_snapshot) method returning current value per contract.
- Validation on construction: all four legs share one expiry; wings are
  symmetric unless explicitly constructed otherwise; credit is positive.

Create data/chain.py:
- snap_strike(x, grid=5.0) — the single strike-rounding helper used everywhere.
- A ChainSnapshot class: timestamp, spot price, and strike -> (bid, ask) lookup
  for calls and puts. Include a mid() helper and a spread() helper.
- ChainSnapshot must be constructible from a plain dict so it can be used in
  tests with no data files.

Create config/default.yaml containing every numeric parameter from AGENTS.md
sections 3, 4 and 5. Group by section. Comment each with its AGENTS.md
reference. Nothing is hard-coded anywhere else.

Write tests/test_fly.py covering construction, validation failures, max_risk,
breakevens, and snap_strike at every add offset used in AGENTS.md 3.2.
```

✅ **ACCEPTANCE** — `pytest tests/test_fly.py` green. Open `config/default.yaml`
and confirm you can find `be_pct`, the three tier trigger distances, the add
offset, `bp_utilization` and `target_flies` without reading any Python.

### M1.2 — Pure rules

```
▶ PROMPT

Build core/rules.py as PURE functions. Every function takes an explicit state
snapshot and returns a decision object. No I/O, no datetime.now(), no globals.
Current time is always passed in as a parameter.

Implement exactly per AGENTS.md section 3:

- select_anchor(open_flies, spx_price) -> Fly          # 3.2, closest center
- select_tier(anchor, now) -> Tier                     # 3.2 table; 15.00 and
                                                       # 20.00 both select mid;
                                                       # after 14:00 ET the
                                                       # tight tier overrides
- should_add(state) -> AddDecision                     # 3.2
- should_drop(state) -> DropDecision                   # 3.3, unconditional on
                                                       # P&L, never gated on
                                                       # profitability
- should_close_breakeven(fly, spx_price, be_pct)       # 3.6
- should_close_stop(fly, current_value, cfg)           # 3.7
- entry_allowed(state) -> EntryDecision                # 3.1 price and spread
                                                       # bounds

Every decision object carries a human-readable `reason` string. These reasons
become the event log, so they must be specific enough to debug from.

Write tests/test_rules.py. Mandatory cases from AGENTS.md section 8:
- be_pct exit: credit 15.00, center 5430 -> exits at 5417.25 and 5442.75 exactly
- tier selection at credit exactly 15.00 and exactly 20.00 -> both mid tier
- two-fly cap suppresses a third add at 13:59 ET, permits it at 14:01 ET
- AM-settled SPX root rejected at entry
- stop loss at credit 15.00 -> -$375 per contract
- stop unreachability: at credit 27.4, 50-wide, the 3.6 exit triggers at a
  smaller SPX distance than the 3.7 stop at 5h, 3h, 1h and 0.2h to expiry

That last test encodes a measured finding. If it fails, do not adjust the test —
tell me, because either the pricing model or the rule is wrong.
```

✅ **ACCEPTANCE** — `pytest tests/test_rules.py` green, **including the stop
unreachability test**. Then grep `core/rules.py` for `now()`, `open(`, `requests`
and `import datetime` — all should be absent.

### M1.3 — Sizing

```
▶ PROMPT

Build core/sizing.py as PURE functions, per AGENTS.md section 5.

- weekly_contract_update(current, weekly_pnl, equity, cfg) -> int
     +1 winning week, -1 losing week, unchanged flat, hard floor 1
- equity_required(n_contracts, cfg) -> float            # 5.1 ladder
- flies_by_bp(equity, contracts, bp_in_use, cfg) -> int # 5.3
- effective_fly_cap(now, equity, contracts, bp_in_use, cfg) -> int
     min(strategy_cap, flies_by_bp, 10); strategy_cap is 2 before 14:00 ET
- pdt_blocked(equity) -> bool                           # 5.5, threshold 25000
- sizing_mode dispatch for: fixed_1, weekly_scaling, equity_banded,
  unconstrained  # 5.6

Write tests/test_sizing.py:
- win -> +1, loss -> -1, floor holds at 1
- contract step-up to 2 blocked below $50,000 at target_flies=5
- at $25,000 / 1 contract, flies_by_bp >= 5
- at $25,000 / 2 contracts, flies_by_bp <= 2
- a drop releases full reserve and raises flies_by_bp on the next call
- BP_BLOCKED_ADD on 3 days in one week forces contracts -= 1 next week
- equity moving $25,000 -> $50,000 raises the cap with no config change
- pdt_blocked is True at 24,999 and False at 25,000
- below $50,000 equity, fixed_1 and weekly_scaling produce identical contract
  sequences over a simulated 26-week win/loss pattern
```

✅ **ACCEPTANCE** — all green. The last test is the important one: below $50k
those two modes **must** be identical, per `AGENTS.md` §5.4.

### M1.4 — Combo-fill integrity

```
▶ PROMPT

Build core/integrity.py per AGENTS.md section 3.9. This is a correctness
requirement, never a tunable.

- verify_fill(order, fill_report) -> IntegrityResult
     Assert all four legs present at expected ratios and correct sides.
     Any mismatch raises FILL_INTEGRITY_BREACH.
- remediate(breach, adapter) -> Action
     Attempt immediate completion at market; if that fails, flatten the partial.
     Never return a state where an unbalanced position persists to the next
     cycle.

Write tests/test_integrity.py:
- a clean 4-of-4 fill passes
- a 3-of-4 fill raises FILL_INTEGRITY_BREACH
- a 4-leg fill with wrong ratios raises FILL_INTEGRITY_BREACH
- after remediation fails, the resulting book contains no unbalanced position
- assert no config key can disable integrity checking
```

✅ **ACCEPTANCE** — all green. Confirm there is **no** config flag anywhere that
turns this off.

🛑 **M1 GATE** — full suite green with zero data files present. If any test needs
a data file, the purity requirement has been violated somewhere.

---


## M3 — Engine and backtest adapter

```
▶ PROMPT

Build core/engine.py and adapters/backtest.py.

The engine runs the cycle loop from AGENTS.md section 3.5, in this exact order:

  1. risk_halts()
  2. breakeven_close()
  3. stop_loss()
  4. drop_check()      <- MUST precede adds
  5. add_check()
  6. eod_handling()

The ordering is a correctness requirement, not a style choice. A fast move
otherwise opens a third fly in the same pass that should have closed one.

The engine calls pure functions from core/rules.py and core/sizing.py and talks
to the outside world only through an adapter interface. It must not know whether
it is backtesting or live.

Every decision appends to a structured event log: timestamp, event type, SPX
price, affected flies, the reason string from the decision object. Event types
at minimum: ENTRY, ADD, DROP, CLOSE_BREAKEVEN, CLOSE_STOP, SETTLEMENT,
ADD_SUPPRESSED_CAP, BP_BLOCKED_ADD, HALT_PREEMPTED_DROP, PDT_LOCKOUT,
NEAR_TOUCH_SUPPRESSED, ENTRY_REJECTED_SPREAD, FILL_INTEGRITY_BREACH.

adapters/backtest.py:
- Replays ChainSnapshots minute by minute.
- Implements the three fill modes from AGENTS.md 6.5: optimistic, realistic
  (mid + 0.10 against per fly per side), pessimistic.
- Implements intrabar_assumption: conservative (adverse event first) and
  optimistic, per AGENTS.md 7.1.
- Models no-fill when combo spread > 1.50 and logs ENTRY_REJECTED_SPREAD.
- Settles held flies at the official SPXW PM settlement value, NOT the 16:00
  quote (AGENTS.md 7.2).
- Applies commissions and exchange fees.

Write a test asserting evaluation order directly from the event log: construct a
bar that triggers both a drop and an add, and assert DROP appears before ADD.
```

✅ **ACCEPTANCE** — one full day replays end to end. Read the event log by hand
and confirm the narrative makes sense. **Verify the ordering test passes.**

---

## M4 — Scenario validation

This is where you find out whether the engine is actually right.

```
▶ PROMPT

Build tests/test_scenarios.py per AGENTS.md section 8. These are the source
author's hand-worked days and are ground truth for the add/drop cycle.

Scenario A — chaotic winner, target +$849 per contract:
  open 4570 -> +7 -> add 4580 -> touch 4570 -> drop 4580
  -> -7 -> add 4560 -> touch 4560 -> drop 4570
  -> +7 -> add 4570 -> touch 4560 -> drop 4570
  -> +7 -> add 4570 -> 14:00 ET switch -> +2.5 -> add 4565
  -> hold 4560 / 4565 / 4570 into settlement
  Assert: never more than 2 flies before the tight switch; exactly 3 at
  settlement.

Scenario B — drifting down, target +$112 per contract:
  open 4570 -> add 4560 -> tight switch -> add 4565
  -> touch 4560 -> drop 4570 -> drift down -> add 4550 -> add 4555
  -> touch 4550 -> drop 4560 -> add 4545
  -> hold 4545 / 4550 / 4555 into settlement

Scenario C — synthetic EKG day, +/-20 point oscillation. Assert the engine
churns flies and loses money. The test is that it degrades as described, not
that it hits a specific number.

Construct synthetic ChainSnapshots with a Black-Scholes surface so these run
with no data subscription.

If the fly SEQUENCE matches but the P&L does not, that is expected — the source
gives strike paths, not the exact vol surface. Assert the sequence strictly and
the P&L within a wide tolerance. Report the actual P&L you get for each.

If the SEQUENCE does not match, stop and report the first divergence with full
state. Do not adjust the scenario to fit the engine.
```

✅ **ACCEPTANCE** — Scenarios A and B reproduce the **fly sequence exactly**.
P&L within a loose band is fine; sequence divergence is an engine bug. Scenario C
loses money.

🛑 **M4 GATE** — sequences reproduce. If they don't, the engine is wrong and
every downstream number is meaningless.

---

## M5 — Path-sensitivity gate 🛑

**The most important gate in the project.** Do not buy the full ThetaData
subscription before this passes.

```
▶ PROMPT

Run Stage 1 from AGENTS.md section 13.1 — the path-sensitivity pilot.

Two runs, 20 trading days from the TRAIN window only, baseline config, 1-minute
data, fill_mode=realistic:
  1.1  intrabar_assumption = conservative
  1.2  intrabar_assumption = optimistic

Report for each: total P&L, profit factor, and the count of bars where both a
drop trigger and an add trigger occurred within the same bar.

Then compute the spread: abs(pnl_1.1 - pnl_1.2) / mean(pnl_1.1, pnl_1.2).

GATE per AGENTS.md 13.1: if that spread exceeds 50%, 1-minute data cannot
resolve this strategy. Report the number and STOP. Do not proceed to Stage 2.
Do not suggest workarounds. This is a data-resolution finding, not a bug.

Also build backtest/ledger.csv now, per AGENTS.md 13.11, appended by the runner
and never hand-edited. These two runs are its first entries.
```

✅ **ACCEPTANCE** — spread **< 50%** of mean P&L.

🛑 **HARD STOP** — if the spread exceeds 50%, the project pauses here. Options
are tick-level data or abandonment. Nothing downstream is salvageable at 1-min.
The same-bar collision count tells you how close to the edge you are even on a
pass.

---

## M6 — Staged runs 2 through 7

Run these **one stage at a time**. Review each before starting the next — that
sequencing is what keeps degrees of freedom low.

### M6.0 — Noise margin first

```
▶ PROMPT

Before Stage 2, establish the noise margin per AGENTS.md 13.9.

Run the baseline config on 10 bootstrap resamples of the TRAIN window. Record
the standard deviation of profit factor across them.

That sigma is the adoption threshold for every subsequent stage: any improvement
smaller than 1 sigma is noise and must not be adopted. Print it prominently and
write it to the ledger.
```

✅ **ACCEPTANCE** — you have a number for σ. Write it down; every later decision
references it.

### M6.1 — Stage 2, baseline surface

```
▶ PROMPT

Run Stage 2 per AGENTS.md 13.2. Six runs: fill_mode (optimistic, realistic,
pessimistic) x intrabar_assumption (conservative, optimistic). TRAIN window.

Report the full section 9 output for each, and in particular the required
diagnostics: whipsaw cost, exit-reason split, profitable-breakeven-exit share by
hour, fly-count distribution, rejected entries, HALT_PREEMPTED_DROP.

The headline cell is realistic x conservative. Report it as the result.

Then run the section 13.13 defect checklist against that cell and tell me
plainly:
- Do the scenario tests still pass?
- Is stop_25 firing at all? (any material count is a BUG SIGNAL per 3.7)
- Is the fly count routinely 1 rather than 2-3?
- Are the quality gates clean for these days?
- Is settlement using the official PM value?

Do not tune anything. Do not proceed to Stage 3. Report and stop.
```

✅ **ACCEPTANCE** — a clean defect checklist. **If `stop_25` fires materially, or
fly count is routinely 1, stop and debug** — that is not a result. If the
baseline is unprofitable but the checklist is clean, consult `AGENTS.md` §13.13
for whether to continue.

### M6.2 — Stage 3, exit family

```
▶ PROMPT

Run Stage 3 per AGENTS.md 13.3. Twelve runs: stop_basis (credit, max_risk, none)
x be_mode (static_85, static_80, time_scaled, dual_mode). TRAIN window, judged at
realistic x conservative.

Implement time_scaled and dual_mode first if not already present:
- time_scaled: be_pct widens through the session so the exit stays a constant
  fraction of credit LOST rather than a constant distance. This targets the
  late-session inversion documented in AGENTS.md 3.6.
- dual_mode: loss exit before 14:00; after 14:00 the same breach becomes an
  explicit profit-lock with its own threshold.

For every variant report the exit-reason split and the share of breakeven exits
that closed at a PROFIT, bucketed by hour. That share is the direct measurement
of the 3.6 inversion.

Adopt a winner only if it beats the Stage 2 base by more than 1 sigma. On ties,
prefer the simpler configuration. Tell me which you would adopt and why, but do
not carry it forward until I confirm.
```

✅ **ACCEPTANCE** — a recommendation with the σ margin shown. Expect
`static_85`'s profitable-exit share to rise sharply in the final two hours — that
confirms the documented inversion.

### M6.3 — Stage 4, calendar filters

```
▶ PROMPT

Run Stage 4 per AGENTS.md 13.4, on top of the confirmed Stage 3 winner. Four
runs: Skip CPI off/on x Skip Triple Witching off/on. TRAIN window.

Judge on the LEFT TAIL, not the mean: worst-5-day P&L and max drawdown. These
filters remove few days and should barely move the average. If they cut the
tail, adopt regardless of mean impact.

Report how many trading days each filter removed, so I can see the cost.
```

✅ **ACCEPTANCE** — tail metrics compared, not just means.

### M6.4 — Stage 5, volatility halts

```
▶ PROMPT

Run Stage 5 per AGENTS.md 13.5 on the confirmed base. Six runs:
vol_halt_mode (flatten, halt_adds_only, off) x max_vix (30, none).

Implement halt_adds_only: stop opening new positions but keep existing flies
under normal exit rules. This removes forced liquidation at dislocated prices
while retaining risk control.

Report HALT_PREEMPTED_DROP counts and, critically, realized slippage on
halt-driven exits versus normal exits. If halt exits slip materially worse, that
prices what the flatten rule costs.

max_vix=none quantifies the sample censoring — report how many extra days it
admits and how they performed.
```

✅ **ACCEPTANCE** — the slippage comparison is the key output. It converts a
doctrinal argument into a number.

### M6.5 — Stage 6, structure

```
▶ PROMPT

Run Stage 6 per AGENTS.md 13.6 on the confirmed base.

First check Stage 1's path spread. If it was NOT narrow, drop touch_confirm from
the matrix entirely — it is unresolvable at 1-minute granularity and must be
settled live. Tell me which case applies before running.

Axes: wing_cap_mode (premium_only, vol_capped) x touch_confirm (instant, 10s,
30s, if admissible).

Implement vol_capped: cap wing width by volatility regime rather than by the
$0.40 premium rule alone. This addresses the risk-scaling defect in AGENTS.md
3.2 where high vol widens wings and raises max risk exactly when it should fall.

Report max risk per fly distribution under both wing modes, and whipsaw cost
under each touch_confirm setting.
```

✅ **ACCEPTANCE** — if Stage 1's spread was wide, confirm `touch_confirm` was
correctly dropped rather than tested on data that can't resolve it.

### M6.6 — Stage 7, sizing

```
▶ PROMPT

Run Stage 7 per AGENTS.md 13.7 on the confirmed base. Six runs:
sizing_mode (fixed_1, weekly_scaling, equity_banded) x start_equity (25000,
40000).

Report max drawdown as a PERCENTAGE of equity, PDT_LOCKOUT days, days spent
within 10% of the $25,000 threshold, BP_BLOCKED_ADD days, and the date equity
first supported 2 contracts.

Sanity check per AGENTS.md 5.4: below roughly $50,000 equity, fixed_1 and
weekly_scaling must produce IDENTICAL curves. If they differ in the train
window, that is a bug — stop and investigate before reporting results.
```

✅ **ACCEPTANCE** — the `fixed_1` vs `weekly_scaling` identity check passes below
$50k. If it fails, it's a bug, not a finding.

🛑 **M6 GATE** — ≤42 runs used, ledger complete, and you have **3 candidate
configs** for the holdout.

---

## M7 — Holdout confirmation 🛑 ONE SHOT

```
▶ PROMPT

Run Stage 8 per AGENTS.md 13.8. This is the ONE permitted use of the holdout
window. Read 13.8 before starting.

Take the top 3 configs from Stages 3 through 7 combined, plus the unmodified
baseline. Run on the untouched holdout window (2026-01 to 2026-09) at
fill=realistic, intrabar=conservative. Pass stage=8 to the loader.

Six runs. No tuning. No re-runs. No "one more variant." If something looks
wrong, report it — do not fix and re-run, because a second pass burns the
holdout permanently.

Acceptance per 13.8: a config is adopted only if it beats baseline on the
holdout in the same DIRECTION and rough MAGNITUDE as it did in train.

If train and holdout rankings INVERT, the train result was noise. In that case
the recommendation is the BASELINE, not the train winner. State this plainly if
it occurs.

Append all six runs to the ledger with stage=8.
```

✅ **ACCEPTANCE** — six runs, one pass, ledger updated.

🛑 **The holdout is now spent.** Any further tuning against it produces numbers
you cannot trust, and there is no recovery short of waiting for new market data.

---

## M8 — Written verdict 🛑 HARD STOP

```
▶ PROMPT

Write VERDICT.md answering the research question from PROJECT.md section 3:

  Does the add/drop recentering cycle generate edge, or does it merely reshape a
  fair-value premium sale into many small wins and rare large losses?

Structure it as:

1. The answer, in one paragraph, stated plainly.
2. Evidence: headline metrics at realistic x conservative, train and holdout.
3. Whipsaw cost — what the add/drop cycle actually cost across the period. This
   is the most direct measurement of whether the cycle earns its keep.
4. Which stages improved results and by how many sigma. Which were noise.
5. Where the P&L actually came from: tail avoidance, exit discipline, or the
   add/drop cycle. Be specific.
6. Whether the null hypothesis in PROJECT.md section 3 was disproved, and how
   confidently.
7. Known limitations: intrabar ambiguity, touch confirmation unresolvable at
   1-min, sample censoring from max_vix, sequence risk, the discretion gap
   versus the source operator.
8. Recommendation: trade it, trade it modified, or do not trade it.

Be direct. If the honest answer is that this does not survive realistic fills,
say exactly that. PROJECT.md section 5 states that a negative result is a
SUCCESSFUL MVP. Do not soften it, do not suggest further tuning to rescue it,
and do not propose new parameters.
```

✅ **ACCEPTANCE** — a verdict you would be comfortable showing someone who wants
the strategy to fail.

🛑 **HARD STOP — human sign-off required.** Do not begin Phase 2 without it.

---

## M9 — Tradier paper adapter

Only after M8 sign-off, and only if the verdict supports trading.

```
▶ PROMPT

Build adapters/tradier.py implementing the same adapter interface as
adapters/backtest.py. The engine must not change at all — if it needs to know
which adapter it is running against, the abstraction is wrong.

Requirements:
- Paper trading endpoint only. Live credentials must not be loadable in this
  milestone.
- Combo-only multi-leg orders per AGENTS.md 3.9. Never leg in.
- Post-fill verification on every fill, with FILL_INTEGRITY_BREACH handling and
  remediation from core/integrity.py.
- Real touch_confirm_seconds enforcement (default 10) — this is the rule that
  could not be tested in backtest, so it gets its first real evaluation here.
- Identical event logging to the backtest adapter so live and replay logs are
  directly comparable.
- A pre-flight check at startup: clock sync, market calendar, account equity,
  buying power, and PDT status. Refuse to start if any fail.

Add a kill switch: a single command that flattens everything and halts.
```

✅ **ACCEPTANCE** — **≥20 paper sessions** before any live discussion. Compare
paper event logs against backtest expectations. Divergence in touch-drop
behaviour is expected and is exactly what this milestone measures.

---

## M10 — Live

```
▶ PROMPT

Live readiness review. Do not enable live trading yet — produce a report.

Compare the 20+ paper sessions against backtest predictions:
- Fill quality: realized slippage versus the modelled mid + 0.10
- Touch-drop behaviour: how often did the 10-second confirmation change the
  decision versus the backtest's instant-touch approximation?
- Any FILL_INTEGRITY_BREACH events, and how remediation performed
- Whipsaw cost, paper versus backtest
- Any divergence in fly count or exit-reason split

Then state whether paper results are consistent with the backtest. If they are
not, quantify the gap and recommend whether to proceed.

Live starts at 1 contract regardless of what the sizing rules permit.
```

✅ **ACCEPTANCE** — paper consistent with backtest. **Start at 1 contract**,
whatever the equity ladder allows.

---

## Anti-drift checks

Run these periodically. Coding agents drift toward making things work rather than
making them correct.

```
▶ PROMPT — audit

Audit the codebase against AGENTS.md and report violations. Do not fix anything
yet — list findings first.

1. Purity: any I/O, clock access, network or globals in core/rules.py or
   core/sizing.py?
2. Hard-coded numbers: any value from AGENTS.md sections 3, 4 or 5 appearing in
   logic rather than config?
3. Evaluation order: does the engine still run drop_check before add_check?
4. Guards intact: does the loader still refuse the holdout without stage=8? Can
   combo-fill integrity be disabled by any config path?
5. Drift: any rule, threshold or default that differs from AGENTS.md? Quote both
   versions.
6. Dead axes: any config key that no longer affects behaviour?
7. Ledger integrity: any run in ledger.csv without a git_sha or config_hash? Any
   stage=8 entries beyond the permitted six?
```

Run after M1, M4, M6 and before M7.

```
▶ PROMPT — result sanity

For the most recent run, check for results that are too good to be true:

1. Win rate above 85% on a short-gamma strategy — usually a settlement or
   marking bug.
2. Any day with zero losing flies across many trades.
3. Fills better than the modelled fill mode.
4. Max drawdown smaller than the largest single-day loss.
5. Profit factor above 2.5 at realistic fills.
6. Settlement P&L inconsistent with the official PM value.

Report anything that trips. On this strategy, a surprisingly good number is
evidence of a bug until proven otherwise.
```

---

## Quick reference

| Milestone | Prompts | Gate |
|---|---|---|
| D0 | Both subscriptions live; **§6.7 answered** | Terminal reports both |
| D1 | Probe shape; SPX field names | Coverage confirmed |
| D2 | 20 pilot days on disk | Extrapolate full-pull time |
| D3 | Six quality gates; holdout guard raises | Data trustworthy |
| D4 | Manifest complete | **Blocked until §6.7 resolved** |
| M1 | 1.1 – 1.4 | Full suite green, **no data files** |
| M3 | single | Day replays; ordering test passes |
| M4 | single | 🛑 Scenario sequences reproduce |
| M5 | single | 🛑 **Path spread < 50%** — or stop |
| M6 | 6.0 – 6.6 | ≤42 runs; 3 candidates |
| M7 | single | 🛑 One shot; holdout spent |
| M8 | single | 🛑 **Human sign-off** |
| M9 | single | ≥20 paper sessions |
| M10 | single | 1 contract |

**Three things that will wreck this project if you let them:**
1. Running the holdout more than once.
2. Adopting improvements smaller than 1σ.
3. Changing a rule to make a test or a result cooperate.
