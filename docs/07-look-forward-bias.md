# 7. Look-forward bias & leakage

This is the section that separates a backtest you can act on from one that will
quietly destroy capital. **Look-forward bias** (lookahead, leakage) is the use,
in simulation, of information that would not have been available at that moment in
live trading. It is the single most common reason a beautiful backtest fails in
production. Every realism assumption from the [engine
design](06-backtesting-engine.md#what-assumptions-determine-backtest-realism) is
ultimately a defense against it.

The reference throughout is [López de Prado](sources.md#lopezdeprado), with
[QuantStart](sources.md#quantstart) on event-driven discipline.

## What is look-forward bias?

**Look-forward bias** is using information in a simulation *before it would have
been available* in real time. The backtest "peeks" at the future — even by a few
seconds — and acts on knowledge the live system could not have had. Because the
peeked-at information is correlated with the very outcome you are trying to
predict, lookahead almost always *flatters* results, often dramatically.

It is insidious because it usually arises from innocent-looking code, not
deliberate cheating: joining a feature table on the wrong timestamp, using a
revised data value instead of the originally-reported one, normalizing with
statistics computed over the whole sample, or assuming a fill at a price that was
only known later. [López de Prado](sources.md#lopezdeprado) and
[QuantStart](sources.md#quantstart) both treat avoiding lookahead as a central
design goal — and the event-driven architecture is partly a *structural* defense
against it.

## What is the difference between event time, exchange time, receive time, decision time, and execution time?

Leakage is fundamentally a **timestamp** problem, so you must be precise about
*which* time you mean. A single market occurrence carries several timestamps, and
they are not equal:

- **Event time** — when the thing happened in the world (e.g. an order matched).
- **Exchange time** — when the exchange's matching engine recorded and stamped it.
- **Receive time** — when *your* system actually received the message, after
  network and feed-handler latency.
- **Decision time** — when your strategy evaluates and chooses an action.
- **Execution time** — when the resulting order reaches the exchange and fills.

The discipline that prevents leakage: **every feature and label must be tagged
with an availability timestamp**, and a decision at time _t_ may only consume
inputs whose availability time is `≤ t`. In live trading you can only act on
*receive* time, never *exchange* time — so a backtest that uses exchange
timestamps as if they were instantly available understates latency and leaks. This
follows the internal data-model discipline and [Databento's schema
conventions](sources.md#databento) for the various timestamps.

## Why is same-bar execution dangerous?

**Same-bar execution** is the most common lookahead bug. As established in
[Section 5](05-bars-and-aggregation.md#why-can-bars-create-look-forward-risk), a
bar's **High, Low, Close, and Volume are only known when the bar closes.** If you
generate a signal from a bar's close — or worse, use its high/low — and then book
a trade *within that same bar* (at its open, or at a favorable intrabar price),
you have used end-of-interval information to act at the start of the interval. You
are trading on the future.

The damage is severe because the leaked quantities are exactly the ones most
correlated with profit: knowing a bar's high before "buying at the open" lets the
backtest sell at the peak it could not have foreseen. The fix is the rule from
Section 5: **decide on closed bars only, act on the next bar.** [OHLC
definitions](05-bars-and-aggregation.md#what-is-an-ohlc-bar) make the timing
explicit, and [López de Prado](sources.md#lopezdeprado) catalogs same-bar
execution among the backtesting dangers.

## How can continuous futures construction leak information?

Because [futures expire](02-futures-contracts.md#what-does-expiry-mean), analysts
stitch successive contracts into a single **continuous** series — and that
stitching is a sneaky leakage vector in two ways.

First, **roll selection**: if you decide *which* contract is active using
information that was not available in real time — say, picking the roll date by
the day volume or open interest peaked, which you only know in hindsight — your
[active-contract series](02-futures-contracts.md#how-should-a-backtester-select-the-active-equity-index-futures-contract)
encodes the future. The roll must be driven by a rule evaluable from
*past* data only (or by a fixed schedule like the [CME roll
date](sources.md#cme)).

Second, **back-adjustment**: to remove the price jump at each roll, continuous
series are often *back-adjusted* — historical prices are shifted by the roll gap.
But the size of that gap is only known *at* the roll, so a naively back-adjusted
history embeds future roll information into past prices, and absolute price levels
(and anything derived from them, like fixed-price thresholds) become
contaminated. The safe approach is to define continuous-contract construction with
**historical availability** in mind: only roll on past-observable signals, and be
explicit about whether features use raw or adjusted prices.

## What is data snooping or backtest overfitting?

**Data snooping** (backtest overfitting) is a leakage of a different kind — not
through time, but through *repeated trial*. If you test many strategies, or tune
many parameters, against the same historical data and keep the best, the winner's
strong "out-of-sample" performance is partly luck that will not repeat. You have
implicitly fit the noise of that specific history. The more configurations you
try, the more inflated the apparent best result, even with no single-run
lookahead.

[López de Prado's backtesting section](sources.md#lopezdeprado) covers exactly
this — the dangers of backtesting and the cross-validation methods meant to
detect it. Practical defenses: limit the number of trials, track how many you
ran, prefer simple/economically-motivated strategies, hold out data you truly
never touch, and discount in-sample performance accordingly. This connects to the
honest evaluation discussed in [Section 9](09-prediction-to-actions.md#how-do-you-evaluate-the-strategy-after-execution-costs)
and the audit in the [homework](10-homework.md#can-you-prove-the-system-does-not-use-future-data).

## How should validation be done for time series?

Standard random **K-fold** cross-validation is *wrong* for trading data, for two
reasons: it trains on future data to predict the past (lookahead), and overlapping
labels let information bleed between folds. Time-series validation must respect the
arrow of time:

- **Chronological splits** — always train on the past and test on a later,
  strictly subsequent period.
- **Walk-forward testing** — repeatedly train up to time _t_, test on the next
  window, then roll _t_ forward, mimicking how a model is periodically retrained
  live.
- **Purging** — drop training samples whose label horizon overlaps the test
  period, so a label that "knows" about the test window cannot leak into training.
- **Embargoing** — additionally exclude a buffer of samples immediately after the
  test period to block serial-correlation leakage across the boundary.
- **Avoid random K-fold when labels overlap** — overlapping multi-step return
  labels share information across folds and break the i.i.d. assumption K-fold
  relies on.

These methods come from [López de Prado](sources.md#lopezdeprado) and the broader
purged-cross-validation literature ([overfitting in the ML
era](sources.md#overfitting-paper)). They are what make [Section 8's
labels](08-feature-engineering.md#what-is-the-prediction-target) evaluable without
fooling yourself.

---

Previous: **[← Backtesting engine](06-backtesting-engine.md)** · Next: **[Feature
engineering & regression framing →](08-feature-engineering.md)**
