# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Pine Script v6 TradingView indicator** — a single-file intraday momentum trading system, **forked from SuperLazyTrade with the SuperTrend signal anchor removed**. The sole source file is `SLT.pine`. There is no build system, package manager, or test runner; development means editing the `.pine` file and pasting it into TradingView's Pine Editor to compile and validate.

SLT keeps two signal anchors: **SMA** (default) and **EMA Cross**. Everything else — the 5-component scoring engine, the 4 risk gates, both P&L tracking systems, win-rate tracking, and the dashboard — is unchanged from the parent project.

## Development Workflow

1. Edit `SLT.pine`
2. Copy the full file contents
3. Open TradingView → Pine Editor → paste → Save → Add to Chart
4. Compilation errors appear immediately in the Pine Editor console
5. Test on a 2-minute chart with a liquid instrument (NVDA, TSLA, SPY, QQQ, SOXX, Silver Futures)

## Architecture

The script is organized into sequential sections (read top-to-bottom, order matters in Pine Script):

1. **Constants** — Gate penalty values, dashboard sizing (`DASHBOARD_MAX_ROWS = 32`)
2. **Inputs** — All user-configurable parameters grouped by function:
   - `grp_anchor`: Signal Anchor (EMA Cross / SMA) + EMA slow period
   - `grp_sma`: SMA anchor length mode (AUTO / MANUAL), manual length, ATR buffer multiplier
   - `grp_sig`: Signal timing, quality filter, min scores, RTH session filter, gate enforcement
   - `grp_orb`: Opening Range Breakout module — enable, range minutes, entry cutoff, breakout buffer, min RVOL, VWAP alignment, min setup quality, plot levels
   - `grp_fade`: VWAP Fade module — enable (default OFF), fade band (ATR), max RVOL, min setup quality, plot bands
   - `grp_pnl`: P&L exit signals, target %, signal P&L history
   - `grp_success`: Success rate tracking, rolling window size
   - `grp_vis`: Dashboard visibility, extended metrics, circles, background
3. **Asset profile assignment** — A `switch` on `syminfo.type` first computes `asset_category` (STOCK/FUND/FUTURES/CRYPTO/MARKET); a separate `if/else if` chain keyed on `asset_category` then sets `profile_name` and all threshold variables used throughout scoring, including the SMA anchor length `sma_len` (it keys on `asset_category`, so it is set here rather than per-ticker). See [Profile Parameters Table](#profile-parameters-table).
4. **Core indicator calculations** — EMAs (9, `emaSlow`=user-choice 20 or 30, 20 fixed for scoring, 50), SMA anchor, VWAP, ATR(14), RSI(14), ADX(14,14), MACD(12,26,9), relative volume (`rvolMode`: Time-of-Day default, Rolling 20-bar SMA fallback/alternative — see [Time-of-Day Relative Volume](#key-design-decisions))
5. **Market regime detection** — Squeeze state (BB(20,2) inside KC(20,1.5×ATR)), squeeze release + bias, stretch factor (EMA distance + trend move since flip), velocity override (ADX>35 rising 3 bars), ATR fuel gauge (session range vs daily ATR-14)
6. **Dual-anchor trend classification** — EMA Cross or SMA mode; `is_bull`/`is_bear`/`trendUp`/`trendDown` unify both anchors behind a single interface, and `anchor_line_price`/`anchor_short` unify the *display* side. `trend_start_price` updated on every `trendUp`/`trendDown`. See [SMA Anchor](#sma-anchor).
7. **Scoring engine** — 5 components summed to `raw_score` (capped at 100). Max theoretical total = 105 across all profiles. See [Scoring Components](#scoring-components).
8. **Risk gates** — 4 gates calculate penalties; applied to `raw_score` → `final_score` only when `gateOn = true`; always shown as warnings regardless. See [Risk Gates](#risk-gates).
9. **Signal generation** — Four sub-stages (regime-router refactor, **Phase 3**: three modules):
   - **Regime classification** — `regime` string (`SQUEEZE` / `BREAKOUT` / `EXHAUSTED` / `TREND` / `RANGE`) derived from `is_squeezing`, `sqz_release`, `stretch_factor`, `adx`. Feeds the dashboard **and** gates the VWAP Fade module's eligibility (see below); still descriptive for Trend Pullback and ORB.
   - **Strategy modules** — each is fully self-gated and exposes `<name>_long` / `<name>_short` / `<name>_quality`:
     - **Trend Pullback** (`tp_*`) — the anchor + 5-component confluence + `passes_quality_filter` logic, condition-for-condition unchanged from pre-refactor. `sigTime` ("Trend" fires off `trendUp`/`trendDown`; "Score" fires off `is_bull`/`is_bear` + `entry_is_fresh` + explicit `barstate.isconfirmed`) is a **mode of this module**. Both modes AND in `in_session` and `anchor_ready`. `tp_quality = final_score`.
     - **ORB** (`orb_*`) — Opening Range Breakout. See [ORB Module](#orb-module).
     - **VWAP Fade** (`fade_*`) — counter-trend mean reversion. See [VWAP Fade Module](#vwap-fade-module).
   - **Regime router** — `active_strategy` / `strat_long` / `strat_short` / `strat_quality`, resolved last-write-wins: **Trend Pullback (default) → VWAP Fade → ORB**. VWAP Fade sits above Trend Pullback because they are opposite stances; the fade's `RANGE`/`EXHAUSTED` + `adx < 1.5×adx_min` gate is what keeps both from being live in the same backdrop. An ORB breakout tops both. The router only chooses between modules — it never loosens a module's own gating.
   - **Signal dispatch** — `signal_buy = strat_long and not b_fired`, `signal_sell = strat_short and not s_fired`. `b_fired`/`s_fired` enforce strict BUY/SELL alternation **across all modules and any number of regime/trend flips**; they clear only when the opposite signal fires or at RTH session open (`useSessionFilter`-gated) — **not** on a trend-direction change by itself.
   Signals are non-repainting; `barstate.isconfirmed` guards every flip/breakout condition. `stars` / `setup_quality` (chart labels) are computed just after the router off `strat_quality`, so they reflect the **active** module. RTH filter via `time(timeframe.period, "0930-1600:23456")` — `in_session`/`session_just_opened` are computed in their own **RTH Session Window** section placed *before* P&L exit tracking, because the session-open reset must flatten stale position state before `current_pnl` is evaluated on that same bar. Score-mode signals can also be constrained to fire only within `maxBarsFromFlip` bars of the anchor flip (default 0 = disabled); Trend mode is unaffected.
10. **P&L tracking** — Two independent systems: (a) live dashboard P&L using `pnl_entry_price`/`pnl_direction`, reset on every new signal and flattened at RTH session open; (b) signal-to-signal `signal_pnl` history shown on labels. PROFIT/LOSS exit triggers are Fixed %-only: `current_pnl` vs `±pnlTarget`. Target/stop resolution (`target_reached`/`stop_reached`) is tracked internally regardless of `enablePnL`; `enablePnL` only gates the visible `signal_profit`/`signal_loss`.
11. **Success rate tracking** — Rolling arrays (`buy_results`/`sell_results`), capped at `maxSignalsToTrack`; every signal is scored won/lost when the next opposite-direction signal closes it — won only if the target was reached during the entry, lost otherwise. **Module-blind:** an ORB signal and a Trend Pullback signal both feed the same BUY/SELL arrays; per-strategy win rates are Phase 4.
12. **Visuals** — Conditional plots, one branch per anchor: EMA9 dynamic line (green/red), or SMA line (green/red) plus its grey ATR buffer band; flip circles at transition bars; BUY/SELL labels anchored to `anchor_line_price` (with a `⚡ORB` / `🔄FADE` tag when the router picked that module). Plus the locked **ORB high/low** (orange) and the **VWAP fade band edges** (teal), all `plot.style_linebr` (break between sessions), gated on `orbShowLevels` / `fadeShowBands`.
13. **Dashboard** — `table.new` at `position.bottom_right` with `DASHBOARD_MAX_ROWS = 32`; rendered only on `barstate.islast`. The table is `var`, so every render starts with `table.clear(d, 0, 0, 1, DASHBOARD_MAX_ROWS - 1)` — without it, rows written by the variable-length gate-detail loop persist after a gate deactivates and keep showing a stale warning. The gate detail loop is the last section written; it has an explicit `if row >= DASHBOARD_MAX_ROWS: break` guard because it is the only variable-length section. All preceding rows are bounded by design. The first data row reports the **active anchor** (`Trend (EMA9)` or `Trend (SMA)`) and its own price. Both that row and the four BUY/SELL label sites read the shared `anchor_line_price`/`anchor_short` globals rather than each re-deriving the anchor. The **Strategy** row (formerly "Market State") shows `active_strategy · <regime>`. The **ORB** row (only when `orbEnabled`) shows `needs RTH filter` / `waiting for open` / `forming H/L` / `armed H/L Q<n>` / `fired this session`. The **VWAP Fade** row (only when `fadeEnabled`) shows `idle (<regime>)` / `watching ±<n> ATR` / `stretched ↑/↓ <n> ATR [· done]`. `DASHBOARD_MAX_ROWS` was bumped 28→30→32 for those two optional rows; a user with both modules off sees the original row count. `dashboard_score` reads `strat_quality` (the active module's quality).
14. **Alerts** — Four `alertcondition` calls with plaintext messages: `"BUY"`, `"SELL"`, `"PROFIT"`, `"LOSS"`.

---

## Scoring Components

All 5 components sum to `raw_score`, capped at 100. Component maxes vary by profile but always total 105 before the cap.

| # | Component | Max Points | Varies by Profile? |
|---|-----------|-----------|-------------------|
| 1 | EMA Cascade Alignment | `ema_max` (20–30) | Yes |
| 2 | VWAP Value Anchor | `vwap_max` (15–25) | Yes |
| 3 | Volume Intensity | 25 (fixed) | No |
| 4 | ADX Trend Strength | 15 (fixed) | No |
| 5 | Momentum Confluence | 20 (fixed) | No |

**Component 1 — EMA Cascade (profile-adaptive):**
- Full (`ema_max`): `ema9 > ema20 > ema50` + close on right side of ema9
- Partial (60%): close beyond ema20 but cascade incomplete
- Weak (30%): close beyond ema50 only
- Zero: counter-trend
- Always uses `ema20` (not `emaSlow`) — slow period choice only affects anchor, not scoring

**Component 2 — VWAP Value (profile-adaptive):**
- BOUNCE/REJECTION (full `vwap_max`): within 0.33× `vwap_h_limit` of VWAP, correct side
- HEALTHY (72%): within `vwap_h_limit`, correct side
- EXTENDED (40%): within `vwap_e_limit`, correct side
- REVERSION RISK (0): beyond `vwap_e_limit`, correct side
- GRACE ZONE (20%): slightly wrong side — bull: within 20% of `vwap_h_limit`; bear: within 10% (intentionally stricter)
- WRONG SIDE (0): too far on wrong side

**Component 3 — Volume Intensity:**
- 25pts: RVOL ≥ 2.5× (EXPLOSIVE)
- 20pts: ≥ 2.0× | 15pts: ≥ 1.5× | 10pts: ≥ 1.2× | 5pts: ≥ 1.0× | 0pts: below
- RVOL denominator is `rvolMode`-dependent — point tiers themselves never changed. See [Time-of-Day Relative Volume](#key-design-decisions).

**Component 4 — ADX Strength (scales with profile `adx_min`, nested if/elseif — rising (2-bar) qualifies each tier):**
- 15pts: ADX ≥ 1.6× `adx_min` AND rising | 12pts: ADX ≥ 1.6× `adx_min` and NOT rising
- 12pts: ADX in [1.3×, 1.6×) `adx_min` AND rising | 9pts: same range, NOT rising
- 9pts: ADX in [1.0×, 1.3×) `adx_min` (no rising check)
- 6/3/0: moderate (≥0.8×) / weak (≥0.6×) / choppy (below)

**Component 5 — Momentum Confluence (3 sub-components, max 20):**
- 5A MACD (8pts): MACD line same sign as trend direction
- 5B RSI (7pts): regime-aware — bull wants RSI >60 rising; bear wants RSI <45 falling
- 5C Squeeze Release (5pts): sqz_release + RVOL >1.5 → 5pts; release alone → 2pts

**Setup quality rating** (applied to `final_score` after any gate enforcement):

`threshold` = `minScoreBuy` (bull) or `minScoreSell` (bear) — both are user inputs defaulting to 50, range 0–70. Max 70 ensures ⭐⭐⭐ EXCELLENT (threshold+30) is always reachable.

- ⭐⭐⭐ EXCELLENT: `final_score ≥ threshold + 30`
- ⭐⭐ STRONG: `≥ threshold + 15`
- ⭐ GOOD: `≥ threshold`
- ⚠️ WEAK: below threshold

---

## Risk Gates

4 gates total. In **Advisory mode** (`gateOn = false`): shown in dashboard but not applied. In **Enforcement mode** (`gateOn = true`, default): penalties subtract from `final_score`; all use `math.max(..., 0)` floor before the final clamp `[0, 100]`.

| Gate | Trigger | Penalty | Constant |
|------|---------|---------|----------|
| 1 — Squeeze | BB(20,2) inside KC(20,1.5×ATR) | −30 | `SQUEEZE_PENALTY` |
| 2 — Stretch Factor | Unified: MAX(EMA stretch, trend stretch) > `ext_scale` | −10 or −20 | `STRETCH_MODERATE/EXTREME_PENALTY` |
| 3 — ATR Fuel | Session range > `fuel_max`% of daily ATR-14 AND no velocity override | −25 | `FUEL_PENALTY` |
| 4 — Liquidity | RVOL < `rvol_gate` AND ADX < 90% of `adx_min` | −25 | `LIQUIDITY_PENALTY` |

**Stretch Factor detail:** `stretch_factor = max(ema_stretch, trend_stretch)` where `ema_stretch = |close - ema20| / ATR(14)` and `trend_stretch = |close - trend_start_price| / ATR(14)`. Moderate penalty when `> ext_scale`; extreme when `> ext_scale × 1.5`. Velocity override (`ADX > 35` rising 3 bars) bypasses Gate 3 only.

---

## SMA Anchor

The default `anchorMode` option (the other is EMA Cross). Selected when `anchorMode = "SMA"`.

**Length** (`sma_len`) is assigned per asset profile in the profile block — see the `sma_len` column in the [Profile Parameters Table](#profile-parameters-table) — and resolved to `smaLenActive` by `smaMode`:

- `smaMode = "AUTO"` (default) → per-profile `sma_len`
- `smaMode = "MANUAL"` → `smaLen_manual` (default 70, range 10–300)

**Flip semantics — ATR buffer band with latched state.** A raw `ta.crossover(close, sma)` whipsaws badly on a 2-min chart, and in Trend mode `trendUp` fires the signal directly off the flip, so the raw cross is not usable as-is. Instead:

```
sma_band = smaBufferATR × ATR(14)          // default 0.25, range 0.0–2.0
close > smaAnchor + sma_band  → latch BULL
close < smaAnchor - sma_band  → latch BEAR
inside the band               → hold previous state
```

`sma_state_bull` is a `var bool` initialized `true`, updated only under `barstate.isconfirmed` (so the state cannot flip intrabar and revert — same non-repainting guarantee as the EMA Cross anchor) and guarded by `not na(smaAnchor) and not na(sma_band)` for the warmup bars before the SMA is valid.

**Why latched rather than a true neutral zone:** several downstream consumers are written as `is_bull ? … : …` with no third case (dashboard trend row, anchor line color). If `is_bull` and `is_bear` were both `false` inside the band, those sites would silently render bearish. Latching preserves the strictly-binary `is_bull`/`is_bear` contract that EMA Cross also satisfies. `sma_is_bear` is defined as `not sma_state_bull`, never as an independent test.

Flips derive from the latch by comparing against `sma_prev_bull`, a second `var bool` assigned from `sma_state_bull` *before* the update block each bar. This is deliberately not `sma_state_bull[1]`: that reads `na` on bar 0, making the flip booleans `na` rather than `false`, and `nz()` has no bool overload to default it with (`CE10123` — `nz` expects a numeric `source`). Because flips come from the latch, `trend_start_price` and `bars_since_flip` stay correct with no extra work, and `maxBarsFromFlip` applies to SMA mode in Score mode exactly as it does to EMA Cross.

Setting `smaBufferATR = 0` degrades to a raw price/SMA cross (the source behavior) — expect substantially more flips.

**Scoring interaction to be aware of:** the SMA anchor overlaps conceptually with Component 1 (EMA Cascade), which already scores price against `ema20`/`ema50`. In SMA mode the anchor is partly being scored by a related measure. Nothing breaks — the periods differ and the other four components are independent — but SMA-mode scores skew slightly higher on trend continuation, which matters when comparing win rates across the two anchor modes.

---

## ORB Module

Second strategy module (Phase 2 of the regime-router refactor). Lives in the STRATEGY MODULES section between Trend Pullback and the router. `grp_orb` inputs. **`orbEnabled` defaults to `true`**, so on a chart with the RTH filter on it is live out of the box — disable it to get pre-Phase-2 behavior.

**State (`var`, reset on `session_just_opened`):** `orb_open_time` (epoch ms of the RTH open bar — the session anchor), `orb_high` / `orb_low`, `orb_locked`, `orb_done` (one breakout per session).

**Range build → latch.** On the session-open bar, seed `orb_high/orb_low` = that bar's `high`/`low`. On each subsequent **confirmed** in-session bar while `orb_mins < orbMinutes` (where `orb_mins = (time - orb_open_time) / 60000`), extend the high/low. The first confirmed bar at/after `orbMinutes` sets `orb_locked := true` and stops extending. Confirmed-bar-only updates keep it non-repainting, same as the anchors.

**Needs `useSessionFilter = ON`.** `session_just_opened` is `useSessionFilter and …` — with the filter off it never fires, `orb_open_time` stays `na`, the range never locks, and `orb_long`/`orb_short` (both AND `orb_locked`) can't fire. The module is inert, not erroring. Dashboard ORB row shows `⚠️ needs RTH filter`.

**Breakout trigger** (`orb_long` / `orb_short`), all required, all on a confirmed bar:
- `orb_locked` and not `orb_done` (one per session)
- `orb_mins <= orbEntryCutoffMin` (0 = no cutoff)
- `close > orb_high + orbBufferATR×ATR(14)` (long) / `close < orb_low - …` (short)
- `rel_vol >= orbMinRvol` (0 disables)
- VWAP alignment if `orbRequireVwapSide` (long: `close > vwap`)
- `orb_quality >= orbMinQuality`

`orb_done := true` on the firing bar (evaluated after the trigger booleans, so the signal still passes through that bar).

**`orb_quality` (0–100)** — additive, `math.min(…, 100)`: range geometry (30 for range 0.75–2.0 ATR / 18 for 0.5–3.0 / else 8) + breakout `rel_vol` (30/22/14/6/0 at ≥2.0/1.5/1.2/1.0/else) + VWAP-side alignment (12) + anchor-direction agreement (8) + ATR fuel left (20/12/5 at `fuel_used` <50/<75/<`fuel_max`). Feeds `strat_quality` (and thus `stars` / dashboard Setup Score) when ORB is the active strategy.

**Router interaction:** `if orb_long or orb_short` → `active_strategy := "ORB"` and `strat_*` take the `orb_*` values, overriding Trend Pullback for that bar. Everything downstream (dispatch alternation, P&L, win-rate, labels `⚡ORB`, alerts) is module-agnostic and needs no ORB-specific code.

**Not modelled yet:** failed-breakout / reversal entries (only the first break each session is taken), a separate ORB stop/target (P&L exits are still the global Fixed-% system), and per-module win-rate (Phase 4).

---

## VWAP Fade Module

Third strategy module (Phase 3). **Counter-trend** mean reversion — the first module with a direction stance opposite to Trend Pullback. `grp_fade` inputs. **`fadeEnabled` defaults to `false`** — opt in deliberately.

**Idea:** price stretches past `VWAP ± fadeBandATR × ATR(14)`, stalls, then closes back inside the band on a reversal bar → fade the excursion toward VWAP.

**State (`var`, reset on `session_just_opened`):** `fade_armed_up`/`fade_armed_dn` (in an above/below-band excursion), `fade_peak_up`/`fade_peak_dn` (max `|price − vwap|` reached during it), `fade_fired_up`/`fade_fired_dn` (fade already taken for this excursion).

**Excursion tracking (confirmed bars only):** `high > up_edge` arms the up side and grows `fade_peak_up`; `close < vwap` is the **deep reset** — clears arm + peak + fired so a fresh stretch is needed for another fade. Mirror for the down side (`low < dn_edge`, `close > vwap`). Because the deep reset keys on the VWAP cross, not the session boundary, the module also works with `useSessionFilter` off (VWAP itself still re-anchors daily).

**Eligibility gate (`fade_regime_ok`):** `regime ∈ {RANGE, EXHAUSTED}` **and** `adx < 1.5 × adx_min`. This is where `regime` stops being purely descriptive. Never fades `TREND` / `BREAKOUT` / `SQUEEZE`.

**Trigger** (`fade_short_trig` / `fade_long_trig`), all required, all confirmed-bar:
- armed on that side and not yet fired for this excursion
- `close` back inside the band edge
- reversal bar: `close < open` (short) / `close > open` (long)
- momentum rolling over: `rsiVal < rsiVal[1]` (short) / `rsiVal > rsiVal[1]` (long)

Plus `fade_calm` (`rel_vol < fadeMaxRvol`, 0 disables — a volume thrust is a breakout, not exhaustion) and `fade_quality >= fadeMinQuality`. `fade_fired_up/dn` is set on the firing bar.

**`fade_quality` (0–100)** — additive, `math.min(…,100)`: excursion size (`fade_peak/ATR`; 35 for 1.5–4.0, 20 for 1.0–6.0, else 8) + reversal-bar strength (25 if the close is in the reverting third of the bar's range, else 10) + volume calm (`rel_vol` <1.0 → 20, <1.5 → 10) + regime (`EXHAUSTED` → 20, `RANGE` → 12).

**Router:** `if fade_long or fade_short` sets `active_strategy := "VWAP Fade"` and `strat_*` to the `fade_*` values, above Trend Pullback but below ORB. Downstream (dispatch alternation, P&L, win-rate, `🔄FADE` label tag, alerts) is unchanged and module-agnostic.

**Not modelled yet:** a VWAP-target exit (P&L exits are still global Fixed-%), a re-arm short of the full VWAP round-trip, and any user control to relax the regime gate.

---

## Profile Parameters Table

All threshold variables are assigned once per bar based on `syminfo.type`. The `switch` expression maps every possible `syminfo.type` value: `"stock"` → STOCK, `"fund"` → FUND, `"futures"` → FUTURES, `"crypto"` → CRYPTO, and **everything else** (index, forex, unknown) → MARKET via the switch default. The MARKET profile is the true catch-all; there is no separate DEFAULT profile in the code.

| Profile | `vwap_h_limit` | `vwap_e_limit` | `adx_min` | `rvol_gate` | `ext_scale` | `fuel_max` | `ema_max` | `vwap_max` | `sma_len` |
|---------|----------------|----------------|-----------|-------------|-------------|------------|-----------|------------|-----------|
| MARKET INDEX 🏦 *(index, forex, unknown)* | 0.7% | 1.5% | 25 | 1.0× | 2.0 ATR | 80% | 22 | 23 | 90 |
| ETF / FUND 📊 | 1.0% | 2.0% | 18 | 1.0× | 2.2 ATR | 75% | 20 | 25 | 90 |
| STOCK 🚀 | 1.8% | 3.5% | 20 | 1.2× | 3.0 ATR | 85% | 20 | 25 | 70 |
| FUTURES ⚡ | 1.5% | 3.0% | 15 | 0.6× | 2.5 ATR | 90% | 30 | 15 | 70 |
| CRYPTO 🪙 | 3.0% | 6.0% | 28 | 0.6× | 4.0 ATR | 95% | 25 | 20 | 120 |

---

## Key Design Decisions

**Profile-adaptive scoring:** `ema_max` and `vwap_max` vary by asset type (e.g., FUTURES gets `ema_max=30`, `vwap_max=15`), so the raw 100-point maximum is assembled differently per profile. All profiles sum to 105 before the cap. When modifying component weights, check all five profiles.

**Strategy router (phased refactor, currently Phase 3):** Signal generation is structured as *regime classification → strategy modules → router → dispatch*. Three modules exist: **Trend Pullback** (`tp_*`, the anchor + confluence logic verbatim), **ORB** (`orb_*`, see [ORB Module](#orb-module)), **VWAP Fade** (`fade_*`, see [VWAP Fade Module](#vwap-fade-module)). The router (`active_strategy`/`strat_long`/`strat_short`/`strat_quality`) resolves last-write-wins Trend Pullback → VWAP Fade → ORB. `regime` now gates the VWAP Fade module's eligibility (its first non-descriptive use). Every downstream consumer (P&L, win-rate, labels, alerts, dashboard) fires off the dispatched `signal_buy`/`signal_sell` / the `strat_*` surface, so modules are added without touching them. **When adding a module:** expose `<name>_long`/`<name>_short`/`<name>_quality`, keep it fully self-gated, register it in the router (opposite-stance modules go *above* Trend Pullback and must carry a regime gate), and let dispatch apply alternation — do not re-implement session/alternation per module. Still to come: Phase 4 (per-module win-rate via generalized results arrays; `regime` as an explicit eligibility matrix for every module; HOD/LOD-break and prior-day-level modules).

**Dual-anchor unification:** After anchor selection, all downstream logic uses `is_bull`, `is_bear`, `trendUp`, `trendDown` — never `ema_*` or `sma_*` directly. The scoring engine, all 4 risk gates, signal generation, both P&L systems, win-rate tracking, and `maxBarsFromFlip` are all anchor-agnostic through this interface. The EMA slow period (`emaSlowPeriod` = 20 or 30, default 30) affects cross detection but NOT Component 1 scoring, which always uses `ema20`. Score mode and Trend mode are both gated through the same `is_bull`/`is_bear` flags.

The anchor selectors are **binary ternary chains** (`anchorMode == "SMA" ? … : …`), every one ending in the EMA Cross branch — so any `anchorMode` value other than `"SMA"` resolves to EMA Cross behavior rather than erroring. A new anchor would have to be added to all four selectors plus `anchor_line_price`, `anchor_short`, and `anchor_ready` explicitly. Only the *drawing* sites (`plot`, `plotshape`, `fill`, dashboard `anchor_display`) use explicit `anchorMode == "..."` equality, so an unhandled mode draws no anchor line at all — which is the symptom to look for.

**`anchor_ready` — anchor warm-up guard:** `ta.sma` returns `na` until `smaLenActive` bars exist (70–120 by profile), and `sma_state_bull` is initialized `true` with its update block skipped while `smaAnchor` is `na`. The SMA anchor therefore reports `is_bull = true` for its whole warm-up window regardless of price direction. `anchor_ready` (`not na(smaAnchor)` in SMA mode, `true` otherwise) is AND'd into both the Trend-mode and Score-mode signal branches to block that. It matters most in the default configuration — Score timing with `maxBarsFromFlip = 0` — where nothing else would stop a warm-up BUY from firing on score alone in a downtrend, seeding the win-rate array and drawing its label at a `na` price. Both EMAs produce usable values from the first bars, so EMA Cross mode passes `anchor_ready` unconditionally.

**Non-repainting:** *Signals* are non-repainting — `barstate.isconfirmed` guards every flip condition and all signal assignments. The daily ATR (`atr_14`) uses `request.security(..., lookahead_off)`, which reads the *developing* daily bar, so `fuel_used` drifts upward through the session. This is **not** display-only — it feeds Gate 3 (ATR Fuel) and therefore `final_score` and signal generation. It is still non-repainting: `lookahead_off` guarantees the value is computed from data available at that bar, so a historical bar recomputes to the same number it had live.

**Gate vs advisory mode:** Setting `gateOn = true` (default/recommended) subtracts gate penalties from `final_score`, directly suppressing low-ADX/range-bound chop entries (Gates 1 and 4 in particular). Setting `gateOn = false` shows gate warnings in the dashboard but does NOT subtract from `final_score`. Component 5C (Squeeze Release) is always active regardless of gate mode, to keep scoring consistent.

**Signal blocking (strict alternation):** `b_fired`/`s_fired` block a same-direction signal from firing again — in both "Trend" and "Score" signal-timing modes — until the opposite signal actually fires, no matter how many trend flips happen in between. E.g. BUY fires, trend flips bear but SELL's score/quality filter never clears, trend flips bull again → BUY stays blocked. The flags do **not** reset on a trend direction change by itself (`is_bull` vs `is_bull[1]`); they only clear when `signal_buy`/`signal_sell` actually fires (`b_fired := true, s_fired := false` and vice versa) or at RTH session open (`useSessionFilter`-gated, so 24h instruments with the filter off never get a daily reset). Live P&L tracking (`pnl_entry_price`, dashboard P&L, signal-to-signal P&L history) does not consult these flags and keeps accumulating across the held direction regardless.

**RTH filter on 24h instruments:** `time(timeframe.period, "0930-1600:23456")` always evaluates against ET hours regardless of instrument type. The filter does NOT auto-disable for CRYPTO or 24h FUTURES. If `useSessionFilter = true` on those instruments, signals will be blocked outside 9:30–4:00 ET Mon–Fri with no warning. Recommended: disable `useSessionFilter` for crypto and around-the-clock futures contracts.

**Entry freshness constraint:** `maxBarsFromFlip` (0-50, default 0 = disabled) constrains Score-mode entries to fire only within N bars of the most recent anchor flip, via `entry_is_fresh = maxBarsFromFlip <= 0 or bars_since_flip <= maxBarsFromFlip` AND'd into the Score-mode branch only. `bars_since_flip` (`var int`) resets to 0 on the flip bar itself and increments every bar after, tracked alongside the existing `trend_start_price` reset so both stay in sync with the same flip event. Trend mode is deliberately unaffected — it already fires directly on the flip bar by construction, so there's nothing to constrain. This is independent of (and complements) the Stretch gate, which only actually penalizes extension when `gateOn = true`; `maxBarsFromFlip` works regardless of gate mode. Calibrate using the Extended Metrics "Bars From Flip" row.

**Two P&L tracking systems (independent):**
- *Live dashboard P&L* — tracks position from `pnl_entry_price`, reset on every new `signal_buy`/`signal_sell`. Since strict alternation (see Signal Blocking above) means there's only ever one signal per held direction, entry always corresponds to the signal that opened the current position — a separate "First Signal" reset-on-direction-change mode is no longer meaningful and was removed.
- *Signal P&L History* — shown on BUY/SELL labels; calculates `%` move from `last_signal_price` to current close using the previous signal's direction. Completely separate state from dashboard P&L.

**P&L exits are Fixed %-only:** `target_reached`/`stop_reached` fire when `current_pnl` (computed from `pnl_entry_price`) crosses `±pnlTarget`, once per entry, on a confirmed bar. `signal_profit`/`signal_loss` are those flags AND `enablePnL`, and drive only the labels and alerts.

**`enablePnL` is display-only — it must not gate resolution:** win-rate scoring reads `target_reached`, *not* `signal_profit`. If it read `signal_profit`, turning off "Enable P&L Exit Signals" (presented purely as a label/alert toggle) would make every tracked entry resolve as a loss and pin both win rates at 0% with no warning. Keep any future outcome/statistics logic on `target_reached`/`stop_reached`.

**Position state is flattened at RTH session open:** when `session_just_opened`, `pnl_entry_price`/`pnl_direction`/`pnl_exit_fired`/`pnl_exit_type` are all reset, matching the win-rate tracker which already closes any carried-over entry at that boundary. Without this, yesterday's entry price survives into the new session and an opening gap can immediately fire a PROFIT or LOSS label *and alert* against a position the tracker already considers closed. This is why the RTH Session Window section sits *above* P&L exit tracking in the file — the reset has to land before `current_pnl` is computed on the session-open bar. `useSessionFilter`-gated, so 24h instruments running with the filter off never flatten.

**Win-rate tracking — every signal, scored on close:** `buy_results`/`sell_results` count every BUY/SELL signal exactly once, not just ones that hit a PROFIT/LOSS target. An entry's outcome isn't known until it's closed by the next opposite-direction signal (per strict alternation, there's exactly one open position per direction at a time), so `buy_won_this_entry`/`sell_won_this_entry` (`var bool`) accumulate whether `target_reached` fired at any point during the entry; when the opposite signal fires, that flag is pushed into the results array (`true` = won, `false` = everything else — a `stop_reached` exit, or the position simply being reversed with no target ever hit) and then reset for the new entry. `buy_entry_open`/`sell_entry_open` (`var bool`) guard against scoring before any entry has actually opened. Each results array is an independent rolling `array<bool>` capped at `maxSignalsToTrack` via FIFO (`array.shift` on overflow). `calc_win_rate(arr, enabled) => [wins, total, rate]` computes stats for both from one shared read-only function. The currently-open entry is never counted until it closes — the win rate reflects only resolved signals.

**Dashboard row budget:** `DASHBOARD_MAX_ROWS = 32` (rows 0–31), bumped 28→30 (Phase 2) →32 (Phase 3). Two optional fixed rows — **ORB** (when `orbEnabled`) and **VWAP Fade** (when `fadeEnabled`) — each +1; the +4 cap keeps ~1 row of headroom at max config (all options + 4 gates + Extended Metrics + Success Rate + ORB + Fade). The gate detail loop is the last section; it has an explicit `if row >= DASHBOARD_MAX_ROWS: break` guard because it is the only variable-length section (all other rows are fixed-count by design). Success Rate Tracking occupies exactly 3 rows (separator + BUY Win Rate + SELL Win Rate) whenever enabled, regardless of Extended Metrics.

**VWAP grace zone asymmetry:** Bull grace zone tolerance = `vwap_h_limit × 0.2`; bear grace zone = `vwap_h_limit × 0.1`. Bears are held to half the tolerance — intentional, reflecting that short-side entries near VWAP carry more reversion risk.

**Time-of-Day Relative Volume:** `rel_vol` (used by Component 3, Gate 4 liquidity, and Component 5C squeeze-release) is `rvolMode`-dependent. Default `"Time-of-Day"` compares current bar volume to the average volume of the *same bar-slot* (bars since session open) across the prior `rvolLookbackDays` completed sessions — this removes the U-shaped intraday volume bias that a trailing SMA has (overstates RVOL near the open, understates it at lunch). History is kept in `array<TodSession>`, where `TodSession` is a UDT wrapping `array<float>` — Pine cannot nest arrays directly (`array<array<float>>` is not supported), so a UDT wrapper is the standard workaround. Session boundaries reuse the existing `is_new_session` (calendar-day change) detector, so the same extended-hours caveat applies as the ATR Fuel gauge. Falls back to the `ta.sma(volume, 20)` method (`"Rolling 20-bar"`) when fewer than 3 historical sessions exist at a slot, or when the user selects Rolling 20-bar explicitly. The active mode is shown inline in the dashboard's Volume row (`TOD`, `20-bar`, or `20-bar*` for an in-flight TOD→fallback).

---

## Pine Script v6 Gotchas

Common failure modes when editing this script:

- **`var` variables** persist across bars and only initialize on bar 0. Do not use `var` for values that must recompute each bar (e.g., profile thresholds). The profile threshold variables (`vwap_h_limit`, `ema_max`, etc.) deliberately omit `var` so they reassign every bar.
- **`request.security` series vs simple:** The daily ATR (`atr_14`) uses `lookahead = barmerge.lookahead_off`. Omitting this causes future-bar lookahead (repainting). Always pass the expression directly — do not pre-compute into a `var` before passing to `request.security`.
- **`na` propagates through arithmetic:** `math.max(na, 0)` returns `na`, not 0. Always guard with `not na(x)` or `nz(x, 0)` before arithmetic involving potentially uninitialized series. This script uses explicit `not na(...)` checks before VWAP and session range calculations.
- **`barstate.isconfirmed` on historical bars:** All historical bars are "confirmed" during replay. `barstate.islast` is true only on the most recent bar. The dashboard uses `barstate.islast`; signals use `barstate.isconfirmed`.
- **`ta.crossover`/`ta.crossunder` are single-bar events:** They return `true` for exactly one bar. No multi-bar guard is needed; however, both require at least 2 bars of history.
- **`array.get` throws on out-of-bounds:** Always check `array.size(arr) > 0` before accessing index 0. The success rate arrays are guarded by `totalTrades_buy > 0` before the win-count loop.
- **`time()` session strings use ET for US equities** but the exchange timezone for other instruments. `"0930-1600:23456"` is hardcoded against ET — this is correct for US stocks but not appropriate for crypto or 24h futures without user awareness (see RTH filter note above).
- **`str.split` includes empty strings** if the delimiter appears at the start/end of the string. `gate_message` is built with a trailing space, so `str.split(gate_message, " ")` always yields a trailing empty token. The word-wrap loop never turns this into a visible row, though: an empty word either merges into the current line as a harmless trailing space (line stays under the 30-char cap) or, if a line break is forced first, becomes `current_line := ""`, which the final `if current_line != "": array.push(...)` guard then excludes. No blank row is ever pushed to `gate_lines`.
- **`var table` cells persist across renders** — `table.cell` only overwrites the cells you write this bar. Any row written on a previous render and *not* rewritten keeps its old text forever. The dashboard therefore calls `table.clear(...)` before writing, because the gate-detail loop writes a variable number of rows.
- **`nz()` has no bool overload** — `nz(someBool[1], someBool)` fails with `CE10123` (`nz` expects a numeric `source`). To default a `bool`'s previous value, keep a second `var bool` and assign it from the first *before* the current bar's update, rather than reading `[1]`. `sma_prev_bull` in the SMA anchor does this; note that `[1]` on a `var bool` is `na` on bar 0, so an unguarded history read makes derived booleans `na` rather than `false`.
- **Arrays cannot nest** — `array<array<float>>` does not compile. To build a 2D/ragged structure (e.g. `TodSession` in the Time-of-Day RVOL feature), wrap the inner array in a user-defined `type` and use `array<TodSession>` instead.
- **Functions cannot reassign a value-type global variable** (`float`/`int`/`bool`/`string`), even one declared with `var` — attempting `pnl_entry_price := close` inside a `set_pnl_entry(dir) =>` function fails to compile with `CE10088 Cannot modify global variable`. Only reference types (`array`/`matrix`/`map`/UDT) can be mutated from inside a function, and only via a parameter (e.g. `add_result(array<bool> arr, ...)` calling `array.push(arr, ...)`). This is why the P&L entry/re-arm logic (`pnl_entry_price`, `pnl_direction`, `pnl_exit_fired`) stays inlined at each of the 8 signal branches instead of being factored into a helper function — wrapping that state in a UDT to allow a helper would touch every read site across the file.

---

## Compile-Check Checklist

No test runner exists. After any edit, verify manually in this order:

1. **Paste into Pine Editor → zero compilation errors** before proceeding
2. **Load NVDA 2-min chart** → confirm dashboard appears at bottom-right
3. **Score ≤ 100** on dashboard at all times (raw_score is capped, but confirm no overflow)
4. **Signal alternation:** let a BUY fire → confirm next BUY is blocked until a SELL fires, including across multiple trend flips where SELL's quality filter never clears (BUY → flip bear, no SELL → flip bull again → still no second BUY)
5. **No double-count on same-bar PROFIT + new signal:** when a PROFIT fires on the same bar as a new signal, confirm success rate increments by 1, not 2
6. **Gate enforcement:** toggle `Enable Risk Gates` ON → confirm score drops when gates are active; toggle OFF → score unchanged but warnings visible
7. **SMA mode:** anchor defaults to SMA → confirm the SMA line appears (not EMA9) with its grey ATR buffer band, flip circles appear where the latched state changes; switch to EMA Cross → confirm EMA9 line appears instead, flip circles at crossover bars
8. **Dashboard row count:** with Success Rate Tracking + all 4 gates active + Extended Metrics ON + ORB + VWAP Fade modules enabled, confirm no runtime error (row overflow guard working; `DASHBOARD_MAX_ROWS = 32`)
9. **RVOL mode:** toggle `Relative Volume Mode` between `Rolling 20-bar` and `Time-of-Day` on the same chart → confirm the Volume row's RVOL multiplier and mode label (`TOD` / `20-bar`) both change, and that a chart with less than `RVOL Lookback Sessions` of history shows `20-bar*` (fallback) instead of `na` or a stale value
10. **P&L Target:** with `Enable P&L Exit Signals` on, confirm PROFIT/LOSS fires at exactly `±P&L Target (%)` from `pnl_entry_price` and labels show the static target text (e.g. `PROFIT +1.0%`)
11. **Win rate:** with `Enable Success Rate Tracking` on, confirm the dashboard shows `BUY Win Rate`/`SELL Win Rate`; let a BUY hit PROFIT then get closed by the next SELL → confirm it counts as a win; let a BUY hit LOSS then get closed by the next SELL → confirm it counts as a loss; let a BUY get closed by the next SELL without ever hitting PROFIT or LOSS → confirm it also counts as a loss; confirm the currently open (not-yet-closed) signal is never counted
12. **Stale dashboard rows:** let a gate go active (gate-detail rows appear), then wait for it to clear → confirm the Gate Details rows disappear entirely rather than freezing on the last warning text
13. **`enablePnL` independence:** turn `Enable P&L Exit Signals` OFF with Success Rate Tracking ON → confirm PROFIT/LOSS labels and alerts stop, but BUY/SELL win rates keep resolving normally (they must NOT collapse to 0%)
14. **Session-open flatten:** with the RTH filter ON, hold a position into the close → confirm at the next 9:30 open the dashboard P&L reads `—` and no PROFIT/LOSS label fires off the overnight gap
15. **Anchor row:** with anchor = EMA Cross → confirm the first dashboard row reads `Trend (EMA9)` with the EMA9 price; switch to SMA → `Trend (SMA)` with the SMA price, and the Signal Anchor row reads e.g. `SMA (70) AUTO ±0.25A`
16. **SMA anchor:** switch anchor to SMA → confirm the SMA line plots (not EMA9) with a grey buffer band either side, flip circles appear only where the latched state changes, and the dashboard Trend row reads `Trend (SMA)`; set `SMA Buffer = 0` → confirm flips become noticeably more frequent (raw cross); set `SMA Buffer = 1.0` → confirm flips become rare and price must clear the band before the line changes color; confirm no flip ever fires on a bar where price sits inside the band
17. **Entry freshness:** in Score mode, set `Max Bars From Flip = 5` → confirm no BUY/SELL fires on bar 6+ after a flip (watch the "Bars From Flip" Extended Metrics row to confirm the count itself resets to 0 on each flip bar); confirm Trend mode is unaffected by the same setting; confirm worst case (Success Rate + Extended Metrics + all 4 gates + ORB + Fade on) still shows no dashboard row overflow with `DASHBOARD_MAX_ROWS = 32`
18. **Strategy router — no behavior change with all extra modules off:** with `orbEnabled = false` and `fadeEnabled = false`, the **Strategy** row reads `Trend Pullback · <regime>`, no ORB/Fade rows render, and signals fire on exactly the same bars as a Phase-1 build (spot-check BUY/SELL bars in both `sigTime` modes).
19. **ORB range + lock:** on an RTH chart (NVDA 2-min, filter ON), at the open the ORB row reads `forming …`; at `Opening Range (minutes)` past the open it flips to `armed H/L Q<n>` and the orange ORB High/Low lines appear and stop moving; confirm the lines break (don't connect) into the next session and a fresh range forms.
20. **ORB breakout fires once:** let price close beyond the locked high by `Breakout Buffer (ATR)` with `rel_vol ≥ Min RVOL` → confirm a `BUY … ⚡ORB` label fires, the Strategy row shows `ORB · …` on that bar, and the ORB row flips to `✅ fired this session`; confirm no second ORB entry that session even on a bigger break; confirm strict alternation still holds (next same-direction signal, ORB or Trend Pullback, blocked until a SELL).
21. **ORB needs RTH filter:** turn `RTH Session Filter` OFF with ORB enabled → confirm the ORB row reads `⚠️ needs RTH filter`, no ORB label ever fires, and Trend Pullback signals are unchanged.
22. **ORB off = pre-Phase-2:** set `Enable ORB Module` OFF → confirm no ORB row, no ORB lines, and identical signals to a Phase-1 build.
23. **VWAP Fade arm/reset:** enable the module on a ranging chart → teal band edges plot at `VWAP ± Fade Band × ATR`; when price pushes past an edge the Fade row reads `stretched ↑/↓ <n> ATR`; when price closes back through VWAP the row returns to `watching …` (arm cleared).
24. **VWAP Fade fires counter-trend, once per excursion:** in a `RANGE`/`EXHAUSTED` regime, after an up-excursion let a bearish bar close back inside the upper band with RSI ticking down → confirm a `SELL … 🔄FADE` label fires with the Strategy row showing `VWAP Fade · …`; confirm no second fade off the same excursion until price has reverted through VWAP and stretched again; confirm nothing fires while `regime` is `TREND`/`BREAKOUT` (row shows `idle (<regime>)`).
25. **VWAP Fade off = unchanged:** `Enable VWAP Fade Module` OFF (the default) → no Fade row, no bands, signals identical to a two-module (Phase 2) build.

---

## Version

The script file is `SLT.pine`. It was forked from **SuperLazyTrade V3** with the SuperTrend signal anchor removed entirely: the input option, the SuperTrend Engine settings group (`atrMode` / `atrLen_manual` / `factor_manual`), the adaptive ATR/factor selection block, the `ta.supertrend` call, the `st_*` anchor, its plots/fills/flip circles, and the dashboard branch. Anchor selection is now binary — SMA (default) or EMA Cross. The on-chart `indicator()` title is `SLT V1` (from `VERSION = "V1"`). No changelog history is tracked in the file — treat the current source as the reference behavior going forward.
