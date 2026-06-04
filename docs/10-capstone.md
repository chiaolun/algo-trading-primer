# 10. Capstone system

This final section is not new material — it is the **integration test**. Everything
from [market structure](01-market-structure.md) through [evaluating net
P&L](09-prediction-to-actions.md#how-do-you-evaluate-the-strategy-after-execution-costs)
comes together into one working system, and four demonstrations prove you actually
understand it. If you can build the system, prove it doesn't cheat, explain its
failures, and compare execution variants, you have met the goal of this wiki.

References pull from across the curriculum: [CME roll
materials](sources.md#cme), the [Databento](sources.md#databento) /
[AlgoSeek](sources.md#algoseek) schemas, [QuantStart](sources.md#quantstart),
[Jansen](sources.md#jansen), [CS 7646](sources.md#cs7646),
[Harris](sources.md#harris), [Cartea et al.](sources.md#cartea), and [López de
Prado](sources.md#lopezdeprado).

## Can you build a minimal end-to-end futures trading system?

The first demonstration is the system itself: an
[event-driven](06-backtesting-engine.md#what-is-an-event-driven-backtester) replay
loop that performs, in time order, every step of the [minimum viable
backtester](06-backtesting-engine.md#what-is-the-minimum-viable-futures-backtester):

1. **Read futures market data** ([trades, quotes, or
   bars](04-market-data.md)) in timestamp order.
2. **Select the active contract** by an explicit [roll
   rule](02-futures-contracts.md#how-should-a-backtester-select-the-active-equity-index-futures-contract).
3. **Construct features** known as of each decision time
   ([Section 8](08-feature-engineering.md#what-features-are-reasonable-for-futures-intraday-prediction)).
4. **Make predictions** with a fitted model.
5. **Generate orders** from the
   [target position](09-prediction-to-actions.md#how-does-a-target-position-become-orders).
6. **Simulate fills** under an explicit [fill
   model](03-orders-and-execution.md#what-assumptions-are-needed-to-simulate-limit-order-fills).
7. **Report P&L**, net of fees and slippage.

The roll mechanics come from [CME](sources.md#cme), the schemas from
[Databento/AlgoSeek](sources.md#databento), the event loop from
[QuantStart](sources.md#quantstart), and the modeling from
[Jansen](sources.md#jansen) and [CS 7646](sources.md#cs7646). "Minimal" is the
operative word: a simple signal and crude fill model are fine, as long as every
stage is present and honest.

## Can you prove the system does not use future data?

Building the system is not enough — you must *prove* it has no
[look-forward bias](07-look-forward-bias.md). The deliverable is an **audit table**
that, for representative decisions, lays out the timestamps side by side:

| Field | What it shows |
| --- | --- |
| **Feature availability time** | The latest time any input feature was known |
| **Decision time** | When the strategy acted |
| **Order time** | When the order was sent |
| **Fill time** | When it executed |
| **Label horizon** | The future window the label covers |

The proof is in the ordering: every feature-availability time must be **≤** the
decision time, the order and fill times must follow it, and the label horizon must
lie entirely in the *future* of the decision — never overlapping the features.
This operationalizes the [timestamp
discipline](07-look-forward-bias.md#what-is-the-difference-between-event-time-exchange-time-receive-time-decision-time-and-execution-time)
and is exactly what the [execution
log](09-prediction-to-actions.md#what-should-the-execution-simulator-log) was
designed to make possible. [López de Prado](sources.md#lopezdeprado) treats this
audit as standard practice.

## Can you explain the strategy's failures?

A trader who can only explain wins doesn't understand the strategy. The third
demonstration is a **failure analysis**: dig into where and why the system lost.
Examine:

- **Loss days** — what conditions preceded them.
- **Missed fills** — [limit
  orders](03-orders-and-execution.md#what-is-a-limit-order) that never executed and
  the opportunity cost.
- **Adverse selection** — fills that immediately went against you.
- **Slippage** — where realized prices diverged from decision prices.
- **Roll periods** — performance around the [roll](02-futures-contracts.md#what-does-it-mean-to-roll-a-futures-position),
  where liquidity and prices shift.
- **High-volatility periods** — whether the strategy breaks when the market moves
  fast.
- **Model decay** — whether the edge fades over time as the market adapts.

This synthesizes [Harris](sources.md#harris) on microstructure, [López de
Prado](sources.md#lopezdeprado) on backtest pitfalls, and
[Cartea et al.](sources.md#cartea) on execution. Naming the failure modes is the
difference between a fragile strategy you trust blindly and a robust one you
understand.

## Can you compare market order and limit order variants of the same signal?

The final demonstration isolates the *execution* decision from the *signal*. Take
one signal and run two versions — a [market-order
variant](09-prediction-to-actions.md#when-should-the-system-use-market-orders) and a
[limit-order
variant](09-prediction-to-actions.md#when-should-the-system-use-limit-orders) —
under identical conditions, and compare:

- **Fill rate** — limits miss fills; markets don't.
- **Implementation shortfall** — the gap between the price when you decided and the
  price you achieved.
- **Adverse selection** — worse for passive limit fills.
- **Turnover** and **net P&L** — the bottom line after costs.

The market-order version trades the spread for certainty; the limit version tries
to earn the spread but risks non-fill and adverse selection — and which wins is an
*empirical* question for your signal and market, not something you can settle on
paper. Running both, under the same signal and [evaluation
metrics](09-prediction-to-actions.md#how-do-you-evaluate-the-strategy-after-execution-costs),
is the experiment that ties the whole curriculum together. The execution theory is
in [Harris](sources.md#harris) and [Cartea et al.](sources.md#cartea); the data to
simulate it honestly is the [L2 depth](04-market-data.md#what-is-level-two-market-data)
from [Databento](sources.md#databento).

---

You've reached the end of the path. Revisit any section from **[Home](index.md)**,
or go deeper with the **[Sources](sources.md)**.
