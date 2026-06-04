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

A tight spread and substantial size at the inside market signal a liquid,
competitive market; a wide spread signals that immediacy is expensive. These
quantities are exactly what [level-one market
data](04-market-data.md#what-is-level-one-market-data) delivers, and Databento
defines its MBP-1 schema as updates to this best bid and offer, including trades
and depth changes ([Databento](sources.md#databento)).

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
