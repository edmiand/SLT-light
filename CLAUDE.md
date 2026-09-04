# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Pine Script v6 TradingView indicator** — a single-file intraday momentum trading system, **forked from SuperLazyTrade with the SuperTrend signal anchor removed**. The sole source file is `SLT.pine`. There is no build system, package manager, or test runner; development means editing the `.pine` file and pasting it into TradingView's Pine Editor to compile and validate.

SLT keeps two signal anchors: **SMA** (default) and **EMA Cross**. The 5-component scoring engine, the 4 risk gates, both P&L tracking systems, win-rate tracking, and the dashboard are carried over from the parent project.

**Light build — minimal input surface.** The settings panel exposes: **Signal Anchor** (EMA Cross / SMA), **Score Stars** (`sigQuality`), **P&L Exit** (`enablePnL` + `pnlTarget`), and a **Dashboard & Visuals** group (`showDash`, `extDash`). Every other parameter that was a user input in the parent script is either **auto-tuned** from the asset profile + anchor choice (see [Auto-Tune](#auto-tune-light-build) and the [Profile Parameters Table](#profile-parameters-table)) or **pinned** to its tuned default in a `FIXED BEHAVIOUR` block near the top of the file. The **VWAP Fade** module is removed entirely; the **ORB** module is retired but left inert (`orbEnabled = false` constant) so its router/dashboard/plot code still compiles. To change a tuned value, edit the profile `if/else` chain — not the panel.

## Development Workflow

1. Edit `SLT.pine`
2. Copy the full file contents
3. Open TradingView → Pine Editor → paste → Save → Add to Chart
4. Compilation errors appear immediately in the Pine Editor console
5. Test on a 2-minute chart with a liquid instrument (NVDA, TSLA, SPY, QQQ, SOXX, Silver Futures)

## Architecture

The script is organized into sequential sections (read top-to-bottom, order matters in Pine Script):

1. **Constants** — Gate penalty values, dashboard sizing (`DASHBOARD_MAX_ROWS = 36`)
2. **Inputs** — minimal surface (see [Auto-Tune](#auto-tune-light-build)):
   - `grp_anchor`: **Signal Anchor** (EMA Cross / SMA) — the mode selector; drives auto-tune
   - `grp_sig`: **Score Stars** (`sigQuality`) — label selectivity on top of the auto min score
   - `grp_pnl`: **P&L Exit** — `enablePnL` + `pnlTarget`
   - `grp_vis`: **Dashboard & Visuals** — `showDash`, `extDash`
   - `FIXED BEHAVIOUR` block: `showLabels`, `sigTime="Score"`, `gateOn=true`, `rvolMode="Time-of-Day"`, `rvolLookbackDays=10`, `enableSuccessRate`, `maxSignalsToTrack`, `showSignalPnL`, `showCircles=true`, `showBg=true` — all pinned constants, no inputs.
   - ORB inert-constant block: `orbEnabled=false` + the 7 ORB tuning constants (module code still present).
3. **Asset profile assignment** — A `switch` on `syminfo.type` first computes `asset_category` (STOCK/FUND/FUTURES/CRYPTO/MARKET); a separate `if/else if` chain keyed on `asset_category` then sets `profile_name`, all scoring threshold variables, the SMA anchor length `sma_len`, **and the four auto-tune fields** `ema_slow_auto` / `sma_buf_auto` / `min_score_auto` / `rth_auto`. Immediately after the chain, a **TIMEFRAME LENGTH SCALING** block defines `tf_scale`/`scale_len()` (see [Timeframe Length Scaling](#timeframe-length-scaling)), then an **AUTO-TUNE RESOLUTION** block maps the profile fields (plus the anchor choice) to the effective globals the rest of the script reads: `emaSlowPeriod` (= `scale_len(ema_slow_auto)`), `smaBufferATR`, `minScoreBuy`/`minScoreSell` (both = `min_score_auto`), `useSessionFilter` (= `rth_auto`), and `maxBarsFromFlip` (`anchorMode=="SMA" ? 0 : scale_len(10)`). See [Profile Parameters Table](#profile-parameters-table).
4. **Core indicator calculations** — EMAs (9, `emaSlow`=profile-auto 20 or 30, 20 fixed for scoring, 50), SMA anchor (`smaLenActive = sma_len`, no MANUAL mode), VWAP, ATR(14), RSI(14), ADX(14,14), relative volume (`rvolMode` pinned to Time-of-Day, Rolling 20-bar SMA is the automatic thin-history fallback — see [Time-of-Day Relative Volume](#key-design-decisions))
5. **Market regime detection** — Squeeze state (BB(20,2) inside KC(20,1.5×ATR)), squeeze release, stretch factor (EMA distance + trend move since flip), velocity override (ADX>35 rising 3 bars), ATR fuel gauge (session range vs daily ATR-14)
6. **Dual-anchor trend classification** — EMA Cross or SMA mode; `is_bull`/`is_bear`/`trendUp`/`trendDown` unify both anchors behind a single interface, and `anchor_line_price`/`anchor_short` unify the *display* side. `trend_start_price` updated on every `trendUp`/`trendDown`. See [SMA Anchor](#sma-anchor).
7. **Scoring engine** — 5 components summed to `raw_score` (capped at 100). Max theoretical total = 105 across all profiles. See [Scoring Components](#scoring-components).
8. **Risk gates** — 4 gates calculate penalties; applied to `raw_score` → `final_score` only when `gateOn = true`; always shown as warnings regardless. See [Risk Gates](#risk-gates).
9. **Signal generation** — Three sub-stages (regime-router refactor; light build runs **one live module**):
   - **Strategy modules** — each is fully self-gated and exposes `<name>_long` / `<name>_short` / `<name>_quality`:
     - **Trend Pullback** (`tp_*`) — the anchor + 5-component confluence + `passes_quality_filter` logic. `sigTime` is pinned to `"Score"` (fires off `is_bull`/`is_bear` + `entry_is_fresh` + explicit `barstate.isconfirmed`); `"Trend"` mode code is still present but unreachable. Both modes AND in `in_session` and `anchor_ready`. `tp_quality = final_score`.
     - **ORB** (`orb_*`) — Opening Range Breakout, **inert** (`orbEnabled = false`). Code retained. See [ORB Module](#orb-module).
     - ~~**VWAP Fade**~~ — removed entirely from the light build.
   - **Regime router** — `active_strategy` / `strat_long` / `strat_short` / `strat_quality`, last-write-wins: **Trend Pullback (default) → ORB**. With ORB inert, the router always resolves to Trend Pullback. The router only chooses between modules — it never loosens a module's own gating.
   - **Signal dispatch** — `signal_buy = strat_long and not b_fired`, `signal_sell = strat_short and not s_fired`. `b_fired`/`s_fired` enforce strict BUY/SELL alternation **across all modules and any number of regime/trend flips**; they clear only when the opposite signal fires or at RTH session open (`useSessionFilter`-gated) — **not** on a trend-direction change by itself.
   Signals are non-repainting; `barstate.isconfirmed` guards every flip/breakout condition. `stars` (BUY/SELL chart labels) is computed just after the router off `strat_quality`, so it reflects the **active** module. RTH filter via `time(timeframe.period, "0930-1600:23456")` — `in_session`/`session_just_opened` are computed in their own **RTH Session Window** section placed *before* P&L exit tracking, because the session-open reset must flatten stale position state before `current_pnl` is evaluated on that same bar. Score-mode signals can also be constrained to fire only within `maxBarsFromFlip` bars of the anchor flip (default 0 = disabled); Trend mode is unaffected.
10. **P&L tracking** — Two independent systems: (a) live dashboard P&L using `pnl_entry_price`/`pnl_direction`, reset on every new signal and flattened at RTH session open; (b) signal-to-signal `signal_pnl` history shown on labels. PROFIT/LOSS exit triggers are Fixed %-only: `current_pnl` vs `±pnlTarget`. Target/stop resolution (`target_reached`/`stop_reached`) is tracked internally regardless of `enablePnL`; `enablePnL` only gates the visible `signal_profit`/`signal_loss`.
11. **Success rate tracking** — Rolling arrays (`buy_results`/`sell_results`), capped at `maxSignalsToTrack`; every signal is scored won/lost when the next opposite-direction signal closes it — won only if the target was reached during the entry, lost otherwise. **Phase 4:** the same result is *also* pushed into a per-module store (`mod_results`, a 3-slot `array<ModResults>` indexed by `strat_id()`), keyed by the module that **opened** the entry (`buy_entry_strategy`/`sell_entry_strategy`, stamped from `active_strategy` at open). Overall arrays stay module-blind. The dashboard shows the per-module breakdown only in a multi-module build; the light build (one live module) shows just the overall BUY/SELL rates.
12. **Visuals** — Conditional plots, one branch per anchor: EMA9 dynamic line (green/red), or SMA line (green/red) plus its grey ATR buffer band; flip circles at transition bars; BUY/SELL labels anchored to `anchor_line_price` (with a `⚡ORB` tag if ORB were ever active). The locked **ORB high/low** plot (orange, `plot.style_linebr`) is retained but gated on `orbShowLevels = false`, so it never draws. The VWAP fade band plots are deleted.
13. **Dashboard** — `table.new` at `position.bottom_right` with `DASHBOARD_MAX_ROWS = 36`; rendered only on `barstate.islast`. The table is `var`, so every render starts with `table.clear(d, 0, 0, 1, DASHBOARD_MAX_ROWS - 1)`. The gate detail loop is the last section written; it has an explicit `if row >= DASHBOARD_MAX_ROWS: break` guard because it is the only variable-length section. The first data row reports the **active anchor** (`Trend (EMA9)` or `Trend (SMA)`) and its price. The **Signal Anchor** row shows the resolved auto values, e.g. `SMA (90) auto ±0.20A` or `EMA Cross (9/30) auto`. (The **Strategy** row — `active_strategy · <regime>` — was removed; `active_strategy` is always `Trend Pullback` in the light build and the `regime` descriptor had no other consumer, so both it and `sqz_bias` are deleted.) The **ORB** row renders only `if orbEnabled` → never. The **VWAP Fade** row is deleted. Under the BUY/SELL Win Rate rows, the **per-module win-rate** loop draws one row per module **only in a multi-module build** (`multi_module = orbEnabled`) — with just Trend Pullback live it would restate the BUY/SELL Win Rate rows, so it is suppressed. The `mod_results` / `record_module` plumbing keeps running, so the rows return intact when a second module goes live; they are then styled to match the BUY/SELL Win Rate rows (white left-aligned label, value `B <r>% <w>/<t>   S <r>% <w>/<t>` colour-coded lime/yellow/red by the module's combined rate, grey until resolved). `dashboard_score` reads `strat_quality`. `DASHBOARD_MAX_ROWS` stays at 36 (headroom; the light build renders well under that).
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
- **Direction is the selected anchor** (`is_bull`/`is_bear`). Component 1 measures how well the EMA cascade confirms the anchor's direction — it is a directional confluence term. (A short-lived experiment keyed it off `ema9` vs `ema20` to make scores numerically comparable across anchors; reverted because it made the term non-directional — see [SMA Anchor](#sma-anchor) → Scoring interaction.)

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

**Component 5 — Momentum Confluence (2 sub-components, max 20):**
- 5A RSI (12pts): regime-aware — bull wants RSI >60 rising (12) / RSI 45–60 (8) / stalling >60 not rising (3); bear is the mirror
- 5B Squeeze Release (8pts): sqz_release + RVOL >1.5 → 8pts; release alone → 3pts
- **MACD sub-component removed** (was 8pts, "MACD line same sign as trend direction"). MACD-line sign ≈ `EMA12 > EMA26`, a near-duplicate of Component 1's EMA cascade, so it carried almost no independent signal. Its 8 points were redistributed proportionally (old 7:5 split) across the two survivors: RSI 7→12, Squeeze 5→8. Component 5 max stays 20; the 105 pre-cap total is unchanged. `ta.macd` call and `macd_aligned`/`macd_pts` are deleted from `SLT.pine`.

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
| 2 — Stretch Factor | Unified: MAX(EMA stretch, trend stretch) > `ext_scale` AND no velocity override | −10 or −20 | `STRETCH_MODERATE/EXTREME_PENALTY` |
| 3 — ATR Fuel | Session range > `fuel_max`% of daily ATR-14 AND no velocity override | −25 | `FUEL_PENALTY` |
| 4 — Liquidity | RVOL < `rvol_gate` AND ADX < 90% of `adx_min` | −25 | `LIQUIDITY_PENALTY` |

**Stretch Factor detail:** `stretch_factor = max(ema_stretch, trend_stretch)` where `ema_stretch = |close - ema20| / ATR(14)` and `trend_stretch = |close - trend_start_price| / ATR(14)`. Moderate penalty when `> ext_scale`; extreme when `> ext_scale × 1.5`. **Velocity override** (`is_high_velocity` = `ADX > 35` rising 3 bars) waives the Gate 2 *and* Gate 3 penalties — a genuine high-conviction momentum regime is allowed to run past the standard extension band. The raw stretch tier (`stretch_penalty`/`stretch_status`) is still computed and shown as advisory in the gate message (`gate2_raw`), annotated `🔥 VELOCITY` instead of a `-N`; only the `final_score` subtraction and the dashboard `active_gate_count` are suppressed (`gate2_stretched = gate2_raw and not is_high_velocity`).

---

## SMA Anchor

The default `anchorMode` option (the other is EMA Cross). Selected when `anchorMode = "SMA"`.

**Length** (`sma_len`) is assigned per asset profile in the profile block — see the `sma_len` column in the [Profile Parameters Table](#profile-parameters-table). In the light build there is no MANUAL mode: `smaLenActive = sma_len` directly.

**Flip semantics — ATR buffer band with latched state.** A raw `ta.crossover(close, sma)` whipsaws badly on a 2-min chart, so the raw cross is not usable as-is. Instead:

```
sma_band = smaBufferATR × ATR(14)          // smaBufferATR = sma_buf_auto (0.20–0.35 by profile)
close > smaAnchor + sma_band  → latch BULL
close < smaAnchor - sma_band  → latch BEAR
inside the band               → hold previous state
```

`sma_state_bull` is a `var bool` initialized `true`, updated only under `barstate.isconfirmed` (so the state cannot flip intrabar and revert — same non-repainting guarantee as the EMA Cross anchor) and guarded by `not na(smaAnchor) and not na(sma_band)` for the warmup bars before the SMA is valid.

**Why latched rather than a true neutral zone:** several downstream consumers are written as `is_bull ? … : …` with no third case (dashboard trend row, anchor line color). If `is_bull` and `is_bear` were both `false` inside the band, those sites would silently render bearish. Latching preserves the strictly-binary `is_bull`/`is_bear` contract that EMA Cross also satisfies. `sma_is_bear` is defined as `not sma_state_bull`, never as an independent test.

Flips derive from the latch by comparing against `sma_prev_bull`, a second `var bool` assigned from `sma_state_bull` *before* the update block each bar. This is deliberately not `sma_state_bull[1]`: that reads `na` on bar 0, making the flip booleans `na` rather than `false`, and `nz()` has no bool overload to default it with (`CE10123` — `nz` expects a numeric `source`). Because flips come from the latch, `trend_start_price` and `bars_since_flip` stay correct with no extra work, and `maxBarsFromFlip` applies to SMA mode in Score mode exactly as it does to EMA Cross.

(`sma_buf_auto` is always > 0 in the light build; setting a profile value to 0 would degrade to a raw price/SMA cross with substantially more flips.)

**Scoring interaction — be aware of it (not corrected):** Component 1 (EMA Cascade) conceptually overlaps the SMA anchor — both measure price against a slow average. In SMA mode `is_bull` (price above a 70–120-bar SMA) holds through pullbacks that would already have flipped an EMA-cross anchor, so Component 1 keeps awarding PARTIAL/WEAK points during choppy continuation where EMA-cross mode would score the bar 0 or as the opposite side. Net: **SMA-mode `raw_score` skews slightly higher on trend continuation** — factor this in when comparing win rates across the two anchors; the score distributions are not strictly comparable.

An experiment (commit `c0d85a1`, reverted) keyed Component 1's bull/bear branch off `ema9` vs `ema20` instead of `is_bull`/`is_bear` to normalize this. It backfired: `t_pts` is added to `raw_score` unconditionally with no re-check against the signal's direction, so decoupling made Component 1 a non-directional term — it suppressed with-trend pullback BUYs (PARTIAL→COUNTER-TREND) and added full-cascade points to counter-trend SELLs when the EMAs were bullish. If normalization is revisited, do it by shifting a few points from `ema_max` to `vwap_max` in SMA mode (VWAP/price is genuinely independent of a price/SMA anchor), never by changing the direction selector.

---

## ORB Module

> **Light build status: INERT.** `orbEnabled` is a constant `false` (not an input). The module code below, its dashboard row (`if orbEnabled`), and its plots (`orbShowLevels = false`) all remain in the file and compile, but nothing activates. The 7 former ORB inputs are kept as constants so those references resolve. Re-enabling means turning the constant back into an input. The description below documents behaviour *if re-enabled*.

Second strategy module (Phase 2 of the regime-router refactor). Lives in the STRATEGY MODULES section between Trend Pullback and the router.

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

**Not modelled yet:** failed-breakout / reversal entries (only the first break each session is taken), and a separate ORB stop/target (P&L exits are still the global Fixed-% system).

---

## VWAP Fade Module

> **Removed in the light build.** The module block, its router branch, its two band plots, its dashboard row, and its inputs (`grp_fade`) are deleted from `SLT.pine`. The `regime` string that gated it was later deleted too (it had no consumer once the dashboard Strategy row was removed). `mod_results` still has 3 slots and `strat_id("VWAP Fade") → 2` still maps, but nothing ever writes slot 2 — left in place to avoid renumbering the ORB slot. The dispatch/P&L/win-rate machinery was module-agnostic, so removing Fade required no changes there. History for a re-add: it was a counter-trend mean-reversion module gated on `regime ∈ {RANGE, EXHAUSTED} and adx < 1.5×adx_min`, sitting above Trend Pullback in the router.

---

## Profile Parameters Table

All threshold variables are assigned once per bar based on `syminfo.type`. The `switch` expression maps every possible `syminfo.type` value: `"stock"` → STOCK, `"fund"` → FUND, `"futures"` → FUTURES, `"crypto"` → CRYPTO, and **everything else** (index, forex, unknown) → MARKET via the switch default. The MARKET profile is the true catch-all; there is no separate DEFAULT profile in the code.

**Scoring / gate thresholds:**

| Profile | `vwap_h_limit` | `vwap_e_limit` | `adx_min` | `rvol_gate` | `ext_scale` | `fuel_max` | `ema_max` | `vwap_max` | `sma_len` |
|---------|----------------|----------------|-----------|-------------|-------------|------------|-----------|------------|-----------|
| MARKET INDEX 🏦 *(index, forex, unknown)* | 0.7% | 1.5% | 25 | 1.0× | 2.0 ATR | 80% | 22 | 23 | 90 |
| ETF / FUND 📊 | 1.0% | 2.0% | 18 | 1.0× | 2.2 ATR | 75% | 20 | 25 | 90 |
| STOCK 🚀 | 1.8% | 3.5% | 20 | 1.2× | 3.0 ATR | 85% | 20 | 25 | 70 |
| FUTURES ⚡ | 1.5% | 3.0% | 15 | 0.6× | 2.5 ATR | 90% | 30 | 15 | 70 |
| CRYPTO 🪙 | 3.0% | 6.0% | 28 | 0.6× | 4.0 ATR | 95% | 25 | 20 | 120 |

**Auto-tune fields** (light build — set in the same `if/else` chain, resolved right after it):

| Profile | `ema_slow_auto` | `sma_buf_auto` | `min_score_auto` | `rth_auto` |
|---------|-----------------|----------------|------------------|------------|
| MARKET INDEX 🏦 | 30 | 0.20 | 55 | `true` |
| ETF / FUND 📊 | 30 | 0.20 | 50 | `true` |
| STOCK 🚀 | 20 | 0.25 | 50 | `true` |
| FUTURES ⚡ | 30 | 0.30 | 45 | `false` |
| CRYPTO 🪙 | 20 | 0.35 | 55 | `false` |

`emaSlowPeriod = ema_slow_auto`; `smaBufferATR = sma_buf_auto`; `minScoreBuy = minScoreSell = min_score_auto`; `useSessionFilter = rth_auto`. `maxBarsFromFlip` is not a profile field — it's `anchorMode == "SMA" ? 0 : 10`, applied in the resolution block.

---

## Auto-Tune (light build)

The parent script exposed ~30 inputs. The light build keeps a handful — Anchor, Score Stars, P&L Exit, and two Dashboard & Visuals toggles (`showDash`, `extDash`) — and derives or pins the rest.

**Derived from the asset profile + anchor** — resolved in the `AUTO-TUNE RESOLUTION` block immediately after the profile `if/else` chain:

| Effective global | Source | Notes |
|---|---|---|
| `emaSlowPeriod` | `ema_slow_auto` (profile) | 20 for fast movers (STOCK/CRYPTO), 30 for grinders. Pins to `ema20` for Component 1 regardless. |
| `smaBufferATR` | `sma_buf_auto` (profile) | Wider latch band on choppier / higher-vol profiles. |
| `minScoreBuy` / `minScoreSell` | `min_score_auto` (profile), single value both sides | 45 (FUTURES) … 55 (MARKET/CRYPTO). The [Component 1 normalization](#scoring-components) is what lets one value serve both anchors. |
| `useSessionFilter` | `rth_auto` (profile) | `false` for FUTURES + CRYPTO — the RTH filter would black out most of a 24h instrument's day. |
| `maxBarsFromFlip` | `anchorMode == "SMA" ? 0 : 10` | EMA9/slow crosses lag the turn → cap entry distance; the SMA latch already fires on the flip bar, so no cap. **This is the one parameter that changes when you switch anchors** — keep it in mind for cross-anchor win-rate comparison. |

**Pinned constants** (`FIXED BEHAVIOUR` block, top of file): `showLabels=true`, `sigTime="Score"`, `gateOn=true`, `rvolMode="Time-of-Day"`, `rvolLookbackDays=10`, `enableSuccessRate=true`, `maxSignalsToTrack=50`, `showSignalPnL=true`, `showCircles=true`, `showBg=true`. (`showDash` / `extDash` are inputs in `grp_vis`.)

**Retired modules:** `orbEnabled=false` + 7 ORB tuning constants kept so the (still-present) ORB module, its dashboard row, and its plots compile without ever activating. The VWAP Fade module, its router branch, its plots, its dashboard row, and its inputs are deleted; `mod_results` slot 2 and the `strat_id("VWAP Fade")→2` mapping are left as harmless vestiges to avoid renumbering.

**To hand-tune:** edit the profile `if/else` chain (or the resolution block for `maxBarsFromFlip`). There is no in-panel override — re-adding an input is a one-line change if experimentation demands it.

---

## Timeframe Length Scaling

Every bar-count lookback in the script (EMA9/20/50, the profile SMA length, ATR/RSI/ADX(14), the squeeze BB/KC(20) lengths, the 20-bar RVOL fallback, `maxBarsFromFlip`) was tuned to cover a specific amount of real time on a **2-minute chart** (the tested interval — see [Development Workflow](#development-workflow)). A fixed bar count covers a different amount of wall-clock time on any other interval — e.g. the STOCK profile's `sma_len=70` covers ~2.3h on a 2-min chart but ~5.8h on a 5-min chart if left unscaled.

`tf_scale = BASE_TF_SECONDS / timeframe.in_seconds()` (`BASE_TF_SECONDS = 120`, the 2-min reference) rescales every such length via `scale_len(base_len) => math.max(1, math.round(base_len * tf_scale))`, so each one keeps covering the same wall-clock window regardless of chart interval. On a 2-min chart `tf_scale = 1.0` exactly, so behavior is byte-for-byte unchanged from before this feature — 2-min remains the reference calibration and the only interval this script's thresholds are actually validated against. Non-intrabar chart types (Range/Tick/Renko) report `timeframe.in_seconds() == 0`; `tf_scale` falls back to `1.0` (unscaled) in that case, since there's no wall-clock equivalent to scale against.

**Scaled:** `ema9Period` (= `scale_len(9)` — kept as a named variable rather than inlined so the dashboard's `EMA Cross (n/…)` label reflects the real period instead of a hardcoded "9"), `emaSlowPeriod` (= `scale_len(ema_slow_auto)`), `ema50`, `ema20` (Component 1 scoring), `smaLenActive` (= `scale_len(sma_len)`), the `atr`/`rsiVal`/`adx` 14-periods, the squeeze `ta.bb`/`ta.atr` 20-periods, the `vol_ma20` RVOL fallback 20-period, and `maxBarsFromFlip` (`scale_len(10)` in EMA Cross mode; still `0` — unlimited — in SMA mode).

**Not scaled (deliberately):** the profile threshold/percentage values (`vwap_h_limit`, `vwap_e_limit`, `adx_min`, `ext_scale`, `fuel_max`, the RVOL point tiers) — these were tuned against 2-min bar noise/volatility characteristics and have no comparable scaling law, so switching interval still means re-validating those by eye; they are not addressed by this feature. Also left alone: `rvolLookbackDays` / `maxSignalsToTrack` (session/signal counts, not bar-lookback windows) and the 2–3 bar `ta.rising()` momentum checks (`adx_rising_2`, `adx_rising_3`, `rsi_rising`) — at that length, scaling either collapses to a meaningless 1-bar check or adds complexity for no real calibration benefit. The daily ATR (`atr_14`, `request.security(..., "D", ...)`) is unaffected by chart interval by construction.

`scale_len()`'s output is `series int` (computed from `timeframe.in_seconds()` inside variables reassigned every bar, so it's series-qualified even though constant for the whole chart run) — Pine v6 accepts `series int` length arguments for `ta.ema`/`ta.sma`/`ta.atr`/`ta.rsi`/`ta.dmi`/`ta.bb`, the same relaxation the profile-driven `smaLenActive`/`emaSlowPeriod` already relied on before this feature existed.

---

## Key Design Decisions

**Profile-adaptive scoring:** `ema_max` and `vwap_max` vary by asset type (e.g., FUTURES gets `ema_max=30`, `vwap_max=15`), so the raw 100-point maximum is assembled differently per profile. All profiles sum to 105 before the cap. When modifying component weights, check all five profiles.

**Strategy router (light build):** Signal generation is still structured as *strategy modules → router → dispatch*, but only **Trend Pullback** (`tp_*`, the anchor + confluence logic) is live. **ORB** (`orb_*`) code is present but inert (`orbEnabled = false`); **VWAP Fade** is deleted. The router (`active_strategy`/`strat_long`/`strat_short`/`strat_quality`) resolves last-write-wins Trend Pullback → ORB, so with ORB off it always yields Trend Pullback. (The descriptive `regime` classification sub-stage and its dashboard Strategy row were removed — nothing consumed `regime` after VWAP Fade.) Every downstream consumer (P&L, win-rate, labels, alerts, dashboard) still fires off the dispatched `signal_buy`/`signal_sell` / the `strat_*` surface, so the module machinery is intact. Per-module win-rate (`mod_results`) still exists; only slot 0 (Trend Pull) ever fills. **When re-adding a module:** expose `<name>_long`/`<name>_short`/`<name>_quality`, keep it fully self-gated, register it in the router (opposite-stance modules go *above* Trend Pullback and must carry a regime gate), let dispatch apply alternation, and wire a `strat_id()` slot + dashboard row. The `sigTime="Trend"` path in Trend Pullback is also dead code (pinned to `"Score"`).

**Dual-anchor unification:** After anchor selection, all downstream logic uses `is_bull`, `is_bear`, `trendUp`, `trendDown` — never `ema_*` or `sma_*` directly. The scoring engine, all 4 risk gates, signal generation, and both P&L systems are anchor-agnostic through this interface. `maxBarsFromFlip` is the deliberate exception: it resolves to `anchorMode == "SMA" ? 0 : 10`. The EMA slow period (`emaSlowPeriod = ema_slow_auto`, 20 or 30 by profile) affects cross detection but NOT Component 1 scoring, which always uses `ema20`. Component 1's bull/bear branch is selected by `is_bull`/`is_bear` like the other directional components (a `c1_bull`/`c1_bear` decoupling was tried and reverted — see [SMA Anchor](#sma-anchor)). Only the Score-mode signal branch is live (`sigTime` pinned to `"Score"`); the Trend-mode branch is retained but unreachable.

The anchor selectors are **binary ternary chains** (`anchorMode == "SMA" ? … : …`), every one ending in the EMA Cross branch — so any `anchorMode` value other than `"SMA"` resolves to EMA Cross behavior rather than erroring. A new anchor would have to be added to all four selectors plus `anchor_line_price`, `anchor_short`, and `anchor_ready` explicitly. Only the *drawing* sites (`plot`, `plotshape`, `fill`, dashboard `anchor_display`) use explicit `anchorMode == "..."` equality, so an unhandled mode draws no anchor line at all — which is the symptom to look for.

**`anchor_ready` — anchor warm-up guard:** `ta.sma` returns `na` until `smaLenActive` bars exist (70–120 by profile), and `sma_state_bull` is initialized `true` with its update block skipped while `smaAnchor` is `na`. The SMA anchor therefore reports `is_bull = true` for its whole warm-up window regardless of price direction. `anchor_ready` (`not na(smaAnchor)` in SMA mode, `true` otherwise) is AND'd into both the Trend-mode and Score-mode signal branches to block that. It matters most in the default configuration — Score timing with `maxBarsFromFlip = 0` — where nothing else would stop a warm-up BUY from firing on score alone in a downtrend, seeding the win-rate array and drawing its label at a `na` price. Both EMAs produce usable values from the first bars, so EMA Cross mode passes `anchor_ready` unconditionally.

**Non-repainting:** *Signals* are non-repainting — `barstate.isconfirmed` guards every flip condition and all signal assignments. The daily ATR (`atr_14`) uses `request.security(..., lookahead_off)`, which reads the *developing* daily bar, so `fuel_used` drifts upward through the session. This is **not** display-only — it feeds Gate 3 (ATR Fuel) and therefore `final_score` and signal generation. It is still non-repainting: `lookahead_off` guarantees the value is computed from data available at that bar, so a historical bar recomputes to the same number it had live.

**Gate vs advisory mode:** Setting `gateOn = true` (default/recommended) subtracts gate penalties from `final_score`, directly suppressing low-ADX/range-bound chop entries (Gates 1 and 4 in particular). Setting `gateOn = false` shows gate warnings in the dashboard but does NOT subtract from `final_score`. Component 5B (Squeeze Release) is always active regardless of gate mode, to keep scoring consistent.

**Signal blocking (strict alternation):** `b_fired`/`s_fired` block a same-direction signal from firing again — in both "Trend" and "Score" signal-timing modes — until the opposite signal actually fires, no matter how many trend flips happen in between. E.g. BUY fires, trend flips bear but SELL's score/quality filter never clears, trend flips bull again → BUY stays blocked. The flags do **not** reset on a trend direction change by itself (`is_bull` vs `is_bull[1]`); they only clear when `signal_buy`/`signal_sell` actually fires (`b_fired := true, s_fired := false` and vice versa) or at RTH session open (`useSessionFilter`-gated, so 24h instruments with the filter off never get a daily reset). Live P&L tracking (`pnl_entry_price`, dashboard P&L, signal-to-signal P&L history) does not consult these flags and keeps accumulating across the held direction regardless.

**RTH filter on 24h instruments:** `time(timeframe.period, "0930-1600:23456")` always evaluates against ET hours regardless of instrument type. In the light build `useSessionFilter = rth_auto`, which the profile block sets to `false` for FUTURES and CRYPTO — so the 24h blackout hazard is handled automatically. The MARKET profile (index / forex / unknown) still resolves `rth_auto = true`; a 24h forex or index-future symbol that lands in MARKET would get the RTH blackout. If that comes up, set `rth_auto := false` in the MARKET branch or add a dedicated profile.

**Entry freshness constraint:** `maxBarsFromFlip` is auto-set from the anchor: `anchorMode == "SMA" ? 0 : 10` (SMA latches on the flip bar so it needs no cap; EMA9/slow crosses lag the turn so entries are capped at 10 bars past the flip). It constrains Score-mode entries via `entry_is_fresh = maxBarsFromFlip <= 0 or bars_since_flip <= maxBarsFromFlip`. `bars_since_flip` (`var int`) resets to 0 on the flip bar and increments after, in sync with the `trend_start_price` reset. This is independent of (and complements) the Stretch gate. Calibrate a profile-specific value by editing the resolution-block ternary; observe via the Extended Metrics "Bars From Flip" row.

**Two P&L tracking systems (independent):**
- *Live dashboard P&L* — tracks position from `pnl_entry_price`, reset on every new `signal_buy`/`signal_sell`. Since strict alternation (see Signal Blocking above) means there's only ever one signal per held direction, entry always corresponds to the signal that opened the current position — a separate "First Signal" reset-on-direction-change mode is no longer meaningful and was removed.
- *Signal P&L History* — shown on BUY/SELL labels; calculates `%` move from `last_signal_price` to current close using the previous signal's direction. Completely separate state from dashboard P&L.

**P&L exits are Fixed %-only:** `target_reached`/`stop_reached` fire when `current_pnl` (computed from `pnl_entry_price`) crosses `±pnlTarget`, once per entry, on a confirmed bar. `signal_profit`/`signal_loss` are those flags AND `enablePnL`, and drive only the labels and alerts.

**`enablePnL` is display-only — it must not gate resolution:** win-rate scoring reads `target_reached`, *not* `signal_profit`. If it read `signal_profit`, turning off "Enable P&L Exit Signals" (presented purely as a label/alert toggle) would make every tracked entry resolve as a loss and pin both win rates at 0% with no warning. Keep any future outcome/statistics logic on `target_reached`/`stop_reached`.

**Position state is flattened at RTH session open:** when `session_just_opened`, `pnl_entry_price`/`pnl_direction`/`pnl_exit_fired`/`pnl_exit_type` are all reset, matching the win-rate tracker which already closes any carried-over entry at that boundary. Without this, yesterday's entry price survives into the new session and an opening gap can immediately fire a PROFIT or LOSS label *and alert* against a position the tracker already considers closed. This is why the RTH Session Window section sits *above* P&L exit tracking in the file — the reset has to land before `current_pnl` is computed on the session-open bar. `useSessionFilter`-gated, so 24h instruments running with the filter off never flatten.

**Win-rate tracking — every signal, scored on close:** `buy_results`/`sell_results` count every BUY/SELL signal exactly once, not just ones that hit a PROFIT/LOSS target. An entry's outcome isn't known until it's closed by the next opposite-direction signal (per strict alternation, there's exactly one open position per direction at a time), so `buy_won_this_entry`/`sell_won_this_entry` (`var bool`) accumulate whether `target_reached` fired at any point during the entry; when the opposite signal fires, that flag is pushed into the results array (`true` = won, `false` = everything else — a `stop_reached` exit, or the position simply being reversed with no target ever hit) and then reset for the new entry. `buy_entry_open`/`sell_entry_open` (`var bool`) guard against scoring before any entry has actually opened. Each results array is an independent rolling `array<bool>` capped at `maxSignalsToTrack` via FIFO (`array.shift` on overflow). `calc_win_rate(arr, enabled) => [wins, total, rate]` computes stats for every array from one shared read-only function. The currently-open entry is never counted until it closes — the win rate reflects only resolved signals.

**Per-module win-rate (Phase 4):** every place that pushes into `buy_results`/`sell_results` — the two closes in *UPDATE SUCCESS RATE TRACKING* and the session-open carry-over close — also calls `record_module(mod_results, strat_id(<dir>_entry_strategy), is_buy, won, cap)`. `mod_results` is a `var array<ModResults>` (UDT wrapping `array<bool> buys` / `array<bool> sells`, because Pine arrays can't nest — same trick as `TodSession`), populated once with 3 slots. `strat_id()` still maps `0` Trend Pullback / `1` ORB / `2` VWAP Fade (anything unknown → 0) — the ORB and Fade mappings are vestigial in the light build. `<dir>_entry_strategy` (`var string`) is stamped from `active_strategy` at the moment the entry opens. `record_module` mutates the UDT's inner array in place (reference type reachable through the `store` parameter). The per-module dashboard rows are gated on `multi_module` (`= orbEnabled`) — in the single-module light build they are suppressed entirely because they would duplicate the BUY/SELL Win Rate rows. `record_module` still runs, so slot 0 accumulates and the rows would render correctly the moment a second module is live; when shown they match the BUY/SELL Win Rate style (white label, value colour-coded by the module's combined rate via `calc_win_rate` on `mr.buys` + `mr.sells`).

**Dashboard row budget:** `DASHBOARD_MAX_ROWS = 36` (rows 0–35). The `28→30→32→36` bumps came from the router phases (optional ORB row, Fade row, up to 3 per-module rows). The light build deleted the Fade row, made the ORB row unreachable, and suppresses the per-module rows in single-module mode, so the live worst case is well under 36. The gate detail loop is the last section; it has an explicit `if row >= DASHBOARD_MAX_ROWS: break` guard because it is the only variable-length section (all other rows are fixed-count by design). Success Rate Tracking occupies 3 fixed rows (separator + BUY Win Rate + SELL Win Rate), regardless of Extended Metrics. The constant stays at 36 as slack for re-adding a module (which brings its per-module rows back).

**VWAP grace zone asymmetry:** Bull grace zone tolerance = `vwap_h_limit × 0.2`; bear grace zone = `vwap_h_limit × 0.1`. Bears are held to half the tolerance — intentional, reflecting that short-side entries near VWAP carry more reversion risk.

**Time-of-Day Relative Volume:** `rel_vol` (used by Component 3, Gate 4 liquidity, and Component 5B squeeze-release) is `rvolMode`-dependent. Default `"Time-of-Day"` compares current bar volume to the average volume of the *same bar-slot* (bars since session open) across the prior `rvolLookbackDays` completed sessions — this removes the U-shaped intraday volume bias that a trailing SMA has (overstates RVOL near the open, understates it at lunch). History is kept in `array<TodSession>`, where `TodSession` is a UDT wrapping `array<float>` — Pine cannot nest arrays directly (`array<array<float>>` is not supported), so a UDT wrapper is the standard workaround. Session boundaries reuse the existing `is_new_session` (calendar-day change) detector, so the same extended-hours caveat applies as the ATR Fuel gauge. Falls back to the `ta.sma(volume, 20)` method (`"Rolling 20-bar"`) when fewer than 3 historical sessions exist at a slot, or when the user selects Rolling 20-bar explicitly. The active mode is shown inline in the dashboard's Volume row (`TOD`, `20-bar`, or `20-bar*` for an in-flight TOD→fallback).

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
- **Arrays cannot nest** — `array<array<float>>` does not compile. To build a 2D/ragged structure (`TodSession` for Time-of-Day RVOL, `ModResults` for per-module win-rate), wrap the inner array in a user-defined `type` and use `array<TodSession>` / `array<ModResults>` instead. A function *can* mutate such an inner array in place (`array.push`/`array.shift` on `array.get(store, i).field`) — reference types reached through a parameter are fair game; only reassigning a value-type global is forbidden.
- **Functions cannot reassign a value-type global variable** (`float`/`int`/`bool`/`string`), even one declared with `var` — attempting `pnl_entry_price := close` inside a `set_pnl_entry(dir) =>` function fails to compile with `CE10088 Cannot modify global variable`. Only reference types (`array`/`matrix`/`map`/UDT) can be mutated from inside a function, and only via a parameter (e.g. `add_result(array<bool> arr, ...)` calling `array.push(arr, ...)`). This is why the P&L entry/re-arm logic (`pnl_entry_price`, `pnl_direction`, `pnl_exit_fired`) stays inlined at each of the 8 signal branches instead of being factored into a helper function — wrapping that state in a UDT to allow a helper would touch every read site across the file.

---

## Compile-Check Checklist

No test runner exists. After any edit, verify manually in this order:

1. **Paste into Pine Editor → zero compilation errors** before proceeding
2. **Settings panel shows only:** Signal Anchor, Score Stars, P&L Exit (Enable + Target %), and Dashboard & Visuals (Show Dashboard + Extended Metrics). No SMA / Signal-timing / RVOL / ORB / Fade / Success groups; no flip-circle / background toggles.
3. **Load NVDA 2-min chart** → dashboard appears bottom-right; the **Signal Anchor** row reads `SMA (70) auto ±0.25A` (NVDA = STOCK profile). No **Strategy** row.
4. **Score ≤ 100** on the dashboard at all times; confirm the 105-max component sum never overflows the clamp.
5. **Signal alternation:** let a BUY fire → next BUY blocked until a SELL fires, across multiple trend flips where SELL's quality filter never clears.
6. **No double-count on same-bar PROFIT + new signal:** success rate increments by 1, not 2.
7. **Gates always enforced:** `gateOn` is a pinned constant `true` — confirm the dashboard shows `Gates` (not `Risks`) and that an active gate drops `final_score`.
8. **Anchor swap:** default SMA → SMA line + grey buffer band, flip circles at latched-state changes, Trend row `Trend (SMA)`, Signal Anchor row `… auto ±<buf>A`. Switch to EMA Cross → EMA9 line instead, flip circles at crossover bars, Signal Anchor row `EMA Cross (9/<20|30>) auto`, and `Max Bars From Flip` behaviour changes (see 12).
9. **Dashboard row budget:** worst case (Extended Metrics ON, all 4 gates active) → no runtime row-overflow error; `DASHBOARD_MAX_ROWS = 36`. (ORB/Fade rows never render.)
10. **RVOL fallback:** `rvolMode` is pinned to Time-of-Day → Volume row shows `TOD` on a chart with ≥ `rvolLookbackDays` (10) sessions of history, and `20-bar*` (auto fallback) on a short chart — never `na` or a stale value.
11. **P&L Target:** with `enablePnL` on, PROFIT/LOSS fires at exactly `±pnlTarget %` from `pnl_entry_price`; labels show the static target text (e.g. `PROFIT +1.0%`).
12. **Auto entry-freshness by anchor:** in **EMA Cross** mode, confirm no BUY/SELL fires more than 10 bars after a flip (watch the "Bars From Flip" Extended Metrics row — count resets to 0 on each flip bar); in **SMA** mode the "Bars From Flip" row reads `(unlimited)` and entries can fire at any distance. `sigTime` is pinned to Score, so there is no Trend-mode comparison.
13. **Win rate:** dashboard shows `BUY Win Rate` / `SELL Win Rate`; a BUY that hits PROFIT then is closed by the next SELL counts as a win; hits LOSS → loss; closed with neither → loss; the currently open signal is never counted. No per-module win-rate rows render in the light build (`multi_module` is false); `mod_results` slot 0 still accumulates in the background.
14. **`enablePnL` independence:** turn P&L Exit OFF → PROFIT/LOSS labels + alerts stop, but BUY/SELL win rates keep resolving normally (must NOT collapse to 0%).
15. **Stale dashboard rows:** let a gate go active (gate-detail rows appear), then clear → the Gate Details rows disappear entirely, not freeze on stale text.
16. **Session-open flatten:** on a STOCK/FUND/MARKET symbol (`rth_auto = true`), hold a position into the close → at the next 9:30 open the dashboard P&L reads `—` and no PROFIT/LOSS fires off the overnight gap.
17. **RTH auto-off for 24h profiles:** load a CRYPTO or FUTURES symbol → confirm `useSessionFilter` resolves `false` (signals fire outside 9:30–16:00 ET; no session-open flatten). Load a STOCK → resolves `true`.
18. **Profile auto-tune resolves:** on a STOCK chart the Signal Anchor row shows buffer `±0.25A` and (EMA mode) `9/20`; on a MARKET-INDEX chart `±0.20A` and `9/30`; on CRYPTO `±0.35A` and `9/20`. Min-score base moves 45↔55 by profile (check the `⭐`/`⚠️` threshold in Extended Metrics).
19. **ORB fully inert:** no ORB row, no orange ORB lines, no `⚡ORB` label ever; `active_strategy` is always `Trend Pullback`. Signals identical to a build with the ORB module physically deleted.
20. **Timeframe length scaling:** on a STOCK symbol at 2-min, Signal Anchor reads `EMA Cross (9/20) auto` (EMA mode) or `SMA (70) auto ±0.25A` (SMA mode) — unchanged from before this feature (`tf_scale = 1.0` at the 2-min baseline). Switch the chart to 5-min → the EMA period numbers and the SMA bar count drop proportionally (e.g. `EMA Cross (4/8) auto`, `SMA (28) auto ±0.25A`); switch back to 2-min → they return to exactly `9/20` and `70`. Load a Range/Tick chart → no error; lengths hold at their 2-min values (`tf_scale` falls back to `1.0`).
21. **Component 1 direction follows the anchor:** the bull/bear branch is chosen by `is_bull`/`is_bear` and always scores against `ema20` (not `emaSlow`). Spot-check on any chart: with `is_bull`, `ema9>ema20>ema50` + `close>ema9` → `✅ FULL CASCADE` (`ema_max`); `close>ema20` but cascade incomplete → `⚠️ PARTIAL` (~60%); `close` below `ema20`/`ema50` in a bull → `🔴 COUNTER-TREND` (0). Component 1's status never contradicts the Trend row's direction. Confirm `raw_score ≤ 100` and the 105-max invariant hold in both anchor modes. (SMA-mode `raw_score` skews slightly higher on continuation — expected, see [SMA Anchor](#sma-anchor).)

---

## Version

The script file is `SLT.pine`. It was forked from **SuperLazyTrade V3** with the SuperTrend signal anchor removed entirely (input option, SuperTrend Engine settings group, adaptive ATR/factor block, `ta.supertrend` call, `st_*` anchor, its plots/fills/circles, dashboard branch). Anchor selection is binary — SMA (default) or EMA Cross.

**Light-build simplification (current):** the input surface is reduced to Signal Anchor, Score Stars, P&L Exit, and two Dashboard & Visuals toggles (`showDash` / `extDash`). Former inputs are auto-tuned from the asset profile + anchor (`ema_slow_auto` / `sma_buf_auto` / `min_score_auto` / `rth_auto` → `emaSlowPeriod` / `smaBufferATR` / `minScore*` / `useSessionFilter`, plus `maxBarsFromFlip` from the anchor) or pinned as constants (`sigTime="Score"`, `gateOn=true`, `rvolMode="Time-of-Day"`, SMA MANUAL mode dropped, success-rate + `showSignalPnL` + `showLabels` + `showCircles` + `showBg`). The **VWAP Fade** module is deleted; the **ORB** module is retained but inert (`orbEnabled=false` constant). (A Component 1 anchor-decoupling — commit `c0d85a1` — was tried for cross-anchor score comparability and reverted; it broke the term's directionality.) The on-chart `indicator()` title is `SLT V1` (from `VERSION = "V1"`).

**Gate change — velocity override extended to Gate 2:** `is_high_velocity` (`ADX > 35` rising 3 bars) now waives the Stretch penalty as well as the ATR-Fuel penalty. A strong, still-accelerating trend that has run past `ext_scale` ATR keeps its `final_score` intact instead of being docked −10/−20 and pushed below the star filter. The raw stretch tier is still shown as advisory (`gate2_raw`, gate message reads `🔥 VELOCITY` in place of `-N`); only the score subtraction and `active_gate_count` are suppressed. Implemented as `gate2_stretched = gate2_raw and not is_high_velocity`, mirroring the pre-existing `gate3_fuel` pattern.

**Dashboard — Strategy row removed:** the `active_strategy · <regime>` row is deleted. In the light build `active_strategy` is always `Trend Pullback`, and `regime` (the `SQUEEZE`/`BREAKOUT`/`EXHAUSTED`/`TREND`/`RANGE` string) had no consumer left after VWAP Fade — so `regime` and `sqz_bias` are deleted from `SLT.pine` too. Signal generation is now three sub-stages (strategy modules → router → dispatch).

**Timeframe length scaling (current):** every bar-count lookback (EMA9/20/50, the profile SMA length, ATR/RSI/ADX(14), the squeeze BB/KC(20) lengths, the 20-bar RVOL fallback, `maxBarsFromFlip`) is now rescaled by `tf_scale = 120 / timeframe.in_seconds()` (`scale_len()`) so each keeps covering the same wall-clock window on any chart interval, not just the tested 2-min chart. At `tf_scale = 1.0` (i.e. on a 2-min chart) nothing changes — see [Timeframe Length Scaling](#timeframe-length-scaling). Profile threshold/percentage values (`vwap_h_limit`, `adx_min`, `ext_scale`, `fuel_max`, RVOL tiers) are explicitly *not* rescaled — they have no comparable scaling law and stay validated only at 2-min.

**Scoring change — MACD dropped (commit `9284b73`):** Component 5's MACD sub-component (was 5A, 8 pts, "MACD line same sign as trend direction") is removed. MACD-line sign ≈ `EMA12 > EMA26`, a near-duplicate of Component 1's EMA cascade, so it carried almost no independent signal. Its 8 points were redistributed proportionally (old 7:5 split) across the two survivors — RSI 7→12 (now 5A), Squeeze Release 5→8 (now 5B). Component 5 max stays 20 and the 105 pre-cap total is unchanged. The `ta.macd(close,12,26,9)` call, `macd_aligned` / `macd_pts`, and the `MACD: ✓/✗` field in `r_status` are all deleted; `r_status` tier thresholds re-cut to `16 / 12 / 8 / 3` for the new point distribution (cosmetic — the label never feeds `raw_score`). No changelog history is tracked in the file — treat the current source as the reference behavior going forward.
