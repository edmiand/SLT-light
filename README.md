# SLT-light

A trimmed-down fork of SuperLazyTrade — a Pine Script v6 TradingView indicator for
intraday momentum trading.

**What's different from SuperLazyTrade:** the SuperTrend signal anchor has been
removed entirely. SLT-light offers two anchors:

- **EMA Cross** (default) — signals on EMA9 crossing the user-selected slow EMA (20 or 30)
- **SMA** — signals on price crossing a slow structural SMA, with an ATR buffer band
  and latched state to suppress whipsaw

Everything else — the 5-component scoring engine, the 4 risk gates, both P&L
tracking systems, win-rate tracking, and the dashboard — is unchanged.

## Usage

1. Open TradingView → Pine Editor
2. Paste the full contents of `SLT-light.pine`
3. Save → Add to Chart
4. Test on a 2-minute chart with a liquid instrument (NVDA, TSLA, SPY, QQQ, SOXX)

There is no build system or test runner: development is editing the `.pine` file
and compiling it in the Pine Editor.

## Disclaimer

Educational reference only. Does not guarantee any specific trading results.
