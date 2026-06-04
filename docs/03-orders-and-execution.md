# 3. Orders & execution

Trading is done with **orders**. The two fundamental types — market and limit —
embody a single trade-off: do you want certainty of *execution* or certainty of
*price*? You cannot have both. This section defines the order types, shows how
they interact with the [limit order
book](01-market-structure.md#what-is-a-double-sided-auction), and ends with the
assumptions you need to *simulate* fills — the hinge between this section and the
[backtesting engine](06-backtesting-engine.md).

The reference throughout is [Harris](sources.md#harris), with execution detail
from [Cartea et al.](sources.md#cartea)

## What is a market order?

A **market order** is an instruction to trade *immediately* at the best price
currently available, however far it has to reach into the book to do so. It
**demands liquidity** (it "takes" liquidity from resting orders) and so it trades
the spread away in exchange for **immediacy**. Its defining property is execution
certainty: as long as there is *any* liquidity on the other side, a market order
fills.

The cost is price uncertainty, called **slippage**. You are guaranteed to trade
but not at what price: a buy market order pays the best ask, and if its size
exceeds the depth there, it walks up to the next price level and the next,
filling at progressively worse prices. So execution certainty is *conditional on
available liquidity* — in a thin or fast market the realized average price can be
far from the quote you saw. Georgia Tech's ML-for-Trading course frames the
strategy pipeline as running from information gathering all the way to placing
market orders ([CS 7646](sources.md#cs7646)).

## What is a limit order?

A **limit order** specifies a worst acceptable price — a ceiling for a buy, a
floor for a sell — and will not trade beyond it. If it cannot trade immediately
at an acceptable price, it **rests** in the book as displayed liquidity at its
limit price, waiting. It gives you **price control** and the chance to **earn**
the spread rather than pay it.

The costs are subtler than a market order's:

- **Queue position / non-execution risk.** A resting limit order only fills when
  the market trades through its price *and* every order ahead of it in the
  [price-time queue](01-market-structure.md#how-does-price-time-priority-work)
  has been consumed. It may never fill at all if the market moves away.
- **Adverse selection.** Precisely *when* your limit order does fill tends to be
  the worst time for you. Your resting buy is most likely to be hit when sellers
  are aggressively pushing the price down — i.e. just before it falls further. So
  the spread you earn is partly compensation for being run over by informed flow.
  Deciding where in the book to post to balance fill probability against adverse
  selection is a core topic in [Cartea et al.](sources.md#cartea)

This execution-certainty-versus-price-control trade-off is the reason
[Section 9](09-prediction-to-actions.md#when-should-the-system-use-market-orders)
spends so long on *when* to use each type.

## How do market orders interact with the limit order book?

A **marketable** order (a market order, or a limit order priced through the
opposite side) executes by **consuming resting liquidity from the opposite side
of the book**. A buy lifts the best offers; a sell hits the best bids. Matching
follows [price-time
priority](01-market-structure.md#how-does-price-time-priority-work): the incoming
order trades against the best-priced resting order first, then the next-oldest at
that price, and so on.

If the incoming order is larger than the size resting at the best price, it
"walks the book": it fills what it can at the best level, then continues into the
next price level, and the next, until it is fully filled or runs out of book.
This is why a single market order can execute at several prices and why large
orders move the market. Databento's L2 schema is defined precisely to capture
this — trades together with aggregated book-depth updates
([Databento](sources.md#databento)) — which is what lets you reconstruct how an
order would have swept the book.

## What is a partial fill?

A **partial fill** occurs when only part of an order executes and the rest
remains. There are two common causes. For a **marketable** order, the size
available at acceptable prices is less than the order size — it consumes all the
liquidity it can and either walks further into the book (worse prices) or, for a
limit order, stops at its limit and rests the remainder. For a **resting limit
order**, an incoming aggressor may be smaller than your displayed size, filling
part of your order and leaving the balance in the queue.

Partial fills matter because order size can easily exceed the depth at the best
price — the displayed [depth](01-market-structure.md#what-makes-a-market-liquid)
at the inside market is often small relative to a meaningful position. A
realistic system must therefore track *remaining quantity*, decide whether to
chase the rest with more aggressive orders, and account for the blended fill
price across [multiple price
levels](04-market-data.md#what-is-market-by-price-data). Understanding partials
requires data that exposes depth — L2 and MBP documentation
([Databento](sources.md#databento)).

## What assumptions are needed to simulate limit order fills?

Simulating a **market** order fill is comparatively easy: you know there was
liquidity and you sweep the recorded book. Simulating a **limit** order fill is
hard, because whether your resting order *would have* traded depends on events
you can only partially observe. A credible fill simulator must take an explicit
stand on:

- **Queue model.** Where does your order sit at its price level, and how does the
  queue ahead of it deplete? Conservative rules assume you are at the *back* of
  the queue and only fill once cumulative volume at your price exceeds the size
  that was ahead of you.
- **Latency.** The delay between your decision and the order reaching the
  exchange, during which the market moves and your intended queue spot changes.
- **Cancellation timing.** When resting orders ahead of you cancel (improving
  your position) versus when you cancel or reprice your own order.
- **Trade-through logic.** Whether a trade at or through your price counts as
  filling you, and for how much.
- **Data granularity** — the decisive constraint. What you can simulate is bounded
  by what the data shows: with **MBO** (every order event) you can reconstruct the
  queue and model position precisely; with **MBP** (price-level aggregates) you
  can approximate it; with only **bars** you cannot model queue position at all and
  must fall back to crude rules. Databento's schemas distinguish MBO, MBP-10,
  MBP-1, and TBBO, and that choice determines what is even simulatable
  ([Databento](sources.md#databento)).

These assumptions are not cosmetic — they are often the difference between a
backtest that looks profitable and one that is. They reappear as core
[realism assumptions](06-backtesting-engine.md#what-assumptions-determine-backtest-realism)
in the engine design, and as a prime source of
[look-forward bias](07-look-forward-bias.md) if the simulator is optimistic about
when your orders fill.

---

Previous: **[← Futures contracts](02-futures-contracts.md)** · Next: **[Market
data →](04-market-data.md)**
