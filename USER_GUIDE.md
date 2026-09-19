# PROJECT.md — NFH 0DTE SPX Iron Fly Engine

**Status:** pre-build, data phase · **Revision:** v2 · 2026-09-18
**Companion files:** `AGENTS.md` v5 (rules spec — authoritative) · `PROMPTS.md` (build prompts) · `USER_GUIDE.md` v2 (operating manual)

---

## 1. What this project is

A **standalone Python engine** that backtests, and later executes, a 0DTE SPX
iron fly strategy known as "Not for the Faint of Heart" (NFH).

Two layers:

1. A **rules engine** implementing the strategy as pure, testable functions.
2. **Adapters** that feed it data — historical replay first, broker second.

### What this project is NOT

| Not this | Why it matters |
|---|---|
| An Options Alpha bot | OA bots are built in a visual no-code editor and cannot be authored as source. The OA template was only the origin of the parameter defaults in `AGENTS.md` §4. Do not build an OA integration, API client, or replication layer. |
| A general options backtesting framework | Single strategy, single instrument. Resist generalising. |
| A live trading system (yet) | Live execution is Phase 2 and is **blocked** on backtest sign-off. |
| A parameter optimiser | See §5. The goal is a defensible answer, not a maximised number. |

---

## 2. The strategy in plain terms

Sell an at-the-money 0DTE iron fly on SPX at 11:00 ET, once the open has settled.
As SPX drifts away from the fly's center strike, add a second fly in the
direction of travel. When price comes back and touches the center of the nearer
fly, drop the farther one. Repeat all afternoon. After 14:00 ET, switch to
tighter spacing and deliberately stack 2–3 flies into cash settlement, where most
of the day's remaining theta decays. Cut any individual fly while it is still
inside its expiration breakeven so losses stay small relative to wins.

The source author's analogy: you are pouring water from a moving pitcher into a
glass, and when the pitcher drifts past the rim you add a second glass rather
than chase with the first.

**Mechanically:** short gamma at the highest-gamma point on the surface, with a
rules-based recentering hedge, held into cash settlement.

---

## 3. The actual research question

> Does the add/drop recentering cycle generate edge, or does it merely reshape a
> fair-value premium sale into many small wins and rare large losses?

**Null hypothesis to disprove — not confirm:**
NFH is a fair-value premium-selling strategy whose reported results came from
operator discretion and a favourable volatility regime, and which does not
survive mechanical execution at realistic fills.

The source operator openly overrides his own rules intraday. Expect the
mechanical version to underperform the anecdote. **Build the backtest to try to
kill the strategy.** A build that can only produce flattering numbers has failed
regardless of what it prints.

---

## 4. MVP definition

The narrowest artifact that can answer §3 credibly.

### In scope

| # | Deliverable | Done when |
|---|---|---|
| 1 | Core rules engine (`core/`) | Pure functions; all `AGENTS.md` §8 unit tests pass |
| 2 | ThetaData v3 downloader (Options + Index) | 2yr SPXW 1-min NBBO + SPX/VIX cached, resumable, all §6.8 gates pass |
| 3 | Backtest adapter + engine loop | Replays a full day; §3.5 evaluation order provably enforced |
| 4 | Scenario validation | §8 Scenarios A and B reproduce the fly sequence |
| 5 | Reporting | §9 metrics + `backtest/ledger.csv` audit trail |
| 6 | Staged experiment runner | §13 stages 1–8 executable and reproducible from config |

### Explicitly out of scope for MVP

- Tradier / live / paper execution (Phase 2)
- Any UI, dashboard, web service, or notebook front-end
- Multi-symbol, multi-strategy, or non-0DTE support
- Real-time streaming or websockets
- Portfolio-level or cross-strategy risk management
- Machine learning, signal discovery, or automated parameter search

If a task does not move §3 closer to an answer, it is out of scope.

---

## 5. Success criteria

**The MVP succeeds if it produces a trustworthy answer — including an
unfavourable one.**

A result of "this strategy does not survive realistic fills" is a **successful
MVP**. It saves real capital and answers the question asked. Do not treat a
negative result as a failed build or a prompt to widen the search.

| Criterion | Requirement |
|---|---|
| Correctness | §8 scenarios reproduce the source author's hand-worked days |
| Honesty | Headline judged at `fill=realistic` × `intrabar=conservative` only |
| Reproducibility | Any run regenerable from `run_id` + `git_sha` + `config_hash` |
| Discipline | Holdout window touched exactly once, provable from the ledger |
| Restraint | ≤48 tuning runs total (`AGENTS.md` §13.12) |

### Anti-criteria — signs the project has gone wrong

- Headline figures quoted at `optimistic` fills
- The holdout window run more than once
- New parameters added because the pre-registered ones didn't find a winner
- Stage 2 re-run with different defaults until it clears
- A "best combination" adopted on an improvement smaller than the 1σ noise margin

---

## 6. Non-negotiables

Correctness and safety requirements. **Never** toggled to improve a result
(`AGENTS.md` §13.10). There are **five**:

1. **Combo-fill integrity.** A partial 4-leg fill leaves you naked short an SPX
   option. At ~20 flies/day × 8 leg-sides this *will* occur. Combo-only orders,
   immediate detection, forced completion or flatten.
2. **AM/PM settlement guard.** Third-Friday standard SPX is AM-settled and **SET
   is that symbol, not SPXW** (vendor-confirmed). Trade SPXW only.
3. **Evaluation order.** Drop check *before* add check, every cycle.
4. **Data quality gates.** A replay that has not passed `AGENTS.md` §6.8 produces
   numbers, not results.
5. **Train/holdout guard.** The loader refuses the holdout unless `stage=8`. The
   one guardrail that cannot be restored once broken.

Plus two architectural rules: the rules layer stays **pure** (state snapshot in,
decision out — no I/O, no clock, no broker calls), and the engine runs **one code
path** whether backtesting or live.

---

## 7. Architecture

```
  ThetaData Options ──┐
  (SPXW 1-min NBBO)   │      ┌──────────────┐
                      ├────► │ data/        │ ──┐  ChainSnapshot
  ThetaData Index ────┘      │ loader,      │   │  + spot / VIX
  (SPX spot, VIX,            │ index,       │   │
   eod.close)                │ quality, iv  │   │
                             └──────────────┘   │
                                                ▼
  ┌───────────────┐          ┌─────────────┐          ┌──────────────┐
  │ adapters/     │ ◄──────► │ core/engine │ ◄──────► │ core/rules   │
  │ backtest.py   │          │ (cycle loop)│          │ core/sizing  │
  │ tradier.py    │          │             │          │ (PURE)       │
  └───────────────┘          └─────────────┘          └──────────────┘
                                    │
                                    ▼
                            ┌──────────────┐
                            │ backtest/    │
                            │ runner,report│──► ledger.csv
                            └──────────────┘
```

**Design rule:** the engine orchestrates; the rules decide; the adapters supply
prices and accept orders. A pure decision layer is what allows the §8 scenarios
to run with no data subscription and no network.

Full module layout in `AGENTS.md` §12.

---

## 8. Data

**Two subscriptions required** — Index is a separate product and an Options plan
does not include SPX or VIX.

| Product | Tier | Provides | First access |
|---|---|---|---|
| Options | Value (~$40/mo) | SPXW chain: EOD, OHLC, **Quote**, Open Interest | 2020-01-01 |
| **Index** | Value (price TBC) | **SPX 1-min spot, VIX 1-min, SPX eod.close** | 2023-01-01 |

| Item | Value |
|---|---|
| Instrument | SPXW 0DTE, PM cash-settled |
| Granularity | 1-minute NBBO snapshots (not bars) |
| Test window | 2024-09 → 2026-09 |
| IV lookback | back to 2023-09 |
| Strike coverage | `strike_range=30` ≈ ±150 points |
| Volume | ~24M rows / ~1–2 GB partitioned Parquet |
| API | **v3** |

**Settlement:** SPXW settles to the SPX index close from `index/history/eod.close`.
The index prints until ~16:04–16:05 ET, so the **16:00:00 quote is not
settlement** — using it biases every held fly in the same direction.

**Why ThetaData over Massive:** Massive's entry tier supplies bars derived from
qualifying *trades* and produces no bar when no eligible trade occurs. Wings sit
20–50 points OTM and routinely go minutes without trading. NBBO quotes exist
continuously. Full rationale in `AGENTS.md` §6.6.

**Bulk resolved:** the full chain for one underlying in one request
(`expiration=*`) is available at Value. Flat Files are whole-market dumps limited
to the 7 most recent days — irrelevant here. **Do not upgrade to Professional.**

### Open blocking question (`AGENTS.md` §6.7)

> Does `index/history/price` at `interval=1m` return an **OHLC bar** or a
> **point-in-time snapshot** per minute?

Every trigger in this strategy reads SPX, not option prices. If snapshot-only,
touches are detectable **only at minute boundaries** and any touch that reverts
inside a minute is invisible — a systematic bias that under-counts both drops and
adds. This determines how M5 is defined. **Resolve before D4.**

---

## 9. Known hazards

| Hazard | Consequence |
|---|---|
| **§6.7 unresolved** | Determines whether touch detection is intrabar or minute-boundary. Blocks M5's definition. |
| **Snapshot latency** | Value quotes are 1-min snapshots; a trigger between them prices at the next one. Systematic drag, must be modelled. |
| **Fill assumptions dominate** | ~20 flies/day × 8 leg-sides. Edge surviving only at mid does not exist. |
| **Whipsaw is the core cost** | The drop rule is structurally buy-high/sell-low. Every oscillation pays a toll. Measured directly, never engineered away. |
| **Censored sample** | Max VIX 30 conditions results on calm days — and VIX < 30 does not preclude a 2% intraday move. |
| **Negative skew + win-streak sizing** | Adding a contract after a winning week puts maximum size on immediately before the tail event. This is what forced the source operator's own size cut. |
| **Sequence risk** | Outcomes depend heavily on start date. Rolling 3-month windows, never a single CAGR. |
| **Overfitting surface** | Full grid = 186,624 runs against ~500 days. Hence the staged protocol. |

Detail in `AGENTS.md` §7.

---

## 10. Method: staged sequential testing

Vary **one family of parameters at a time**, fix the winner, then test the next
family against that base.

- Full grid: **186,624 runs** — the best of which is almost certainly the
  luckiest, not the best.
- Staged: **48 runs** — same axes, far fewer degrees of freedom.

Accepted trade-off: sequential testing can miss genuine interaction effects. A
grid capable of finding real ones would also surface thousands of false ones,
with no way to distinguish them at ~500 days. A missed interaction costs upside;
a fitted one costs capital.

**Data split — fixed before the first run:**

| Window | Period | Use |
|---|---|---|
| Train | 2024-09 → 2025-12 | All tuning |
| Holdout | 2026-01 → 2026-09 | Touched **once**, Stage 8 |

Stage 1 is a **gate**. Stage 8 is **one shot**. If train and holdout rankings
invert, ship the baseline — the train result was noise.

---

## 11. Milestones — data-first

| # | Milestone | Gate |
|---|---|---|
| D0 | Both subscriptions; Terminal v3; **resolve §6.7** | Terminal reports access for both products |
| D1 | Probe one day: chain + SPX index sample | ~122 contracts × 390 min; SPX shape known |
| D2 | Downloader, 20 pilot days | Pilot on disk; full-pull time extrapolated |
| D3 | Quality gates + holdout guard | Six gates pass; guard provably raises |
| D4 | **Full pull**, unattended | Manifest complete; gates pass across range |
| M1 | Rules engine + unit tests | §8 unit tests green — **build during D4** |
| M3 | Engine loop + backtest adapter | One real day replays; ordering provable |
| M4 | Scenario validation | Sequences A and B reproduce |
| M5 | **Path-sensitivity gate** | **Spread < 50% of mean P&L, or STOP** |
| M6 | Staged runs 2–7 | ≤42 runs, ledger complete |
| M7 | Holdout confirmation | One shot, 6 runs |
| M8 | **Written verdict** | **Human sign-off — hard stop** |
| M9 | Tradier paper adapter | ≥20 sessions |
| M10 | Live, 1 contract | — |

**M1 runs concurrently with D4** — the rules layer is pure and needs no data.
**M8 is a hard stop.** Do not begin Phase 2 without explicit sign-off.

**If M5 fails:** upgrade to ThetaData Standard (~$80/mo) for tick-level data on
the same API — only the loader changes — or stop. It is not a vendor migration.

---

## 12. Glossary

| Term | Meaning |
|---|---|
| **0DTE** | Zero days to expiration — opens and expires the same session |
| **SPXW** | Weekly SPX options, **PM** cash-settled. What this strategy trades |
| **SET** | AM settlement symbol for standard third-Friday SPX. Must never enter the book |
| **Iron fly** | Short ATM straddle + long protective wings; 4 legs, one expiry |
| **Center** | The shared short strike `K` of a fly. All triggers reference this |
| **Wings** | Long outer legs. Exist **only** for buying-power efficiency — no rule may reference a wing strike |
| **Credit** | Net premium received per contract, in points, positive |
| **Anchor fly** | The open fly whose center is closest to current SPX. Drives tier selection and add distance |
| **Add** | Opening a new fly after SPX travels a tier-defined distance from the anchor's center |
| **Drop** | Closing the farther fly when SPX touches the nearer fly's center. Unconditional on P&L |
| **Tight mode** | Post-14:00 ET regime: 2.5-point trigger, 5-point spacing, fly cap lifted |
| **Breakeven** | `center ± credit` at expiration |
| **`be_pct`** | Fraction of breakeven distance at which a fly is cut (default 0.85) |
| **Snapshot latency** | Delay between a trigger firing and the next 1-min quote that can price it |
| **Whipsaw cost** | Realized P&L on add-then-drop round trips closing within 30 min — the strategy's recurring toll |
| **PDT** | Pattern day trader. Below $25k equity, intraday round-trips are blocked — and the drop rule round-trips constantly |
| **Profit factor** | Gross profit ÷ gross loss. The primary metric |

---

## 13. Where the detail lives

| Question | File |
|---|---|
| What are the exact rules, thresholds, tests? | `AGENTS.md` §3–§9 |
| What variants do we test, in what order? | `AGENTS.md` §13 |
| What do I tell opencode to build, step by step? | `PROMPTS.md` |
| How do I run it and read the results? | `USER_GUIDE.md` |

**`AGENTS.md` is authoritative on all rules and parameters.** Where this document
summarises, it simplifies; on any conflict, `AGENTS.md` wins.
