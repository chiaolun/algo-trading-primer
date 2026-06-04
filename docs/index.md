# Algo Trading Wiki

This wiki is a self-study path for building one thing: a **minimal, leakage-free
futures backtester driven by a machine-learning signal**. It is organized as a
sequence of *driving questions*. If you can answer every question in your own
words and defend the answer against an interview-style "why?", you understand
enough to build — and to trust — a simple systematic trading system.

The path runs from the ground up: what a market *is*, how futures contracts and
their expiries work, how orders interact with the order book, what market data
actually contains, how raw data becomes the bars and features a model consumes,
how a backtester replays history without cheating, and finally how a prediction
becomes a position, an order, and a profit-and-loss statement you can believe.

## How to use this wiki

- **Read in order.** Each section assumes the previous ones. Later sections
  (leakage, feature engineering, execution) only make sense once you have the
  microstructure vocabulary from the early ones.
- **Answer the driving questions out loud.** Each `##` heading is a question.
  Cover the answer, attempt it, then read.
- **Follow the citations.** Every factual claim links to an entry on the
  [Sources](sources.md) page, which tells you where to go deeper.
- **Build as you go.** By [Section 10](10-capstone.md) you should be assembling
  the pieces into a working end-to-end system.

## The learning path

1. **[Market structure & double-sided auctions](01-market-structure.md)** —
   traders, brokers, exchanges, the bid/ask spread, price-time priority, and
   what makes a market liquid.
2. **[Futures contracts & expiries](02-futures-contracts.md)** — standardized
   contracts, expiration, rolling, term structure, and selecting the active
   contract.
3. **[Orders & execution](03-orders-and-execution.md)** — market vs. limit
   orders, how they hit the book, partial fills, and what it takes to *simulate*
   a fill.
4. **[Market data](04-market-data.md)** — L1, L2, market-by-price,
   market-by-order, and trade data — and what each can and cannot tell you.
5. **[Bars & aggregation](05-bars-and-aggregation.md)** — OHLCV bars, how trades
   become bars, empty bars, and the look-forward risk bars introduce.
6. **[Backtesting engine design](06-backtesting-engine.md)** — what a backtest
   is, the event-driven architecture, the state to maintain, and the minimum
   viable futures backtester.
7. **[Look-forward bias & leakage](07-look-forward-bias.md)** — the timestamps
   that matter, same-bar execution, continuous-contract leakage, overfitting,
   and time-series validation.
8. **[Feature engineering & regression framing](08-feature-engineering.md)** —
   turning a time series into a supervised-learning problem, choosing targets
   and features, stationarity, and forecasts vs. rules.
9. **[From prediction to actions](09-prediction-to-actions.md)** — predictions
   to positions to orders, when to use which order type, what to log, and how to
   evaluate net of costs.
10. **[Capstone system](10-capstone.md)** — assemble, prove there is no leakage,
    explain the failures, and compare execution variants.

When you want the underlying references, see the **[Sources](sources.md)** page.
