# 8. Feature engineering & regression framing

With the data and timing discipline in place, the modeling question becomes: how
do you turn a continuous stream of market events into a **supervised-learning
problem** a regression can solve? This section frames the time series as a table
of rows and columns, picks a prediction target, surveys sensible features for
intraday futures, and stresses two ideas that beginners get wrong — stationarity,
and the distinction between a *forecast* and a *trading rule*.

The course spine is [CS 7646](sources.md#cs7646); the Python ML workflow follows
[Jansen](sources.md#jansen); labeling discipline follows [López de
Prado](sources.md#lopezdeprado).

## How do you convert a time series problem into a regression problem?

You convert it into a table. The key design decision is what a **row** is:

- **Rows are decision times** — the moments at which the strategy could act (every
  bar close, every _N_ seconds, or every event). Each row is one prediction
  opportunity.
- **Columns are features known at the decision time** — every input must have an
  [availability timestamp](07-look-forward-bias.md#what-is-the-difference-between-event-time-exchange-time-receive-time-decision-time-and-execution-time)
  at or before the row's time. This is where leakage is most often introduced, so
  it is the column rule you guard most carefully.
- **The label is a future outcome over a specified horizon** — e.g. the return
  from this decision time to _h_ steps ahead.

Once in this shape, the problem is ordinary supervised learning: fit a function
from features to label. Georgia Tech's CS 7646 lists linear regression among the
statistical ML approaches used for trading decisions
([CS 7646](sources.md#cs7646)). The framing is simple; the discipline is in the
timestamps and in [how you
validate](07-look-forward-bias.md#how-should-validation-be-done-for-time-series).

## What is the prediction target?

The **target** (label) is the future quantity you predict, and the choice shapes
everything downstream. Common options:

- **Future return** — percentage change in price over the horizon. The default,
  and naturally [stationary](#what-does-stationarity-mean-in-this-context).
- **Future midprice change** — change in the
  [midprice](01-market-structure.md#what-is-the-bid-ask-spread-and-midprice),
  avoiding the noise of which side traded.
- **Future trade-price change** — change in executed price, closer to what you
  realize but noisier and contaminated by the spread.
- **Execution-adjusted P&L** — the outcome *after* modeled
  [costs](09-prediction-to-actions.md#how-do-you-evaluate-the-strategy-after-execution-costs),
  the most honest but hardest target.
- **Probability of hitting a threshold** — a classification framing (e.g. will
  price rise by _x_ before falling by _y_), which leads to triple-barrier-style
  labels.

The horizon must match how long you can hold and how fast your signal decays.
[Jansen](sources.md#jansen) and [López de Prado](sources.md#lopezdeprado) both
treat target construction; choosing a target that you can actually *trade* (net of
costs) is what keeps the model honest.

## What features are reasonable for futures intraday prediction?

Good intraday features summarize recent price, flow, and liquidity in forms a
model can generalize from. A reasonable starter set:

- **Lagged returns** — recent returns over several windows, capturing momentum or
  reversal.
- **Rolling / realized volatility** — recent variability, for sizing and as a
  conditioning variable.
- **Spread** — the current [bid–ask
  spread](01-market-structure.md#what-is-the-bid-ask-spread-and-midprice), a
  liquidity and cost signal.
- **Depth imbalance** — relative
  [bid vs. ask depth](#what-is-an-order-book-imbalance-feature), a short-horizon
  pressure signal.
- **Order-flow imbalance / signed volume** — net
  [aggressive buying vs. selling](#what-is-signed-order-flow).
- **Time of day** — intraday seasonality (open, lunch lull, close).
- **Roll-state features** — proximity to the
  [roll](02-futures-contracts.md#what-does-it-mean-to-roll-a-futures-position),
  where liquidity and behavior shift.

[Jansen's book and repository](sources.md#jansen) cover building exactly such
features and feeding them into models from linear regression up to deep
reinforcement learning. Every one of these must be computed from data available
*at* the decision time — rolling windows and lags are convenient precisely because
they look only backward.

## What is an order-book imbalance feature?

**Order-book imbalance** measures the relative weight of resting demand versus
supply. The simplest version uses the top of book:

```
imbalance = (bid_size − ask_size) / (bid_size + ask_size)
```

which ranges from −1 (all offers) to +1 (all bids). A positive imbalance — much
more size resting on the bid than the ask — is often a short-horizon signal that
price will tick up, since there is more demand to absorb. The feature generalizes
to multiple levels by summing (optionally distance-weighting) depth across the top
_k_ price levels on each side.

Computing it requires depth, i.e.
[L2 / MBP-10](04-market-data.md#what-is-market-by-price-data) data — [trade
data alone cannot show resting
depth](04-market-data.md#what-can-level-two-data-show-that-trade-data-cannot).
[Databento's L2 and MBP-10 docs](sources.md#databento) describe the depth updates
this feature is built from.

## What is signed order flow?

**Signed order flow** classifies each trade by which side *initiated* it — a buy
(the aggressor lifted the offer) is +size, a sell (the aggressor hit the bid) is
−size — and aggregates the signed sizes over a window. It measures net buying or
selling *pressure*, which is more informative than raw volume because it has a
direction.

When the feed provides an **aggressor flag**, you use it directly. When it does
not, you must **infer** the side with a rule — the **tick rule** (a trade above
the previous trade is a buy, below is a sell) or the **Lee–Ready** style of
comparing the trade price to the prevailing
[midprice](01-market-structure.md#what-is-the-bid-ask-spread-and-midprice). Such
inference is imperfect, so signed flow built from inferred sides is noisier than
from a true aggressor flag. [AlgoSeek's futures bars](sources.md#algoseek) include
buy/sell aggressor statistics for some products, giving you the signed flow
without inference.

## What does stationarity mean in this context?

A series is (loosely) **stationary** if its statistical properties — mean,
variance — do not drift over time. Raw price levels are emphatically *not*
stationary: a futures price wanders over months and never revisits old levels, so
a model trained on one price regime generalizes badly to another, and absolute
price is meaningless as a feature.

The fix is to transform inputs into stationary forms: **returns** or **log
differences** instead of prices, **normalized spreads** (spread relative to price
or to recent average), **z-scores** of features over a rolling window, and
**relative-depth** measures (the dimensionless [imbalance
ratio](#what-is-an-order-book-imbalance-feature) rather than raw lot counts).
Stationarizing is standard in [Jansen](sources.md#jansen) and the broader
statistical-learning literature; it is also what lets a model fit on one period
remain meaningful in another — provided the transform uses only
[backward-looking](07-look-forward-bias.md#what-is-look-forward-bias) statistics.

## What is the difference between fitting a model and defining a trading rule?

These are two distinct steps, and conflating them is a classic beginner error. A
**model** produces a **forecast** — a number, like "expected return over the next
minute is +3 bps." That is all it does; it has no opinion about positions, risk,
or orders. A **trading rule** (the strategy) maps forecasts into **actions** —
target positions and the orders to reach them — incorporating thresholds, sizing,
risk limits, and costs.

The same forecast can drive very different strategies: trade only when the
predicted edge exceeds costs, scale position size with conviction, or impose
[no-trade bands](09-prediction-to-actions.md#how-does-a-prediction-become-a-position)
to limit turnover. Separating the two lets you evaluate the model (is the forecast
accurate?) independently from the strategy (does acting on it make money net of
costs?). [CS 7646](sources.md#cs7646) and [Jansen](sources.md#jansen) both draw
this line, and it is the bridge into [Section 9](09-prediction-to-actions.md),
which is entirely about turning forecasts into actions.

---

Previous: **[← Look-forward bias](07-look-forward-bias.md)** · Next: **[From
prediction to actions →](09-prediction-to-actions.md)**
