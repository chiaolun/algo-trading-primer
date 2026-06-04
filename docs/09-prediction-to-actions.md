# 9. From prediction to actions

A [forecast](08-feature-engineering.md#what-is-the-difference-between-fitting-a-model-and-defining-a-trading-rule)
is not a trade. This section closes the loop: how a prediction becomes a **target
position**, how a target position becomes **orders**, when to use each order type,
what the execution layer must **log**, and how to **evaluate** the strategy once
real-world costs are subtracted. This is where microstructure, execution, and
modeling all come together.

References: [Harris](sources.md#harris) and [Cartea et al.](sources.md#cartea) for
execution; [CS 7646](sources.md#cs7646) and [Jansen](sources.md#jansen) for the
strategy logic; [López de Prado](sources.md#lopezdeprado) for evaluation.

## How does a prediction become a position?

The strategy maps a forecast to a **target position** — the holding you *want* to
have right now. The mapping typically combines:

- **Thresholds** — only take a position when the predicted edge is large enough to
  beat costs; small forecasts map to flat.
- **Target position / scaling rule** — how conviction translates to size. This may
  be binary (±1 unit), proportional to the forecast, or volatility-scaled so each
  position carries similar risk.
- **Risk limits** — hard caps on position size, gross exposure, and loss, which
  override the raw mapping.
- **No-trade bands** — a hysteresis zone around the current position so small
  forecast wiggles don't trigger churn. Without bands, a noisy forecast generates
  constant [turnover](#how-do-you-evaluate-the-strategy-after-execution-costs) and
  bleeds costs.

The art is mapping a continuous, noisy forecast to a position that is responsive
enough to capture edge but stable enough not to be eaten by trading costs.
[CS 7646](sources.md#cs7646) and [Jansen](sources.md#jansen) both treat this
forecast-to-position step.

## How does a target position become orders?

Orders come from the **difference** between your current position and your target
position. If you hold +2 and want +5, you must buy 3; if you hold +2 and want 0,
you sell 2. The execution layer turns that delta into concrete orders, deciding:

- **Order type** — [market](#when-should-the-system-use-market-orders) for
  immediacy or [limit](#when-should-the-system-use-limit-orders) for price.
- **Price and size** — where to post and how much, possibly slicing a large delta
  into smaller child orders to limit impact.
- **Cancellation and replacement** — managing resting limit orders as the market
  and the target move: cancel stale orders, reprice to stay competitive, and chase
  or back off based on urgency.

This is exactly the execution problem [Cartea, Jaimungal and
Penalva](sources.md#cartea) formalize — trading algorithms for working large
orders, VWAP schedules, market making, and the like. Even a minimal system needs a
basic version: compute the delta, choose a type, send, and reconcile fills against
the target.

## When should the system use market orders?

Reach for a [market
order](03-orders-and-execution.md#what-is-a-market-order) when **immediacy is
worth more than the spread**. Typical cases:

- **Urgency** — the signal is short-lived and decays before a limit order would
  likely fill.
- **Risk reduction** — you need *out* of a position now (a stop, a risk-limit
  breach), where certainty of execution dominates.
- **Crossing the spread to participate** — entering a fast-moving market where
  waiting means missing the move entirely.

You pay the spread and accept [slippage](03-orders-and-execution.md#what-is-a-market-order),
but you get execution certainty conditional on liquidity. Georgia Tech frames the
strategy pipeline as running all the way from information gathering to placing
market orders ([CS 7646](sources.md#cs7646)). The rule of thumb: use market orders
when the cost of *not* trading exceeds the spread.

## When should the system use limit orders?

Reach for a [limit
order](03-orders-and-execution.md#what-is-a-limit-order) when **price matters more
than speed** and you can tolerate non-execution. Typical cases:

- **Spread capture** — you aim to *earn* the spread by providing liquidity rather
  than paying it, the core of market-making-style strategies.
- **Patience** — the signal persists long enough that waiting for a fill is
  acceptable.
- **Cost-sensitive entries** — when the modeled edge is thin and paying the spread
  would erase it.

The costs to manage are [queue
risk](01-market-structure.md#how-does-price-time-priority-work), [adverse
selection](03-orders-and-execution.md#what-is-a-limit-order) (you fill exactly
when the market is about to move against you), and **non-fill risk** (the trade
you wanted never happens), all governed by your **cancellation rules**. Deciding
where in the book to post to balance these is the central question in
[Cartea et al.](sources.md#cartea) and [Harris](sources.md#harris). The
[homework](10-homework.md#can-you-compare-market-order-and-limit-order-variants-of-the-same-signal)
asks you to compare both order types on the same signal precisely because the
trade-off is not obvious in advance.

## What should the execution simulator log?

To evaluate and debug a strategy — and to *prove* it didn't cheat — the simulator
must log a full audit trail for every decision and fill. At minimum:

- **Decision timestamp** and **feature timestamp** — when the decision was made and
  the latest time any input was available (the heart of the [leakage
  audit](07-look-forward-bias.md#what-is-the-difference-between-event-time-exchange-time-receive-time-decision-time-and-execution-time)).
- **Model version** and **prediction** — which model produced what forecast.
- **Target position** — what the strategy wanted.
- **Order timestamp, type, price, quantity** — the order that was sent.
- **Fill timestamp, fill price, remaining quantity** — what actually executed and
  what was left.
- **Fees** and **post-trade state** — costs accrued and the resulting position,
  cash, and P&L.

This is the engineering standard for an auditable execution layer, mirroring the
[state](06-backtesting-engine.md#what-state-should-the-backtester-maintain) the
event-driven engine already maintains ([QuantStart](sources.md#quantstart)). The
feature-timestamp-versus-decision-timestamp pair is what lets you later produce the
[no-future-data audit table](10-homework.md#can-you-prove-the-system-does-not-use-future-data).

## How do you evaluate the strategy after execution costs?

A strategy is only as good as its **net** performance, so evaluation must subtract
every real cost and look beyond a single headline number. Compute:

- **Gross vs. net P&L** — before and after fees and slippage; the gap is the cost
  of trading, and many gross-profitable strategies are net losers.
- **Turnover** — how much you trade; high turnover multiplies costs and signals
  over-trading.
- **Fees and slippage** — commissions, exchange fees, and the difference between
  decision price and fill price.
- **Drawdown** — worst peak-to-trough loss, the survivability measure.
- **Sharpe-like statistics** — [risk-adjusted
  return](11-portfolio-optimization.md#why-risk-adjusted-return), the standard way
  to compare strategies on a common footing.
- **Hit rate and average win/loss** — how often you're right and the payoff
  asymmetry.
- **Inventory exposure** — how much risk you carried to earn the return.
- **Capacity proxies** — how much size the strategy could absorb before its own
  impact erodes the edge.

[López de Prado's backtesting chapters](sources.md#lopezdeprado) and
[Jansen](sources.md#jansen) cover these metrics and the statistical caveats around
them — remembering that an impressive backtest can still be an artifact of
[overfitting](07-look-forward-bias.md#what-is-data-snooping-or-backtest-overfitting).
Net-of-cost evaluation is the honest test the whole pipeline has been building
toward.

---

Previous: **[← Feature engineering](08-feature-engineering.md)** · Next:
**[Homework: build a standalone system →](10-homework.md)**
