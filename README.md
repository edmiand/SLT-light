# SLT-light

A single-file **Pine Script v6 TradingView indicator** — an intraday momentum
trading system for the 2-minute chart. Forked from **SuperLazyTrade V3** with the
SuperTrend anchor removed and the input surface reduced to a handful of controls;
everything else is auto-tuned from the instrument type.

- Source file: `SLT.pine` (on-chart title `SLT V1`)
- No build system, package manager, or test runner. "Development" = edit the
  `.pine` file, paste into TradingView's Pine Editor, compile, eyeball on a chart.
- Test instruments: NVDA, TSLA, SPY, QQQ, SOXX, Silver/CL futures on a 2-min chart.

---

## 1. What the script does, in one paragraph

Every bar it (a) detects the instrument type and loads a matching parameter
**profile**, (b) computes a trend direction from the selected **anchor** (SMA or
EMA Cross), (c) scores the setup 0–100 with a **5-component engine**, (d) subtracts
**risk-gate** penalties, (e) if the score clears the profile minimum and the
star filter, emits a non-repainting **BUY/SELL** that strictly alternates, (f)
tracks **P&L** and **win rate** for those signals, and (g) draws the anchor line,
flip circles, labels, and a **dashboard**. Four `alertcondition`s expose BUY /
SELL / PROFIT / LOSS.

---

## 2. Inputs (the entire settings panel)

| Group | Input | Default | Meaning |
|---|---|---|---|
| 📍 Signal Anchor | `anchorMode` | **SMA** | `SMA` or `EMA Cross` — the trend engine; also drives auto-tune |
| Signal | `sigQuality` | `⭐ +` | Label selectivity on top of the auto min score: `All Signals` / `⭐ +` / `⭐⭐ +` / `⭐⭐⭐ Only` |
| 💰 P&L Exit | `enablePnL` | `true` | Show PROFIT/LOSS exit markers + alerts (display only) |
| 💰 P&L Exit | `pnlTarget` | `1.0%` | PROFIT/LOSS fires at ±this % from entry |
| 📊 Dashboard & Visuals | `showDash` | `true` | Bottom-right table |
| 📊 Dashboard & Visuals | `extDash` | `false` | Add per-component breakdown + win-rate + diagnostic rows |

**Pinned constants** (`FIXED BEHAVIOUR` block — were inputs in the parent script):
`showLabels=true`, `sigTime="Score"`, `gateOn=true`, `rvolMode="Time-of-Day"`,
`rvolLookbackDays=10`, `enableSuccessRate=true`, `maxSignalsToTrack=50`,
`showSignalPnL=true`, `showCircles=true`, `showBg=true`.

**Retired modules:** `orbEnabled=false` (ORB code still compiles, never fires);
VWAP Fade deleted entirely.

Everything else is derived — see §3–4. **To hand-tune, edit the profile `if/else`
chain in `SLT.pine`, not the panel.**

---

## 3. Asset profile (automatic, from `syminfo.type`)

A `switch` maps the symbol to a category, then an `if/else` chain sets ~15
parameters. `MARKET` is the catch-all (index, forex, anything unrecognised).

| `syminfo.type` | Profile |
|---|---|
| `stock` | STOCK 🚀 |
| `fund` | ETF / FUND 📊 |
| `futures` | FUTURES ⚡ |
| `crypto` | CRYPTO 🪙 |
| everything else | MARKET INDEX 🏦 |

**Scoring / gate thresholds per profile:**

| Profile | `vwap_h_limit` | `vwap_e_limit` | `adx_min` | `rvol_gate` | `ext_scale` | `fuel_max` | `ema_max` | `vwap_max` | `sma_len` |
|---|--|--|--|--|--|--|--|--|--|
| MARKET 🏦 | 0.7% | 1.5% | 25 | 1.0 | 2.0 | 80% | 22 | 23 | 90 |
| FUND 📊 | 1.0% | 2.0% | 18 | 1.0 | 2.2 | 75% | 20 | 25 | 90 |
| STOCK 🚀 | 1.8% | 3.5% | 20 | 1.2 | 3.0 | 85% | 20 | 25 | 70 |
| FUTURES ⚡ | 1.5% | 3.0% | 15 | 0.6 | 2.5 | 90% | 30 | 15 | 70 |
| CRYPTO 🪙 | 3.0% | 6.0% | 28 | 0.6 | 4.0 | 95% | 25 | 20 | 120 |

**Auto-tune fields per profile:**

| Profile | `ema_slow_auto` | `sma_buf_auto` | `min_score_auto` | `rth_auto` |
|---|--|--|--|--|
| MARKET 🏦 | 30 | 0.20 | 55 | true |
| FUND 📊 | 30 | 0.20 | 50 | true |
| STOCK 🚀 | 20 | 0.25 | 50 | true |
| FUTURES ⚡ | 30 | 0.30 | 45 | **false** |
| CRYPTO 🪙 | 20 | 0.35 | 55 | **false** |

---

## 4. Auto-tune resolution (profile + anchor → effective globals)

Runs immediately after the profile chain. These names are what the rest of the
script reads:

| Effective global | Source |
|---|---|
| `emaSlowPeriod` | `ema_slow_auto` (EMA Cross slow leg, 20 or 30) |
| `smaBufferATR` | `sma_buf_auto` (SMA latch band, × ATR-14) |
| `minScoreBuy` = `minScoreSell` | `min_score_auto` (single value both sides) |
| `useSessionFilter` | `rth_auto` (`false` for FUTURES + CRYPTO — a 24h instrument would be blacked out) |
| `maxBarsFromFlip` | `anchorMode == "SMA" ? 0 : 10` — **the one parameter that changes with the anchor** |

---

## 5. Computation pipeline (top-to-bottom; order matters in Pine)

1. **Constants** — gate penalties (`SQUEEZE 30`, `FUEL 25`, `LIQUIDITY 25`,
   `STRETCH 10/20`), `DASHBOARD_MAX_ROWS = 36`.
2. **Types** — `TodSession` (wraps `array<float>`), `ModResults` (wraps two
   `array<bool>`). Pine arrays can't nest, so UDT wrappers are used.
3. **Inputs** (§2).
4. **Profile assignment** (§3) + **auto-tune resolution** (§4).
5. **Core indicators** — `ema9`, `emaSlow` (period `emaSlowPeriod`), `ema20`
   (always used for scoring), `ema50`, `smaAnchor` (`ta.sma(close, sma_len)`),
   session `vwap`, `atr = ta.atr(14)`, `rsiVal = ta.rsi(14)`, `adx = ta.dmi(14,14)`,
   `adx_rising_2 / _3`.
6. **Regime detection** — squeeze state (BB(20,2) inside KC(20,1.5×ATR)),
   `sqz_release`, `sqz_bias`, `ema_stretch = |close−ema20|/atr`, velocity
   override (`adx > 35 and adx_rising_3`).
7. **ATR Fuel gauge** — session high/low tracked from `is_new_session`
   (daily-bar boundary); `fuel_used = session_range / daily_ATR14 × 100`
   (clamped 0–200). Daily ATR via `request.security(..., lookahead_off)` — reads
   the *developing* daily bar, so `fuel_used` drifts up through the session but
   is still non-repainting.
8. **Time-of-Day relative volume** — `rel_vol` = current bar volume ÷ average
   volume of the *same bar-slot since session open* across the prior
   `rvolLookbackDays` (10) sessions. Removes the U-shaped intraday bias of a
   trailing SMA. Falls back to `ta.sma(volume,20)` when < 3 historical sessions
   exist at that slot, or for volume-less index feeds (`rel_vol → 1.0`).
   Dashboard Volume row shows `TOD` / `20-bar` / `20-bar*` (in-flight fallback).
9. **Dual-anchor trend classification** (§6).
10. **Scoring engine** (§7).
11. **Risk gates** (§8).
12. **Signal generation** (§9).
13. **P&L tracking** (§10) and **win-rate tracking** (§11).
14. **Visuals** (§12), **dashboard** (§13), **alerts** (§14).

---

## 6. Anchors

Both anchors expose the same binary interface — `is_bull`, `is_bear`,
`trendUp`, `trendDown`, `anchor_line_price`, `anchor_short`, `anchor_ready` — so
every downstream consumer is anchor-agnostic (the sole exception is
`maxBarsFromFlip`).

### EMA Cross (`anchorMode = "EMA Cross"`)
- `is_bull = ema9 > emaSlow`; flips on `ta.crossover/crossunder(ema9, emaSlow)`.
- `emaSlow` period is 20 or 30 by profile. **This does not affect scoring** —
  Component 1 always uses `ema20`.
- Crosses lag the turn, so `maxBarsFromFlip = 10` caps how late a Score-mode
  entry may fire.
- `anchor_ready` is always `true` (EMAs are valid from the first bars).

### SMA (`anchorMode = "SMA"`, default)
- Slow structural SMA (length `sma_len`, 70–120 by profile) with an **ATR buffer
  band** and **latched state**:
  ```
  sma_band = smaBufferATR × ATR(14)
  close > smaAnchor + sma_band  → latch BULL
  close < smaAnchor − sma_band  → latch BEAR
  inside band                   → hold previous state
  ```
- `sma_state_bull` is a `var bool` (init `true`), updated only under
  `barstate.isconfirmed` → non-repainting. `sma_is_bear = not sma_state_bull`
  (never an independent test — keeps `is_bull`/`is_bear` strictly binary).
- Flips derived by comparing to `sma_prev_bull` (a second `var bool` captured
  before the update — not `[1]`, which is `na` on bar 0).
- `maxBarsFromFlip = 0` (latch already fires on the flip bar).
- `anchor_ready = not na(smaAnchor)` — blocks signals during the 70–120-bar
  warm-up, where the latch would otherwise report `is_bull = true` regardless of
  price.

**Scoring interaction:** in SMA mode `is_bull` holds through pullbacks, so
Component 1 keeps awarding PARTIAL/WEAK points where EMA-cross mode would score 0
or flip. SMA-mode `raw_score` therefore skews slightly higher on trend
continuation — the two score distributions are **not strictly comparable**.

---

## 7. Scoring engine — 5 components, `raw_score` capped at 100

Component maxes are assembled differently per profile but always total **105
before the cap**.

| # | Component | Max | Direction-sensitive? |
|---|---|--|--|
| 1 | EMA Cascade Alignment | `ema_max` 20–30 | Yes (`is_bull`/`is_bear`) |
| 2 | VWAP Value Anchor | `vwap_max` 15–25 | Yes |
| 3 | Volume Intensity | 25 | No |
| 4 | ADX Trend Strength | 15 | No |
| 5 | Momentum Confluence | 20 | Yes (RSI half) |

**Component 1 — EMA Cascade** (branch chosen by `is_bull`/`is_bear`, scored vs
`ema9/ema20/ema50`):
- FULL (`ema_max`): `ema9 > ema20 > ema50` and close on the right side of `ema9`
- PARTIAL (60%): close beyond `ema20`, cascade incomplete
- WEAK (30%): close beyond `ema50` only
- COUNTER-TREND (0): otherwise

**Component 2 — VWAP Value** (correct side = above for bull, below for bear):
- BOUNCE/REJECTION (100%): within `0.33 × vwap_h_limit` of VWAP
- HEALTHY (72%): within `vwap_h_limit`
- EXTENDED (40%): within `vwap_e_limit`
- REVERSION RISK (0): beyond `vwap_e_limit`
- GRACE ZONE (20%): slightly wrong side — bull tolerance `0.2 × vwap_h_limit`,
  bear `0.1 ×` (intentionally stricter)
- WRONG SIDE (0): otherwise

**Component 3 — Volume Intensity** (`rel_vol` tiers):
25 / 20 / 15 / 10 / 5 / 0 at `rel_vol ≥` 2.5 / 2.0 / 1.5 / 1.2 / 1.0 / below.

**Component 4 — ADX Strength** (thresholds scale with profile `adx_min`; 2-bar
rising qualifies each tier):
15 (≥1.6× & rising) / 12 (≥1.6×) / 12 (≥1.3× & rising) / 9 (≥1.3×) / 9 (≥1.0×) /
6 (≥0.8×) / 3 (≥0.6×) / 0.

**Component 5 — Momentum Confluence** (max 20, **2 sub-components** — MACD was
removed as a near-duplicate of Component 1; its 8 points were redistributed 7:5):
- **5A RSI (12)** — regime-aware. Bull: `>60 rising` → 12, `45–60` → 8,
  `>60 not rising` → 3, else 0. Bear: mirrored (`<45 falling` → 12, `40–55` → 8,
  `<40 rising` → 3).
- **5B Squeeze Release (8)** — `sqz_release and rel_vol > 1.5` → 8;
  `sqz_release` alone → 3; else 0. Always active regardless of gate mode.

`raw_score = t_pts + v_pts + vol_pts + a_pts + r_pts`, then `min(…, 100)`.

**Setup quality stars** (from the active strategy's quality vs the direction's
threshold): `⭐⭐⭐` at `+30`, `⭐⭐` at `+15`, `⭐` at threshold, `⚠️` below.

---

## 8. Risk gates — 4 penalties on `final_score`

`gateOn` is pinned `true` (enforcement). Each uses a `math.max(…, 0)` floor;
`final_score` is finally clamped `[0, 100]`. Gate warnings always render in the
dashboard regardless of `gateOn`.

| Gate | Trigger | Penalty |
|---|---|--|
| 1 — Squeeze | BB(20,2) inside KC(20,1.5×ATR) | −30 |
| 2 — Stretch | `max(ema_stretch, trend_stretch) > ext_scale` (−10) or `> ext_scale × 1.5` (−20) | −10 / −20 |
| 3 — ATR Fuel | `fuel_used > fuel_max` **and not** velocity override | −25 |
| 4 — Liquidity | `rel_vol < rvol_gate` **and** `adx < 0.9 × adx_min` | −25 |

`trend_stretch = |close − trend_start_price| / atr`, where `trend_start_price` is
stamped at each anchor flip.

---

## 9. Signal generation (non-repainting)

Structured as **regime → modules → router → dispatch**. In the light build only
one module is live.

1. **Regime** — descriptive string `SQUEEZE / BREAKOUT / EXHAUSTED / TREND /
   RANGE` from `is_squeezing`, `sqz_release`, `stretch_factor`, `adx`. Feeds the
   dashboard only; gates nothing.
2. **Modules** (each fully self-gated, exposes `_long` / `_short` / `_quality`):
   - **Trend Pullback** (`tp_*`) — live. `sigTime` pinned to `"Score"`:
     `tp_long = in_session and anchor_ready and entry_is_fresh and
     barstate.isconfirmed and is_bull and passes_quality_filter(final_score,
     minScoreBuy)` (short is the mirror). `entry_is_fresh = maxBarsFromFlip <= 0
     or bars_since_flip <= maxBarsFromFlip`. `tp_quality = final_score`. The
     `"Trend"` branch is retained but unreachable.
   - **ORB** (`orb_*`) — inert (`orbEnabled = false`). Opening-range breakout
     code kept so the router / dashboard / plots compile.
3. **Router** — `active_strategy` / `strat_long` / `strat_short` /
   `strat_quality`, last-write-wins `Trend Pullback → ORB`. With ORB off it
   always resolves to Trend Pullback. `stars` is computed here off
   `strat_quality`.
4. **Dispatch** — `signal_buy = strat_long and not b_fired`,
   `signal_sell = strat_short and not s_fired`.

**Strict alternation:** `b_fired` / `s_fired` block a same-direction repeat no
matter how many trend/regime flips occur, until the opposite signal fires (or
RTH session open, `useSessionFilter`-gated). A trend-direction change alone does
**not** clear them.

**RTH window:** `in_session` from `time(period, "0930-1600:23456")` (always ET).
`session_just_opened` flattens position state and clears `b_fired`/`s_fired`
before P&L is evaluated on that bar — hence the RTH section sits *above* P&L
tracking in the file. Off for FUTURES/CRYPTO (`useSessionFilter = false`).

`passes_quality_filter(score, min)` applies `sigQuality`: `All Signals` → `≥ 0`;
`⭐ +` → `≥ min`; `⭐⭐ +` → `≥ min + 15`; `⭐⭐⭐ Only` → `≥ min + 30`.

---

## 10. P&L — two independent systems

- **Live dashboard P&L** — `pnl_entry_price` / `pnl_direction` set on every new
  signal, flattened at RTH open. `current_pnl` = % move from entry. PROFIT/LOSS
  fire once per entry when `current_pnl` crosses `±pnlTarget` on a confirmed bar.
  `target_reached` / `stop_reached` are tracked **regardless of `enablePnL`**;
  `enablePnL` only gates the visible `signal_profit` / `signal_loss` labels +
  alerts.
- **Signal-to-signal P&L history** — `signal_pnl` = % move from
  `last_signal_price` using the previous signal's direction. Shown on the
  BUY/SELL label. Completely separate state.

---

## 11. Win-rate tracking

Rolling `array<bool>` `buy_results` / `sell_results`, FIFO-capped at
`maxSignalsToTrack` (50). Every BUY/SELL is scored exactly once, when the next
opposite-direction signal (or RTH session-open carryover) closes it:
`buy_won_this_entry` / `sell_won_this_entry` accumulate whether `target_reached`
fired at any point during the entry → pushed as `true` (won) / `false` (stop, or
reversed with no target hit). The open entry is never counted until it closes.
`calc_win_rate(arr, enabled) => [wins, total, rate]`.

**Per-module (`mod_results`)** — the same result is mirrored into a 3-slot
`array<ModResults>` keyed by `strat_id(<dir>_entry_strategy)` (0 Trend Pullback /
1 ORB / 2 VWAP Fade), stamped from `active_strategy` at entry open. Only slot 0
ever fills in the light build; the per-module dashboard rows are gated on
`multi_module = orbEnabled` and thus suppressed (they would duplicate the
BUY/SELL Win Rate rows).

---

## 12. Visuals

- **Anchor line** — one branch per anchor. EMA mode: `ema9`, green/red by
  `is_bull`, `linewidth 3`. SMA mode: `smaAnchor` green/red + grey ATR buffer
  band (`smaAnchor ± sma_band`) with a translucent fill.
- **Flip circles** — tiny green/red circles at `trendUp` / `trendDown` bars
  (`showCircles` pinned on).
- **BUY/SELL labels** — anchored to `anchor_line_price`, text `BUY ⭐⭐…`
  (+ `⚡ORB` tag if ORB were ever active) and an optional `signal_pnl` line.
- **PROFIT/LOSS labels** — blue, static text `PROFIT +1.0%` / `LOSS -1.0%`,
  side chosen from the closed entry's direction.
- **ORB high/low** — orange `plot.style_linebr`, gated on `orbShowLevels = false`
  → never drawn.

---

## 13. Dashboard

`table.new` at `position.bottom_right`, 2 columns × `DASHBOARD_MAX_ROWS = 36`,
rendered only on `barstate.islast`. The table is `var`, so every render starts
with `table.clear(...)` (the gate-detail loop writes a variable number of rows).

Fixed rows: **Header** (`SLT V1 | PnL ±x%`) → **Trend (`EMA9`/`SMA`)** with
price → **Profile** → **Signal Anchor** (`SMA (70) auto ±0.25A` /
`EMA Cross (9/20) auto`) → **Strategy** (`Trend Pullback · <regime>`) → *(ORB row
— only `if orbEnabled`, never)* → **ATR Fuel** → **Setup Score** → **Gates**
(active count).

If `enableSuccessRate`: separator → **BUY Win Rate** → **SELL Win Rate**
(colour-coded lime/yellow/red at 60/50) → *(per-module rows — only if
`multi_module`, never)*.

If `extDash`: separator → **EMA Cascade** / **VWAP Value** / **Volume** /
**ADX Strength** / **Momentum** (each `status (points)`) → **Volatility**
(ATR-14 $ + session range $) → **Bars From Flip** (`n / 10 max` or
`n (unlimited)`) → **Gate Details** (word-wrapped at 30 chars, with an explicit
`if row >= DASHBOARD_MAX_ROWS: break` guard).

---

## 14. Alerts

Four `alertcondition` calls with plaintext messages: `"BUY"`, `"SELL"`,
`"PROFIT"`, `"LOSS"`.

---

## 15. Known redundancies / design notes

- **Component 1 ≈ SMA anchor** (SMA mode) — both measure price vs a slow average;
  documented, not corrected (see §6).
- **`rel_vol` is counted three times** — Component 3 tiers, Component 5B gating,
  and Gate 4. A dead-volume bar loses points in all three.
- **`adx` is counted twice** — Component 4 tiers and Gate 4's `is_choppy` test.
- **Extension penalised twice** — Component 2 EXTENDED/REVERSION tiers and Gate 2
  `ema_stretch` (both keyed on price vs a mean; VWAP and `ema20` track closely).
- **MACD removed** — its sign was ≈ `EMA12 > EMA26`, duplicating Component 1;
  its 8 points went to RSI (7→12) and Squeeze (5→8).
- The **105 → 100 pre-cap** still discards 5 points off a max-score bar.

---

## 16. Development workflow

1. Edit `SLT.pine`.
2. Copy the full file → TradingView → Pine Editor → paste → Save → Add to Chart.
3. Compilation errors show immediately in the Pine console.
4. Verify on a 2-min chart of a liquid instrument. See the **Compile-Check
   Checklist** in `CLAUDE.md` for the 20-point manual verification (panel
   contents, score ≤ 100, alternation, gate enforcement, anchor swap, RVOL
   fallback, session flatten, profile auto-tune, ORB inertness, …).

`CLAUDE.md` is the exhaustive internal reference; this README is the summary.

---

## Disclaimer

Educational reference only. Does not guarantee any specific trading results.
