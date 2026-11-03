# AI Trader V1 — Paper-Only Machine Learning Trader

A $0-budget educational prototype that trains a **logistic-regression** model on historical 5-minute price bars, evaluates it on **chronologically held-out** sessions and can submit **PAPER** stock bracket orders through Alpaca. No live-trading code path exists. The model is a modest ML baseline, **not a proven profitable trading strategy**.

## Requirements

Python 3.11+, internet for Alpaca mode, a free Alpaca paper account, an always-on computer for continuous paper-loop operation. Works on Chromebook with Linux development environment enabled; on Android, use a remote Python environment/PC rather than assume this will run natively in the ChatGPT mobile app.

## Quick start (offline, no account required)

1. Unzip this folder and open a terminal in it.
2. Create a Python environment, install packages:

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

3. Test the project:

   ```bash
   python ai_trader.py demo
   python -m pytest -q
   ```

`demo` creates *fictional* candles and an offline `output/demo/dashboard.html`; **never treat demo metrics as trading performance**. It cannot be used for paper execution.

## Get real historical candles and run an honest holdout backtest

1. Create a free Alpaca **paper** account at <https://app.alpaca.markets/>. Generate PAPER keys (not live keys).
2. Copy `.env.example` to `.env`; replace both values with PAPER keys. Never share or commit `.env`.
3. Run:

   ```bash
   python ai_trader.py fetch --days 180 --symbol SPY
   python ai_trader.py train --csv output/iex_bars.csv --source alpaca_iex --symbol SPY
   ```

This creates `output/real/model.joblib`, `output/real/report.json`, `output/real/trades.csv` and an offline-openable `output/real/dashboard.html`. Training uses the earlier 65% of trading sessions, reserves the next 15% as a validation diagnostic, and reports backtest performance on the last 20% of sessions. The probability threshold (60%) is fixed in code before testing. Session-level data splitting reduces but cannot eliminate bias, overfitting or regime drift. A backtest is not a promise of future results.

**Market data limitations:** The free IEX feed covers only one trading venue (not the consolidated market); historical bars may be sparse. Strategy performance can change on consolidated data. The current software ignores stocks splits/dividends, fees, taxes, partial fills, halts and complex corporate actions. Backtest entry, stop, take-profit and slippage modeling is approximate. Tiny sample sizes or too few trades mean there is no credible evidence of an edge.

## Run a PAPER trading signal check

First run the `fetch` and `train` commands using Alpaca historical data, then:

```bash
python ai_trader.py paper-once --symbol SPY
```

This is a **dry run** even with keys. It checks the paper account, market hours, historical bars, live IEX quote, ML signal, bid/ask spread, prior orders/positions, risk limits and signal freshness. It does **not** submit orders.

To allow **simulated** paper orders:

```bash
python ai_trader.py paper-once --symbol SPY --execute-paper
```

Or run a loop that rechecks each minute and attempts to flatten the *symbol's* paper position at 3:45 p.m. Eastern on trading days:

```bash
python ai_trader.py paper-loop --symbol SPY --execute-paper
```

**Critical:** The loop must remain running. If your machine sleeps, internet disconnects, API requests fail, trading halts occur, or cancellation/flatten orders fail, paper positions may remain open overnight. A stop order does not guarantee a stop price. Verify orders and account positions in Alpaca directly. The EOD mechanism attempts to cancel any open orders for the selected symbol and close its position; use a **dedicated paper account**, since orders for this symbol might have been created manually.

## Built-in guards

- **PAPER-ONLY**: Broker client is hard-coded to `paper=True`; no live path or live toggle.
- **Long-only stocks** (default SPY); no futures, options, shorting or margin leverage.
- **No entries before 10:40 a.m. or after 3 p.m. ET**. EOD flatten attempt at 3:45 p.m. ET if loop stays active.
- **Max one position and no concurrent open order for the symbol.**
- **Up to 3 bot entry attempts per day** per symbol. Resubmission on the same 5-minute bar is blocked by broker order IDs.
- **0.25% nominal account equity at risk per trade** based on stop width, not a guaranteed loss cap.
- **10% of account equity max position notional**; whole shares only.
- **1% daily equity-loss guard** relative to broker `last_equity`. It blocks new entries; it does not forcibly cut losses or guarantee a loss limit.
- **IEX quote freshness, spread and bar age checks**; skips uncertain market data.
- **Bracket orders** with stop and take-profit, though filling and timing can differ from modeled results.
- **Synthetic-data trained models are refused for paper trading**.

These are software guardrails, not guarantees. P&L may exceed calculated risk due to gap/slippage, unfilled orders, connectivity or software failures. Account capital in a paper account is simulated.

## Strategy and ML transparency

Inputs: 1-bar return, 3-bar return, candle range, candle body, relative volume, 12-bar moving average gap, VWAP gap and 14-bar intraday average true range percentage. Predictions estimate whether a next-open-to-three-bars-later-close move exceeds 0.14%, before the other execution assumptions. A buy also requires price above VWAP and above the 12-bar average. The model is deliberately simple and interpretable enough to test; **there is no evidence yet it has predictive value**.

The backtest enters at the *next* bar open and estimates stop-first execution when both stop and target are touched. Model training uses past-only candles. Data are split by session to avoid training on the same day's test candles. There's a single held-out period, not rolling retraining or robust multi-regime walk-forward analysis. Before trusting results, add purged walk-forward evaluation, higher-quality market-wide data, extensive paper forward tests and out-of-sample monitoring.

## Security and operation

- Do not put API keys in chat, GitHub, public videos, screenshots or uploaded files.
- `.env` is ignored by Git; zipped package contains only `.env.example`.
- Only load `model.joblib` you generated yourself: Python's joblib uses pickle, which can run malicious code from untrusted files.
- Test everything in paper mode for an extended period; verify live broker settings independently before even considering trading with real money.
- Day trading may be restricted by broker margin rules; those are **not implemented** here because this software cannot place live orders. See SEC/FINRA and your broker for current treatment.

## Future V2 ideas

1. Independent benchmark (buy-and-hold, random and rules-based signals), transaction-level trade logs, walk-forward retraining and embargo.
2. Reliable broker-stream ingestion, heartbeat, persistent reconciliation, circuit breaker, actual fill tracking and robust calendar/EOD fail-safe.
3. Free dashboard with equity curve, precision/recall by regime, and live paper performance.
4. Optional research-only news sentiment. Avoid using an LLM as the sole buy/sell decision engine.

Built November 2026. Educational software. Not financial advice.
