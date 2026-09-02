# Scalper Bot

Phone/Termux-run MT5 scalping bot. Logs into a broker account (KBS, FBS,
Deriv, Exness — anything running MT5) via MetaAPI, runs an EMA/RSI/Bollinger
signal engine against live price data, and shows status on a mobile
dashboard styled like the MT5 "account connected" flow.

## What's actually in here

- `server.js` — Express server: serves the dashboard, exposes the
  connect/stop/remove/status/quotes API, runs the 5-second trading loop.
- `src/metaapiConnector.js` — talks to MetaAPI: provisions the broker
  account connection, streams prices, places/closes orders.
- `src/strategy.js` — EMA9/EMA21 cross + RSI14 + Bollinger(20,2) signal
  logic.
- `src/riskManager.js` — position sizing (risk % of balance per trade) and
  a daily loss circuit breaker that auto-stops the bot.
- `public/index.html` — the dashboard UI, matching the reference screens.

## Before you run this for real

This is a working skeleton with a real, coherent strategy — not a proven
edge. Two non-optional steps before any live money touches it:

1. **Backtest** the strategy logic in `src/strategy.js` against historical
   data for your actual target symbol/broker.
2. **Demo-run** it on a demo MT5 account for at least a couple of weeks and
   look at the real win rate, drawdown, and average trade — then tune the
   EMA/RSI/BB periods and the stop-loss/take-profit pip values in `.env`
   based on what you actually see, not on the defaults here.

There's a `DailyLossGuard` in `riskManager.js` that halts trading if the
account drops 5% in a day — treat that as a last-resort safety net, not a
substitute for the two steps above.

## Setup (Termux)

```bash
pkg update && pkg install nodejs git -y
git clone <your-github-repo-url>
cd scalper-bot
npm install
cp .env.example .env
```

Edit `.env`:
- `META_API_TOKEN` — from app.metaapi.cloud → API access → generate token.
  This is your MetaAPI account token, not a broker password.
- `SYMBOL` — the exact symbol name your broker uses (varies — check MT5
  Market Watch on that broker, e.g. `EURUSD` vs `EURUSDm` vs `EURUSD.s`).
- `RISK_PERCENT`, `STOP_LOSS_PIPS`, `TAKE_PROFIT_PIPS` — starting values,
  expect to change these after demo testing.

Run it:

```bash
node server.js
```

Then open `http://127.0.0.1:3000` in your phone's browser. Log in with the
broker account's actual login/password/server — those get sent to your own
local server, which does the real MetaAPI login. Nothing goes anywhere else.

## Broker-specific gotcha: provisioning profiles

MetaAPI needs to know how to reach your broker's MT5 servers. For major,
widely-used brokers it resolves this automatically from the server name.
For a smaller or regional broker (a "KBS"-style server MetaAPI hasn't seen
before) you may need to create a **provisioning profile** manually first in
the MetaAPI dashboard, then pass its id into `createAccount()` in
`src/metaapiConnector.js` as `provisioningProfileId`. If `/api/connect`
fails with a provisioning-related error, that's what's happening.

## Pushing to GitHub

```bash
git init
git add .
git commit -m "Initial scalper bot"
git remote add origin <your-repo-url>
git push -u origin main
```

`.gitignore` already excludes `.env` and `node_modules` — never commit your
real MetaAPI token or a broker password into the repo.

## Running multiple accounts at once

This supports connecting any number of broker accounts simultaneously —
different brokers, different logins, all independent of each other.

- `src/accountManager.js` holds one record per connected account: its own
  `MetaapiConnector`, candle buffer, daily loss guard, and running/stopped
  flag. One shared 5-second timer drives all of them, but each account's
  trade decisions, risk sizing, and daily-loss check only ever look at that
  account's own data.
- The dashboard opens on an **accounts list** — tap "+ Connect another
  account" to add one (KBS, FBS, Deriv, whatever), tap an existing card to
  open its own login/dashboard/stop/remove controls.
- Removing or stopping one account has zero effect on the others.

Nothing about the per-account strategy or risk logic changed — it's the
same EMA/RSI/Bollinger engine and same position sizing, just run once per
connected account instead of once globally.
