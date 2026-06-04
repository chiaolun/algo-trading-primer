# Sources

This page is the bibliography for the wiki. Pages link to the anchored entries
below (for example, `[Harris](sources.md#harris)`). Each entry says what it is
best used for.

## Required sources

### Larry Harris, *Trading and Exchanges* {#harris}

The backbone for market structure: the [double-sided
auction](01-market-structure.md#what-is-a-double-sided-auction), liquidity,
[market orders](03-orders-and-execution.md#what-is-a-market-order) and [limit
orders](03-orders-and-execution.md#what-is-a-limit-order), the bid–ask spread,
brokers, exchanges, and the trading rules that govern it all. Oxford describes
the book as covering trading, traders, marketplaces, and the rules that govern
trading.

- <https://global.oup.com/academic/product/trading-and-exchanges-9780195144703>

### CME Group Education & product pages {#cme}

The authority for [futures contracts](02-futures-contracts.md), expiration, the
[roll](02-futures-contracts.md#what-does-it-mean-to-roll-a-futures-position),
equity-index roll dates, contract specifications, and product-specific
conventions. CME states that futures contracts have a limited lifespan and that
expiration and rollover are central terms; for equity-index futures it gives the
roll date as the Monday before the third Friday of the expiration month.

- Introduction to Futures / Understanding Futures Expiration & Contract Roll:
  <https://www.cmegroup.com/education/courses/introduction-to-futures/understanding-futures-expiration-contract-roll>
- Equity Index Roll Dates:
  <https://www.cmegroup.com/trading/equity-index/rolldates.html>

### Databento documentation {#databento}

Vendor documentation for market-data schemas: L1/BBO,
[L2](04-market-data.md#what-is-level-two-market-data),
[MBP-10](04-market-data.md#what-is-market-by-price-data),
[MBO](04-market-data.md#what-is-market-by-order-data), trades, and TBBO — and how
data granularity determines [what a simulator can
do](03-orders-and-execution.md#what-assumptions-are-needed-to-simulate-limit-order-fills).
Databento defines MBP-1 as updates to the best bid and offer including trades and
depth changes; L2 as all trades and updates to aggregated book depth for a fixed
number of price levels; and MBP-10 as order-book events across the top ten price
levels.

- What's a schema?:
  <https://databento.com/docs/schemas-and-data-formats/whats-a-schema>
- Level 2 (L2) market data:
  <https://databento.com/microstructure/level-2-market-data>
- Market by price (MBP-10):
  <https://databento.com/docs/schemas-and-data-formats/mbp-10>

### QuantStart, event-driven backtesting series {#quantstart}

The architecture reference for an [event-driven
backtester](06-backtesting-engine.md#what-is-an-event-driven-backtester): market
data events, signal events, order events, fill events, and portfolio updates.
QuantStart describes drip-feeding market data as events to replicate how an
order-management and portfolio system behaves, which also helps avoid
[lookahead bias](07-look-forward-bias.md).

- Event-Driven Backtesting with Python — Part I:
  <https://www.quantstart.com/articles/Event-Driven-Backtesting-with-Python-Part-I/>

### Marcos López de Prado, *Advances in Financial Machine Learning* {#lopezdeprado}

The reference for [backtesting risks](07-look-forward-bias.md): leakage,
overfitting, data snooping, and time-series cross-validation (purging,
embargoing). O'Reilly's table of contents lists a full backtesting section
covering the dangers of backtesting, cross-validation, synthetic data, backtest
statistics, and strategy risk.

- <https://www.oreilly.com/library/view/advances-in-financial/9781119482086/p03.xhtml>

### Georgia Tech CS 7646: Machine Learning for Trading {#cs7646}

A course spine for the implementation exercises — regression, ML, and turning
information into trading decisions. Georgia Tech describes the course as
addressing the real-world challenges of implementing ML-based trading
strategies, and lists linear regression among the approaches used.

- <https://omscs.gatech.edu/cs-7646-machine-learning-trading>

## Optional advanced sources

### Cartea, Jaimungal & Penalva, *Algorithmic and High-Frequency Trading* {#cartea}

For execution: [limit-order
placement](09-prediction-to-actions.md#when-should-the-system-use-limit-orders),
market making, VWAP-style scheduling, and microstructure modeling. The Cambridge
front matter notes the book discusses how market makers choose where to post
limit orders in the book; the publisher describes coverage of large-order
execution, VWAP schedules, pairs trading, and dark pools.

- Front matter:
  <https://assets.cambridge.org/97811070/91146/frontmatter/9781107091146_frontmatter.pdf>
- Overview: <https://books.google.nl/books?id=5dMmCgAAQBAJ>

### Stefan Jansen, *Machine Learning for Algorithmic Trading* {#jansen}

For the Python ML workflow: [feature
engineering](08-feature-engineering.md#what-features-are-reasonable-for-futures-intraday-prediction),
model-driven strategies, and backtesting examples. The repository says the book
covers techniques from linear regression to deep reinforcement learning and
shows how to build, backtest, and evaluate strategies driven by model
predictions.

- <https://github.com/stefan-jansen/machine-learning-for-trading>

### AlgoSeek documentation & catalog {#algoseek}

For examples of futures [trade
bars](05-bars-and-aggregation.md#how-do-trades-become-ohlcv-bars), quote bars,
trade counts, aggressor statistics, volume, and data-format conventions. The
AlgoSeek futures catalog lists trade-only 1-minute and 1-second OHLC bars with
volume, dollar volume, trade count, and buy/sell aggressor statistics; the
dataset includes price, volume, open interest, and expiry.

- <https://algoseek.com/data-sets/list>

## Portfolio construction & risk

### Harry Markowitz — *Portfolio Selection* (1952) {#markowitz}

The origin of [mean-variance
optimization](11-portfolio-optimization.md#markowitz-mean-variance-optimization):
treat expected return as desirable and variance as undesirable, and choose
portfolio weights on the efficient frontier. The foundation of modern portfolio
theory.

- <https://www.jstor.org/stable/2975974>

### William F. Sharpe — *The Sharpe Ratio* (1994) {#sharpe}

The reward-to-variability ratio for [risk-adjusted
return](11-portfolio-optimization.md#the-sharpe-ratio): excess return per unit of
standard deviation, and the lingua franca for comparing return streams.

- <https://web.stanford.edu/~wfsharpe/art/sr/sr.htm>

### Bailey & López de Prado — Probabilistic & Deflated Sharpe Ratio {#psr}

For the [Probabilistic Sharpe
Ratio](11-portfolio-optimization.md#the-probabilistic-sharpe-ratio): a Sharpe
estimate is uncertain, so PSR gives the probability the true Sharpe exceeds a
benchmark given sample length, skew, and kurtosis; the Deflated Sharpe Ratio
corrects for the number of trials, connecting to
[overfitting](07-look-forward-bias.md#what-is-data-snooping-or-backtest-overfitting).
Also developed in [*Advances in Financial Machine
Learning*](sources.md#lopezdeprado).

- <https://papers.ssrn.com/sol3/papers.cfm?abstract_id=1821643>

### Grinold & Kahn — *Active Portfolio Management* {#grinoldkahn}

For the [Information
ratio](11-portfolio-optimization.md#sharpes-cousins), the fundamental law of
active management, and [factor
models](11-portfolio-optimization.md#factor-analysis-what-a-stream-is-made-of) for
risk decomposition.

- <https://www.mhprofessional.com/active-portfolio-management-a-quantitative-approach-for-producing-superior-returns-and-controlling-risk-9780070248823-usa>

### Andrew Ang — *Asset Management: A Systematic Approach to Factor Investing* {#ang}

For [factor
analysis](11-portfolio-optimization.md#factor-analysis-what-a-stream-is-made-of) as
a way to characterize and commoditize the risk exposures embedded in a return
stream, and the [alpha-vs-beta](11-portfolio-optimization.md#alpha-vs-beta)
distinction.

- <https://global.oup.com/academic/product/asset-management-9780199959327>

## Additional reference

### Investopedia — *Understanding an OHLC Chart* {#investopedia}

Plain-language definition of the [OHLC
bar](05-bars-and-aggregation.md#what-is-an-ohlc-bar): the open, high, low, and
close prices for each period.

- <https://www.investopedia.com/terms/o/ohlcchart.asp>

### *Backtest overfitting in the machine learning era* {#overfitting-paper}

Purged-cross-validation context for [time-series
validation](07-look-forward-bias.md#how-should-validation-be-done-for-time-series).

- <https://www.sciencedirect.com/science/article/abs/pii/S0950705124011110>
