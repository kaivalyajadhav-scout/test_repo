# AGENTS.md — NFH 0DTE SPX Iron Fly Engine

**Authoritative rules specification.** Where `PROJECT.md` summarises, it
simplifies; on any conflict, **this file wins**.

**Revision:** v6 · 2026-09-19
**Companions:** `PROJECT.md` (scope/MVP) · `PROMPTS.md` (build prompts) · `USER_GUIDE.md` (operating manual)

Read end to end before writing code.

### Changes in v6 — D0 results

D0 environment verification completed against SPXW/SPX 2026-09-17. §6.7 is
**resolved**, and resolved favourably.

| § | Change |
|---|---|
| 6.7 | **RESOLVED.** `index/history/price` supports **`interval=1s`** on Index Value — 23,403 dense rows/session, repeats *not* omitted. Touch detection is **sub-minute**, not minute-boundary. |
| 6.2 | SPX and VIX move to **1-second**. Lookback year stays 1-minute. |
| 6.5 | **`snapshot_latency` promoted to a primary parameter.** It is now the dominant modelling hazard, not a minor adjustment. |
| 6.6 | Confirmed: CSV responses, column names, `strike_range = 2N+1`, base URL, both subscriptions active. |
| 6.8 | **New gate #7** — reject index price of 0.0 or >5% deviation. The 09:30:00 print is literally `0.0`. |
| 3.8, 7.2 | Settlement empirically confirmed: `last_trade` 16:04:58, close 7637.76 vs 16:00 price 7637.90 — **0.14 pts = $14/contract**. |
| 7.1 | Intrabar hazard for **triggers** largely dissolved; replaced by trigger/fill asymmetry. |
| 13.1 | M5 redefined a third time — now measures **option-quote latency sensitivity**. Downgraded from hard gate to expected formality. |
| 13.6 | **`touch_confirm` restored as a first-class axis** — directly testable at 1s. |

### Changes in v5
Two subscriptions confirmed required; settlement corrected to
`index/history/eod.close`; `snapshot_latency` introduced; bulk question resolved;
build on v3.

### Changes in v4 / v3
Data-first build order. §3.7 corrected — the −25% stop is measurably
*unreachable*. §3.9 combo-fill integrity added. Train/holdout split fixed
in-spec.

---

## 0. Resolved decisions — do not re-open

| # | Question | Resolution |
|---|---|---|
| 1 | Add reference point | **Anchor fly (recenter after every drop).** Transcript Q&A disregarded. No `first_fly` mode. |
| 2 | Data vendor | **ThetaData.** Options Value (~$40/mo) **+ Index Value** (§6.1). |
| 3 | Stop-loss basis | **−25% of credit received** — but see §3.7, measurably unreachable. Fate decided by Stage 3. |
| 4 | Position sizing | **Equity-scaled.** Flies take priority over contracts. §5. |
| 5 | Account size | **$25,000 starting equity.** §5.4. |
| 6 | API version | **v3.** Base URL `http://127.0.0.1:25503`. |
| 7 | **SPX trigger resolution** | **1-second** (§6.7). Confirmed available on Index Value. |

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
    *not* SPXW. Trade SPXW only. Assert on root/expiry (§8 test).
- Strike grid: 5.00 points near ATM. One helper, `snap_strike(x, grid=5.0)`.
- Multiplier: 100. P&L in dollars per contract unless suffixed `_total`.
- Timezone: compute and store in **US/Eastern**. All times below are ET.

### Iron fly definition

| Leg | Strike | Side |
|---|---|---|
| Short put | `K` | Sell |
| Short call | `K` | Sell |
| Long put | `K − w` | Buy |
| Long call | `K + w` | Buy |

- `credit` — net premium received per contract, in points, positive.
- `center` — `K`. `max_risk` — `(w − credit) × 100` dollars per contract.
- Wings exist **only** for buying-power efficiency. **No trigger may reference a
  wing strike.**

---

## 3. Core rules

### 3.1 Entry — first fly of the day

- Entry time **11:00 ET** — let the open settle.
- Center at-the-money: `center = snap_strike(spx_price)`.
- Wing width **$50** for the first fly.
- Reject if fly mid price outside `[3.00, 50.00]`.
- Reject if combo bid/ask spread > `1.50`.
- Record `open_credit`. Tier selection reads the **anchor fly's** credit.

> **D0 validation.** The 09:30 SPXW quote sampled `bid=5.00 / ask=22.70` — a
> $17.70 spread. The 11:00 entry and the 1.50 spread filter are both doing real
> work. Any staleness or crossed-quote gate must exclude the opening minutes or
> it will fire constantly.

### 3.2 Adds — distance from the anchor fly's center

Anchor fly = the open fly whose center is closest to current SPX.

```
distance = abs(spx_price - anchor.center)
```

| Anchor credit | Trigger distance | Target spacing | Add offset |
|---|---|---|---|
| `< 15` | `> 7.0` | 10 apart | `3.00` |
| `15 ≤ c ≤ 20` | `> 10.0` | 15 apart | `3.00` |
| `> 20` | `> 15.0` | 20 apart | `3.00` |
| after 14:00 ET | `> 2.5` | 5 apart | `0.01` |

Boundaries inclusive on the mid tier — `15.00` and `20.00` both select mid.

```
center_new = snap_strike(spx_price + sign(spx_price - anchor.center) * add_offset)
```

New-fly wing width is **priced, not fixed**: long put strike whose mid is nearest
**$0.40**, clamp `20 ≤ w ≤ 50`, mirror on the call side.

> **Recentering is the defined behaviour.** After a drop, the survivor becomes the
> anchor. No "distance from first fly" mode.
>
> **Spacing note.** One shared offset; `3.00` lands the common tier exactly on
> 10-apart after snapping. The 15/20 tiers land one strike tight in some cases.
> Accepted — do not make the offset tier-dependent.
>
> **Known risk-scaling defect.** The $0.40 wing rule widens wings in high vol,
> *raising* max risk exactly when it should fall. Tested as `wing_cap_mode:
> vol_capped` in Stage 6.

**Evaluation cadence:** adds are evaluated against **1-second** SPX (§6.7), so a
trigger is detected within ~0.5s on average. Pricing the resulting fill is a
separate, coarser problem — see §6.5.

### 3.3 Drops — touch the center

When SPX trades **at or through** the `center` of any open fly, close every other
open fly whose center is **farther from SPX** than the touched fly.

- **Unconditional on P&L.** A dropped fly is usually near its worst mark. That is
  the point — risk reduction, not a loss exit. Never gate on profitability.
- **Touch tolerance:** the touch must hold `touch_confirm_seconds` (default
  **10**). **At 1-second SPX resolution this is directly observable** — 10
  samples per window — and is a genuine Stage 6 axis (§13.6), not an
  approximation.
- Log every drop **and** every suppressed near-touch with SPX, both centers, both
  marks.

> **Structural note — do not "fix" this.** The drop rule is inherently
> buy-high/sell-low: you add after a 7-point move, then drop when price returns.
> Every oscillation pays a toll. This is the strategy's core recurring cost and
> must be measured (§9 whipsaw cost), not engineered away.

### 3.4 Two-fly cap

- **11:00–14:00 ET:** never more than **2** open flies. A third-creating add is
  **suppressed and logged** — not queued.
- **From 14:00 ET:** cap lifts; tight adds run freely, targeting 2–3 flies into
  settlement.
- Hard ceiling **10 positions/day**.

### 3.5 Evaluation order — correctness requirement

```
1. risk_halts()        # VIX change, rapid move, day PT / day max loss
2. breakeven_close()   # §3.6
3. stop_loss()         # §3.7
4. drop_check()        # §3.3 — MUST precede adds
5. add_check()         # §3.2
6. eod_handling()      # settlement
```

Assert the ordering via the event log in tests.

### 3.6 Loss exit — inside breakeven

Breakeven is `center ± credit`. Exit while SPX is still **inside** it, at
`be_pct` (default **0.85**, configurable in `[0.80, 0.90]`):

```
exit_distance = be_pct * credit
close if spx_price <= center - exit_distance or spx_price >= center + exit_distance
```

Computed per-fly from that fly's own credit — a single float, **not** dollar
tiers.

Unit-test verbatim: credit `15.00`, center `5430` → BE `5415 / 5445` → exit at
**`5417.25 / 5442.75`**.

> **Known defect — the late-session inversion.** `be_pct` is a fixed *distance*,
> but the fly's value at that distance changes as extrinsic decays. At 11:00 the
> exit realises a genuine loss; by ~15:45 the same distance is nearly all
> intrinsic and the exit closes at a **profit**. Pre-registered for Stage 3.
> **Required diagnostic:** share of `breakeven` exits closing at a profit, by
> hour.

### 3.7 Stop loss — measurably unreachable

Specified as **−25% of credit received**. Credit `15.00` → stop at `−$375`.

Black-Scholes at 13% vol, 50-wide, credit ≈ 27.4 — the price exit (§3.6) triggers
13–15 points earlier at *every* hour:

| Time left | 85%-BE exit distance | Fly value there | Stop fires at |
|---|---|---|---|
| 5.0h | 23.3 pts | 30.3 | 37.9 pts |
| 3.0h | 23.3 pts | 27.7 | 38.8 pts |
| 1.0h | 23.3 pts | 24.3 | 36.2 pts |
| 0.2h | 23.3 pts | 23.3 | 34.3 pts |

1. Implement as specified. Do **not** delete pre-emptively.
2. **`stop_25` firing is a bug signal.** Any material count → stop and
   investigate: either the price exit is not evaluating or ordering is wrong.
3. Fate decided by Stage 3 (`stop_basis: credit / max_risk / none`).

### 3.8 End of day — settlement (D0-confirmed)

- Survivors are **held into PM cash settlement**. Do not flatten at 15:59.
- **SPXW settles to the official SPX index close**, from `index/history/eod` →
  `close`.
- **Empirically confirmed on 2026-09-17:**

| Field | Value |
|---|---|
| `last_trade` | **16:04:58 ET** |
| `close` (settlement) | 7637.76 |
| 16:00:00 index price | 7637.90 |
| **Difference** | **0.14 pts = $14/contract** |

The gap is small but **one-directional and systematic** across every held fly.
With 2–3 flies into settlement on most days, it compounds. Using the 16:00 print
is a silent bias, not a rounding error.

- Settlement P&L from intrinsic value of all four legs at `eod.close`.
- No new positions after **15:45 ET**.
- Note `index/history/eod` *does* return `open/high/low/close` — daily OHLC exists
  even though intraday does not. Useful for calendar and quality layers.

### 3.9 Combo-fill integrity — mandatory, not tunable

A partial 4-leg fill leaves the account **naked short an SPX option** — undefined
risk on a cash-settled index. At ~20 flies/day × 8 leg-sides, certain.

1. **Combo-only orders.** Never leg in. Atomic or rejected.
2. **Post-fill verification.** All four legs, expected ratios. Mismatch →
   `FILL_INTEGRITY_BREACH`.
3. **Breach handling.** Immediate completion at market; else flatten the partial.
   Never carry an unbalanced position to the next cycle.
4. **Backtest modelling.** No-fill when combo spread > `max_ba_spread_open`
   (1.50). Count and report rejected entries.

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

> **Rapid-move rules now measure precisely.** At 1-second SPX, "$25 in 1 minute"
> is evaluated against a true rolling window rather than two minute-boundary
> samples. Expect these to fire **more often** than a 1-minute implementation
> would — that is correctness, not a regression.

**Three known conflicts — log them, do not silently resolve them:**

1. **Vol halts vs. the drop rule.** They flatten the book exactly where the
   touch-drop would recenter, and market-exit 3–5 flies into a dislocation. Emit
   `HALT_PREEMPTED_DROP`; report halt-exit slippage vs. normal. Stage 5.
2. **Calendar defaults contradict source guidance.** The author sits out CPI and
   month-end/witching; defaults trade them. Stage 4.
3. **Censored sample.** Max VIX 30 conditions results on calm days. Stage 5 via
   `max_vix: none`.

---

## 5. Position sizing — equity-scaled

| Dimension | Controls | Set when |
|---|---|---|
| `contracts` | Leverage per fly | **Weekly**, never intraweek |
| `max_concurrent_flies` | How many flies the book may hold | **Live**, from remaining BP |

```
bp_budget   = account_equity * bp_utilization      # default 0.80
bp_per_fly  = contracts * reserve_per_fly
flies_by_bp = floor(bp_available / bp_per_fly)
```

`reserve_per_fly` = worst-case max risk of a not-yet-opened fly,
`(max_wing_width − min_expected_credit) × 100`. Default **$4,000**. An open fly
reserves its actual `(w − credit) × 100`, typically ~$3,500. Reserve at entry,
release on exit.

### 5.1 Priority rule — flies before contracts

When BP is scarce, **buy fly capacity first, contract count second.** The
add/drop cycle and the close stack *are* the edge; contract count is only
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

### 5.2 Weekly contract scaling

```
winning week -> contracts += 1
losing  week -> contracts -= 1
flat         -> unchanged
contracts = max(1, min(scaled, floor(bp_budget / (target_flies * reserve_per_fly))))
```

Week = Mon–Fri, evaluated after Friday settlement. Start **1**, hard floor **1**.
Applies to the *next* week; never resize intraweek. P&L realized **including
settlement**, net of costs.

> **Known defect, pre-registered for Stage 7.** Adding a contract after a winning
> week places maximum size immediately before the tail event on a negatively
> skewed strategy — what forced the source operator's own size cut.
> `equity_banded` is the alternative.

### 5.3 Live fly cap

```
strategy_cap  = 2 if time < 14:00 ET else 10
effective_cap = min(strategy_cap, flies_by_bp, 10)
```

The drop rule is a **BP recycler** — every drop frees a full reserve. An add
blocked by `flies_by_bp` emits `BP_BLOCKED_ADD` and is **strategy degradation,
not risk control**: it leaves a fly drifting ITM with no offsetting position.

**Feedback loop:** `BP_BLOCKED_ADD` on more than `bp_block_tolerance_days`
(default **2**) in a week → force `contracts -= 1` next week regardless of P&L.

### 5.4 The $25k account

| Contracts | Flies supported | Verdict |
|---|---|---|
| 1 | **5** | Full strategy, comfortable |
| 2 | 2 | Degraded — cap binds before the close stack |
| 3 | 1 | Not the strategy |

**$25k runs the strategy properly at 1 contract.** Fly capacity is not the
constraint; contract scaling is. First step to 2 contracts needs ~$50,000.
Consequence: **weekly scaling contributes nothing until equity roughly doubles.**
Below ~$50k, `fixed_1` and `weekly_scaling` must produce identical curves — a
difference is a bug.

### 5.5 PDT constraint

```
if account_equity < 25000: no intraday round-trips permitted
```

Emit `PDT_LOCKOUT`. $25k is simultaneously the minimum for the strategy to
function *and* the level below which it stops. No buffer. Report days within 10%
of the threshold.

### 5.6 Required sizing modes

`fixed_1` · `weekly_scaling` · `equity_banded` · `unconstrained`

---

## 6. Data requirements

### 6.1 Subscriptions — TWO required, both CONFIRMED ACTIVE

| Product | Tier | Provides | First access |
|---|---|---|---|
| **Options** | VALUE | SPXW chain: EOD, OHLC, **Quote**, Open Interest | **2020-01-01** |
| **Index** | VALUE | **SPX 1s/1m spot, VIX, SPX eod.close** | **2023-01-01** |
| Stock | FREE | unused | — |
| Rate | FREE | unused | — |

Both cover the 2023-09 lookback. Index is **mandatory** — spot, VIX and
settlement all live there.

### 6.2 Datasets and windows

| # | Dataset | Source | Granularity | Window |
|---|---|---|---|---|
| 1 | SPX index price | Index | **1-second** | 2 yrs |
| 2 | SPXW 0DTE NBBO | Options `option/history/quote` | 1-min | 2 yrs |
| 3 | Implied volatility | computed from #2 | 1-min | 2 yrs **+ 1 yr** |
| 4 | VIX | Index | **1-second** + daily open | 2 yrs |
| 5 | **SPX EOD close (= settlement)** | Index `index/history/eod` | daily | 2 yrs |
| 6 | Econ calendar flags | external | daily | 2 yrs |

**Lookback year (2023-09 → 2024-09):** 1-minute suffices — it feeds IV Rank only.
Use `strike_range=5`.

**Volume:** SPX 1s ≈ 23,400 rows/day × 504 ≈ **11.8M rows ≈ 0.19 GB**. Options
1-min ≈ 47.6k rows/day ≈ **24M rows ≈ 1.4 GB**. Total comfortably under 2 GB.

**Train/holdout split — fixed, before any run:**

| Window | Period | Use |
|---|---|---|
| **Train** | 2024-09 → 2025-12 | All tuning, Stages 1–7 |
| **Holdout** | 2026-01 → 2026-09 | Touched **once**, Stage 8 |

The loader takes an explicit window parameter and **refuses the holdout unless
`stage=8`**. Enforce in code — the one guardrail that cannot be restored once
broken.

### 6.3 Strike coverage

**Confirmed:** `strike_range=N` returns `2N+1` strikes (`strike_range=2` → 5).
So **`strike_range=30` → 61 strikes ≈ ±150 points**, matching the requirement.

### 6.4 Greeks and IV

**Greeks are not required** — every trigger is price-distance based. Trade, Trade
Quote and Greeks endpoints require Standard+; we need none.

**Compute IV from the quote midpoint** with a fixed rate/dividend assumption.
0DTE IV inversion goes numerically unstable in the final hour — return `None` and
log, never a garbage number. Report failure rate by hour. Fallback: **VIX
percentile rank** via `iv_rank_source: "computed" | "vix_proxy"`.

### 6.5 Fill modelling — the dominant hazard

**Resolving §6.7 did not remove the modelling risk; it relocated it.**

```
SPX triggers:   1-second resolution   (precise)
Option pricing: 1-minute snapshots    (coarse)
```

A touch detected at 14:32:07 cannot be priced until the **14:33:00** snapshot —
up to **59 seconds** of latency between decision and priceable fill. This is
**systematically adverse**: price has usually continued moving in the direction
that triggered the exit.

At Value, `interval=1m` returns the **last NBBO at that timestamp** — a
point-in-time snapshot, not a bar. The option OHLC endpoint is included but its
high/low come from **trades**, not quotes; there is no NBBO high/low aggregate.

**`snapshot_latency` — a primary parameter, not an adjustment:**

| Mode | Assumption |
|---|---|
| `next_snapshot` | Price at the next 1-min option snapshot — **default, realistic** |
| `same_snapshot` | Price at the preceding snapshot — optimistic bound |

**Fill modes:**

| Mode | Assumption |
|---|---|
| `optimistic` | Mid |
| `realistic` | Mid + `0.10` against you, per fly per side |
| `pessimistic` | Full bid/ask |

Also model commissions (per-leg × 4 legs × 2 sides), exchange fees, and no-fill
when combo spread > `1.50`.

**Edge that survives only at `optimistic` / `same_snapshot` does not exist.**

### 6.6 Confirmed API shapes

**Base URL:** `http://127.0.0.1:25503` · **Responses: CSV, not JSON.** Parse
accordingly; cache raw CSV before any transformation.

```
GET /v3/option/history/quote
    root=SPXW  expiration=*  strike_range=30  max_dte=0
    right=both  interval=1m  date=YYYYMMDD
→ root,date,strike,right,timestamp,bid_size,bid_price,ask_price,ask_size

GET /v3/index/history/price
    root=SPX  interval=1s  date=YYYYMMDD
→ timestamp,price

GET /v3/index/history/eod
    root=SPX  date=YYYYMMDD
→ created,last_trade,open,high,low,close,volume,count,
  bid_size,bid_exchange,bid,bid_condition,ask_size,ask_exchange,ask,ask_condition
```

**Sub-1m intervals are single-day requests only** — 504 separate calls for the 1s
index pull. Routine, but the downloader must checkpoint and resume.

**Bulk — resolved.** Full chain for one underlying in one request
(`expiration=*`) works at Value. Flat Files are whole-market dumps limited to the
7 most recent days — irrelevant. **Do not upgrade to Professional.**

**Operational:** Theta Terminal v3 must be running. The downloader must fail
loudly when it is not, and must checkpoint/resume. **Cache raw responses to disk
before any transformation.**

### 6.7 SPX trigger resolution — RESOLVED

`index/history/price` returns only `timestamp,price` — **no OHLC at any intraday
interval**. But the interval enum reaches `1s`, and **`interval=1s` is permitted
on Index Value**.

**D0 measurement, SPX 2026-09-17:**

| Metric | Value |
|---|---|
| Rows returned (09:30–16:00) | **23,403** |
| Distinct timestamps | 23,402 |
| Gaps | 23,400 × 1s, 1 × 2s |
| Repeated prices omitted? | **No** — series is dense, contrary to the docs |

**Therefore touch detection is sub-minute, not minute-boundary.**

| Metric | 1-minute | **1-second** |
|---|---|---|
| Observations/session | 390 | **23,400** |
| Mean time-to-detect a touch | 30 s | **0.5 s** |
| `touch_confirm_seconds = 10` | unresolvable | **directly testable** |

Consequences: §7.1 intrabar ambiguity for *triggers* largely dissolves; §13.1
drops from hard gate to expected formality; `touch_confirm` returns as a
first-class Stage 6 axis; **no Standard-tier upgrade is required.**

### 6.8 Data quality gates — SEVEN, build before the first backtest run

1. **Missing strikes** — flag any minute where a needed strike is absent.
2. **Crossed/locked quotes** (bid ≥ ask) — drop or forward-fill; log counts.
3. **Zero-bid wings** — common deep OTM; decide fill policy explicitly.
4. **Stale quotes** — unchanged bid/ask > N minutes on a near-ATM strike.
   **Exclude the opening minutes** or this fires constantly (§3.1).
5. **Half days** — 13:00 ET closes break the 14:00 tight switch and settlement.
6. **0DTE availability** — verify expiry availability per date; never assume.
7. **Index price validity (new).** Reject any index price of `0.0`, or any print
   deviating more than **5%** from the prior valid print.

> Gate 7 exists because the **09:30:00.000 SPX print is literally `0.0`**, with
> the first valid price at 09:31:00. Entry is at 11:00 so it should never reach
> the engine — but a phantom 7,600-point move must be rejected explicitly, not
> avoided by luck.

Do not trust a replay that has not passed all seven.

---

## 7. Modelling hazards

### 7.1 Trigger/fill asymmetry — the dominant hazard
Triggers resolve at 1-second; fills price at 1-minute snapshots. Up to 59s of
systematically adverse latency (§6.5). This **replaces** intrabar ordering as the
primary risk and is what §13.1 now measures.

### 7.2 Settlement, not last trade
SPXW settles to `index/history/eod.close`. D0-confirmed: final print 16:04:58,
and the 16:00:00 value differs by 0.14 pts = $14/contract, one-directional across
every held fly. See §3.8.

### 7.3 Touch confirmation — now measurable
`touch_confirm_seconds = 10` is directly observable at 1s resolution (10 samples
per window). No longer an approximation; it is a Stage 6 axis. The remaining
live-vs-backtest gap is reaction and order latency, not detection.

### 7.4 Sequence risk
Outcomes depend heavily on start date. Report rolling 3-month windows and the
worst start date — never a single aggregate CAGR.

### 7.5 Overfitting surface
Enough knobs to fit anything. **Fix all parameters at documented values for the
primary run.** §13.0 governs.

### 7.6 The structural prior
Selling an ATM 0DTE straddle is selling gamma at the highest-gamma point on the
surface; the premium is approximately fair. The add/drop cycle does not obviously
*create* edge — it **reshapes the distribution**. Judge results against that
prior.

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
churns flies and loses.

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
- **Settlement uses `eod.close`, not the 16:00 print** — assert the $14/contract
  difference is captured
- **`snapshot_latency`:** a trigger at 14:32:07 prices at the 14:33:00 snapshot
- **Gate 7:** an index price of `0.0` is rejected, not propagated
- **`touch_confirm`:** a 6-second touch does not fire at `10s`; an 11-second one
  does
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
| **Snapshot-latency cost** | P&L delta `next_snapshot` vs `same_snapshot`. Isolates §6.5, the dominant hazard. |
| **Exit-reason split** | Any material `stop_25` is a **bug signal** (§3.7). |
| **Profitable-breakeven-exit share, by hour** | Detects the §3.6 late-session inversion. |
| **`HALT_PREEMPTED_DROP`** + halt-exit slippage | Prices what the vol-flatten rule costs. |
| **Rejected entries** (spread > 1.50) | A strategy that cannot get filled has not traded. |
| **Fly-count distribution** | 2 pre-14:00, 2–3 at settlement. Routinely 1 = adds not firing. |
| **Sub-minute touch count** | Touches that occur and revert within 60s — the population a 1-minute feed would have missed entirely. |

**Sizing section:** realized vs. unconstrained contract count, `BP_BLOCKED_ADD`
days, forced size-downs, `PDT_LOCKOUT` days, days within 10% of $25k, peak/mean
BP utilization, date equity first supported 2 contracts.

**Headline reported at `fill=realistic` × `snapshot_latency=next_snapshot` only.**

---

## 10. Remaining operational questions

1. **Commission schedule** — default $0.65/contract/leg ($5.20/fly round trip)
   plus SPX exchange fees. Confirm actual Tradier rates.
2. **Half days / no-0DTE dates** — default **skip entirely**, log as excluded.
3. **Partial first week** — ignore weeks with < 3 trading days for scaling.
4. **Paper-trade duration before live** — minimum **20 sessions**.
5. **VIX at 1s** — confirm permitted and dense in D1 (SPX is; VIX untested).

---

## 11. Build order — data-first

| # | Step | Gate | Status |
|---|---|---|---|
| D0 | Subscriptions, Terminal v3, **resolve §6.7** | Both products active | ✅ **DONE** |
| D1 | Probe one full day at production settings; time it | Coverage + wall-clock known | ← next |
| D2 | Downloader, 20 pilot days | Pilot on disk | |
| D3 | Seven quality gates + holdout guard | All pass; guard raises | |
| D4 | **Full pull**, unattended | Manifest complete | |
| M1 | `core/` rules, sizing, integrity + unit tests | §8 green. **No data needed — build during D4** | |
| M3 | `adapters/backtest.py` + `core/engine.py` | One real day replays; ordering provable | |
| M4 | §8 scenario tests | Sequences A and B reproduce | |
| M5 | **§13.1 latency-sensitivity gate** | **Spread < 50%** | |
| M6 | Stages 2–7 | ≤42 runs, ledger complete | |
| M7 | Stage 8 holdout | One shot, 6 runs | |
| M8 | **Written verdict** | **Human sign-off — hard stop** | |
| M9 | Tradier paper | ≥20 sessions | |
| M10 | Live, 1 contract | — | |

**M1 runs concurrently with D4.** **M8 is a hard stop.**

---

## 12. Module layout

```
nfh/
  config/default.yaml       # every number in §3–§5; base_url; nothing hard-coded
  core/
    fly.py                  # Fly: legs, center, credit, breakevens, max_risk, mark()
    book.py                 # open flies, anchor selection, cap enforcement
    rules.py                # PURE: should_add / should_drop / should_close
    sizing.py               # PURE: weekly scaling, equity-scaled fly cap, PDT
    engine.py               # §3.5 cycle loop — adapter-agnostic
    integrity.py            # §3.9 combo-fill verification
    calendar.py             # FOMC/CPI/PPI/PCE/NFP/witching/OpEx/EOQ + half days
  data/
    thetadata.py            # v3 CSV downloader: checkpoint, resume, raw cache
    loader.py               # parquet → ChainSnapshot; train/holdout guard
    chain.py                # strike → (bid, ask); snap_strike
    index.py                # SPX 1s spot, VIX, eod.close settlement
    iv.py                   # IV from mid; IV Rank; VIX proxy fallback
    quality.py              # §6.8 — seven gates
  adapters/
    backtest.py             # fill modes + snapshot_latency
    tradier.py
  backtest/
    runner.py               # §13 stage orchestration
    report.py               # §9 metrics
    ledger.csv              # append-only audit trail
  tests/
```

Hard requirements:
- `core/rules.py` and `core/sizing.py` are **pure functions** — no I/O, no clock,
  no network. Testable with no subscription.
- The engine must not know whether it is backtesting or live. **One code path.**
- Every number in §3–§5 comes from config.

---

## 13. Experiment matrix

### 13.0 Selection protocol — non-negotiable

A full grid is **186,624 runs** against ~500 trading days. The best of 186k runs
is overfit with near-certainty.

1. **Split the data before the first run** (§6.2). Holdout touched **once** at
   §13.8. Run it twice and it is burned.
2. **Test sequentially, not as a grid.** Each stage fixes its winner. **48 runs
   total.** Adopt only if it beats the base by more than 1σ (§13.9).
3. **Judge every stage at `fill=realistic`, `snapshot_latency=next_snapshot`.**

**Accepted trade-off:** sequential testing can miss interaction effects. A grid
able to find real ones would surface thousands of false ones. A missed
interaction costs upside; a fitted one costs capital.

### 13.1 Stage 1 — Latency-sensitivity pilot (2 runs)

**Redefined.** With §6.7 resolved, this no longer tests intrabar ordering or
sub-minute invisibility. It now measures **option-quote latency sensitivity**:

| Run | `snapshot_latency` |
|---|---|
| 1.1 | `next_snapshot` (realistic) |
| 1.2 | `same_snapshot` (optimistic bound) |

20 train days, baseline config, `fill=realistic`.

**Threshold unchanged:** if the P&L spread exceeds **50% of mean P&L**, the
strategy's measured edge is dominated by fill-timing assumptions and no
downstream result is trustworthy. Stop and reconsider.

**Expectation has changed.** Previously a hard go/no-go that could kill the
project; now expected to pass. Treat a failure as genuinely surprising and
investigate for bugs before accepting it.

Also report the **sub-minute touch count** — touches that occurred and reverted
within 60 seconds. That number is exactly what a 1-minute feed would have missed,
and it quantifies what the 1s pull bought you.

### 13.2 Stage 2 — Baseline surface (6 runs)
`fill_mode` (3) × `snapshot_latency` (2). Run the §13.13 defect checklist before
interpreting any P&L.

### 13.3 Stage 3 — Exit family (12 runs)
`stop_basis` (credit / max_risk / none) × `be_mode` (static_85 / static_80 /
time_scaled / dual_mode). Report exit-reason split and profitable-breakeven-exit
share by hour. Expect `none` ≈ `credit` (confirms §3.7).

### 13.4 Stage 4 — Calendar filters (4 runs)
Skip CPI × Skip Triple Witching. **Judge on the left tail**, not the mean.

### 13.5 Stage 5 — Volatility halts (6 runs)
`vol_halt_mode` (flatten / halt_adds_only / off) × `max_vix` (30 / none). Report
halt-exit slippage vs. normal. Note the rapid-move rules now evaluate on true
rolling 1s windows (§4).

### 13.6 Stage 6 — Structure (6 runs)

**`touch_confirm` restored as a first-class axis** — directly measurable at 1s.

| Axis | Values |
|---|---|
| `wing_cap_mode` | `premium_only` / `vol_capped` |
| `touch_confirm` | `instant` / `5s` / `10s` / `30s` |

This directly prices the whipsaw toll — the strategy's core recurring cost — and
tests the exact mechanism the source author describes reacting to manually. It is
arguably the most informative single axis in the matrix.

### 13.7 Stage 7 — Sizing (6 runs)
`sizing_mode` (fixed_1 / weekly_scaling / equity_banded) × `start_equity` (25000 /
40000). Below ~$50k, `fixed_1` and `weekly_scaling` **must** match — a difference
is a bug.

### 13.8 Stage 8 — Holdout **— ONE SHOT** (6 runs)
Top 3 configs + unmodified baseline, holdout window, `stage=8`. Adopted only if
it beats baseline in the same **direction and rough magnitude** as in train.
**Rank inversion ⇒ ship the baseline.**

### 13.9 Metrics and margins

Primary: **profit factor at `realistic` × `next_snapshot`.** Not total P&L.

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
snapshot_latency_cost, exit_reason_split, n_days, timestamp
```

The audit trail proving the holdout was touched once.

### 13.12 Total cost

| Stage | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | **Total** |
|---|---|---|---|---|---|---|---|---|---|
| Runs | 2 | 6 | 12 | 4 | 6 | 6 | 6 | 6 | **48** |

≈1.2 hours at 90s/run, vs. ~4,666 hours for the full grid — and **more reliable**.

### 13.13 If Stage 2 fails

**First, rule out defects:** scenario tests passing? Any `stop_25` firing? Fly
count routinely 1? All seven quality gates clean? Settlement using `eod.close`?
Any index price of 0.0 reaching the engine? If flies-into-settlement ≈ 1 or
`stop_25` fires at all, **stop and fix** — that is not a result.

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
1. Stage 3's best still unprofitable at `realistic` × `next_snapshot` → stop.
2. Recovery requires `optimistic` fills or `same_snapshot` → stop.
3. Profitability hinges on one parameter value with no mechanism → stop.
4. Stages 3–7 recover it but Stage 8 does not confirm → stop. Definitive.
