# CLAUDE.md

Guidance for Claude Code working in this repository. This file describes the
**current state** of the script. Release-to-release history, reverted
experiments, and commit hashes live in **[HISTORY.md](HISTORY.md)** (not loaded
by default).

## Project Overview

A **Pine Script v6 TradingView indicator** — a single-file intraday momentum
trading system, forked from **SuperLazyTrade** with the SuperTrend anchor
removed. The sole source file is `SLT.pine`. No build system, package manager, or
test runner: development means editing the `.pine` file and pasting it into
TradingView's Pine Editor to compile.

Two signal anchors: **SMA** (default) and **EMA Cross**. Carried over from the
parent: the 5-component scoring engine, 5 risk gates, both P&L tracking systems,
win-rate tracking, and the dashboard.

**Light build — minimal input surface.** The panel exposes only:

| Group | Inputs |
|---|---|
| Signal Anchor | EMA Cross / SMA (drives auto-tune) |
| Score Stars | `sigQuality` — label selectivity on top of the auto min score |
| P&L Exit | `enablePnL` + `pnlTarget` |
| Trading Hours | `tradingHours` (`input.session`, default `"0930-1600:23456"`) |
| Dashboard & Visuals | `showDash`, `extDash` |

Everything else is **auto-tuned** from the asset profile + anchor (see
[Auto-Tune](#auto-tune)) or **pinned** in the `FIXED BEHAVIOUR` block near the
top of the file: `showLabels=true`, `sigTime="Score"`, `gateOn=true`,
`rvolMode="Time-of-Day"`, `rvolLookbackDays=10`, `enableSuccessRate=true`,
`maxSignalsToTrack=50`, `showSignalPnL=true`, `showCircles=true`,
`IB_MINUTES=30`, `IB_EARNINGS_MULT=2`, `PNL_FRICTION_PCT=0.05`. There is no
`showBg` (no `bgcolor()` in this build).

**Retired modules:** VWAP Fade is deleted entirely. ORB is inert
(`orbEnabled=false` constant + 7 tuning constants) — its module/router/dashboard/
plot code still compiles but never activates. The **Index Data Proxy** ETF is
auto-picked by symbol family — no input; to hand-pick, edit the `proxy_ticker`
chain. To change any tuned value, edit the profile `if/else` chain, not the panel.

## Development Workflow

1. Edit `SLT.pine`
2. Paste full contents into TradingView → Pine Editor → Save → Add to Chart
3. Compilation errors appear immediately in the Pine Editor console
4. Test on a **2-minute chart** with a liquid instrument (NVDA, TSLA, SPY, QQQ,
   SOXX, Silver Futures)

## Architecture

Sequential sections — order matters in Pine Script:

1. **Constants** — gate penalties, `DASHBOARD_MAX_ROWS = 36`
2. **Inputs** — the 5 groups above + `FIXED BEHAVIOUR` block + ORB inert constants
3. **Asset profile assignment** — `switch` on `syminfo.type` → `asset_category`
   (STOCK/FUND/FUTURES/CRYPTO/MARKET); an `if/else if` on that sets
   `profile_name`, all scoring thresholds, `sma_len`, and the auto-tune fields
   `ema_slow_auto` / `sma_buf_auto` / `min_score_auto`. Then **TIMEFRAME LENGTH
   SCALING** (`tf_scale`, `len9/len14/len20/len50/len10`) and **AUTO-TUNE
   RESOLUTION** (maps profile fields + anchor to `emaSlowPeriod`, `smaBufferATR`,
   `minScoreBuy`/`minScoreSell`, `maxBarsFromFlip`).
3b. **Index data proxy resolution** — `chart_has_volume` latch, auto-picked
   `proxy_ticker`, one `request.security`, and `use_proxy` / `vol_eff` /
   `proxy_vwap_scaled`. Engages only for `syminfo.type == "index"` with no
   volume and a recognized family.
4. **Core calculations** — EMAs (9, `emaSlow` 20/30, 20 fixed for scoring, 50),
   SMA anchor, VWAP (reassigned once before Component 2 — proxy-scaled or session
   TWAP for a volume-less index) plus its standard deviation (`vwap_stdev`,
   feeds Component 2 — see [VWAP Standard Deviation
   Bands](#key-design-decisions)), cumulative volume delta (`cvd`, feeds Gate 5 —
   see [CVD Divergence gate](#key-design-decisions)), ATR/RSI(14), ADX(14,14),
   relative volume (Time-of-Day, fed by `vol_eff`; Rolling 20-bar SMA is the
   thin-history fallback)
5. **Market regime detection** — squeeze (BB(20,2) inside KC(20,1.5×ATR)),
   squeeze release, stretch factor, velocity override (ADX>35 rising 3 bars),
   ATR fuel gauge (session range vs daily ATR-14)
6. **Dual-anchor trend classification** — `is_bull`/`is_bear`/`trendUp`/
   `trendDown` unify both anchors; `anchor_line_price`/`anchor_short` unify
   display. `trend_start_price` updated on each flip.
7. **Scoring engine** — 5 components → `raw_score` (capped 100; 105 max pre-cap)
8. **Risk gates** — 5 gates → penalties subtract from `final_score` when
   `gateOn` (pinned `true`); always shown as warnings. Gate 5 (CVD Divergence)
   is the one directional gate — see [Risk Gates](#risk-gates)
9. **Signal generation** — modules → router → dispatch:
   - **Trend Pullback** (`tp_*`) — the live module: anchor + 5-component
     confluence + quality filter. `sigTime` pinned to `"Score"` (fires off
     `is_bull`/`is_bear` + `entry_is_fresh` + `barstate.isconfirmed`). ANDs in
     `in_session`, `anchor_ready`, and `ib_gate_long`/`ib_gate_short` (see
     [Initial Balance rejection filter](#key-design-decisions)). `tp_quality =
     final_score`. The `"Trend"` mode branch is retained but unreachable.
   - **ORB** (`orb_*`) — inert.
   - **Router** — `active_strategy`/`strat_long`/`strat_short`/`strat_quality`,
     last-write-wins Trend Pullback → ORB; always resolves to Trend Pullback.
   - **Dispatch** — `signal_buy = strat_long and not b_fired` (mirror for sell).
     `b_fired`/`s_fired` enforce strict BUY/SELL alternation across any number of
     flips; clear only on the opposite signal or at session open — not on a
     trend-direction change alone.
   Signals are non-repainting; `barstate.isconfirmed` guards every flip/breakout.
   `in_session`/`session_just_opened` are computed in their own **TRADING HOURS
   SESSION WINDOW** section *before* P&L tracking.
10. **P&L tracking** — two independent systems: (a) live dashboard P&L
    (`pnl_entry_price`/`pnl_direction`, reset on each new signal, flattened at
    session open); (b) signal-to-signal `signal_pnl` history on labels. Exits:
    `target_hit`/`stop_hit` when `current_pnl` crosses `±pnlTarget` (`in_session`-
    gated), or a **day-end forced resolution** by the sign of P&L if still open
    when the session ends. `enablePnL` gates only the visible `signal_profit`/
    `signal_loss` — never resolution.
11. **Success rate tracking** — every closed BUY/SELL trade lands in one of three
    per-direction `var array<int>` (W = `target_reached`, L = `stop_reached`,
    U = same-day reversal with neither). Front-trimmed every bar to the trailing
    `WINRATE_LOOKBACK_SESSIONS` (20) trading sessions. The numbers compose:
    win rate = `W/(W+L)`, resolved = `W+L`, fired = `W+L+U`. The **Trade Signal**
    verdict row turns the last-fired direction's stats into TRADE/CAUTION/SKIP/
    WAIT. Per-module store (`mod_results`) still runs but its rows are suppressed
    (`multi_module = orbEnabled` = false).
12. **Visuals** — one branch per anchor: EMA9 dynamic line, or SMA line + grey
    ATR buffer band; flip circles; BUY/SELL labels at `anchor_line_price`. ORB
    high/low plot gated off. VWAP fade plots deleted.
13. **Dashboard** — `table.new` bottom-right, `DASHBOARD_MAX_ROWS = 36`, rendered
    on `barstate.islast`, `table.clear`ed each render. First data row = active
    anchor + price. Signal Anchor row shows resolved auto values (e.g.
    `SMA (90) auto ±0.20A` or `EMA Cross (9/30) auto`). Win Rate rows read
    `<rate>%  ·  <W>W <L>L <U>↺`, coloured by the win rate. Net P&L (BUY/SELL),
    ATR Fuel, Initial Balance, and CVD Slope rows are in Extended Metrics
    only — Net P&L reads `±x.xx%` (V2, the Σ net-P&L figure — see [Σ net
    P&L](#key-design-decisions)), coloured by its own sign, not the rate;
    Initial Balance reads `forming (<n>/30m)` while the range is still
    building, then `<low>-<high>  ·  blocked <n>L <n>S`; CVD Slope reads
    `n/a (...)` / `warming up (n/len14)` / a `▲`/`▼` `format.volume` reading,
    orange when Gate 5 is active.
14. **Alerts** — 4 `alertcondition` calls: `"BUY"`, `"SELL"`, `"PROFIT"`, `"LOSS"`.

## Scoring Components

All 5 sum to `raw_score`, capped at 100. Component maxes total 105 pre-cap and
are assembled differently per profile — check all five when changing weights.

| # | Component | Max | Varies by profile? |
|---|-----------|-----|-------------------|
| 1 | EMA Cascade Alignment | `ema_max` (20–30) | Yes |
| 2 | VWAP Value Anchor | `vwap_max` (15–25) | Yes |
| 3 | Volume Intensity | 25 | No |
| 4 | ADX Trend Strength | 15 | No |
| 5 | Momentum Confluence | 20 | No |

- **C1 — EMA Cascade:** Full (`ema_max`) = `ema9>ema20>ema50` + close on right
  side of ema9; Partial (60%) = close beyond ema20, cascade incomplete; Weak
  (30%) = close beyond ema50 only; Zero = counter-trend. Direction is the
  selected anchor (`is_bull`/`is_bear`) — it's a directional confluence term.
  Always scores against `ema20`, never `emaSlow`.
- **C2 — VWAP Value:** by distance from VWAP, correct side — BOUNCE/REJECTION
  (full, within `0.33 × vwap_h`), HEALTHY (72%, within `vwap_h`), EXTENDED
  (40%, within `vwap_e`), REVERSION RISK (0, beyond `vwap_e`). Slightly wrong
  side = GRACE ZONE (20%): bull within `0.2 × vwap_h`, bear within `0.1 ×`
  (intentionally stricter — short-side entries near VWAP carry more reversion
  risk); further wrong = WRONG SIDE (0). `vwap_h`/`vwap_e` are **standard
  deviations** (`VWAP_STDEV_HEALTHY`/`VWAP_STDEV_EXTENDED`, 1.0/2.0, shared
  across all profiles) when a real session-cumulative volume-weighted stdev is
  available (`vwap_use_stdev`) — the common case for any volume-bearing
  symbol; **fixed percentages** (`vwap_h_limit`/`vwap_e_limit`, still
  profile-specific) otherwise. See [VWAP Standard Deviation
  Bands](#key-design-decisions). For a volume-less cash index `vwap` is the
  proxy's VWAP rescaled to index units, or a session TWAP if no proxy
  resolves — both stay on the fixed-% path (no proxy-borrowed stdev yet).
  Volume-less non-index (forex) → `na`, C2 scores 0.
- **C3 — Volume Intensity:** 25/20/15/10/5/0 at RVOL ≥2.5/2.0/1.5/1.2/1.0/below.
  RVOL from `vol_eff` (proxy or chart volume); `1.0` fallback (fixed 5 pts) with
  no working proxy.
- **C4 — ADX Strength:** nested tiers vs profile `adx_min`, rising (2-bar)
  qualifies: 15 at `≥1.6×` rising / 12 not rising; 12 at `[1.3×, 1.6×)` rising /
  9 not; 9 at `[1.0×, 1.3×)` (no rising check); 6/3/0 at `≥0.8×` / `≥0.6×` / below.
- **C5 — Momentum Confluence (max 20):** 5A RSI (12) regime-aware — bull: RSI >60
  rising (12) / 45–60 (8) / stalling >60 not rising (3) / else 0; bear mirrors
  about 50: RSI <40 not rising (12) / 40–55 (8) / stalling <40 rising (3) / else 0.
  5B Squeeze Release (8) — release + RVOL >1.5 → 8, release alone → 3. MACD
  sub-component removed (near-duplicate of C1); its points went to RSI (7→12) and
  Squeeze (5→8).

**Setup quality rating** (on `final_score`, `threshold = min_score_auto`,
45–55 by profile):

- ⭐⭐⭐ EXCELLENT: `≥ threshold + 30`
- ⭐⭐ STRONG: `≥ threshold + 15`
- ⭐ GOOD: `≥ threshold`
- ⚠️ WEAK: below threshold

## Risk Gates

`gateOn` is pinned `true`: penalties subtract from `final_score`, all with a
`math.max(..., 0)` floor before the final `[0, 100]` clamp.

| Gate | Trigger | Penalty |
|------|---------|---------|
| 1 — Squeeze | BB(20,2) inside KC(20,1.5×ATR) | −30 |
| 2 — Stretch | `max(ema_stretch, trend_stretch) > ext_scale`, no velocity override | −10 / −20 |
| 3 — ATR Fuel | session range > `fuel_max`% of daily ATR-14, no velocity override | −25 |
| 4 — Liquidity | RVOL < `rvol_gate` AND ADX < 90% of `adx_min` | −25 |
| 5 — CVD Divergence | trend stance contradicted by the CVD slope over `len14` bars | −20 |

`ema_stretch = |close - ema20| / ATR(14)`, `trend_stretch = |close -
trend_start_price| / ATR(14)`; extreme tier at `ext_scale × 1.5`. **Velocity
override** (`is_high_velocity` = ADX > 35 rising 3 bars) waives the Gate 2 *and*
Gate 3 penalties (still shown as advisory). Component 5B (Squeeze Release) is
always active regardless of gate mode. **Gate 5 is directional** (unlike 1-4):
`gate5_divergence = (is_bull and cvd_slope < 0) or (is_bear and cvd_slope > 0)`
— see [CVD Divergence gate](#key-design-decisions) for the full mechanics.

## Profile Parameters Table

Assigned once per bar from `syminfo.type` via a `switch` — `"stock"`→STOCK,
`"fund"`→FUND, `"futures"`→FUTURES, `"crypto"`→CRYPTO, **everything else** (index,
forex, unknown) → MARKET (the true catch-all). MARKET/forex symbols are usually
volume-less; for `type == "index"` the [Index Data Proxy](#index-data-proxy)
restores VWAP + RVOL, forex stays volume-less.

**Scoring / gate thresholds:**

`vwap_h_limit`/`vwap_e_limit` below are the **fixed-% fallback only** — active
when `vwap_use_stdev` is false (a volume-less symbol: proxy-fed index,
TWAP-fallback index, or forex). Any volume-bearing symbol (the common case for
every profile) uses the shared `VWAP_STDEV_HEALTHY`/`VWAP_STDEV_EXTENDED`
(1.0σ/2.0σ) instead — see [VWAP Standard Deviation
Bands](#key-design-decisions). Notably, each profile's `vwap_e_limit /
vwap_h_limit` ratio is already ≈2.0 — these percentages were themselves an
empirical approximation of "roughly 1σ / roughly 2σ" for each asset class
before a real stdev was available to measure directly.

| Profile | `vwap_h_limit` (fallback) | `vwap_e_limit` (fallback) | `adx_min` | `rvol_gate` | `ext_scale` | `fuel_max` | `ema_max` | `vwap_max` | `sma_len` |
|---------|------|------|------|------|------|------|------|------|------|
| MARKET INDEX 🏦 | 0.7% | 1.5% | 25 | 1.0× | 2.0 | 80% | 22 | 23 | 90 |
| ETF / FUND 📊 | 1.0% | 2.0% | 18 | 1.0× | 2.2 | 75% | 20 | 25 | 90 |
| STOCK 🚀 | 1.8% | 3.5% | 20 | 1.2× | 3.0 | 85% | 20 | 25 | 70 |
| FUTURES ⚡ | 1.5% | 3.0% | 15 | 0.6× | 2.5 | 90% | 30 | 15 | 70 |
| CRYPTO 🪙 | 3.0% | 6.0% | 28 | 0.6× | 4.0 | 95% | 25 | 20 | 120 |

## Auto-Tune

Set in the same profile `if/else` chain, resolved in the `AUTO-TUNE RESOLUTION`
block right after it.

| Profile | `ema_slow_auto` | `sma_buf_auto` | `min_score_auto` |
|---------|------|------|------|
| MARKET INDEX 🏦 | 30 | 0.20 | 55 |
| ETF / FUND 📊 | 30 | 0.20 | 50 |
| STOCK 🚀 | 20 | 0.25 | 50 |
| FUTURES ⚡ | 30 | 0.30 | 45 |
| CRYPTO 🪙 | 20 | 0.35 | 55 |

| Effective global | Source |
|---|---|
| `emaSlowPeriod` | `ema_slow_auto`, timeframe-scaled. C1 still uses `ema20`. |
| `smaBufferATR` | `sma_buf_auto` |
| `minScoreBuy` / `minScoreSell` | `min_score_auto`, single value both sides |
| `maxBarsFromFlip` | `anchorMode == "SMA" ? 0 : len10` — **the one parameter that changes with the anchor** |

`tradingHours` is **not** profile-derived — it's the user input, applied
identically to every profile. To hand-tune anything else, edit the profile
chain (or the resolution-block ternary for `maxBarsFromFlip`); there is no
in-panel override.

## Timeframe Length Scaling

Every bar-count lookback was tuned for a **2-minute chart**. `tf_scale =
120 / timeframe.in_seconds()` rescales each so it covers the same wall-clock
window on any interval; on 2-min `tf_scale = 1.0` exactly (byte-for-byte
unchanged). Non-intrabar charts (Range/Tick/Renko) report `0` → falls back to
`1.0`.

**Scaled:** `ema9Period` (=`len9`), `emaSlowPeriod`, `ema50`, `ema20`,
`smaLenActive`, the ATR/RSI/ADX 14-periods (`len14`), the squeeze BB/ATR
20-periods (`len20`), the RVOL fallback 20-period, `maxBarsFromFlip` (`len10`).
**Not scaled (deliberate):** profile threshold/percentage values, the RVOL point
tiers, `rvolLookbackDays` / `maxSignalsToTrack` / `WINRATE_LOOKBACK_SESSIONS`
(session/signal counts), the 2–3 bar `ta.rising()` checks, the daily ATR.

**No shared helper:** `ta.*` length args need `simple int`; a user function
widens its return to `series` (`CE10123`). Compute each scaled length inline as
a plain expression assigned to a `simple int` variable:
`int(math.max(1.0, math.round(base * tf_scale)))`.

## SMA Anchor

Default `anchorMode` (other is EMA Cross). `sma_len` per profile;
`smaLenActive = sma_len` (no MANUAL mode).

**Flip semantics — ATR buffer band with latched state:**

```
sma_band = smaBufferATR × ATR(14)          // smaBufferATR = sma_buf_auto, 0.20–0.35
close > smaAnchor + sma_band  → latch BULL
close < smaAnchor - sma_band  → latch BEAR
inside the band               → hold previous state
```

`sma_state_bull` (`var bool`, init `true`) updates only under
`barstate.isconfirmed` and guarded by `not na(smaAnchor) and not na(sma_band)`.
Flips derive from `sma_prev_bull` (a second `var bool` assigned *before* the
update block each bar — not `sma_state_bull[1]`, which is `na` on bar 0).
`sma_is_bear = not sma_state_bull`, never an independent test — several
downstream sites are `is_bull ? … : …` with no third case, so the latch keeps
the strictly-binary contract EMA Cross also satisfies.

**Scoring interaction (known, not corrected):** in SMA mode `is_bull` holds
through pullbacks that would have flipped an EMA-cross anchor, so C1 keeps
awarding PARTIAL/WEAK points during choppy continuation. **SMA-mode `raw_score`
skews slightly higher on trend continuation** — factor this in when comparing
win rates across anchors.

## Key Design Decisions

**Strategy router (light build):** structured as *modules → router → dispatch*,
but only **Trend Pullback** is live. ORB is inert; VWAP Fade deleted. The router
always yields Trend Pullback. Every downstream consumer (P&L, win-rate, labels,
alerts, dashboard) fires off the dispatched `signal_buy`/`signal_sell` /
`strat_*` surface, so the module machinery is intact for re-adding a module:
expose `<name>_long`/`_short`/`_quality`, keep it self-gated, register it in the
router (opposite-stance modules go *above* Trend Pullback with a regime gate),
wire a `strat_id()` slot + dashboard row.

**Dual-anchor unification:** after anchor selection, all downstream logic uses
`is_bull`/`is_bear`/`trendUp`/`trendDown` — never `ema_*`/`sma_*` directly.
`maxBarsFromFlip` is the deliberate exception (`anchorMode == "SMA" ? 0 : 10`).
`emaSlowPeriod` affects cross detection but NOT C1 scoring (always `ema20`). The
anchor selectors are binary ternary chains ending in EMA Cross, so any non-`"SMA"`
value resolves to EMA Cross; only the *drawing* sites use explicit `== "..."`
equality, so an unhandled mode draws no anchor line.

**`anchor_ready` — warm-up guard:** `ta.sma` is `na` until `smaLenActive` bars
exist, and `sma_state_bull` starts `true`, so the SMA anchor reports `is_bull =
true` for its whole warm-up. `anchor_ready` (`not na(smaAnchor)` in SMA mode,
`true` otherwise) is AND'd into the signal branches to block a warm-up BUY in a
downtrend. EMA Cross passes unconditionally.

**Non-repainting:** *signals* are non-repainting — `barstate.isconfirmed` guards
every flip. The daily ATR (`atr_14`) uses `lookahead_off`, reads the developing
daily bar, so `fuel_used` drifts up through the session — still non-repainting
(a historical bar recomputes to the same value).

**Signal blocking (strict alternation):** `b_fired`/`s_fired` block a
same-direction repeat until the opposite signal actually fires, no matter how
many trend flips happen between. They clear on the opposite signal or at
`session_just_opened` — **not** on a trend-direction change alone. Live P&L
tracking ignores these flags.

**Trading Hours — applies to every asset type:** `tradingHours` drives
`in_session`/`session_just_opened` uniformly for every profile, including 24h
futures/crypto. There is no per-profile bypass and no toggle to disable it —
widen the session (`"0000-2359:1234567"`) if a symbol needs 24h. **One
continuous range only:** a comma-separated multi-range string makes
`session_just_opened`/`_closed` fire at every internal gap, prematurely
force-scoring positions at e.g. a lunch break. Not auto-corrected — keep
`tradingHours` to one range.

**Entry freshness:** `maxBarsFromFlip` = `anchorMode == "SMA" ? 0 : 10`
(SMA latches on the flip bar; EMA crosses lag). Constrains Score-mode entries via
`entry_is_fresh = maxBarsFromFlip <= 0 or bars_since_flip <= maxBarsFromFlip`.

**Initial Balance (IB) rejection filter:** the first `ib_minutes_effective`
(`IB_MINUTES`, 30, doubled to 60 on an earnings-day STOCK session — see
[Earnings-day IB widening](#key-design-decisions) below) minutes of each
session latch a reference range (`ib_high`/`ib_low`), tracked with the
same extend-then-lock pattern the ORB module uses (`ib_mins < ib_minutes_effective`
while building, `ib_locked` once closed) but keyed off wall-clock `time` rather
than a tf-scaled bar count. `ib_up_broken`/`ib_down_broken` latch the first confirmed
close beyond `ib_high`/`ib_low` each session — only that first poke is guarded:
`ib_fresh_up_break`/`ib_fresh_down_break` require RVOL ≥ `rvol_gate` (the same
liquidity bar Gate 4 already applies) via `ib_gate_long`/`ib_gate_short`, ANDed
into both `tp_long`/`tp_short` branches. An unconfirmed break — price probing
past the opening range without volume behind it — is exactly the shape that
tends to fail and reverse intraday; once a side has broken **with adequate
volume**, later signals on that side are unaffected (the gate is one-shot per
session per direction, not a standing filter). **The latch only fires on an
RVOL-confirmed break** — `ib_up_broken`/`ib_down_broken` are set only when
`ib_fresh_up_break`/`ib_fresh_down_break` is true AND `rel_vol >= rvol_gate`
on that same bar (a `/code-review` pass caught the first version latching on
ANY first break attempt regardless of outcome: a blocked, low-RVOL fakeout
still set the latch, so the very next bar `ib_fresh_up_break` was already
false and `ib_gate_long` resolved unconditionally true — permanently
disabling the gate for the rest of the session after a single blocked poke).
A low-RVOL poke now stays "fresh" — still gated — until a bar that actually
clears the volume bar shows up. Reset on `session_just_opened`
alongside the other per-session latches. No panel input — retune `IB_MINUTES`
in `FIXED BEHAVIOUR`. **Observable, not silent:** `tp_long_pre_ib`/
`tp_short_pre_ib` isolate the condition the module would have fired on absent
the IB gate, so `ib_blocked_long_count`/`ib_blocked_short_count` (session-reset
`var int`s) count only genuine blocks — not "didn't qualify anyway" — and
render as the Initial Balance row in Extended Metrics (see checklist item
12b). Without this the filter has zero dashboard footprint and there's no way
to tell whether it ever engages.

**Earnings-day IB widening:** a PEAD-lite addition rather than a standalone
gate — a hard earnings-day block would throw away legitimate post-earnings
continuation trades, but the classic gap-and-reverse whipsaw is the same
failure shape the IB filter already targets, just running longer than a normal
opening range. `earn_actual = request.earnings(syminfo.tickerid,
earnings.actual, barmerge.gaps_on, barmerge.lookahead_off)` — `gaps_on` is
essential here: it returns non-`na` ONLY on the bar where a new earnings
datapoint is published, `na` otherwise, whereas the default `gaps_off`
forward-fills and would read non-`na` on every bar after the first-ever
historical report — useless as an "is TODAY the report day" flag.
`lookahead_off` keeps it non-repainting. The call itself is gated behind
`if asset_category == "STOCK"` — `asset_category` is `simple` (fixed for a
given chart, like `cvd_tf_ok`), so this is safe and skips the call entirely on
the 4 non-STOCK profiles (a `/code-review` pass caught the first version
calling it unconditionally every bar with the result simply discarded
elsewhere, the same "wasted feed" class of issue the compile-perf pass had
already fixed for the Index Data Proxy). `is_earnings_day` (`var bool`, set
`true` when `not na(earn_actual)`) naturally covers both release timings
without special-casing them: a before-market (BMO) report surfaces on the
`session_just_opened` bar itself (the chart's first bar after publication,
since `tradingHours` carries no pre-market bars), and an after-the-close (AMC)
report surfaces on the *next* session's open when the chart shows no extended
hours — exactly the session each report's reaction actually plays out in.
**Resets on `session_just_closed`, not `session_just_opened`** — also caught
in review: if the chart *does* display extended hours, an AMC report's one
non-`na` bar lands on that off-session post-market bar, before
`session_just_opened` fires for the next real RTH open; resetting on the
next open (the original version) would clear the flag on that exact bar
before the `not na(earn_actual)` check could re-set it (the one non-`na` bar
has already passed by then), silently losing the widening for precisely the
session it's meant to cover. Resetting on the *prior* session's close instead
lets the flag persist correctly across that gap. `ib_minutes_effective =
is_earnings_day ? IB_MINUTES * IB_EARNINGS_MULT : IB_MINUTES`
(`IB_EARNINGS_MULT` = 2, so 30 → 60 minutes) feeds the IB extend-while-building
check in place of the bare constant. Dashboard: the Initial Balance row's
label gains a trailing 📅 on an earnings session, so a wider-than-usual range
reads as intentional. No panel input — retune `IB_EARNINGS_MULT` in
`FIXED BEHAVIOUR`.

**CVD Divergence gate (Gate 5):** approximates aggressive buy/sell pressure
from OHLCV alone — real bid/ask trade classification isn't available in Pine,
so each bar is decomposed into its 1-minute constituent bars via
`request.security_lower_tf(syminfo.tickerid, "1", [close, open, volume])` and
each sub-bar is classified up-volume (`close > open`) or down-volume
(`close < open`); summed into `cvd_bar_delta`, then run-summed into a
session-cumulative `cvd`. `cvd_tf_ok = timeframe.isintraday and not
timeframe.isseconds and timeframe.multiplier > 1` is `simple` (fixed for a
given chart), so gating the `request.security_lower_tf` call itself on it is
safe — the same pattern an `input.bool` toggle would use — and matters here
specifically because "1" (1 minute) is invalid as a `lower_tf` whenever it
isn't strictly below the chart's own resolution: a chart at 1 minute or
coarser is the obvious case, but a **seconds-based chart** (e.g. `"30S"`) also
has `timeframe.isintraday = true` and `timeframe.multiplier = 30`, so without
excluding `timeframe.isseconds` explicitly, `cvd_tf_ok` would wrongly pass on
a chart already finer than 1 minute — a **runtime error**, not just a wasted
call (caught in a `/code-review` pass after the initial build). `cvd` gets its own
self-contained session-open reset (`cvd_prev_in_sess`, keyed off the shared
`th_sess_time`) rather than `session_just_opened`, because `session_just_opened`
isn't declared until much later in the file (TRADING HOURS SESSION WINDOW) —
same reason `session_twap` already does this for the TWAP anchor.
`cvd_slope = cvd - cvd[len14]`, gated by `cvd_slope_ready` (`cvd_eligible and
cvd_bar_count > len14`) so the lookback never reaches back across a session
boundary into an unrelated prior session's cumulative value.
`gate5_divergence = (is_bull and cvd_slope < 0) or (is_bear and cvd_slope >
0)` — **directional**, unlike Gates 1-4, in the same sense Component 1's
cascade check is: a bull stance held while aggressive flow has been net
negative over the lookback (or the reverse for bear) is exactly the
unconfirmed-move shape prone to a same-day reversal. `cvd_eligible` additionally
requires `chart_has_volume` (a volume-less index's lower-tf decomposition
still runs — it's gated on `cvd_tf_ok` only, never on a `series` condition —
but its volume values are meaningless, so the gate must not trust them).
**Observable, not silent** (same principle the IB gate uses): a CVD Slope row
in Extended Metrics distinguishes "n/a (chart ≤1m)" / "n/a (no volume)" /
"warming up (n/`len14`)" from an actual `▲`/`▼` reading, turning orange the
moment `gate5_divergence` fires. No panel input — retune `DIVERGENCE_PENALTY`
(`const int`, top of file) directly.

**VWAP Standard Deviation Bands (Component 2):** distance-from-VWAP scoring
switches from a fixed percentage to real standard deviations whenever one is
available. `vwap_sum_pv2`/`vwap_sum_v` (session-cumulative `Σ(volume ×
close²)` / `Σvolume`, reset on the shared `early_session_just_opened` latch —
same idiom as TWAP/CVD) give a population variance `E[X²] - E[X]²` around the
**existing** `vwap` value; `vwap_stdev = sqrt(variance)` when positive AND
`vwap_stdev_ready` (`vwap_sample_count > len14`, `chart_has_volume` implied —
mirrors `cvd_slope_ready`'s exact warm-up guard), else `na`. **The warm-up
guard was added after a `/code-review` pass** caught its absence: without it,
the first bar or two of every session traded a near-zero-sample variance, so
even a tiny price move off VWAP divided down to a wildly inflated sigma count
(e.g. "12σ"), pinning Component 2 to REVERSION RISK/WRONG SIDE at the open on
every real-volume symbol, every single day. Deliberately NOT computed via
`ta.vwap`'s own banded overload (`ta.vwap(source, anchor, mult)`) — that
overload takes an explicit anchor and computes its *own* VWAP value
re-anchored to it, which could subtly diverge from the `vwap = ta.vwap(close)`
value (implicit exchange-session reset) every profile's scoring is already
calibrated against; the manual accumulator bolts a stdev estimate onto the
existing value instead of introducing a second one. **Reset condition also
OR's in `is_new_session`** (calendar-day rollover) alongside
`early_th_session_opened` (the `tradingHours`-derived trigger) — the same
`/code-review` pass caught that a 24h-widened `tradingHours` (this script's
own documented technique for round-the-clock CRYPTO/FUTURES, checklist item
17) makes `in_session` permanently true, so the `tradingHours`-only trigger
would fire ONCE at bar 0 and never again, turning `vwap_sum_pv2`/`vwap_sum_v`
into a running total over the ENTIRE chart history while `vwap` itself kept
resetting daily — a severe E[X²]/E[X] mismatch specifically for the two
real-volume profiles (CRYPTO/FUTURES) this feature is meant to help. Harmless
on a normal (non-widened) RTH chart, where the two triggers coincide on the
same bar. `vwap_use_stdev = vwap_stdev_ready and not na(vwap_stdev)` gates
which unit Component 2 uses: `vwap_h`/`vwap_e` resolve to
`VWAP_STDEV_HEALTHY`/`VWAP_STDEV_EXTENDED` (1.0σ/2.0σ, `const float`, shared
across ALL profiles — not profile-adaptive) when true, or the legacy
`vwap_h_limit`/`vwap_e_limit` (%, still profile-specific) when false. The tier
logic itself (BOUNCE/HEALTHY/EXTENDED/REVERSION RISK, GRACE ZONE fractions)
is completely unchanged — only the unit and the specific threshold values
change; `vwap_dist` replaces the old `vwap_dist_pct` as the unit-agnostic
distance measure feeding those same comparisons. **Scoped to real-volume
symbols only** (`chart_has_volume`): a volume-less index (proxy-fed or
TWAP-fallback) and forex all stay on the fixed-% path — extending
stdev-borrowing to the proxy ETF (mirroring how VWAP/RVOL are already
borrowed there) was scoped out of this pass as materially higher-risk
(would need a `request.security`-evaluated helper function carrying its own
`var`-state accumulator in the proxy's context) and left for a future,
separate change if pursued. **Why not profile-adaptive:** each profile's
pre-existing `vwap_e_limit / vwap_h_limit` ratio was already ≈2.0 across all
5 — those percentages were themselves an empirical stand-in for "roughly 1σ /
roughly 2σ" per asset class; once a symbol's actual volatility is measured
directly, a single statistical convention applies universally, so
`VWAP_STDEV_HEALTHY`/`EXTENDED` are `const` rather than assigned per profile
like every other threshold in the Profile Parameters Table. Dashboard: the
VWAP Value row's numeric suffix reads `(x.xxσ)` or `(x.xx%)` depending on
which path is live, so the mode in effect is always visible, not silent.

**Two P&L systems (independent):** *Live dashboard P&L* from `pnl_entry_price`,
reset on every new signal (strict alternation means entry always = the signal
that opened the position). *Signal P&L History* on labels, `%` from
`last_signal_price` using the previous signal's direction — separate state.

**P&L friction cost:** `current_pnl` and `day_end_pnl` both compute via one
shared `net_pnl_pct(is_buy, entry, px)` function — `raw_pct - PNL_FRICTION_PCT`
where `raw_pct` is the directional `(px - entry)/entry × 100` (or the mirror
for a sell) — rather than each inlining its own copy of the formula (a
`/code-review` pass flagged the original two-inline-copies version as the same
duplication-risk shape as the `live_pnl_pct` bug below, before either had
shipped). `PNL_FRICTION_PCT` (0.05, a single conservative generic estimate —
not profile-scaled) covers the round-trip bid-ask spread + slippage a live
fill would actually pay. Applied at the source so every consumer — `target_hit`/`stop_hit`,
`day_end_win_hit`/`day_end_loss_hit`, the win-rate W/L/U buckets they feed, and
every P&L *display* (dashboard header `PnL <x>%`, day-end PROFIT/LOSS label
text) — reflects a realistic net outcome without being touched individually.
Net effect: a raw move now needs `pnlTarget + PNL_FRICTION_PCT` to trigger
PROFIT (stop triggers `PNL_FRICTION_PCT` sooner), and a position that closes
exactly flat at day's end correctly resolves as a small loss rather than a
trivial win. Deliberately scoped to this system only — the separate *Signal
P&L History* (`signal_pnl`, shown on labels) is purely informational, not used
for target/stop or win-rate resolution, and is left as a raw price move.
Fixed a pre-existing duplicate along the way: the dashboard header's
`live_pnl_pct` re-derived the same raw formula independently instead of
reusing `current_pnl`, which would have bypassed the friction subtraction
silently — now reuses `current_pnl` directly (the day-end-frozen branch was
already correct, since `day_end_frozen_pnl := day_end_pnl`). No panel input —
retune `PNL_FRICTION_PCT` in `FIXED BEHAVIOUR`.

**P&L exits fire on target/stop, or are forced at day's end:**
`normal_target_reached`/`normal_stop_reached` fire when `current_pnl` crosses
`±pnlTarget`, once per entry, `in_session`-gated. `target_reached =
normal_target_reached or day_end_win_hit` (mirror for stop), so all downstream
consumers are covered by two booleans. `day_end_trigger = session_just_closed or
day_end_fallback or day_end_onbar`, three paths: `session_just_closed = not
in_session and prev_in_session` (first off-hours bar, one bar late) is the normal
path; `day_end_fallback` keys off the calendar-day rollover (`is_new_session`)
for a 24h-widened `tradingHours` where `in_session` never goes false;
`day_end_onbar` fires **on the final in-session bar itself, at the live edge
only** (`barstate.islast and barstate.isconfirmed and in_session and
bar_close_mins >= session_end_mins`, where `session_end_mins` is parsed once from
`tradingHours`) — without it, an RTH-only feed viewed live at the close has no
later bar and the bubble waits hours for the next session. Historical days are
untouched (`barstate.islast` is only the dataset's last bar); on reload the same
bar re-resolves via `session_just_closed` at the identical bar/price. At the
trigger, if still unresolved (`pnl_direction != "NONE"`, `not pnl_exit_fired`,
and the entry opened before the resolving bar — `< bar_index` for `day_end_onbar`,
`< bar_index - 1` for the two late paths — else a meaningless 0% win), it
resolves by the sign of `day_end_pnl` (`day_end_onbar` reads `close`, the late
paths `close[1]`). Label/alert anchored to that same bar (`bar_index`/`close`
onbar, `bar_index - 1`/`close[1]` late); dashboard live P&L freezes at
`day_end_frozen_pnl`.
`enablePnL` gates only `signal_profit`/`signal_loss` (labels + alerts), never
resolution or win-rate scoring — keep any future stats logic on
`target_reached`/`stop_reached`.

**Position flattened at session open:** on `session_just_opened`,
`pnl_entry_price`/`pnl_direction`/`pnl_exit_fired`/`pnl_exit_type` reset — this is
why TRADING HOURS SESSION WINDOW sits *above* P&L exit tracking in the file.
Applies to every asset type.

**Win-rate tracking — one session window, three buckets:** every BUY/SELL trade,
**when it closes** (next opposite signal, or `session_just_opened` carryover),
lands in exactly one of `buy_win_s`/`buy_loss_s`/`buy_unr_s` (SELL trio too),
each a `var array<int>` holding the `session_seq` at close time. W = PROFIT fired
while open; L = LOSS fired; U = neither (same-day reversal before `±pnlTarget` —
the only way to stay unresolved since day-end forced resolution).
`buy_won_this_entry`/`buy_lost_this_entry` accumulate during the open trade;
`buy_entry_open` guards against scoring before a trade opens; all flags cleared
at both close paths. **Every bar** the six arrays are front-trimmed (no scan —
`session_seq` is non-decreasing) to the trailing `WINRATE_LOOKBACK_SESSIONS` (20)
trading sessions via `trim_window()`. `buy_w`/`buy_l`/`buy_u` = trimmed
`array.size(...)`, recomputed every bar. `session_seq` counts **trading
sessions** (`is_new_session`), so weekends don't shrink the window. The numbers
compose: rate `= W/(W+L)`, resolved `= W+L`, fired `= W+L+U`.

**Σ net P&L (V2-only, absent in V1):** the win rate hides how much a reversed (U) trade
actually cost, and treats a +1% target and a −1% stop as equal weight. Each
direction also keeps a paired `buy_all_s` (`array<int>` session stamp) /
`buy_all_p` (`array<float>` net %) list, pushed at the same two close sites
as the W/L/U buckets and trimmed in step by `trim_pair()`. The value pushed:
for W/L, `buy_entry_result` — captured the bar `target_reached`/`stop_reached`
fired as `day_end_close ? day_end_pnl : current_pnl` (the exact figure the
PROFIT/LOSS label shows, net of `PNL_FRICTION_PCT`); for U at an
opposite-signal close, `current_pnl` on that bar (the net move to the
reversal, while `pnl_direction` still holds the closing trade); for the
session-open carryover, `nz(result, 0)` (no exposure). `sum_pnl()` sums the
list on the last bar. Displayed as two **Net P&L (BUY)** / **Net P&L (SELL)**
rows in **Extended Metrics** (moved there from the Win Rate rows so the
always-visible Win Rate rows stay purely rate-colored), coloured by the
**sign of Σ** — lime above zero, yellow at exactly zero, red below, grey with
no closed trades yet. Σ is per-trade %, equal size, no compounding — a
setup-comparison figure, not an account return. The Trade Signal verdict is
unchanged (still rate-based).

**Trade Signal verdict:** a dashboard row right after Setup Score (deliberately
*not* grouped with the Win Rate rows), turning the last-fired direction's
(`last_signal_type`) resolved stats into a plain call. Computed by
`trade_verdict(wins, total, fired, target)` right after `last_signal_type` is
updated (so a current-bar signal is reflected immediately). Three gates in order:
1. **Sample size** — `< VERDICT_MIN_SAMPLE` (10) resolved → `WAIT (n/10)`.
2. **Resolution rate** — `(W+L)/(W+L+U) < VERDICT_MIN_RESOLUTION` (0.40) →
   `SKIP (CHOPPY)`.
3. **Wilson-adjusted rate** — `wilson_lower_bound(wins, total)` (95%,
   `WILSON_Z = 1.96`) shrinks toward 50% at small `total`, judged against the
   same 60/50 tier cutoffs as the win-rate row colors: ≥60% `TRADE`, ≥50%
   `CAUTION`, below `SKIP`. Appends per-trade expectancy `target × (2·wlb − 1)`.
All three thresholds are pinned constants. Row reads `—` until a signal has fired.

**Per-module win-rate (Phase 4):** every W/L close also calls `record_module(...)`
keyed by the module that **opened** the entry (U not tracked per-module).
`mod_results` is a 3-slot `var array<ModResults>` (UDT wrapping two `array<bool>`
— Pine arrays can't nest), still FIFO-capped by signal count, **not**
session-windowed — reconcile if a second module goes live. Dashboard rows gated on
`multi_module = orbEnabled` (false), so suppressed; slot 0 still accumulates.

**Time-of-Day Relative Volume:** `rel_vol` (C3, Gate 4, C5B) with `rvolMode`
pinned to `"Time-of-Day"` compares the current bar's volume to the average of the
*same bar-slot* (bars since session open) across the prior `rvolLookbackDays`
(10) completed sessions — removing the U-shaped intraday bias a trailing SMA has
(overstates near the open, understates at lunch). History in `array<TodSession>`
(UDT-wrapped `array<float>` — arrays can't nest). Session boundaries reuse
`is_new_session`. Falls back to `ta.sma(volume, 20)` when a slot has fewer than 3
historical sessions (or if Rolling 20-bar were selected). Dashboard Volume row
shows `TOD` / `20-bar` / `20-bar*` (in-flight TOD→fallback), `proxy·`-prefixed
when the series is a proxy's. Feeds off `vol_eff`.

**Index Data Proxy:** a cash index (SPX, IXIC/NDX, S&P/TSX, ...) usually reports
`volume` as `na`/0, which guts C2 (VWAP → 0), C3 (RVOL pinned to 1.0), C5B, and
Gate 4 — pushing the score ceiling below `min_score_auto`. Fix: when
`syminfo.type == "index"` **and** the chart carries no volume (`chart_has_volume`
— a `var` latch tripping only on a sustained ≥50% of the last 100 bars, not the
first nonzero bar), borrow VWAP + RVOL from a liquid tracking ETF.
- **Proxy selection** (`proxy_ticker`, auto by `syminfo.ticker` substring, **no
  input**): `NDX`/`IXIC`/`NDQ`/`NAS100`/`US100`/`USTEC` → `NASDAQ:QQQ`;
  `TSX`/`TX60`/`SPTSX` → `TSX:XIC`; `SPX`/`SP500`/`US500`/`GSPC` → `AMEX:SPY`.
  Only these set `proxy_family_known` — an unrecognized index (DAX, FTSE, Nikkei)
  falls to the session-TWAP substitute, **not** SPY.
- **One `request.security`** pulls `[ta.vwap, close, volume, ta.sma(volume,
  len20)]` with `lookahead_off`. `use_proxy = proxy_eligible and proxy_ok`.
- **VWAP:** `proxy_vwap_scaled = close * (proxy_vwap / proxy_close)` — proxy's
  fractional deviation at index scale, so C2's math is unchanged. `vwap`
  reassigned once before C2: chart-volume → real `ta.vwap`; volume-less index +
  proxy → `proxy_vwap_scaled`; volume-less index, no proxy → `session_twap`
  (running mean of `hlc3` anchored to the `tradingHours` RTH window); volume-less
  non-index → left `na`.
- **Volume:** `vol_eff` / `vol_ma_eff` feed all RVOL paths. `na` guards on
  `vol_eff` (per-bar proxy gap) and on the Time-of-Day slot history.
- **Dashboard:** Profile row appends `· <proxy_ticker> data`, or orange
  `· ⚠️ volume-less → TWAP`. Volume row mode label gains `proxy·` prefix.

## Pine Script v6 Gotchas

- **`var`** initializes only on bar 0. The profile threshold variables
  deliberately omit `var` so they reassign every bar.
- **`request.security`** — always pass `lookahead = barmerge.lookahead_off` and
  the expression directly (no pre-computed `var`).
- **`request.security_lower_tf`** raises a **runtime error** (not just a wasted
  call) if its `timeframe` argument isn't strictly lower than the chart's — so
  a chart already at 1-minute can't request `"1"`. Guarding the call inside
  `if cvd_tf_ok` is safe specifically because `cvd_tf_ok` is built from `simple`
  values (`timeframe.isintraday`, `timeframe.multiplier` — fixed for a given
  chart), the same class of condition an `input.bool` toggle would use to gate
  a `request.security` call; a `series`-typed condition would need a call in
  every branch instead.
- **`na` propagates:** `math.max(na, 0)` is `na`. Guard with `not na(x)` /
  `nz(x, 0)`.
- **`barstate.isconfirmed`** is true on all historical bars during replay;
  `barstate.islast` only on the most recent. Dashboard uses `islast`, signals use
  `isconfirmed`.
- **`array.get`** throws out-of-bounds — check `array.size() > 0` first.
- **`for i = 0 to N`** infers direction from `0`/`N`, it doesn't skip when
  `0 > N`: an empty array makes `array.size(arr) - 1` equal `-1`, and
  `for i = 0 to -1` still runs once *descending* (`i = 0`) instead of zero
  times, then `array.get(arr, 0)` throws on the actually-empty array (hit by
  the CVD lower-tf loop on bar 0, where `request.security_lower_tf` hasn't
  returned any sub-bars yet). Wrap any `for i = 0 to array.size(arr) - 1`
  in `if array.size(arr) > 0` unless the array's non-emptiness is already
  structurally guaranteed (the `gate_lines`/`words` loops are safe without
  the guard because `str.split` on a non-empty string always returns ≥ 1
  token).
- **`nz()` has no bool overload** (`CE10123`). To default a bool's previous
  value, keep a second `var bool` assigned *before* the update block
  (`sma_prev_bull`). `[1]` on a `var bool` is `na` on bar 0.
- **Arrays can't nest** — `array<array<float>>` won't compile. Wrap the inner
  array in a UDT (`TodSession`, `ModResults`). A function *can* mutate such an
  inner array in place via a parameter; it can't reassign a value-type global
  (`CE10088`) — which is why the P&L entry logic stays inlined at each signal
  branch.
- **A function's return is always `series`-qualified** — breaks `ta.*` `length`
  args (`simple int`). Inline the arithmetic instead (see Timeframe Length
  Scaling).
- **`time_close(timeframe, session)`** does not reliably detect the
  boundary-crossing bar. Use `session_just_closed = not in_session and
  prev_in_session` (fires one bar late; compensate with `close[1]`/`bar_index -
  1`).
- **`str.split`** includes a trailing empty token when the delimiter ends the
  string; the `gate_lines` word-wrap loop's final `if current_line != ""` guard
  drops it.
- **`var table` cells persist** across renders — the dashboard `table.clear`s
  before writing because the gate-detail loop is variable-length (with an
  explicit `if row >= DASHBOARD_MAX_ROWS: break`).

## Compile-Check Checklist

No test runner. After any edit, verify in the Pine Editor:

1. **Zero compilation errors** before proceeding.
2. **Panel shows only:** Signal Anchor, Score Stars, P&L Exit, Trading Hours,
   Dashboard & Visuals. No SMA / signal-timing / RVOL / ORB / Fade / Success
   groups; no proxy field; no flip-circle or background toggles.
3. **NVDA 2-min** → dashboard bottom-right, Signal Anchor row
   `SMA (70) auto ±0.25A`. No Strategy row.
4. **Score ≤ 100** always; 105-max component sum never overflows the clamp.
5. **Alternation:** a BUY blocks the next BUY until a SELL fires, across
   multiple trend flips where SELL's filter never clears.
6. **No double-count** on a same-bar PROFIT + new signal (increments by 1).
7. **Gates enforced:** dashboard says `Gates` (not `Risks`); an active gate drops
   `final_score`.
7b. **Gate 5 (CVD Divergence):** on an intraday chart above 1-minute with real
    volume, the Extended Metrics CVD Slope row reads `▲`/`▼` plus a
    `format.volume`-scaled number once `cvd_bar_count > len14`; a bull stance
    held while it reads `▼` (or bear while `▲`) shows `📉 CVD DIVERGENCE (-20)`
    in Gate Details and drops `final_score` by 20. On a 1-minute chart or a
    volume-less symbol the row reads `n/a (...)` and Gate 5 never fires.
8. **Anchor swap:** SMA → SMA line + grey band, flip circles at latched-state
   changes, `Trend (SMA)`. EMA Cross → EMA9 line, circles at crossover bars,
   `EMA Cross (9/<20|30>) auto`, `maxBarsFromFlip` becomes 10 (SMA = unlimited).
9. **Dashboard row budget:** worst case (Extended Metrics ON, all 5 gates
   active) → no row-overflow error.
10. **RVOL:** Volume row shows `TOD` on a long chart, `20-bar*` on a short one —
    never `na`.
11. **P&L target:** PROFIT/LOSS label text still reads exactly `±pnlTarget %`
    (static, unaffected by friction); the underlying trigger now needs a raw
    price move of `pnlTarget + PNL_FRICTION_PCT` (PROFIT) or
    `-pnlTarget - PNL_FRICTION_PCT`-or-less (LOSS, reached sooner in raw
    terms) since `current_pnl` is net of friction. A position still open at
    the last in-session bar fires `PROFIT/LOSS +<actual%>` with real,
    friction-adjusted P&L; no PROFIT/LOSS label ever appears outside
    `tradingHours`.
11b. **P&L friction cost:** the dashboard header's `PnL <x>%` and every
    PROFIT/LOSS/day-end value are `PNL_FRICTION_PCT` (0.05) lower than the raw
    price move would give — e.g. a position that closes at the literal entry
    price at day's end resolves as a small LOSS, not a breakeven/win. The
    separate Signal P&L History shown on BUY/SELL labels is untouched (still a
    raw price move) — friction applies only to the live-tracking system that
    feeds target/stop and win-rate resolution.
12. **Entry freshness:** EMA Cross → no signal >10 bars after a flip; SMA →
    "Bars From Flip" reads `(unlimited)`.
12b. **Initial Balance rejection:** the first confirmed close beyond the
    first-`IB_MINUTES` (30) range on low RVOL (< `rvol_gate`) fires no
    BUY/SELL label; a break on adequate RVOL fires normally, and once a side
    has broken **with adequate RVOL**, later same-side signals are unaffected
    for the rest of the session. **Specifically verify the fixed latch
    behavior:** if a low-RVOL poke past the range gets blocked, the NEXT
    close still beyond the range must still be gated (checked against RVOL
    again) rather than sailing through unconditionally — it should take an
    actual RVOL-confirmed break to release the gate, not just any first
    attempt. Resets at the next `session_just_opened`. Extended Metrics'
    Initial Balance row confirms the range and increments `blocked <n>L <n>S`
    the moment the filter actually suppresses a signal — check this row, not
    just the absence of a label, to confirm the filter is engaging at all.
12c. **Earnings-day IB widening:** on a STOCK-profile symbol, a session
    following a fresh earnings report widens the range to `IB_MINUTES ×
    IB_EARNINGS_MULT` (60 min) instead of the normal 30; the Initial Balance
    row's label shows a trailing 📅 for that session only. A non-earnings
    session, or a non-STOCK profile, behaves exactly as item 12b — no
    widening, no 📅.
13. **Win rate:** rows read `<rate>%  ·  <W>W <L>L <U>↺` over the trailing 20
    sessions; the three compose (rate `= W/(W+L)`, `W+L+U` = all closed
    trades); counts fall off as close bars age past 20 sessions; window
    survives a weekend gap unshrunk; grey until a trade closes; coloured
    purely by the win rate (≥60% lime, ≥50% yellow, else red). No per-module
    rows.
13b. **Σ (V2 only, Extended Metrics):** `Net P&L (BUY)`/`Net P&L (SELL)` rows
    change only on a close bar, by exactly one trade's net %. A PROFIT +1.0%
    trade adds +1.00 (already net of friction); a U trade reversed at −0.4%
    adds −0.40. Row colour follows the SIGN of Σ, independent of the Win Rate
    row's colour right above it in the main section — e.g. a 70% win rate can
    show a red Net P&L row, a 45% rate a lime one. Reads `—` (grey) until the
    first close; with `enablePnL` off it still accumulates. Only visible with
    Extended Metrics on.
14. **`enablePnL` independence:** turning P&L Exit OFF stops PROFIT/LOSS
    labels + alerts but win rates keep resolving (must NOT collapse to 0%).
15. **Stale rows:** a gate goes active then clears → the Gate Details rows
    disappear, not freeze on stale text.
16. **Session-open flatten:** hold a position into the close → at the next open
    the dashboard P&L reads `—` and no PROFIT/LOSS fires off the gap.
17. **Trading Hours on 24h symbols:** CRYPTO/FUTURES at default `tradingHours` →
    no signals outside 09:30–16:00, position flattens at each session open.
    Widen to `0000-2359:1234567` → signals resume around the clock.
18. **Day-end resolution:** a position short of `±pnlTarget` at the close →
    `PROFIT/LOSS +<actual%>` anchored at the actual last in-session bar, counted
    into the win rate, no further label on later off-hours bars; live P&L freezes
    at the day-end value. On an RTH-only feed watched live at the close the
    bubble appears **on that last bar** the moment it confirms (`day_end_onbar`),
    not only once a later bar prints; scroll back / reload → the label stays on
    the same bar (now via `session_just_closed`).
19. **Day-end fallback:** with `tradingHours` widened to 24h, a position that
    never hits target still force-resolves at the calendar-day rollover; a signal
    opening on that exact bar shows `U`, not a false `W`.
20. **Profile auto-tune:** STOCK → `±0.25A` + `9/20`; MARKET INDEX → `±0.20A` +
    `9/30`; CRYPTO → `±0.35A` + `9/20`. Min-score base 45↔55 by profile.
21. **ORB inert:** no ORB row, no orange lines, no `⚡ORB` label;
    `active_strategy` always `Trend Pullback`.
22. **Timeframe scaling:** 2-min unchanged (`9/20`, `70`); 5-min → periods drop
    proportionally; Range/Tick → no error, lengths hold at 2-min values.
23. **C1 direction follows the anchor** and always scores against `ema20`; its
    status never contradicts the Trend row.
23b. **VWAP Standard Deviation Bands:** on a volume-bearing symbol (NVDA,
    SOXX, SPY, QQQ...), the first `len14` bars of each session read `%` (not
    yet `vwap_stdev_ready`) — no "12σ" spike at the open — then the VWAP Value
    row's numeric suffix switches to `(x.xxσ)`; tiers (BOUNCE/HEALTHY/
    EXTENDED/REVERSION RISK/GRACE ZONE) fire at 1.0σ/2.0σ boundaries (plus the
    existing 0.33/0.2/0.1 fractions) regardless of profile. On a volume-less
    index with a working proxy (e.g. `SPX` → `SPY` data) or the TWAP fallback
    (`⚠️ volume-less → TWAP`), or on forex, the row stays on `%` against the
    profile's `vwap_h_limit`/`vwap_e_limit` for the whole session — unchanged
    from before this feature. On a CRYPTO/FUTURES chart with `tradingHours`
    widened to 24h (`"0000-2359:1234567"`), the σ reading should still look
    sane hours/days into the session (not drifting toward a multi-day/
    multi-year baseline) — the accumulator resets on the calendar-day
    rollover as a fallback trigger specifically for this case.
24. **Trade Signal verdict:** row right after Setup Score; `—` before any signal;
    switches BUY↔SELL verdict on the same bar the new direction fires;
    `WAIT (n/10)` under 10 resolved; `SKIP (CHOPPY)` when
    `(W+L)/(W+L+U) < 0.40`; otherwise tier by the Wilson-adjusted rate (diverges
    from the raw rate at small `total`), with a `(±x.xx%)` expectancy;
    unaffected by `enablePnL`.
25. **Index Data Proxy:** volume-less index → Profile row shows the auto-picked
    proxy (`NASDAQ:IXIC` → `· NASDAQ:QQQ data`), Volume row `proxy·TOD`, C2 scores
    non-zero. Unrecognized index (`TVC:DAX`) → orange `· ⚠️ volume-less → TWAP`,
    C2 falls back to session TWAP, C3 to the 1.0 floor. `SPY`/`QQQ`/`ES1!` → no
    proxy annotation, behaviour unchanged. Forex → no proxy, C2 stays
    `NO VWAP DATA`. Signals stay non-repainting.

## Version

`SLT.pine`, forked from **SuperLazyTrade V3** with the SuperTrend anchor removed
entirely. Anchor selection is binary — SMA (default) or EMA Cross. On-chart
`indicator()` title is `SLT V2` (from `VERSION = "V2"`). **V1 was redefined
(2026-09-13)** to point at an earlier snapshot — the script as it stood before
the Initial Balance rejection filter, CVD Divergence gate (Gate 5), earnings-
day IB widening, P&L friction cost, VWAP standard-deviation bands, and a
compile-performance pass that moved per-bar `str.tostring` calls off the hot
path — frozen as `SLT-V1.pine`. **V2 is everything this file documents**: all
of the above, plus the Σ net-P&L figure (Net P&L rows in Extended Metrics).
Concretely, V1 lacks Gate 5 entirely (only Gates 1-4), has no Initial Balance
row, no P&L friction subtraction, fixed-% VWAP-distance thresholds only (no σ
path), and no Net P&L rows — compare it against V2 on the same chart to see
what each feature actually buys. (A separate single-EMA-anchor rebuild was also tried
under the name "V2" earlier this session and parked after live results — see
HISTORY.md; it was never committed and does not correspond to either file
here.) The changelog is in
**[HISTORY.md](HISTORY.md)** — append new entries at the end, newest last; keep
this file describing only the current state.
