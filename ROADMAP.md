# ROADMAP.md

Candidate future work for `SLT.pine`, newest ideas at the bottom of each
section. Like `HISTORY.md`, this file is **not** loaded into model context by
default. An item moves to `HISTORY.md` (with rationale + commit) when it ships;
delete it here if it's rejected.

Tags: **Effort** S/M/L · **Risk** Low/Med/High (Risk = chance of a behavior
regression or a reversed design decision, not implementation difficulty).

> **Note (2026-09-13):** a V2 EMA-pullback rebuild shipped items 1–4 and
> superseded 8–9, but was parked after live results; the working file is
> back on the V1 base (V1.1 = V1 + Σ). Every item below is therefore still
> open. See HISTORY.md for what V2 tried and why it was parked.

---

## Performance & code size

### 1. Delete the inert ORB module
**Effort M · Risk Med.** ORB is `orbEnabled = false` and unreachable: ~120 lines
across the `FIXED BEHAVIOUR` constants (`orbMinutes`…`orbShowLevels`), the
module logic (`orb_*` range/breakout/quality), the router branch, the dashboard
row, the two `plot()` level lines, and the `strat_id` ORB slot. Removing it is
the single biggest **compile-time** reduction available, also drops 2 plot
outputs and ~1 per-bar block of dead `orb_q` math.
_Trade-off:_ CLAUDE.md documents ORB as deliberately-kept scaffolding for
"re-adding a module". Needs an explicit call to give that up (and a
`Key Design Decisions` / checklist-item-21 edit).

### 2. Delete the dead per-module win-rate plumbing
**Effort M · Risk Med.** `multi_module = orbEnabled` is always false, so the
`ModResults` UDT, `mod_results` array, `record_module()`, `calc_win_rate()`,
`fmt_win_rate()`, `strat_id()`, and the `for mi = 0 to 2` dashboard loop are all
dead (slot 0 still accumulates every trade close, feeding nothing). ~60 lines +
3 UDFs + 1 UDT of compile weight; a few `array.push`/`array.shift` per trade
close of runtime. Pairs naturally with item 1.
_Trade-off:_ same "kept for a second module" rationale as ORB.

### 3. Drop the unreachable `sigTime == "Trend"` branch
**Effort S · Risk Low.** `sigTime` is pinned `"Score"`. The `if sigTime ==
"Trend"` branch of `tp_long`/`tp_short` never runs. Collapse to the Score body
and remove the `sigTime` pin + the Trend-mode notes.

### 4. Merge the 4 flip-circle `plotshape`s into 2
**Effort S · Risk Low.** EMA-up / EMA-down / SMA-up / SMA-down are four separate
`plotshape` calls, two always `na` for the inactive anchor. Compute
`flip_up_px = anchorMode == "SMA" ? (trendUp ? smaAnchor : na) : (trendUp ? ema9
: na)` (mirror for down) and use two. Minor compile win; combine with item 1
removing the 2 ORB `plot()`s to roughly halve plot-output count.

### 5. Full deferral of the remaining per-bar string assembly
**Effort M · Risk Med.** After the Sept optimization pass the hot-path
`str.tostring` calls are gone; what's left is interned-literal tier selection
(`t_status`, `stretch_status`, `sqz_status`) every bar. Converting those to a
compact int/enum tier code resolved to a string only in the dashboard is the
last increment. **Do only if profiling still shows a string-work gap** — the
payoff is now small and it touches the scoring/gate control flow.

### 6. Time-of-Day RVOL: running per-slot sum
**Effort M · Risk Med.** The `for s = 0 to tod_hist_sessions - 1` loop re-sums up
to `rvolLookbackDays` (10) sessions' volume at the current slot every bar. A
maintained running sum per slot (updated on push/shift) removes the loop.
Measure first — ≤10 iterations of array access is likely well below the
`request.security` / string costs already addressed.

### 7. Gate `day_end_onbar` time math behind a cheaper precondition
**Effort S · Risk Low.** `bar_close_mins = hour(time_close, tz) * 60 +
minute(time_close, tz)` and `reached_session_end` compute every bar but only
matter for `day_end_onbar`, which requires `barstate.islast`. Short-circuit the
`hour()`/`minute()` calls when not near the live edge.

---

## Signal behavior

### 8. N-bar signal cooldown as the whipsaw fallback
**Effort M · Risk Med.** HISTORY names this as the fallback if clusters of
*normal-range* bars are still seen whipsawing at the open (the oversized-bar
range test can't catch those). A short post-signal cooldown (tf-scaled, likely
opening-window-scoped like the oversized filter was) would. Design question:
cooldown vs. a minimum-bars-between-signals counter, and whether it interacts
badly with legitimate fast reversals (the reason the oversized filter was
scoped to the open).

### 9. Decide the final disposition of the oversized-bar filter
**Effort S (decision) · Risk Low.** The working tree currently removes it
(degraded observed win rate — see HISTORY). Confirm: stays removed, replaced by
item 8, or reinstated with different scoping. Whatever lands needs a HISTORY
entry and a CLAUDE.md / checklist-item-12b update to match.

### 10. Pieces of the parked V2 worth re-trying one at a time on the V1 base
**Effort S each · Risk Low–Med.** In order of exit-safety: (a) an entry-window
block (no entries in the first/last 30 min — the book is flat at open, so it
cannot suppress an exit); (b) an expectancy-based Trade Signal verdict
(`Σ / n` per direction, now that Σ exists); (c) a 15-minute EMA(20) bias as a
penalty-only Gate 6 with its own dashboard row. Each must show its effect in
the `Σ` figure over several sessions before the next is added.

---

## Done

Shipped items and their rationale live in [HISTORY.md](HISTORY.md) (newest last).
