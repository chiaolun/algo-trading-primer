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

A crucial input to this trade-off is **alpha decay** — the edge lost to the delay
between when a signal fires and when you actually execute; the longer you wait, the
more of the edge is gone. Decay rates vary by strategy and dictate how to trade. A
fast-decaying signal (often high-Sharpe but low-capacity, e.g. statistical
arbitrage) must be traded *aggressively* to capture its edge before it evaporates,
incurring more market impact and [transaction
costs](#how-do-you-evaluate-the-strategy-after-execution-costs); a slow-decaying
signal (moderate-Sharpe, higher-capacity, e.g. macro or long/short equity) can be
worked *patiently* at far lower cost. The real objective is to **maximize net
alpha** — gross alpha minus trading costs: trade too fast and impact eats the edge,
trade too slowly and decay does, and there is a "just right" speed in between that
depends on the decay rate relative to the cost of trading. AQR's [*Transactions
Costs: Practical Application*](sources.md#aqr-transaction-costs) works through this
balancing act.

The first picture is decay itself — different strategies bleed their edge at very
different rates:

<figure style="margin:1.5rem auto;text-align:center">
<svg viewBox="0 0 640 400" role="img" aria-label="Alpha decay curves for a fast- and a slow-decaying signal" style="max-width:100%;height:auto;color:var(--md-default-fg-color)">
  <title>Different strategies exhibit different alpha decay rates</title>
  <!-- axes -->
  <line x1="60" y1="30" x2="60" y2="350" stroke="currentColor" stroke-width="1" opacity="0.5"/>
  <line x1="60" y1="350" x2="620" y2="350" stroke="currentColor" stroke-width="1" opacity="0.5"/>
  <!-- y ticks + labels -->
  <g stroke="currentColor" stroke-width="1" opacity="0.5">
    <line x1="56" y1="350" x2="60" y2="350"/><line x1="56" y1="270" x2="60" y2="270"/>
    <line x1="56" y1="190" x2="60" y2="190"/><line x1="56" y1="110" x2="60" y2="110"/>
    <line x1="56" y1="30" x2="60" y2="30"/>
  </g>
  <g fill="currentColor" font-size="12" opacity="0.75" text-anchor="end">
    <text x="52" y="354">0.0</text><text x="52" y="274">0.4</text>
    <text x="52" y="194">0.8</text><text x="52" y="114">1.2</text><text x="52" y="34">1.6</text>
  </g>
  <!-- x ticks + labels -->
  <g fill="currentColor" font-size="12" opacity="0.75" text-anchor="middle">
    <text x="60" y="368">0</text><text x="247" y="368">10</text>
    <text x="433" y="368">20</text><text x="620" y="368">30</text>
  </g>
  <!-- axis titles -->
  <text x="340" y="392" fill="currentColor" font-size="12.5" opacity="0.85" text-anchor="middle">Days between signal and execution</text>
  <text x="18" y="190" fill="currentColor" font-size="12.5" opacity="0.85" text-anchor="middle" transform="rotate(-90 18 190)">Gross Sharpe ratio</text>
  <!-- slow-decaying signal -->
  <polyline fill="none" stroke="#14b8a6" stroke-width="2.5" points="60,230 200,233 340,238 480,242 620,246"/>
  <!-- fast-decaying signal -->
  <polyline fill="none" stroke="#6366f1" stroke-width="2.5" points="60,50 97,170 135,250 172,294 209,320 247,336 300,344 340,346 620,350"/>
  <!-- inline labels -->
  <text x="150" y="120" fill="#6366f1" font-size="12.5" font-weight="600">Fast decay (e.g. stat-arb)</text>
  <text x="350" y="225" fill="#14b8a6" font-size="12.5" font-weight="600">Slow decay (e.g. macro)</text>
</svg>
<figcaption style="font-size:0.8rem;opacity:0.8">A fast-decaying, high-Sharpe signal is nearly gone within days; a slow-decaying
signal holds its edge for weeks. Original illustration of the concept in
<a href="https://www.aqr.com/Insights/Research/White-Papers/Transactions-Costs-Practical-Application">AQR, <em>Transactions Costs: Practical Application</em></a> (Exhibit 4).</figcaption>
</figure>

The second picture is the consequence — because trading faster costs more market
impact while waiting forfeits decaying alpha, **net** alpha (gross minus cost) is
humped, peaking at an intermediate trading speed:

<figure style="margin:1.5rem auto;text-align:center">
<svg viewBox="0 0 640 400" role="img" aria-label="Gross alpha, transaction cost, and net alpha versus trading speed" style="max-width:100%;height:auto;color:var(--md-default-fg-color)">
  <title>Trading skill is about maximizing net alpha</title>
  <!-- axes -->
  <line x1="60" y1="30" x2="60" y2="350" stroke="currentColor" stroke-width="1" opacity="0.5"/>
  <line x1="60" y1="350" x2="620" y2="350" stroke="currentColor" stroke-width="1" opacity="0.5"/>
  <!-- y ticks + labels -->
  <g stroke="currentColor" stroke-width="1" opacity="0.5">
    <line x1="56" y1="350" x2="60" y2="350"/><line x1="56" y1="243" x2="60" y2="243"/>
    <line x1="56" y1="137" x2="60" y2="137"/><line x1="56" y1="30" x2="60" y2="30"/>
  </g>
  <g fill="currentColor" font-size="12" opacity="0.75" text-anchor="end">
    <text x="52" y="354">0.0</text><text x="52" y="247">0.2</text>
    <text x="52" y="141">0.4</text><text x="52" y="34">0.6</text>
  </g>
  <!-- x end labels -->
  <g fill="currentColor" font-size="12" opacity="0.75">
    <text x="60" y="368" text-anchor="start">fast / aggressive</text>
    <text x="620" y="368" text-anchor="end">slow / patient</text>
  </g>
  <text x="340" y="392" fill="currentColor" font-size="12.5" opacity="0.85" text-anchor="middle">Time taken to complete the trade</text>
  <text x="18" y="190" fill="currentColor" font-size="12.5" opacity="0.85" text-anchor="middle" transform="rotate(-90 18 190)">Alpha (%)</text>
  <!-- gross alpha -->
  <polyline fill="none" stroke="#6366f1" stroke-width="2.5" points="60,83 200,99 368,115 508,126 620,131"/>
  <!-- transaction cost -->
  <polyline fill="none" stroke="#f59e0b" stroke-width="2.5" points="60,243 200,291 368,323 508,323 620,323"/>
  <!-- net alpha -->
  <polyline fill="none" stroke="#14b8a6" stroke-width="3" points="60,190 200,158 368,142 508,153 620,158"/>
  <!-- markers on net curve -->
  <g fill="#14b8a6">
    <circle cx="60" cy="190" r="4"/><circle cx="368" cy="142" r="4.5"/><circle cx="620" cy="158" r="4"/>
  </g>
  <g font-size="12" fill="currentColor" opacity="0.9">
    <text x="72" y="184">too fast</text>
    <text x="368" y="130" text-anchor="middle" font-weight="600" fill="#14b8a6">just right</text>
    <text x="612" y="152" text-anchor="end">too slow</text>
  </g>
  <!-- legend -->
  <g font-size="12.5" font-weight="600">
    <line x1="400" y1="58" x2="424" y2="58" stroke="#6366f1" stroke-width="2.5"/>
    <text x="430" y="62" fill="#6366f1">Gross alpha</text>
    <line x1="400" y1="78" x2="424" y2="78" stroke="#f59e0b" stroke-width="2.5"/>
    <text x="430" y="82" fill="#f59e0b">Transaction cost</text>
    <line x1="400" y1="98" x2="424" y2="98" stroke="#14b8a6" stroke-width="3"/>
    <text x="430" y="102" fill="#14b8a6">Net alpha</text>
  </g>
</svg>
<figcaption style="font-size:0.8rem;opacity:0.8">Net alpha is maximized at an intermediate speed: trade too fast and market impact
eats the edge; trade too slowly and alpha decays away. Original illustration of the
concept in
<a href="https://www.aqr.com/Insights/Research/White-Papers/Transactions-Costs-Practical-Application">AQR, <em>Transactions Costs: Practical Application</em></a> (Exhibit 7).</figcaption>
</figure>

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
