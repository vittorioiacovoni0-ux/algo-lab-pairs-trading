# algo-lab-pairs-trading

Intraday statistical-arbitrage research and paper trading on US equities: cointegrated pairs, 15-minute bars,
out-of-sample evaluation with transaction costs, and an event-driven live runner on an Alpaca **paper** account.

The signal logic lives in a single module (`strategy.py`) that both the research backtest and the live bot import,
so what is backtested is exactly what is executed.

## Pipeline

| Step | File | What it does |
|---|---|---|
| Data | `data.py` | Downloads intraday bars from Alpaca (free IEX feed). |
| Signal | `strategy.py` | Hedge-ratio leg weights (gross exposure = 1), rolling z-score of the log spread, and a state machine (flat / long spread / short spread) with entry, exit and stop thresholds. It never flips sign in a single step. |
| Research | `research_pairs.py` | Engle-Granger cointegration test on the **train** window only, then out-of-sample backtest with costs. Reports Sharpe, net and gross return, max drawdown, number of trades and time in market. |
| Live | `live_pairs.py` | Reads `pair_config.json`, waits for each bar to close, recomputes the state and sends market orders to the paper account, with logging and market-hours handling. |

## Design choices worth noting

- **No look-ahead:** the position decided at the close of bar *t* earns the return from *t+1* (`shift(1)`), and the z-score uses only past data.
- **Costs are explicit:** a per-turnover cost in bps is charged on every position change; net and gross returns are reported separately.
- **Multiple-testing awareness:** the research script prints how many pairs it tested versus how many cointegrated pairs
  you would expect by chance at the 5% level (about 5% of all pairs).
- **Single code path:** research and live share `strategy.py`.

## Setup

```bash
pip install -r requirements.txt
cp .env.example .env          # add your Alpaca PAPER keys
python research_pairs.py      # writes pair_config.json for the best out-of-sample pair
python live_pairs.py          # paper trading, logs to live_pairs.log
```

`live_pairs.py` is hard-wired to `paper=True`. Do not point it at a live account without a proper review.

## Limitations

- The best pair is chosen by out-of-sample Sharpe on the same test window, so the reported figure carries selection bias.
  A second, untouched holdout (or a walk-forward re-selection) is the natural next step.
- IEX-feed volume is a small fraction of consolidated volume; fills in paper trading are optimistic compared with real execution.
- The universe (16 large caps) and the 180-day window are small, and results are sensitive to both.
- Nothing here is investment advice.

## Roadmap

- [x] Phase 1: pairs trading, out-of-sample research and paper trading
- [ ] Phase 2: L2 order-book recording for crypto and microstructure analysis
- [ ] Phase 3: Avellaneda-Stoikov market making with `hftbacktest` (queue position and latency)
- [ ] Phase 4: full pipeline on `nautilus_trader` with a testnet

## License

MIT. Author: Vittorio Iacovoni ([vittorioiacovoni.com](https://vittorioiacovoni.com)).
