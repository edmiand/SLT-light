# SLT

An intraday trading indicator for TradingView (Pine Script v6).

SLT looks for moments where trend, momentum, volume and volatility line up, and
marks them on the chart with **BUY** and **SELL** labels. A small dashboard in the
corner shows the current setup quality, a plain trade call, and a running win rate
for the signals it has printed.

It is built for **intraday trading** — short holds, one position at a time, flat
by the end of the session. It adapts on its own to whatever you load it on
(stocks, ETFs, futures, crypto, index CFDs), so there is nothing to set up per
symbol.

## Settings

The panel is deliberately small: the trend engine, how selective the signal
labels are, an optional fixed profit/loss target, a trading-hours window, and
dashboard visibility. Everything else tunes itself.

## Alerts

BUY, SELL, PROFIT, LOSS.

## Disclaimer

For educational purposes only. Not financial advice, and no trading result is
guaranteed. Trade at your own risk.

---

*Development notes and the full internal reference live in `CLAUDE.md` and
`HISTORY.md`.*
