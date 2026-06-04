# 5. Bars & aggregation

Raw [trades and quotes](04-market-data.md) arrive at irregular, high-frequency
intervals. To make them tractable for analysis and modeling, they are usually
**aggregated into bars** — fixed summaries over a time interval. Bars are
convenient and ubiquitous, but they introduce a specific and dangerous form of
[look-forward risk](07-look-forward-bias.md) that you must understand before you
trust any bar-based backtest.

OHLC definitions follow [Investopedia](sources.md#investopedia); bar construction
and fields follow [AlgoSeek](sources.md#algoseek); the lookahead warning follows
[López de Prado](sources.md#lopezdeprado) and [QuantStart](sources.md#quantstart).

## What is an OHLC bar?

An **OHLC bar** summarizes price activity over a fixed interval with four prices:
the **Open** (first traded price in the interval), the **High** (maximum), the
**Low** (minimum), and the **Close** (last traded price). Add **Volume** — total
quantity traded — and it becomes an **OHLCV** bar. Each bar also carries a
**start time**, an **end time**, and an implied **bar interval** (1 second, 1
minute, 1 day, etc.).

Investopedia defines OHLC as the open, high, low, and close prices for each period
([Investopedia](sources.md#investopedia)). A bar is a *lossy* compression: it
keeps four prices and a sum, discarding the path the price took inside the
interval. That loss is exactly what creates the lookahead trap below.

## How do trades become OHLCV bars?

Building OHLCV bars from a [trade feed](04-market-data.md#what-is-trade-data) is a
deterministic group-and-reduce:

1. **Sort** trades by timestamp.
2. **Group** them into intervals (e.g. all trades in `09:30:00`–`09:30:59` form
   the 09:30 one-minute bar).
3. Within each interval, reduce: **Open** = price of the *first* trade, **High** =
   *max* trade price, **Low** = *min* trade price, **Close** = price of the *last*
   trade, **Volume** = *sum* of trade sizes.

Richer bars carry more reductions over the same trades: **dollar volume** (sum of
price × size), **trade count**, and **buy/sell aggressor** totals. AlgoSeek's
futures catalog lists exactly such trade-only 1-minute and 1-second OHLC bars with
volume, dollar volume, trade count, and aggressor statistics
([AlgoSeek](sources.md#algoseek)). The key point is that a bar is not known until
its interval **closes** — the High, Low, Close and Volume are only final at the
end of the bar, which is the crux of [same-bar
leakage](07-look-forward-bias.md#why-is-same-bar-execution-dangerous).

## What happens if no trades occur during a bar interval?

If no trades print during an interval there is nothing to reduce, and you must
choose a convention. The common options:

- **Emit an empty / null bar** — record the interval with no OHLC and zero volume.
- **Carry forward the prior close** — set O=H=L=C to the last known close and
  volume to zero, producing a flat bar.
- **Omit the interval entirely** — skip it, leaving a gap in the time index.

None is "correct" in the abstract; the right choice depends on what consumes the
bars. A model that assumes a regular time grid needs filled bars; one that keys
off actual activity may prefer omission. This is an internal data-spec decision,
and you should also check the **vendor's** convention so you do not double-handle
gaps. Carrying forward the close is convenient but be careful: a long run of
carried-forward bars can masquerade as a tradable, liquid series when in fact
nothing traded — a subtle realism trap.

## What is the difference between trade bars, quote bars, and book-derived bars?

Bars can summarize different underlying streams, and the choice changes what the
bar *means*:

- **Trade bars** summarize **executions** — OHLCV from the [tape](04-market-data.md#what-is-trade-data).
  They tell you where trades happened and how much traded.
- **Quote bars** summarize **BBO changes** — e.g. the open/high/low/close of the
  bid, the ask, or the midprice over the interval, often with time-weighted
  averages. They describe quoted prices even when little or nothing trades.
- **Book-derived bars** summarize **depth or imbalance** — e.g. average
  [depth imbalance](08-feature-engineering.md#what-is-an-order-book-imbalance-feature)
  or resting size over the interval, built from
  [L2/MBP](04-market-data.md#what-is-market-by-price-data) data.

Trade bars can be stale or empty in quiet periods (no trades), while quote bars
stay informative because quotes update even without executions; book bars capture
liquidity dynamics neither of the others sees. Databento and AlgoSeek expose the
underlying trade and quote streams these bars are built from
([Databento](sources.md#databento), [AlgoSeek](sources.md#algoseek)).

## Why can bars create look-forward risk?

This is the most important point of the section. A bar's **High, Low, Close, and
total Volume are only known once the interval ends.** So any decision made
*inside* a bar cannot legitimately use those values — they lie in that decision's
future. The classic error is to generate a signal from a bar's close and then
assume you traded *at* that same bar's open, high, or low: you have used
information from the end of the interval to act at its start. That single mistake
can manufacture spectacular, entirely fake backtest profits.

The safe rule is: **a decision made at time _t_ may only use bars that have fully
closed at or before _t_.** If you decide on the 09:30 bar, you act on the 09:31
bar at the earliest. [López de Prado](sources.md#lopezdeprado) treats this under
the dangers of backtesting, and [QuantStart](sources.md#quantstart) notes that
**event-driven** backtesting — treating each market-data receipt as an event you
can only react to *after* it arrives — structurally helps avoid lookahead bias.
That insight is the bridge to the [backtesting engine](06-backtesting-engine.md)
and to [Section 7](07-look-forward-bias.md) on leakage.

---

Previous: **[← Market data](04-market-data.md)** · Next: **[Backtesting engine
design →](06-backtesting-engine.md)**
