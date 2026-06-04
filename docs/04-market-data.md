# 4. Market data

Everything a strategy knows about the market arrives as **market data**. The
*kind* of data you have determines what you can measure, what features you can
build, and — critically — how realistically you can [simulate
fills](03-orders-and-execution.md#what-assumptions-are-needed-to-simulate-limit-order-fills).
This section walks up the ladder of granularity from level-one quotes to
per-order events, and contrasts what quotes show versus what trades show.

The reference throughout is [Databento's schema
documentation](sources.md#databento), with bar-level examples from
[AlgoSeek](sources.md#algoseek).

## What is level-one market data?

**Level-one (L1)** data is the top of the book plus trades: the **best bid and
best offer** (the [inside market /
BBO](01-market-structure.md#what-is-the-bid-ask-spread-and-midprice)), usually
with their sizes, together with the **trades** that print. It tells you the best
available price on each side and what actually executed, but nothing about depth
*behind* the best price.

The exact contents depend on the feed definition. Databento's **MBP-1** schema is
defined as updates to the best bid and offer, including trades and changes in the
size at the best levels ([Databento](sources.md#databento)); a pure **BBO** feed
gives the top-of-book quote. L1 is enough to compute the spread, the midprice,
and simple [signed-flow](08-feature-engineering.md#what-is-signed-order-flow)
proxies, and it is the minimum needed to mark a position to market.

## What is level-two market data?

**Level-two (L2)** data adds **depth**: it shows the aggregated size resting at
each of several price levels on both sides of the book, not just the best one. So
instead of "best bid 100.00 × 50 lots," you see 100.00 × 50, 99.75 × 120,
99.50 × 200, and so on, plus the corresponding offers — and the trades.

Databento defines L2 as **all trades and updates to aggregated book depth for a
fixed number of price levels** ([Databento](sources.md#databento)). This is the
first level of data rich enough to see how a [market order would walk the
book](03-orders-and-execution.md#how-do-market-orders-interact-with-the-limit-order-book),
to measure [depth imbalance](08-feature-engineering.md#what-is-an-order-book-imbalance-feature),
and to watch liquidity build and evaporate. Note that L2 *aggregates* by price
level — it tells you the total size at a price but not how many individual orders
make it up or what order you would be behind in the
[queue](01-market-structure.md#how-does-price-time-priority-work).

## What is market-by-price data?

**Market-by-price (MBP)** is the precise name for price-level-aggregated book
data. Each update is keyed by **price level** and reports the **total size** at
that level (and, in some feeds, the **order count** at the level), updating as
orders are added, cancelled, or executed. "L2" and "MBP" describe the same idea;
MBP is the schema-precise term and usually comes with a stated depth.

Databento's **MBP-10** schema is defined as order-book events across the **top
ten price levels**, keyed by price, including trades and aggregate market depth
([Databento](sources.md#databento)). MBP gives you size and (sometimes) order
count per level, which supports book-imbalance features and approximate fill
logic — but because it aggregates orders within a level, you still cannot track
an *individual* order's queue position. For that you need market-by-order.

## What is market-by-order data?

**Market-by-order (MBO)** is the most granular feed: it reports **every
individual order event** — each add, modify, cancel, and execution — typically
with an **order ID**. Because you see each order separately, you can **reconstruct
the full limit order book** and, crucially, each order's **queue position** at its
price level.

That queue reconstruction is why MBO supports the most precise [fill
simulation](03-orders-and-execution.md#what-assumptions-are-needed-to-simulate-limit-order-fills):
you can place a hypothetical order, know exactly how much size sits ahead of it,
and watch that size deplete event by event. Databento lists MBO as a distinct
schema separate from the MBP formats ([Databento](sources.md#databento)). The
cost is volume and complexity — MBO is the largest and most demanding data to
store and process — so a project often chooses the *coarsest* data that still
supports the fidelity its strategy and execution model require.

## What is trade data?

**Trade data** (the "tape," or **prints**) records executions: each trade's
**price**, **size**, and **timestamp**, and — when the feed provides it — the
**aggressor side** (whether the trade was buyer- or seller-initiated). Trades are
fundamentally different from quotes: a quote is an *intention* to trade (a resting
bid or offer), while a trade is a *realized* transaction. The tape tells you what
actually changed hands and at what price.

Trade data is the raw material for [OHLCV
bars](05-bars-and-aggregation.md#how-do-trades-become-ohlcv-bars) and for
volume-based features. AlgoSeek's futures bar datasets, for example, include
trade-derived fields such as volume, dollar volume, trade count, and buy/sell
aggressor statistics ([AlgoSeek](sources.md#algoseek)). When the aggressor side
is present you can construct [signed order
flow](08-feature-engineering.md#what-is-signed-order-flow) directly; when it is
absent you must *infer* it (for example with a tick rule or by comparing the trade
price to the prevailing midprice).

## What can level-two data show that trade data cannot?

Trade data only shows what *executed*; L2 shows the *resting intentions* and how
they change. So L2 reveals things trades never can:

- **Resting depth** — how much size is available at each price, and therefore the
  likely [slippage](03-orders-and-execution.md#what-is-a-market-order) of a large
  order before it trades.
- **Queue changes** — orders joining and leaving the book over time.
- **Cancellations** — liquidity that is pulled *without* ever trading, invisible
  on the tape but often highly informative (e.g. spoofing-like flickering, or
  liquidity vanishing ahead of a move).
- **Book imbalance** — the relative weight of bids versus offers, a workhorse
  predictive feature.

Databento's L2 and MBP-10 docs describe exactly these aggregated-depth dynamics
([Databento](sources.md#databento)).

A cancellation is the clearest example of something only L2 reveals: liquidity
vanishes with no trade, and the best price can step away — a move the tape never
shows.

<figure style="margin:1.5rem 0;width:100%">
<svg viewBox="0 0 680 220" role="img" aria-label="Animation of a cancellation: the best ask is pulled with no trade and the best ask steps to a worse price" style="width:100%;height:auto;display:block;color:var(--md-default-fg-color)">
  <title>A cancellation removes liquidity with no trade and the best ask steps away</title>
  <style>
    .mkcx-stage{animation:mkcx-stage 8s infinite}
    .mkcx-b3{animation:mkcx-b3 8s infinite}.mkcx-p3{animation:mkcx-p3 8s infinite}
    .mkcx-pa{animation:mkcx-pa 8s infinite}.mkcx-pb{animation:mkcx-pb 8s infinite}
    .mkcx-note{animation:mkcx-note 8s infinite}.mkcx-s2{animation:mkcx-s2 8s infinite}.mkcx-s1{animation:mkcx-s1 8s infinite}
    @keyframes mkcx-stage{0%,86%{opacity:1}90%,95%{opacity:0}99%,100%{opacity:1}}
    @keyframes mkcx-b3{0%,24%{opacity:.3}44%,86%{opacity:0}92%,100%{opacity:.3}}
    @keyframes mkcx-p3{0%,24%{opacity:1}44%,90%{opacity:.25}93%,100%{opacity:1}}
    @keyframes mkcx-pa{0%,24%{opacity:1}40%,90%{opacity:0}93%,100%{opacity:1}}
    @keyframes mkcx-pb{0%,44%{opacity:0}52%,86%{opacity:1}90%,100%{opacity:0}}
    @keyframes mkcx-note{0%,26%{opacity:0}34%,86%{opacity:1}90%,100%{opacity:0}}
    @keyframes mkcx-s2{0%,40%{opacity:1}48%,90%{opacity:0}93%,100%{opacity:1}}
    @keyframes mkcx-s1{0%,48%{opacity:0}56%,86%{opacity:1}90%,100%{opacity:0}}
    @media (prefers-reduced-motion:reduce){.mkcx-stage,.mkcx-b3,.mkcx-p3,.mkcx-pa,.mkcx-pb,.mkcx-note,.mkcx-s2,.mkcx-s1{animation:none}}
  </style>
  <g class="mkcx-stage">
    <text x="340" y="20" text-anchor="middle" font-size="13" font-weight="600" fill="currentColor">The lone order at the best ask cancels — no trade — so the best ask steps to 100.04</text>
    <g font-size="13" font-weight="600" text-anchor="end">
      <text x="282" y="60" fill="#e5484d">100.05</text><text x="282" y="100" fill="#e5484d">100.04</text>
      <text class="mkcx-p3" x="282" y="140" fill="#e5484d">100.03</text><text x="282" y="184" fill="#2ca35e">100.01</text>
    </g>
    <rect x="300" y="44" width="75" height="24" fill="#e5484d" opacity="0.3"/><text x="310" y="60" font-size="12" fill="currentColor">60</text>
    <rect x="300" y="84" width="137" height="24" fill="#e5484d" opacity="0.3"/><text x="310" y="100" font-size="12" fill="currentColor">110</text>
    <rect class="mkcx-b3" x="300" y="124" width="94" height="24" fill="#e5484d" opacity="0.3"/><text class="mkcx-p3" x="310" y="140" font-size="12" fill="currentColor">75</text>
    <rect x="300" y="168" width="112" height="24" fill="#2ca35e" opacity="0.3"/><text x="310" y="184" font-size="12" fill="currentColor">90</text>
    <text class="mkcx-pa" x="402" y="140" font-size="12" fill="#e5484d">← best ask 100.03</text>
    <text class="mkcx-pb" x="445" y="100" font-size="12" font-weight="600" fill="#e5484d">← new best ask 100.04</text>
    <text class="mkcx-note" x="402" y="140" font-size="11.5" font-weight="600" fill="currentColor">✗ cancelled — no print on the tape</text>
    <text class="mkcx-s2" x="500" y="184" font-size="12" fill="currentColor" opacity="0.85">spread: 0.02</text>
    <text class="mkcx-s1" x="500" y="184" font-size="12" font-weight="600" fill="currentColor">spread: 0.03 — wider</text>
  </g>
</svg>
<figcaption style="font-size:0.8rem;opacity:0.8">The only order resting at the best ask (100.03) is cancelled — pulled with no trade,
so nothing prints on the tape. The best ask steps up to 100.04 and the spread widens
from 0.02 to 0.03. Trade data would show none of this; L2 shows it plainly.</figcaption>
</figure>

## What can trade data show that level-two snapshots may not?

Conversely, trades capture realized economics that a sequence of book snapshots
can miss — especially if the book is sampled rather than streamed event-by-event:

- **Actual executed prices** — where trades *really* happened, including prints
  inside the spread or through multiple levels, rather than just where orders
  rested.
- **Traded volume** — how much genuinely changed hands, the basis for
  participation and capacity estimates.
- **Realized spread proxies** — comparing trade prices to the surrounding
  midprice estimates the spread actually paid.
- **Signed flow** — when the aggressor side is available, who was initiating
  (buyers lifting offers vs. sellers hitting bids), which a static depth snapshot
  does not reveal.

The two views are complementary, which is why richer feeds bundle both: AlgoSeek's
trade bars carry executed-volume and aggressor statistics
([AlgoSeek](sources.md#algoseek)), and Databento's trade and **TBBO** schemas pair
each trade with the prevailing quote ([Databento](sources.md#databento)). A
serious system usually consumes *both* trades and book data so it can see
realized executions and resting liquidity together.

---

Previous: **[← Orders & execution](03-orders-and-execution.md)** · Next: **[Bars
& aggregation →](05-bars-and-aggregation.md)**
