# 📈 SLT

**An intraday trading indicator for TradingView (Pine Script v6).**

SLT hunts for the moments where trend, momentum, volume and volatility all line
up — and marks them on the chart with **🟢 BUY** and **🔴 SELL** labels. A compact
dashboard in the corner shows the current setup quality, a plain-language trade
call, and a running win rate for the signals it has printed.

## ✨ What it does

- 🎯 **Confluence scoring** — five components (EMA cascade, VWAP, volume, ADX,
  momentum) scored and summed into a single setup rating
- 🛡️ **Risk gates** — squeeze, over-extension, spent volatility and thin
  liquidity all dock the score
- 🧭 **Plain trade call** — a TRADE / CAUTION / SKIP / WAIT verdict, sample-size
  aware, so you don't have to read the raw numbers
- 📊 **Live dashboard** — setup score, gates, win rate and P&L, bottom-right
- 🔔 **Alerts** — `BUY`, `SELL`, `PROFIT`, `LOSS`

## ⚡ Built for intraday

Short holds, one position at a time, flat by the end of the session. SLT adapts
on its own to whatever you load it on — 📊 stocks, ETFs, ⚡ futures, 🪙 crypto,
🏦 index CFDs — so there is nothing to set up per symbol.

## ⚙️ Settings

The panel is deliberately small:

| Setting | What it controls |
|---|---|
| 🧠 **Signal Anchor** | the trend engine (EMA Cross or SMA) |
| ⭐ **Score Stars** | how selective the BUY/SELL labels are |
| 💰 **P&L Exit** | an optional fixed profit/loss target |
| 🕒 **Trading Hours** | the session window signals may fire in |
| 🖥️ **Dashboard & Visuals** | dashboard visibility |

Everything else tunes itself.

## ⚠️ Disclaimer

For educational purposes only. Not financial advice, and no trading result is
guaranteed. Trade at your own risk.

---

*Development notes and the full internal reference live in `CLAUDE.md` and
`HISTORY.md`.*
