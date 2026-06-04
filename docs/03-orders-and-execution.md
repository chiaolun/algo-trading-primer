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

The animation below shows it: a market buy larger than the best-ask level sweeps up
through two price levels, executing at a worse average price (slippage) and leaving
a higher best ask behind.

<figure style="margin:1.5rem 0;width:100%">
<svg viewBox="0 0 680 268" role="img" aria-label="Animation of a large market buy order walking up through several ask price levels" style="width:100%;height:auto;display:block;color:var(--md-default-fg-color)">
  <title>A market order larger than the top level walks the book</title>
  <style>
    .mkwo-stage{animation:mkwo-stage 9s infinite}
    .mkwo-bar{transform-box:fill-box;transform-origin:left center}
    .mkwo-b3{animation:mkwo-b3 9s infinite}
    .mkwo-b4{animation:mkwo-b4 9s infinite}
    .mkwo-arrow{animation:mkwo-arrow 9s infinite}
    .mkwo-n3{animation:mkwo-n3 9s infinite}
    .mkwo-n4a{animation:mkwo-n4a 9s infinite}
    .mkwo-n4b{animation:mkwo-n4b 9s infinite}
    .mkwo-pa{animation:mkwo-pa 9s infinite}
    .mkwo-f1{animation:mkwo-f1 9s infinite}
    .mkwo-f2{animation:mkwo-f2 9s infinite}
    .mkwo-sum{animation:mkwo-sum 9s infinite}
    @keyframes mkwo-stage{0%,86%{opacity:1}90%,95%{opacity:0}99%,100%{opacity:1}}
    @keyframes mkwo-b3{0%,18%{transform:scaleX(1)}26%,89%{transform:scaleX(0)}92%,100%{transform:scaleX(1)}}
    @keyframes mkwo-b4{0%,30%{transform:scaleX(1)}40%,89%{transform:scaleX(.318)}92%,100%{transform:scaleX(1)}}
    @keyframes mkwo-arrow{0%,26%{transform:translateY(0)}40%,86%{transform:translateY(-40px)}92%,100%{transform:translateY(0)}}
    @keyframes mkwo-n3{0%,18%{opacity:1}26%,90%{opacity:0}93%,100%{opacity:1}}
    @keyframes mkwo-n4a{0%,30%{opacity:1}40%,90%{opacity:0}93%,100%{opacity:1}}
    @keyframes mkwo-n4b{0%,30%{opacity:0}40%,86%{opacity:1}90%,100%{opacity:0}}
    @keyframes mkwo-pa{0%,18%{opacity:1}24%,90%{opacity:0}93%,100%{opacity:1}}
    @keyframes mkwo-f1{0%,20%{opacity:0}26%,86%{opacity:1}90%,100%{opacity:0}}
    @keyframes mkwo-f2{0%,34%{opacity:0}40%,86%{opacity:1}90%,100%{opacity:0}}
    @keyframes mkwo-sum{0%,46%{opacity:0}52%,86%{opacity:1}90%,100%{opacity:0}}
    @media (prefers-reduced-motion:reduce){.mkwo-stage,.mkwo-b3,.mkwo-b4,.mkwo-arrow,.mkwo-n3,.mkwo-n4a,.mkwo-n4b,.mkwo-pa,.mkwo-f1,.mkwo-f2,.mkwo-sum{animation:none}}
  </style>
  <g class="mkwo-stage">
    <text x="340" y="20" text-anchor="middle" font-size="13" font-weight="600" fill="currentColor">Market BUY 150 lots — larger than the best-ask size, so it walks the book</text>
    <g font-size="13" font-weight="600" text-anchor="end">
      <text x="120" y="68" fill="#e5484d">100.05</text>
      <text x="120" y="108" fill="#e5484d">100.04</text>
      <text x="120" y="148" fill="#e5484d">100.03</text>
      <text x="120" y="204" fill="#2ca35e">100.01</text>
    </g>
    <rect x="140" y="52" width="174.5" height="24" fill="#e5484d" opacity="0.3"/>
    <text x="150" y="68" font-size="12.5" fill="currentColor">60</text>
    <rect class="mkwo-bar mkwo-b4" x="140" y="92" width="320" height="24" fill="#e5484d" opacity="0.3"/>
    <text class="mkwo-n4a" x="150" y="108" font-size="12.5" fill="currentColor">110</text>
    <text class="mkwo-n4b" x="150" y="108" font-size="12.5" font-weight="600" fill="currentColor">35</text>
    <rect class="mkwo-bar mkwo-b3" x="140" y="132" width="218" height="24" fill="#e5484d" opacity="0.3"/>
    <text class="mkwo-n3" x="150" y="148" font-size="12.5" fill="currentColor">75</text>
    <text class="mkwo-pa" x="366" y="148" font-size="12" fill="#e5484d">← best ask</text>
    <rect x="140" y="188" width="261.8" height="24" fill="#2ca35e" opacity="0.3"/>
    <text x="150" y="204" font-size="12.5" fill="currentColor">90</text>
    <g class="mkwo-arrow" fill="#0ea5e9">
      <path d="M124,137 L124,151 L138,144 Z"/>
    </g>
    <text class="mkwo-f1" x="480" y="148" font-size="12.5" fill="currentColor">✓ 75 filled @ 100.03</text>
    <text class="mkwo-f2" x="480" y="108" font-size="12.5" fill="currentColor">✓ 75 filled @ 100.04</text>
    <text class="mkwo-sum" x="140" y="246" font-size="12.5" font-weight="600" fill="currentColor">Filled 150 @ avg 100.034 — 1 tick of slippage; new best ask 100.04 (35 left)</text>
  </g>
</svg>
<figcaption style="font-size:0.8rem;opacity:0.8">A 150-lot market buy is larger than the 75 resting at the best ask (100.03). It
takes all 75 there, then walks up to 100.04 for the remaining 75 — executing at two
prices for an average of 100.034 (one tick of slippage) and leaving 100.04 as the
new best ask. (Animation loops; it respects <em>reduced-motion</em> settings.)</figcaption>
</figure>

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
