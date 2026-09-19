# AGENTS.md — NFH 0DTE SPX Iron Fly Engine

**Authoritative rules specification.** Where `PROJECT.md` summarises, it
simplifies; on any conflict, **this file wins**.

**Revision:** v5 · 2026-09-18
**Companions:** `PROJECT.md` (scope/MVP) · `PROMPTS.md` (build prompts) · `USER_GUIDE.md` (operating manual)

Read end to end before writing code.

### Changes in v5

Vendor-confirmed data facts folded in (ThetaData support, written).

| § | Change |
|---|---|
| 6.1 | **Two subscriptions required.** Index Value is separate from Options Value and is mandatory — SPX spot, VIX and settlement all live there. |
| 3.8, 7.2 | **Settlement corrected.** SPXW settles to the SPX index close from `index/history/eod.close`. The index prints until ~16:04–16:05 ET; the 16:00:00 quote is **not** settlement. |
| 6.5 | **New: `snapshot_latency`.** Value option quotes are 1-min point-in-time NBBO snapshots, not bars. Triggers price at the next snapshot. |
| 6.6 | Confirmed request shape; build on **v3**; Flat Files irrelevant (7-day window). |
| 7.1, 7.3, 13.1 | **Intrabar hazard reframed and BLOCKED** pending §6.7 — the M5 test definition depends on whether SPX 1-min is OHLC or snapshot. |
| 6.7 | **New: open blocking question.** Resolve before D4. |
| 11 | Build order is **data-first** (D1–D4 precede the engine). |

### Changes in v4
Build order became data-first; §13.1 gate retained as a data-sufficiency check
rather than a purchase decision.

### Changes in v3
§3.7 corrected — the −25% stop is measurably *unreachable*, not "usually fires
first." §3.9 combo-fill integrity added. Train/holdout split fixed in-spec.

---

## 0. Resolved decisions — do not re-open

| # | Question | Resolution |
|---|---|---|
| 1 | Add reference point | **Anchor fly (recenter after every drop).** The transcript Q&A claim that distance is always measured from the day's first fly is disregarded. Single code path — no `first_fly` mode. |
| 2 | Data vendor | **ThetaData.** Options Value (~$40/mo) **+ Index Value** (§6.1). |
| 3 | Stop-loss basis | **−25% of credit received** as specified — but see §3.7, it is measurably unreachable. Fate decided by Stage 3. |
| 4 | Position sizing | **Equity-scaled.** Weekly contract scaling plus a live fly cap from account equity. Flies take priority over contracts. §5. |
| 5 | Account size | **$25,000 starting equity.** Capacity recomputes from live equity. §5.4. |
| 6 | API version | **v3.** Vendor-recommended for new projects; all confirmed parameters are v3. |

---

## 1. Strategy in one paragraph

Sell an at-the-money 0DTE iron fly on SPX after the open settles. As SPX drifts
away from the fly's center, add a second fly in the direction of travel. When SPX
returns and touches the center of the nearer fly, drop the farther one. Repeat
all afternoon. After 14:00 ET, switch to tighter spacing and deliberately stack
2–3 flies into cash settlement at the close, where the bulk of the day's theta
decays. Cut individual flies while they are still inside their expiration
breakeven so losses stay small relative to wins.

---

## 2. Instrument and conventions

- Underlying: **SPX** (cash index, European, no assignment risk).
- Options: **SPXW** 0DTE, **PM cash-settled**.
  - Third-Friday standard SPX is **AM-settled**, and **SET** is that symbol —
    *not* SPXW (vendor-confirmed). Trade SPXW only. Assert on root/expiry so an
    AM-settled contract can never enter the book (§8 test).
- Strike grid: 5.00 points near ATM. One helper, `snap_strike(x, grid=5.0)`,
  used everywhere a strike is computed.
- Multiplier: 100. P&L in dollars per contract unless suffixed `_total`.
- Timezone: compute and store in **US/Eastern**. All times below are ET.

### Iron fly definition

Four legs, same expiry, centered on strike `K`:

| Leg | Strike | Side |
|---|---|---|
| Short put | `K` | Sell |
| Short call | `K` | Sell |
| Long put | `K − w` | Buy |
| Long call | `K + w` | Buy |

- `credit` — net premium received per contract, in points, positive.
- `center` — `K`.
- `max_risk` — `(w − credit) × 100` dollars per contract.
- Wings exist **only** for buying-power efficiency. **No trigger anywhere in this
  engine may reference a wing strike.**

---

## 3. Core rules

### 3.1 Entry — first fly of the day

- Entry time **11:00 ET** — let the open settle.
- Center at-the-money: `center = snap_strike(spx_price)`.
- Wing width **$50** for the first fly.
- Reject if fly mid price outside `[3.00, 50.00]`.
- Reject if combo bid/ask spread > `1.50`.
- Record `open_credit` on the fly. Each fly carries its own credit; tier
  selection reads the **anchor fly's** credit, not the day's first.

### 3.2 Adds — distance from the anchor fly's center

Each cycle, the **anchor fly** is the open fly whose center is closest to current
SPX.

```
distance = abs(spx_price - anchor.center)
```

Tier selected from `anchor.open_credit`:

| Anchor credit | Trigger distance | Target spacing | Add offset |
|---|---|---|---|
| `< 15` | `> 7.0` | 10 apart | `3.00` |
| `15 ≤ c ≤ 20` | `> 10.0` | 15 apart | `3.00` |
| `> 20` | `> 15.0` | 20 apart | `3.00` |
| after 14:00 ET | `> 2.5` | 5 apart | `0.01` |

Boundaries inclusive on the mid tier — `15.00` and `20.00` both select mid. Unit
test both edges exactly.

```
center_new = snap_strike(spx_price + sign(spx_price - anchor.center) * add_offset)
```

New-fly wing width is **priced, not fixed**: pick the long put strike whose mid is
nearest **$0.40**, clamp width to `20 ≤ w ≤ 50`, mirror on the call side.

> **Recentering is the defined behaviour.** After a drop, the survivor becomes the
> anchor and all distances measure from it. No "distance from first fly" mode.
>
> **Spacing note.** `spacing = trigger + add_offset` with one shared offset.
> `3.00` lands the common tier exactly on 10-apart after snapping; the 15/20 tiers
> land one strike tight in some cases. Accepted — do not make the offset
> tier-dependent.
>
> **Known risk-scaling defect.** The $0.40 wing rule pushes the long strike
> further out in high vol, *widening* wings and *raising* max risk exactly when it
> should fall. Not fixed in the baseline; tested as `wing_cap_mode: vol_capped`
> in Stage 6 (§13.6).

### 3.3 Drops — touch the center

When SPX trades **at or through** the `center` of any open fly, close every other
open fly whose center is **farther from SPX** than the touched fly. Two-fly case:
touch the near fly, drop the far one.

- **Unconditional on P&L.** A dropped fly is usually near its worst mark. That is
  the point — a risk-reduction exit, not a loss exit. Never gate on profitability.
- **Touch tolerance:** the touch must hold `touch_confirm_seconds` (default **10**)
  in live. Detection fidelity in backtest depends on §6.7 — see §7.3.
- Log every drop **and** every suppressed near-touch with SPX, both centers, both
  marks.

> **Structural note — do not "fix" this.** The drop rule is inherently
> buy-high/sell-low: you add after a 7-point move, then drop when price returns.
> Every oscillation pays a toll. This is the strategy's core recurring cost and
> must be measured directly (§9 whipsaw cost), not engineered away.

### 3.4 Two-fly cap

- **11:00–14:00 ET:** never more than **2** open flies. An add that would create a
  third is **suppressed and logged** — not queued.
- **From 14:00 ET:** cap lifts; tight adds run freely, targeting 2–3 flies into
  settlement.
- Hard ceiling **10 positions/day** regardless of time.

### 3.5 Evaluation order — correctness requirement

```
1. risk_halts()        # VIX change, rapid move, day PT / day max loss
2. breakeven_close()   # §3.6
3. stop_loss()         # §3.7
4. drop_check()        # §3.3 — MUST precede adds
5. add_check()         # §3.2
6. eod_handling()      # settlement
```

A fast move otherwise opens a third fly in the same pass that should have closed
one. Assert the ordering via the event log in tests.

### 3.6 Loss exit — inside breakeven

Breakeven is `center ± credit`. Exit while SPX is still **inside** it, at
`be_pct` (default **0.85**, configurable in `[0.80, 0.90]`):

```
exit_distance = be_pct * credit
exit_lo, exit_hi = center - exit_distance, center + exit_distance
close if spx_price <= exit_lo or spx_price >= exit_hi
```

Computed per-fly from that fly's own credit — a single float, **not** three
hard-coded dollar tiers.

Unit-test verbatim: credit `15.00`, center `5430` → BE `5415 / 5445` → exit at
**`5417.25 / 5442.75`**.

> **Known defect — the late-session inversion.** `be_pct` is a fixed *distance*,
> but the fly's value at that distance changes as extrinsic decays. At 11:00 the
> exit realises a genuine loss. By ~15:45 the same distance is nearly all
> intrinsic and the exit closes at a **profit**.
>
> The rule therefore stops being a loss-cutter and becomes a profit-taker during
> the exact window the strategy claims most of its theta. Pre-registered for
> Stage 3, where `time_scaled` and `dual_mode` are tested against it. **Required
> diagnostic:** share of `breakeven` exits closing at a profit, bucketed by hour.

### 3.7 Stop loss — measurably unreachable

Specified as **−25% of credit received**. For credit `15.00` (= $1,500), stop at
`−$375`/contract.

Black-Scholes evaluation at 13% vol, 50-wide fly, credit ≈ 27.4 shows the price
exit (§3.6) triggering 13–15 points earlier at *every* hour:

| Time left | 85%-BE exit distance | Fly value there | Distance where −25% stop fires |
|---|---|---|---|
| 5.0h | 23.3 pts | 30.3 | 37.9 pts |
| 3.0h | 23.3 pts | 27.7 | 38.8 pts |
| 1.0h | 23.3 pts | 24.3 | 36.2 pts |
| 0.2h | 23.3 pts | 23.3 | 34.3 pts |

Implications:

1. Implement the stop as specified. Do **not** delete it pre-emptively.
2. **`stop_25` firing in a backtest is a bug signal.** Any material count → stop
   and investigate; either the price exit is not evaluating or ordering (§3.5) is
   wrong.
3. Its fate is decided by Stage 3 (`stop_basis: credit / max_risk / none`).

### 3.8 End of day — settlement (vendor-corrected)

- Survivors are **held into PM cash settlement**. Do not flatten at 15:59.
- **SPXW settles to the official closing value of the SPX index.** ThetaData
  publishes no separate settlement print on the option itself.
- **Source:** `index/history/eod` → `close` field (requires the Index
  subscription, §6.1).
- **Critical timing:** the SPX index keeps printing until roughly **16:04–16:05
  ET** as closing prices arrive. The `eod.close` value is the settlement; the
  **16:00:00 quote is not**. Using the 16:00 print biases every held fly in the
  same direction.
- Settlement P&L from intrinsic value of all four legs at that close.
- No new positions after **15:45 ET**.

### 3.9 Combo-fill integrity — mandatory, not tunable

A partially filled 4-leg combo leaves the account **naked short an SPX option** —
undefined risk on a cash-settled index. At ~20 flies/day × 8 leg-sides, partial
fills are a certainty.

1. **Combo-only orders.** Never leg in. Single multi-leg order, atomic or rejected.
2. **Post-fill verification.** Assert all four legs present at expected ratios.
   Mismatch → `FILL_INTEGRITY_BREACH`.
3. **Breach handling.** Attempt immediate completion at market; if that fails,
   flatten the partial. Never carry an unbalanced position to the next cycle.
4. **Backtest modelling.** Model no-fill when combo spread >
   `max_ba_spread_open` (1.50). Count rejected entries and report them — a
   strategy that cannot get filled has not traded.

Correctness requirement (§13.10). Never relaxed to improve a result.

---

## 4. Carried-over baseline (configurable; defaults as stated)

| Control | Default |
|---|---|
| Skip FOMC | **On** |
| Skip CPI / PPI / PCE / Nonfarm / Triple Witching / OpEx / End of Quarter | Off |
| Max VIX | 30 |
| Max Symbol IV Rank | 30 |
| Max VIX Change from open → halt + flatten | 5 |
| Rapid move $25 in 1 min → halt + flatten | On |
| Rapid move $40 in 2 min → halt + flatten | On |
| Min close RoR | −90% |
| Day PT / Day Max Loss close | Off ($1,000 / −$2,000) |
| Elevated-volatility entry | On, from 13:30 ET |
| Tight trade price threshold | $10 |
| Late tight threshold entry | 15:30 ET |

**Three known conflicts — log them, do not silently resolve them:**

1. **Vol halts vs. the drop rule.** VIX-change and rapid-move rules flatten the
   book in exactly the conditions where the touch-drop would instead recenter,
   and market-exit 3–5 flies into a dislocation with blown-out 0DTE spreads.
   Emit `HALT_PREEMPTED_DROP`; report halt-exit slippage vs. normal. Stage 5.
2. **Calendar defaults contradict source guidance.** The source author sits out
   CPI and month-end/witching; defaults trade them, keeping the fattest
   left-tail days. Stage 4.
3. **Censored sample.** Max VIX 30 means every result is conditional on calm
   days — and VIX < 30 does not preclude a 2% intraday move. Stage 5 via
   `max_vix: none`.

---

## 5. Position sizing — equity-scaled

Two dimensions competing for the same buying power:

| Dimension | Controls | Set when |
|---|---|---|
| `contracts` | Leverage per fly | **Weekly**, never intraweek |
| `max_concurrent_flies` | How many flies the book may hold | **Live**, from remaining BP |

```
bp_budget   = account_equity * bp_utilization      # default 0.80
bp_per_fly  = contracts * reserve_per_fly
flies_by_bp = floor(bp_available / bp_per_fly)
```

`reserve_per_fly` is the **worst-case** max risk of a not-yet-opened fly:
`(max_wing_width − min_expected_credit) × 100`. Default **$4,000**. An open fly
reserves its *actual* `(w − credit) × 100`, typically ~$3,500. Reserve at entry,
release on exit.

### 5.1 Priority rule — flies before contracts

When BP is scarce, **buy fly capacity first, contract count second.**

The add/drop recentering cycle and the 2–3 fly stack into settlement *are* the
edge — cutting fly capacity changes what the strategy is. Contract count is only
leverage on an edge you already have.

```
equity_required(n) = n * target_flies * reserve_per_fly / bp_utilization
```

| Contracts | Equity req. (5-fly target) | (4-fly target) |
|---|---|---|
| 1 | $25,000 | $20,000 |
| 2 | $50,000 | $40,000 |
| 3 | $75,000 | $60,000 |
| 4 | $100,000 | $80,000 |
| 5 | $125,000 | $100,000 |

### 5.2 Weekly contract scaling

```
winning week (realized weekly P&L > 0)  -> contracts += 1
losing  week (realized weekly P&L < 0)  -> contracts -= 1
flat    week                            -> unchanged

contracts = max(1, min(scaled, floor(bp_budget / (target_flies * reserve_per_fly))))
```

- Week = Mon–Fri, evaluated after Friday settlement.
- Start **1 contract**, hard floor **1**.
- Applies to the *next* week. Never resize intraweek.
- P&L realized **including settlement**, net of commissions and fees.

> **Known defect, pre-registered for Stage 7.** Adding a contract after a winning
> week places maximum size immediately before the tail event, on a negatively
> skewed strategy. This is what forced the source operator's own size cut after
> two record losing days. `equity_banded` is the alternative.

### 5.3 Live fly cap

```
strategy_cap  = 2 if time < 14:00 ET else 10          # §3.4
effective_cap = min(strategy_cap, flies_by_bp, 10)
```

The drop rule is a **BP recycler** — every drop frees a full reserve and is often
what permits the next add.

An add blocked by `flies_by_bp` rather than `strategy_cap` emits
`BP_BLOCKED_ADD`. Treat as **strategy degradation, not risk control** — a blocked
add leaves a fly drifting ITM with no offsetting position, the exact scenario the
add exists to handle.

**Feedback loop:** if `BP_BLOCKED_ADD` occurs on more than
`bp_block_tolerance_days` (default **2**) days in a week, force `contracts -= 1`
next week regardless of P&L.

### 5.4 The $25k account

At $25,000, 80% utilization ($20,000 usable), ~$3,500–4,000 per fly:

| Contracts | Flies supported | Verdict |
|---|---|---|
| 1 | **5** | Full strategy, comfortable |
| 2 | 2 | Degraded — cap binds before the close stack |
| 3 | 1 | Not the strategy |

**$25k runs the strategy properly at 1 contract.** Fly capacity is not the
constraint; contract scaling is. First step to 2 contracts needs ~$50,000.

Consequence: **the weekly scaling rule contributes nothing until equity roughly
doubles.** Below ~$50k, `fixed_1` and `weekly_scaling` produce identical curves —
if they differ in the train window, there is a bug.

### 5.5 PDT constraint

```
if account_equity < 25000: no intraday round-trips permitted
```

Emit `PDT_LOCKOUT` days. $25k is simultaneously the minimum for the strategy to
function *and* the level below which it stops functioning. No buffer. Report days
spent within 10% of the threshold.

### 5.6 Required sizing modes

| Mode | Behaviour |
|---|---|
| `fixed_1` | 1 contract always — isolates pure strategy edge |
| `weekly_scaling` | §5.2 with live BP caps — the mandated model |
| `equity_banded` | §5.1 ladder only; no reference to win/loss streaks |
| `unconstrained` | §5.2 with no BP/PDT caps — measures what the caps cost |

---

## 6. Data requirements

### 6.1 Subscriptions — TWO required

Vendor-confirmed: Index is a **separate product** from Options. An Options plan
does **not** include SPX or VIX.

| Product | Tier | Provides | First access |
|---|---|---|---|
| **Options** | Value (~$40/mo) | SPXW chain: EOD, OHLC, **Quote**, Open Interest | **2020-01-01** |
| **Index** | Value | **SPX 1-min spot, VIX 1-min, SPX EOD close** | **2023-01-01** |

Index Value is **mandatory** — three hard requirements depend on it: the spot
series driving every trigger (§3.2, §3.3), VIX for the halt rules (§4), and the
settlement close (§3.8). Both first-access dates cover the 2023-09 lookback.

### 6.2 Datasets and windows

| # | Dataset | Source | Granularity | Window |
|---|---|---|---|---|
| 1 | SPX index price | Index | 1-min | 2 yrs |
| 2 | SPXW 0DTE NBBO | Options `option/history/quote` | 1-min | 2 yrs |
| 3 | Implied volatility | computed from #2 | 1-min | 2 yrs **+ 1 yr lookback** |
| 4 | VIX | Index | 1-min + daily open | 2 yrs (+1 yr) |
| 5 | **SPX EOD close (= settlement)** | Index `index/history/eod` | daily | 2 yrs |
| 6 | Econ calendar flags | external | daily | 2 yrs |

**Train/holdout split — fixed now, before any run:**

| Window | Period | Use |
|---|---|---|
| **Train** | 2024-09 → 2025-12 | All tuning, Stages 1–7 |
| **Holdout** | 2026-01 → 2026-09 | Touched **once**, Stage 8 |

The loader takes an explicit window parameter and **refuses the holdout unless
`stage=8`**. Enforce in code — the one guardrail that cannot be restored once
broken.

### 6.3 Strike coverage

`strike_range=30` returns 30 strikes above + 30 below + ATM = **61 strikes** ≈
**±150 points** on a 5-point grid.

≈122 contracts/day × 390 min ≈ **47.6k rows/day** ≈ **24M rows over 2 years** ≈
**1–2 GB** partitioned Parquet. Partition by date, memory-map per day.

**Lookback-year optimization:** IV Rank needs only ATM implied vol, so pull
`strike_range=5` (±25 points) for 2023-09 → 2024-09. Cuts roughly a third off
total download with no loss of fidelity.

### 6.4 Greeks and IV

**Greeks are not required** — every trigger is price-distance based. Trade, Trade
Quote and Greeks endpoints require Standard+; we need none of them.

**Compute IV from the quote midpoint** with a fixed rate/dividend assumption.
0DTE IV inversion goes numerically unstable in the final hour as extrinsic
collapses — return `None` and log, never a garbage number. Report failure rate by
hour.

Fallback: **VIX percentile rank**, via `iv_rank_source: "computed" | "vix_proxy"`.

### 6.5 Fill modelling — with snapshot latency

At Value, `interval=1m` returns the **last NBBO at that timestamp** — a
point-in-time snapshot, not a bar. No intraminute high/low on quotes. (The option
OHLC endpoint *is* included, but its high/low come from **trades**, not quotes;
there is no NBBO high/low aggregate.)

| Mode | Assumption |
|---|---|
| `optimistic` | Mid |
| `realistic` | Mid + `0.10` against you, per fly per side |
| `pessimistic` | Full bid/ask |

**`snapshot_latency` (new, mandatory).** A trigger firing between snapshots is
priced at the **next available snapshot**, not at the trigger moment — up to 60
seconds of latency. Model it explicitly; it is a systematic drag, not noise, and
it is closer to live behaviour than instantaneous fills.

Also model commissions (per-leg × 4 legs × 2 sides), exchange fees, and no-fill
when combo spread > `1.50` (§3.9).

**Edge that survives only at `optimistic` does not exist.**

### 6.6 Confirmed request shape

```
GET /v3/option/history/quote
    root=SPXW
    expiration=*
    strike_range=30
    max_dte=0
    right=both
    interval=1m
    date=YYYYMMDD
```

Vendor-confirmed: all parameters supported at Value; "exactly the supported
shape." One request per date, ~750 requests total.

**Bulk — resolved, no upgrade needed.** Two meanings were conflated:

| Meaning | Tier | Relevant? |
|---|---|---|
| Full chain for one underlying, one request (`expiration=*`) | **Value** | **Yes — what we need** |
| Flat Files: whole market, one date | Professional, **7 most recent days only** | No |

Flat Files cover only the last 7 calendar days — useless for a 2-year backtest at
any tier. **Do not upgrade to Professional for "bulk."**

**Build on v3.** Vendor-recommended; all parameters above are v3.

**Operational:** requires a local **Theta Terminal v3** process. The downloader
must fail loudly when it is not up, and must checkpoint/resume. **Cache raw
responses to disk before any transformation** — re-downloading two years because
of a parsing bug is the most avoidable time sink in this project. The terminal
prints exact access levels at startup; verify they match both subscriptions.

### 6.7 OPEN BLOCKING QUESTION — resolve before D4

Everything in §6.5 concerns *option* data. But **every trigger in this strategy
reads SPX, not option prices.**

> Does `index/history/price` with `interval=1m` return an **OHLC bar** (open,
> high, low, close) per minute, or a **point-in-time snapshot** like the option
> quote endpoint? If snapshot-only, is there an `index/history/ohlc` at 1-minute
> on Index Value?

| If SPX 1-min is… | Consequence |
|---|---|
| **OHLC** | Intrabar touch detection works. M5 runs as specified (§13.1). |
| **Snapshot only** | Touches and adds detectable **only at minute boundaries**. Any touch that occurs and reverts inside a minute is **invisible**. |

The snapshot-only case is a **systematic bias, not noise** — it under-counts both
drops and adds, and the source author explicitly describes touches that revert
before he can click. That is exactly the population that would vanish.

It does not necessarily kill the project; minute-boundary detection is arguably
closer to what a human operator reacts to. But it changes the M5 test from *"which
order did events occur within the bar?"* to *"how much does the strategy change
when sub-minute touches are invisible?"* — different test, different
implementation.

**Do not implement §13.1 or launch the full pull until this is answered.**

### 6.8 Data quality gates — build before the first backtest run

1. **Missing strikes** — flag any minute where a needed strike is absent.
2. **Crossed/locked quotes** (bid ≥ ask) — drop or forward-fill; log counts.
3. **Zero-bid wings** — common deep OTM; decide fill policy explicitly.
4. **Stale quotes** — unchanged bid/ask > N minutes on a near-ATM strike.
5. **Half days** — 13:00 ET closes break the 14:00 tight switch and settlement.
6. **0DTE availability** — SPX did not always list 0DTE on all five weekdays.
   Verify expiry availability per date; never assume.

Do not trust a replay that has not passed all six.

---

## 7. Modelling hazards

### 7.1 Intrabar / inter-snapshot sequencing
**Definition pending §6.7.** If SPX 1-min is OHLC, you know the high and low but
not their order — default `conservative` (adverse event first). If snapshot-only,
the hazard is instead *unobservable* sub-minute excursions. Either way, expose
`intrabar_assumption` and quantify it at §13.1.

### 7.2 Settlement, not last trade
SPXW settles to the SPX index close from `index/history/eod.close`. The index
prints until ~16:04–16:05 ET; the 16:00:00 quote is **not** settlement. Using it
biases every held fly in the same direction. See §3.8.

### 7.3 Touch confirmation fidelity
`touch_confirm_seconds = 10` cannot be evaluated at 1-min granularity. In
backtest, degrade to the best available detection (§6.7) and **state the
approximation in the report header**. A known live-vs-backtest divergence, not a
bug to hide.

### 7.4 Sequence risk
Outcomes depend heavily on start date. Report rolling 3-month windows and the
worst start date — never a single aggregate CAGR. Weekly sizing amplifies this.

### 7.5 Overfitting surface
Enough knobs to fit anything. **Fix all parameters at documented values for the
primary run.** Any sweep is a separate, labelled sensitivity study. §13.0 governs.

### 7.6 The structural prior
Selling an ATM 0DTE straddle is selling gamma at the highest-gamma point on the
surface; the premium is approximately fair. The add/drop cycle does not obviously
*create* edge — it **reshapes the distribution** into many small wins and rare
large losses. Judge results against that prior.

---

## 8. Acceptance tests

**Scenario A — chaotic winner (+$849/contract)**
```
open 4570 → +7 → add 4580 → touch 4570 → drop 4580
→ −7 → add 4560 → touch 4560 → drop 4570
→ +7 → add 4570 → touch 4560 → drop 4570
→ +7 → add 4570 → 14:00 ET switch → +2.5 → add 4565
→ hold 4560 / 4565 / 4570 into settlement
```
Assert: never > 2 flies before the tight switch; exactly 3 at settlement.

**Scenario B — drifting down (+$112/contract)**
```
open 4570 → add 4560 → tight switch → add 4565
→ touch 4560 → drop 4570 → drift down → add 4550 → add 4555
→ touch 4550 → drop 4560 → add 4545
→ hold 4545 / 4550 / 4555 into settlement
```

**Scenario C — loser.** Synthetic EKG day (±20 pt oscillation); assert the engine
churns flies and loses. The test is that it *degrades as described*.

**Unit tests, minimum:**
- `be_pct` exit: credit 15 / center 5430 → `5417.25 / 5442.75`
- Tier selection at credit exactly `15.00` and `20.00` → both mid tier
- `snap_strike` at every add offset
- Two-fly cap suppresses a third add at 13:59, permits at 14:01
- `drop_check` runs before `add_check` (ordered event log)
- AM-settled SPX root rejected at entry
- Stop loss: credit 15.00 → −$375/contract
- **Stop unreachability:** at credit 27.4, the §3.6 exit triggers before the §3.7
  stop at 5h, 3h, 1h and 0.2h to expiry
- **Combo integrity:** a simulated 3-of-4 leg fill raises
  `FILL_INTEGRITY_BREACH` and never persists to the next cycle
- **Settlement uses `index/history/eod.close`, not the 16:00 quote**
- **`snapshot_latency`:** a trigger between snapshots prices at the next one
- Weekly sizing: win→+1, loss→−1, floor at 1
- Contract step-up blocked below $50k at a 5-fly target
- `BP_BLOCKED_ADD` on 3 days in one week forces `contracts -= 1` next week
- PDT lockout blocks round-trips when equity < $25,000
- **Loader refuses holdout window unless `stage=8`**

---

## 9. Report output

**Per run:** total P&L, win rate, avg win, avg loss, largest win, largest loss,
max drawdown (%), Sharpe, **profit factor (primary metric)**, worst 5 days,
**consecutive-losing-day distribution**.

**Required diagnostics:**

| Diagnostic | Why |
|---|---|
| **Whipsaw cost** | Realized P&L on add-then-drop round trips closing within 30 min. The strategy's recurring toll. |
| **Exit-reason split** | Any material `stop_25` is a **bug signal** (§3.7). |
| **Profitable-breakeven-exit share, by hour** | Detects the §3.6 late-session inversion. |
| **`HALT_PREEMPTED_DROP`** + halt-exit slippage | Prices what the vol-flatten rule costs. |
| **Rejected entries** (spread > 1.50) | A strategy that cannot get filled has not traded. |
| **Fly-count distribution** | 2 pre-14:00, 2–3 at settlement. Routinely 1 = adds not firing. |
| **Snapshot-latency cost** | P&L delta vs. instantaneous fills — isolates §6.5 drag. |

**Sizing section:** realized vs. unconstrained contract count, `BP_BLOCKED_ADD`
days, forced size-downs, `PDT_LOCKOUT` days, days within 10% of $25k, peak/mean
BP utilization, date equity first supported 2 contracts.

**Headline reported at `fill=realistic` × `intrabar=conservative` only.**

---

## 10. Remaining operational questions

1. **Commission schedule** — default $0.65/contract/leg ($5.20/fly round trip)
   plus SPX exchange fees. Confirm actual Tradier rates.
2. **Half days / no-0DTE dates** — default **skip entirely**, log as excluded.
3. **Partial first week** — ignore weeks with < 3 trading days for scaling.
4. **Paper-trade duration before live** — minimum **20 sessions**.
5. **Index Value monthly price** — confirm on the pricing page.

---

## 11. Build order — data-first

| # | Step | Gate |
|---|---|---|
| **D0** | Subscribe Options Value **+ Index Value**; launch Theta Terminal v3; **resolve §6.7** | Terminal prints access for both products |
| **D1** | Probe one day: option chain + SPX index sample | ~122 contracts × 390 min; SPX shape known |
| **D2** | Downloader: raw cache, checkpoint/resume, Parquet | 20 pilot days on disk |
| **D3** | Quality gates (§6.8) + loader with holdout guard | All six pass; guard provably raises |
| **D4** | **Full pull** — unattended | Manifest complete; gates pass across range |
| M1 | `core/` rules, sizing, integrity + unit tests | §8 unit tests green. **No data needed — build during D4** |
| M3 | `adapters/backtest.py` + `core/engine.py` | One real day replays; §3.5 ordering provable |
| M4 | §8 scenario tests (synthetic surfaces) | Sequences A and B reproduce |
| M5 | **§13.1 path-sensitivity gate** | **Spread < 50%, or STOP** |
| M6 | Stages 2–7 (§13) | ≤42 runs, ledger complete |
| M7 | Stage 8 holdout | One shot, 6 runs |
| M8 | **Written verdict** | **Human sign-off — hard stop** |
| M9 | `adapters/tradier.py`, paper | ≥20 sessions |
| M10 | Live, 1 contract | — |

**D1–D3 use 20 pilot days only.** A schema mistake replicated across 500 days is
hours wasted. **M1 runs concurrently with D4** — the rules layer is pure.

---

## 12. Module layout

```
nfh/
  config/default.yaml       # every number in §3–§5; nothing hard-coded
  core/
    fly.py                  # Fly: legs, center, credit, breakevens, max_risk, mark()
    book.py                 # open flies, anchor selection, cap enforcement
    rules.py                # PURE: should_add / should_drop / should_close
    sizing.py               # PURE: weekly scaling, equity-scaled fly cap, PDT
    engine.py               # §3.5 cycle loop — adapter-agnostic
    integrity.py            # §3.9 combo-fill verification
    calendar.py             # FOMC/CPI/PPI/PCE/NFP/witching/OpEx/EOQ + half days
  data/
    thetadata.py            # v3 downloader: checkpoint, resume, raw cache
    loader.py               # parquet → ChainSnapshot; train/holdout guard
    chain.py                # strike → (bid, ask); snap_strike
    index.py                # SPX spot, VIX, settlement close (§6.1 Index product)
    iv.py                   # IV from mid; IV Rank; VIX proxy fallback
    quality.py              # §6.8 gates
  adapters/
    backtest.py
    tradier.py
  backtest/
    runner.py               # §13 stage orchestration
    report.py               # §9 metrics
    ledger.csv              # append-only audit trail
  tests/
```

Hard requirements:
- `core/rules.py` and `core/sizing.py` are **pure functions** — no I/O, no clock,
  no network. This is what makes the strategy testable without a subscription.
- The engine must not know whether it is backtesting or live. **One code path.**
- Every number in §3–§5 comes from config.

---

## 13. Experiment matrix

### 13.0 Selection protocol — non-negotiable

A full grid of the axes below is **186,624 runs** against ~500 trading days. The
best of 186k runs is overfit with near-certainty.

1. **Split the data before the first run** (§6.2). Holdout touched **once** at
   §13.8. Run it twice and it is burned — no recovery short of new market data.
2. **Test sequentially, not as a grid.** Each stage fixes its winner; the next
   varies one family against that base. **48 runs total.** Adopt only if it beats
   the base by more than 1σ (§13.9).
3. **Judge every stage at `fill=realistic`, `intrabar=conservative`.**

**Accepted trade-off:** sequential testing can miss interaction effects. A grid
able to find real ones would surface thousands of false ones. A missed
interaction costs upside; a fitted one costs capital.

### 13.1 Stage 1 — Path-sensitivity pilot **(GATE)** ⚠️ BLOCKED ON §6.7

2 runs, 20 train days, baseline config.

**Test definition depends on §6.7.** If SPX 1-min is OHLC: conservative vs.
optimistic intrabar ordering. If snapshot-only: measure sensitivity to
unobservable sub-minute excursions instead.

**Gate:** if the P&L spread exceeds **50% of mean P&L**, 1-minute data cannot
resolve this strategy. **Stop.** Upgrade to Standard (~$80, tick-level, same API)
or abandon.

Also report same-bar/same-interval collision counts — a high count with a passing
spread means you were lucky, not safe.

### 13.2 Stage 2 — Baseline surface (6 runs)
`fill_mode` (3) × `intrabar_assumption` (2). Run the §13.13 defect checklist
before interpreting any P&L.

### 13.3 Stage 3 — Exit family (12 runs)
`stop_basis` (credit / max_risk / none) × `be_mode` (static_85 / static_80 /
time_scaled / dual_mode). Report exit-reason split and profitable-breakeven-exit
share by hour. Expect `none` ≈ `credit` (confirms §3.7).

### 13.4 Stage 4 — Calendar filters (4 runs)
Skip CPI × Skip Triple Witching. **Judge on the left tail**, not the mean.

### 13.5 Stage 5 — Volatility halts (6 runs)
`vol_halt_mode` (flatten / halt_adds_only / off) × `max_vix` (30 / none). Report
halt-exit slippage vs. normal.

### 13.6 Stage 6 — Structure (6 runs)
`wing_cap_mode` (premium_only / vol_capped) × `touch_confirm` (instant / 10s /
30s). **Drop `touch_confirm` entirely if Stage 1's spread was wide** — it is
unresolvable and must be settled live.

### 13.7 Stage 7 — Sizing (6 runs)
`sizing_mode` (fixed_1 / weekly_scaling / equity_banded) × `start_equity` (25000 /
40000). Below ~$50k, `fixed_1` and `weekly_scaling` **must** match — a difference
is a bug.

### 13.8 Stage 8 — Holdout **— ONE SHOT** (6 runs)
Top 3 configs + unmodified baseline, holdout window, `stage=8`. Adopted only if
it beats baseline in the same **direction and rough magnitude** as in train.
**Rank inversion ⇒ ship the baseline.**

### 13.9 Metrics and margins

Primary: **profit factor at `realistic` × `conservative`.** Not total P&L.

**Noise margin.** Before Stage 2, run the baseline on 10 bootstrap resamples of
train; record profit-factor σ. **Any improvement < 1σ is noise.** Prefer the
simpler config on ties.

### 13.10 Build-only items — never test axes

| Item | Why |
|---|---|
| **Combo-fill integrity (§3.9)** | A partial fill leaves you naked short an SPX option. |
| **AM/PM settlement guard (§2)** | An AM-settled root in the book is a silent catastrophic bug. |
| **Data quality gates (§6.8)** | A failing replay produces numbers, not results. |
| **Evaluation order (§3.5)** | Drop before add is correctness. |
| **Train/holdout guard (§6.2)** | Cannot be restored once broken. |

### 13.11 Run ledger

`backtest/ledger.csv`, appended by the runner, **never hand-edited**:

```
run_id, stage, git_sha, config_hash, data_window, varied params,
profit_factor, total_pnl, max_dd_pct, win_rate, whipsaw_cost,
exit_reason_split, n_days, timestamp
```

The audit trail proving the holdout was touched once.

### 13.12 Total cost

| Stage | Runs |
|---|---|
| 1 gate / 2 / 3 / 4 / 5 / 6 / 7 / 8 | 2 / 6 / 12 / 4 / 6 / 6 / 6 / 6 |
| **Total** | **48** |

≈1.2 hours at 90s/run, vs. ~4,666 hours for the full grid — and **more reliable**.

### 13.13 If Stage 2 fails

**First, rule out defects:** scenario tests passing? Any `stop_25` firing? Fly
count routinely 1? Quality gates clean? Settlement using `eod.close`? If
flies-into-settlement ≈ 1 or `stop_25` fires at all, **stop and fix** — that is
not a result.

**Then judge by magnitude:**

| Profit factor | Action |
|---|---|
| ~0.9–1.0 | Continue. A real defect plausibly eats the edge. |
| ~0.6–0.9 | Continue only with a **mechanically explicable** Stage 3 improvement. |
| < 0.6 | **Stop.** No exit tweak recovers that. |

Check **concentration**: three catastrophic days against 497 profitable ones means
tail management is broken (fixable). Steady daily erosion means you are selling
fair-value gamma and paying spread (not fixable by parameters).

**Kill criteria — written now so they cannot be negotiated later:**
1. Stage 3's best still unprofitable at `realistic` × `conservative` → stop.
2. Recovery requires `optimistic` fills → stop. That edge does not exist.
3. Profitability hinges on one parameter value with no mechanism → stop.
4. Stages 3–7 recover it but Stage 8 does not confirm → stop. Definitive.
