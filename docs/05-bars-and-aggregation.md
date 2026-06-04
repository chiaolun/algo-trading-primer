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

The same trades summarize into one bar per interval. Below, each interval's bar is
drawn **both ways** side by side — the **candlestick** and the leaner **OHLC bar** —
with dotted verticals marking the interval boundaries.

<figure style="margin:1.5rem 0;width:100%">
<svg viewBox="0 0 700 320" role="img" aria-label="A series of trades on the left aggregated into one bar per interval on the right, shown as a candlestick beside an OHLC bar" style="width:100%;height:auto;display:block;color:var(--md-default-fg-color)">
  <title>Trades become one bar per interval — candlestick beside OHLC bar</title>
  <g stroke="currentColor" opacity="0.12"><line x1="60" y1="40" x2="680" y2="40"/><line x1="60" y1="156" x2="680" y2="156"/><line x1="60" y1="271" x2="680" y2="271"/></g>
  <g fill="currentColor" font-size="11" opacity="0.6" text-anchor="end"><text x="50" y="44">100.8</text><text x="50" y="160">100.4</text><text x="50" y="275">100.0</text></g>
  <text x="195" y="22" text-anchor="middle" font-size="13" font-weight="600" fill="currentColor">A series of trades…</text>
  <text x="545" y="22" text-anchor="middle" font-size="13" font-weight="600" fill="currentColor">…one bar per interval, drawn two ways</text>
  <g stroke="currentColor" opacity="0.45" stroke-dasharray="1.5 4"><line x1="70" y1="36" x2="70" y2="300"/><line x1="156" y1="36" x2="156" y2="300"/><line x1="243" y1="36" x2="243" y2="300"/><line x1="330" y1="36" x2="330" y2="300"/></g>
  <polyline fill="none" stroke="#6366f1" stroke-width="2" points="75,213 90,170 105,242 120,141 135,98 148,156 162,156 177,69 192,170 207,271 222,228 236,242 250,242 265,257 282,184 300,127 316,170 326,156"/>
  <g fill="#6366f1"><circle cx="75" cy="213" r="2.4"/><circle cx="90" cy="170" r="2.4"/><circle cx="105" cy="242" r="2.4"/><circle cx="120" cy="141" r="2.4"/><circle cx="135" cy="98" r="2.4"/><circle cx="148" cy="156" r="2.4"/><circle cx="162" cy="156" r="2.4"/><circle cx="177" cy="69" r="2.4"/><circle cx="192" cy="170" r="2.4"/><circle cx="207" cy="271" r="2.4"/><circle cx="222" cy="228" r="2.4"/><circle cx="236" cy="242" r="2.4"/><circle cx="250" cy="242" r="2.4"/><circle cx="265" cy="257" r="2.4"/><circle cx="282" cy="184" r="2.4"/><circle cx="300" cy="127" r="2.4"/><circle cx="316" cy="170" r="2.4"/><circle cx="326" cy="156" r="2.4"/></g>
  <g fill="currentColor" font-size="11" opacity="0.6" text-anchor="middle"><text x="113" y="314">09:30</text><text x="199" y="314">09:31</text><text x="286" y="314">09:32</text></g>
  <text x="362" y="150" text-anchor="middle" font-size="11.5" fill="currentColor" opacity="0.7">aggregate</text>
  <path d="M346,165 L378,165 M371,159 L378,165 L371,171" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.7"/>
  <g><rect x="438" y="41" width="8" height="8" fill="#2ca35e" opacity="0.5" stroke="#2ca35e"/><line x1="442" y1="36" x2="442" y2="52" stroke="#2ca35e" stroke-width="1.2"/><text x="452" y="49" font-size="11" fill="currentColor" opacity="0.75">candlestick</text></g>
  <g stroke="#2ca35e" stroke-width="2"><line x1="566" y1="36" x2="566" y2="52"/><line x1="560" y1="48" x2="566" y2="48"/><line x1="566" y1="40" x2="572" y2="40"/></g>
  <text x="578" y="49" font-size="11" fill="currentColor" opacity="0.75">OHLC bar</text>
  <line x1="443" y1="98" x2="443" y2="242" stroke="#2ca35e" stroke-width="1.5"/><rect x="434" y="156" width="18" height="57" fill="#2ca35e" opacity="0.5" stroke="#2ca35e"/>
  <g stroke="#2ca35e" stroke-width="2" fill="none"><line x1="477" y1="98" x2="477" y2="242"/><line x1="469" y1="213" x2="477" y2="213"/><line x1="477" y1="156" x2="485" y2="156"/></g>
  <line x1="543" y1="69" x2="543" y2="271" stroke="#e5484d" stroke-width="1.5"/><rect x="534" y="156" width="18" height="86" fill="#e5484d" opacity="0.5" stroke="#e5484d"/>
  <g stroke="#e5484d" stroke-width="2" fill="none"><line x1="577" y1="69" x2="577" y2="271"/><line x1="569" y1="156" x2="577" y2="156"/><line x1="577" y1="242" x2="585" y2="242"/></g>
  <line x1="643" y1="127" x2="643" y2="257" stroke="#2ca35e" stroke-width="1.5"/><rect x="634" y="156" width="18" height="86" fill="#2ca35e" opacity="0.5" stroke="#2ca35e"/>
  <g stroke="#2ca35e" stroke-width="2" fill="none"><line x1="677" y1="127" x2="677" y2="257"/><line x1="669" y1="242" x2="677" y2="242"/><line x1="677" y1="156" x2="685" y2="156"/></g>
  <g stroke="currentColor" opacity="0.35" stroke-dasharray="1.5 3"><line x1="425" y1="98" x2="489" y2="98"/><line x1="425" y1="156" x2="489" y2="156"/><line x1="425" y1="213" x2="489" y2="213"/><line x1="425" y1="242" x2="489" y2="242"/></g>
  <g fill="currentColor" font-size="10.5" opacity="0.85" font-weight="600"><text x="491" y="102">H</text><text x="491" y="160">C</text><text x="491" y="217">O</text><text x="491" y="246">L</text></g>
  <g fill="currentColor" font-size="11" opacity="0.6" text-anchor="middle"><text x="460" y="314">09:30</text><text x="560" y="314">09:31</text><text x="660" y="314">09:32</text></g>
</svg>
<figcaption style="font-size:0.8rem;opacity:0.8">Left: the trade line, split by dotted verticals into three one-minute intervals.
Right: each interval reduced to one bar, shown as a <strong>candlestick</strong>
(filled body = open→close, wicks = high/low) beside an <strong>OHLC bar</strong>
(line = low→high, left tick = open, right tick = close). Both encode the same
O/H/L/C — interval 1's four levels are guided across. Green = close ≥ open, red below.</figcaption>
</figure>

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
