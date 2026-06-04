# 1. Market structure & double-sided auctions

Before you can model a market, you need a clear picture of what a market is, who
participates, and how prices are actually formed. This section builds the
vocabulary — traders, brokers, exchanges, the order book, the bid–ask spread,
price-time priority, and liquidity — that every later section depends on. When
you get to [orders and execution](03-orders-and-execution.md) and
[market data](04-market-data.md), these terms are the foundation.

The canonical reference throughout is [Harris, *Trading and
Exchanges*](sources.md#harris).

## What is a market, and what role does an exchange play?

A **market** is any arrangement that lets buyers and sellers find each other and
agree on prices. The participants fall into a few roles. **Traders** are the
principals who want to buy or sell — they include both *liquidity demanders*,
who want to trade now, and *liquidity suppliers* (dealers and market makers), who
stand ready to take the other side. **Brokers** are agents who arrange trades on
behalf of clients rather than trading for their own account; they route orders,
provide access, and owe a duty of best execution. An **exchange** is the
venue and the rule-maker: it operates the matching system, defines the order
types, sets and enforces priority rules, disseminates prices, and provides the
fairness and transparency guarantees that let strangers trade with confidence.

Markets are commonly classified by *how* trades are arranged. In an
**order-driven market**, all participants submit orders into a central book and a
matching engine pairs them according to public rules — there is no privileged
intermediary who must be on every trade. In a **dealer (quote-driven) market**,
designated dealers post bids and offers and clients trade against those quotes;
the dealer is the counterparty to most trades and earns the spread for providing
immediacy. Most modern electronic futures and equity markets are order-driven
central-limit-order-book (CLOB) markets, which is the model assumed throughout
this wiki.

The reason **rules matter** is that price formation is only trustworthy if
everyone knows how orders are ranked, how trades are matched, when information is
released, and how disputes are resolved. The exchange's rule set is what turns a
pile of competing orders into a single, well-defined "market price."

## What is a double-sided auction?

A **double-sided auction** (or two-sided auction) is the mechanism behind an
order-driven market: buyers and sellers submit their interest *simultaneously and
continuously*, rather than the market clearing at a single moment. Buyers submit
**bids** (the prices and sizes at which they are willing to buy) and sellers
submit **offers** / **asks** (the prices and sizes at which they are willing to
sell). A trade occurs whenever a new buy order is priced at or above the best
existing sell order, or vice versa — the two sides cross.

Contrast this with a **single-sided auction** (like a classic art auction), where
one seller faces many competing buyers. In a double-sided auction both sides are
open at once and the book is continuously updated as orders arrive, execute, and
cancel. The collection of all resting bids and offers is the **limit order book**,
and the auction runs continuously through the trading session. This continuous,
two-sided structure is exactly what makes [limit-order
fills](03-orders-and-execution.md#what-assumptions-are-needed-to-simulate-limit-order-fills)
non-trivial to simulate — a resting order's fate depends on everyone else's
orders arriving over time.

## What is the bid, ask, spread, and midprice?

These terms describe the top of the order book at any instant:

- The **best bid** is the highest price any buyer is currently willing to pay.
- The **best ask** (best offer) is the lowest price any seller is currently
  willing to accept.
- The **bid–ask spread** is the gap between them: `ask − bid`. It is the implicit
  cost of immediacy — to buy *now* you pay the ask and could only sell back at the
  bid, so you start every round-trip down by the spread.
- The **midprice** is the average of the best bid and best ask,
  `(bid + ask) / 2`. It is the conventional reference for "the price" and the
  usual basis for fair-value and return calculations, because it is not biased
  toward either side.
- The **inside market** (or top of book / BBO — best bid and offer) is the best
  bid together with the best ask.
- The **tick size** is the smallest increment by which the price may move — the
  spacing between adjacent rungs of the ladder below.

Traders visualize all of this as a **price ladder** (or depth-of-market): price
levels stacked vertically, with resting bid sizes on one side and ask sizes on the
other.

<figure style="margin:1.5rem auto;text-align:center">
<svg viewBox="0 0 720 400" role="img" aria-label="A price ladder showing bids, asks, the spread, the midprice and the tick size" style="max-width:100%;height:auto;color:var(--md-default-fg-color)">
  <title>Price ladder: bids, asks, spread, midprice and tick size</title>
  <!-- column headers -->
  <g fill="currentColor" font-size="12.5" opacity="0.7" text-anchor="middle">
    <text x="245" y="56">Bids — buy orders</text>
    <text x="370" y="56">Price</text>
    <text x="495" y="56">Asks — sell orders</text>
  </g>
  <!-- ladder frame + dividers -->
  <rect x="170" y="70" width="400" height="280" fill="none" stroke="currentColor" stroke-width="1" opacity="0.25"/>
  <g stroke="currentColor" stroke-width="1" opacity="0.12">
    <line x1="320" y1="70" x2="320" y2="350"/><line x1="420" y1="70" x2="420" y2="350"/>
    <line x1="170" y1="110" x2="570" y2="110"/><line x1="170" y1="150" x2="570" y2="150"/>
    <line x1="170" y1="190" x2="570" y2="190"/><line x1="170" y1="230" x2="570" y2="230"/>
    <line x1="170" y1="270" x2="570" y2="270"/><line x1="170" y1="310" x2="570" y2="310"/>
  </g>
  <!-- best ask / best bid row highlights -->
  <rect x="170" y="150" width="400" height="40" fill="#e5484d" opacity="0.10"/>
  <rect x="170" y="150" width="3" height="40" fill="#e5484d"/>
  <rect x="170" y="230" width="400" height="40" fill="#2ca35e" opacity="0.10"/>
  <rect x="170" y="230" width="3" height="40" fill="#2ca35e"/>
  <!-- ask size bars -->
  <g fill="#e5484d" opacity="0.30">
    <rect x="420" y="81" width="45" height="18"/><rect x="420" y="121" width="82.5" height="18"/>
    <rect x="420" y="161" width="56.25" height="18"/>
  </g>
  <!-- bid size bars -->
  <g fill="#2ca35e" opacity="0.30">
    <rect x="252.5" y="241" width="67.5" height="18"/><rect x="185" y="281" width="135" height="18"/>
    <rect x="222.5" y="321" width="97.5" height="18"/>
  </g>
  <!-- size numbers -->
  <g fill="currentColor" font-size="12.5" text-anchor="middle">
    <text x="442" y="94">60</text><text x="461" y="134">110</text><text x="448" y="174">75</text>
    <text x="286" y="254">90</text><text x="252" y="294">180</text><text x="271" y="334">130</text>
  </g>
  <!-- prices -->
  <g font-size="13" text-anchor="middle" font-weight="600">
    <text x="370" y="94" fill="#e5484d">100.05</text>
    <text x="370" y="134" fill="#e5484d">100.04</text>
    <text x="370" y="174" fill="#e5484d" font-weight="700">100.03</text>
    <text x="370" y="214" fill="currentColor" opacity="0.6">100.02</text>
    <text x="370" y="254" fill="#2ca35e" font-weight="700">100.01</text>
    <text x="370" y="294" fill="#2ca35e">100.00</text>
    <text x="370" y="334" fill="#2ca35e">99.99</text>
  </g>
  <!-- midprice line -->
  <line x1="170" y1="210" x2="600" y2="210" stroke="currentColor" stroke-width="1.2" stroke-dasharray="5 4" opacity="0.55"/>
  <g fill="currentColor" text-anchor="end">
    <text x="162" y="206" font-size="12" opacity="0.85">midprice = 100.02</text>
    <text x="162" y="220" font-size="10" opacity="0.6">(bid + ask) / 2</text>
  </g>
  <!-- spread bracket -->
  <g stroke="#000" stroke-width="0"></g>
  <line x1="600" y1="170" x2="600" y2="250" stroke="currentColor" stroke-width="1.5" opacity="0.8"/>
  <line x1="594" y1="170" x2="600" y2="170" stroke="currentColor" stroke-width="1.5" opacity="0.8"/>
  <line x1="594" y1="250" x2="600" y2="250" stroke="currentColor" stroke-width="1.5" opacity="0.8"/>
  <text x="606" y="166" font-size="12" font-weight="600" fill="#e5484d">best ask 100.03</text>
  <text x="606" y="205" font-size="12" font-weight="600" fill="currentColor">spread = 2 ticks</text>
  <text x="606" y="220" font-size="11" fill="currentColor" opacity="0.75">( = 0.02 )</text>
  <text x="606" y="262" font-size="12" font-weight="600" fill="#2ca35e">best bid 100.01</text>
  <!-- tick bracket -->
  <g stroke="currentColor" stroke-width="0.8" opacity="0.3" stroke-dasharray="2 2">
    <line x1="156" y1="290" x2="170" y2="290"/><line x1="156" y1="330" x2="170" y2="330"/>
  </g>
  <line x1="150" y1="290" x2="150" y2="330" stroke="currentColor" stroke-width="1.5" opacity="0.8"/>
  <line x1="150" y1="290" x2="156" y2="290" stroke="currentColor" stroke-width="1.5" opacity="0.8"/>
  <line x1="150" y1="330" x2="156" y2="330" stroke="currentColor" stroke-width="1.5" opacity="0.8"/>
  <g fill="currentColor" text-anchor="end">
    <text x="144" y="306" font-size="12" font-weight="600">1 tick</text>
    <text x="144" y="321" font-size="11" opacity="0.75">= 0.01</text>
  </g>
</svg>
<figcaption style="font-size:0.8rem;opacity:0.8">Resting buy orders (bids, green) sit below resting sell orders (asks, red). The
best bid (100.01) and best ask (100.03) form the inside market; the gap between them
is the <strong>spread</strong> (here 2 ticks = 0.02), the midprice sits halfway, and
each rung is one <strong>tick</strong> (0.01) apart. The 100.02 level is empty —
inside the spread, where no one is resting.</figcaption>
</figure>

A tight spread and substantial size at the inside market signal a liquid,
competitive market; a wide spread signals that immediacy is expensive. These
quantities are exactly what [level-one market
data](04-market-data.md#what-is-level-one-market-data) delivers, and Databento
defines its MBP-1 schema as updates to this best bid and offer, including trades
and depth changes ([Databento](sources.md#databento)).

The tick size matters more than it first appears, because its magnitude *relative to
price and volatility* shapes how a market trades. Contrast two contracts:

- **A large (binding) tick.** When the tick is big relative to the price, the spread
  is pinned at a single tick almost all the time — liquidity providers would quote
  tighter but are not allowed to. Liquidity instead piles into deep queues at the
  best bid and offer, so [queue position](#how-does-price-time-priority-work) becomes
  the dominant edge and prices move in discrete jumps. The E-mini S&P 500 future
  (ES), whose 0.25-point tick is worth \$12.50, is the classic example: it is almost
  always exactly one tick wide, with thousands of lots queued at the touch.
- **A small (non-binding) tick.** When the tick is tiny relative to the price, the
  spread can be several ticks wide, each level holds little size (depth is spread
  thinly across many rungs), prices move almost continuously, and it is cheap to
  **price-improve** by stepping one tick ahead of the queue. A \$500 stock quoted in
  \$0.01 ticks — a tick of 0.002% of price — behaves this way.

On the ladder above, a smaller tick would mean more, finer rungs with orders spread
thinly across them; a larger tick, fewer rungs with large queues pinned at the
touch. Tick size is an exchange design lever: set it too large and traders pay an
artificially wide spread, too small and the book fragments into noise across hundreds
of rungs.

## How does price-time priority work?

**Price-time priority** is the ranking rule most order-driven markets use to
decide which resting order trades first. Orders are sorted **by price first**:
the most aggressive orders (highest bids, lowest offers) have priority because
they offer the best terms to the other side. **Among orders at the same price,
the earliest-submitted order has priority** — time breaks the tie. An incoming
marketable order is matched against the head of the queue at the best price,
then the next, and so on.

This is why **queue position matters** so much. If you post a [limit
order](03-orders-and-execution.md#what-is-a-limit-order) at the best bid, you
join the back of the queue at that price. You only get filled once every order
ahead of you at that price has traded or cancelled. Two traders quoting the
identical price can have very different fill rates and very different [adverse
selection](03-orders-and-execution.md#what-is-a-limit-order) depending on where
they sit in the queue. Any realistic [fill
simulator](06-backtesting-engine.md#what-state-should-the-backtester-maintain)
must therefore model queue position, not just price — which in turn requires data
that exposes individual orders or at least queue dynamics. Cartea, Jaimungal and
Penalva treat exactly this question of where market makers should post in the
book ([Cartea et al.](sources.md#cartea)).

## What makes a market liquid?

**Liquidity** is the ability to trade quickly, in size, without moving the price
much. It is not a single number; it has several dimensions:

- **Tightness** — a narrow [bid–ask
  spread](#what-is-the-bid-ask-spread-and-midprice), so the cost of immediacy is
  low.
- **Depth** — large displayed size at and near the inside market, so you can
  trade meaningful quantity without walking up the book.
- **Resiliency** — how quickly the book refills and the spread re-tightens after a
  large trade consumes liquidity.
- **Volume** — how much actually trades over time; a proxy for how easily you can
  enter and exit.
- **Immediacy** — how fast you can complete a trade of a given size.

These dimensions can disagree: a market can be tight but thin (narrow spread, little
size), or deep but slow to recover. Liquidity is central to everything
downstream — it determines [slippage](03-orders-and-execution.md#what-is-a-market-order)
on market orders, fill probability on limit orders, and ultimately the *capacity*
of a strategy. For equity-index futures it also explains why [trading concentrates
in one contract](02-futures-contracts.md#for-equity-index-futures-why-is-liquidity-concentrated-in-the-front-or-lead-contract).

---

Next: **[Futures contracts & expiries →](02-futures-contracts.md)**
