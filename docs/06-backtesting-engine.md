# 6. Backtesting engine design

A **backtester** is the machine that replays history and asks: "if my strategy
had been running then, what would have happened?" Its design decides whether the
answer is trustworthy. This section defines what a backtest is, why an
**event-driven** architecture is the right shape, the state the engine must
carry, and what the *minimum viable* futures backtester looks like. It closes
with the assumptions that separate a realistic backtest from a fantasy — which
[Section 7](07-look-forward-bias.md) then dissects as sources of leakage.

The architecture reference is [QuantStart](sources.md#quantstart); backtesting
risks follow [López de Prado](sources.md#lopezdeprado); the course spine is
[CS 7646](sources.md#cs7646).

## What is a backtest?

A **backtest** is a simulation of a trading strategy over historical data. It has
four parts: **historical replay** (feed past market data in time order),
**simulated decision-making** (the strategy produces signals and target
positions), **simulated execution** (orders are turned into fills under a [fill
model](03-orders-and-execution.md#what-assumptions-are-needed-to-simulate-limit-order-fills)),
and **performance measurement** (track P&L, risk, and costs). The promise of a
backtest is an estimate of how a strategy *would* have performed; the peril is
that small modeling shortcuts can make that estimate wildly optimistic.

Georgia Tech's CS 7646 frames exactly this — the real-world challenges of
implementing ML-based trading strategies and measuring them
([CS 7646](sources.md#cs7646)). The recurring theme of this wiki is that a
backtest is only as honest as its weakest assumption.

## What is an event-driven backtester?

An **event-driven backtester** processes the world as a stream of discrete
**events**, handled in time order, exactly as a live system would. The core event
types form a loop:

- **Market event** — a new tick, quote, or [bar](05-bars-and-aggregation.md)
  arrives.
- **Signal event** — the strategy, reacting to the market event, emits a desired
  view or [target position](09-prediction-to-actions.md#how-does-a-prediction-become-a-position).
- **Order event** — the portfolio/execution layer translates the target into
  concrete orders.
- **Fill event** — the simulated exchange reports executions, which update
  positions, cash, and P&L.

QuantStart describes this as **drip-feeding** market data as events to replicate
how a real order-management and portfolio system behaves
([QuantStart](sources.md#quantstart)). The architecture's great virtue is that it
*structurally* enforces causality: the strategy can only act on an event *after*
it has been delivered, which makes the [same-bar lookahead
trap](05-bars-and-aggregation.md#why-can-bars-create-look-forward-risk) hard to
fall into by accident. It contrasts with **vectorized** backtesting (compute
signals over a whole price array at once), which is faster to write but far easier
to leak future data through.

## What state should the backtester maintain?

To behave like a live trading system, the engine must carry a complete picture of
the world at each instant. At minimum:

- **Clock** — the current simulated time, advanced by events.
- **Active contract** — which [futures
  contract](02-futures-contracts.md#how-should-a-backtester-select-the-active-equity-index-futures-contract)
  is currently tradable, and the roll schedule.
- **Book / market state** — the latest quotes, depth, or bar needed to price and
  to simulate fills.
- **Positions** — current holdings per instrument.
- **Cash and margin** — available capital, posted [margin](02-futures-contracts.md#what-is-a-futures-contract),
  and buying power.
- **Open orders** — resting orders, their prices, sizes, and queue assumptions.
- **Fills** — the execution record.
- **Realized and unrealized P&L** — closed-trade profit and mark-to-market of open
  positions.
- **Fees** — commissions and exchange fees accrued.
- **Risk limits** — position caps, loss limits, and other guardrails.

This is the state a live order-management-and-portfolio system keeps, mirrored in
simulation ([QuantStart](sources.md#quantstart)). Getting the accounting right —
especially marking positions to the *current* market and accruing fees — is what
makes the final [P&L
evaluation](09-prediction-to-actions.md#how-do-you-evaluate-the-strategy-after-execution-costs)
meaningful.

## What is the minimum viable futures backtester?

The smallest system that is still honest enough to learn from does the following,
in an event loop:

1. **Replay market data** in timestamp order.
2. **Select the active contract** by an explicit
   [roll rule](02-futures-contracts.md#how-should-a-backtester-select-the-active-equity-index-futures-contract).
3. **Generate signals** from features known *as of* the current event.
4. **Send orders** derived from the [target
   position](09-prediction-to-actions.md#how-does-a-target-position-become-orders).
5. **Simulate fills** under an explicit fill model and latency assumption.
6. **Update positions, cash, and margin** on each fill.
7. **Calculate P&L** (realized and unrealized) and accrue fees.
8. **Handle the roll** — close the expiring contract and reopen in the next, with
   its costs.

Steps 2 and 8 are what make it a *futures* backtester specifically; everything
else generalizes. The roll mechanics come from [CME's roll
materials](sources.md#cme) and the event loop from
[QuantStart](sources.md#quantstart). This is precisely the system you assemble in
the [capstone](10-capstone.md).

## What assumptions determine backtest realism?

The gap between a backtest and reality lives in a handful of assumptions. Each one
that is too generous inflates results:

- **Data granularity** — [bars vs. L2 vs.
  MBO](04-market-data.md) bound what you can faithfully simulate.
- **Latency** — the delay between decision, order, and fill.
- **Queue model** — where your [limit
  orders](03-orders-and-execution.md#what-assumptions-are-needed-to-simulate-limit-order-fills)
  sit and how they fill.
- **Transaction costs, slippage, and fees** — the spread paid, market impact, and
  commissions; a strategy that looks great gross can be a loser net.
- **Fill priority** — whether you optimistically assume you trade ahead of others.
- **Roll rules** — when and how you switch contracts, and what that costs.
- **Missing-data handling** — [empty
  bars](05-bars-and-aggregation.md#what-happens-if-no-trades-occur-during-a-bar-interval),
  gaps, and outages.
- **Session boundaries** — opens, closes, halts, and overnight gaps where
  liquidity and behavior differ.

These draw on [Harris](sources.md#harris), the [Databento
schemas](sources.md#databento), and [López de Prado's backtesting
chapters](sources.md#lopezdeprado), whose Part 3 covers the dangers of
backtesting, cross-validation, synthetic data, backtest statistics, and strategy
risk. The single most damaging failure of realism — using information before it
was available — is important enough to get its own section next.

---

Previous: **[← Bars & aggregation](05-bars-and-aggregation.md)** · Next:
**[Look-forward bias & leakage →](07-look-forward-bias.md)**
