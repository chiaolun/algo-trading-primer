# 11. Portfolio optimization: characterizing & combining investments

The earlier sections build **one** strategy. Real money is run as a **portfolio**
of many return streams — strategies, assets, factors — and the central question
becomes: given a pile of candidate investments, how do you decide which are worth
holding and how to combine them?

The key idea underneath everything in this section is **characterization**:
coming up with a *standard way to describe* an investment, so that you can

- **value investments against each other** — is stream A better than stream B? — and
- **reason about combining them** — what happens to risk and return when I hold
  both?

Once you can describe any return stream in a common language, comparison and
combination become arithmetic. This section develops that language along three
axes: how good a stream is **on its own**, what a stream is **made of**, and how
streams **combine**.

References: [Markowitz](sources.md#markowitz), [Sharpe](sources.md#sharpe),
[Bailey & López de Prado](sources.md#psr), [Grinold &
Kahn](sources.md#grinoldkahn), [Ang](sources.md#ang), and [López de
Prado](sources.md#lopezdeprado).

## Part A — Characterizing a single stream (valuing them against each other)

### Why risk-adjusted return?

A raw return number is meaningless without its risk. A stream that returned 30%
by taking wild, near-ruinous swings is not obviously better than one that returned
12% smoothly — and it is certainly not better *per unit of risk taken*. So the
first job of characterization is to summarize a return stream by **both** its
reward and its risk, and the overwhelmingly common choice is the **mean and the
standard deviation** of its returns:

- the **mean** $\mu = E[R]$ stands in for reward, and
- the **standard deviation** $\sigma = \sqrt{\operatorname{Var}(R)}$ stands in for
  risk.

This **mean-variance** characterization is the foundation of modern portfolio
theory ([Markowitz](sources.md#markowitz)). It reduces an entire, complicated
distribution of outcomes to two numbers — which is exactly what makes streams
comparable and combinable.

It is worth being honest about what this throws away. Standard deviation is a
**symmetric** measure: it penalizes big *gains* as much as big *losses*, even
though only the downside hurts. And it fully describes risk only when returns are
roughly **normal** — but financial returns have **fat tails** and **skew**, so σ
systematically *understates* the chance of extreme moves (a theme that returns
with a vengeance under [crisis correlations](#when-correlations-go-to-one)). The
mean-variance summary is a starting language, not the last word — its cousins
below ([Sortino](#sharpes-cousins)) exist precisely to patch the symmetry flaw.

### The Sharpe ratio

The **Sharpe ratio** turns the two-number summary into a single, comparable score:
reward per unit of risk, measured as **excess return over the risk-free rate, per
unit of standard deviation** ([Sharpe](sources.md#sharpe)):

$$ SR = \frac{E[R] - r_f}{\sigma_R} $$

where $r_f$ is the risk-free rate. Subtracting $r_f$ matters because you should
only get credit for return *above* what idle cash earns. A higher Sharpe means
more reward for the risk borne, which is why it is the **lingua franca** for
comparing return streams of different scales: a strategy doing ±2% a day and a
bond fund doing ±0.1% a day can be placed on the same axis.

Sharpe ratios are quoted **annualized** so they are comparable across measurement
frequencies. If you estimate Sharpe from returns sampled $N$ times per year (≈252
for daily), you scale by $\sqrt{N}$:

$$ SR_{\text{annual}} = \sqrt{N}\; SR_{\text{per-period}} $$

The $\sqrt{N}$ (not $N$) comes from mean growing linearly with horizon while
standard deviation grows with its square root. Keep the assumptions in view: the
scaling assumes returns are roughly independent across periods (autocorrelation
breaks it), and the ratio itself inherits σ's blind spots — a strategy that sells
options can show a glorious Sharpe right up until the tail event that σ never saw
coming.

### The Probabilistic Sharpe Ratio

A Sharpe ratio computed from data is an **estimate**, not the truth, and estimates
have error bars. Two managers can report the same 1.5 Sharpe, but if one has ten
years of daily data and the other has three months, you should believe them very
differently. The **Probabilistic Sharpe Ratio (PSR)** makes that precise: instead
of a point estimate, it reports the **probability that the true Sharpe exceeds a
benchmark** $SR^{*}$ ([Bailey & López de Prado](sources.md#psr)):

$$ \widehat{PSR}(SR^{*}) = \Phi\!\left( \frac{(\widehat{SR} - SR^{*})\,\sqrt{n-1}}
{\sqrt{1 - \hat\gamma_3 \widehat{SR} + \frac{\hat\gamma_4 - 1}{4}\widehat{SR}^2}} \right) $$

where $n$ is the number of observations, $\hat\gamma_3$ the skew, $\hat\gamma_4$
the kurtosis, and $\Phi$ the normal CDF. Read it qualitatively: PSR **rises** with
a longer track record ($n$) and a higher observed Sharpe, and it **falls** when
returns are negatively skewed or fat-tailed — i.e. it deflates the headline number
exactly when σ was most likely lying about the risk.

The natural extension is the **Deflated Sharpe Ratio (DSR)**, which additionally
corrects for the fact that you tried **many** strategies and kept the best. The
more configurations you test, the higher a Sharpe you expect to see *by luck
alone*, so DSR raises the bar accordingly. This is the portfolio-level face of
[data snooping and backtest
overfitting](07-look-forward-bias.md#what-is-data-snooping-or-backtest-overfitting):
PSR and DSR are how you defend a reported Sharpe against "you just got lucky."

### Sharpe's cousins

Two close relatives fix specific weaknesses of Sharpe and round out the
single-stream toolkit:

- The **Sortino ratio** replaces total standard deviation with **downside
  deviation** — the volatility of *negative* returns only:
  $\text{Sortino} = (E[R] - r_f) / \sigma_{\text{down}}$. It addresses Sharpe's
  unfair penalty on upside volatility, rewarding strategies whose variability is
  mostly to the good side.
- The **Information ratio (IR)** measures skill *relative to a benchmark*: active
  return (the stream minus its benchmark) divided by **tracking error** (the
  standard deviation of that active return), $IR = \alpha / \omega$. It is the
  natural score when the job is to **beat an index** rather than to earn an
  absolute return, and it is central to the fundamental law of active management
  ([Grinold & Kahn](sources.md#grinoldkahn)). The Information ratio also points
  straight at the next axis: to define "active return" you must first say what the
  benchmark exposure *is* — which is what factor analysis does.

## Part B — Characterizing what a stream is made of (factor analysis)

### Factor analysis: what a stream is made of

Risk-adjusted return scores a stream in isolation, but it does not tell you *why*
the stream moves — and two strategies with identical Sharpe ratios can be driven
by completely different underlying bets. **Factor analysis** characterizes a
return stream by **decomposing it into exposures to common factors** plus a
leftover:

$$ R = \alpha + \sum_{k} \beta_k F_k + \varepsilon $$

Each $F_k$ is a **factor** — a pervasive source of return such as the overall
market, value-vs-growth, momentum, size, or a sector — and each $\beta_k$ is the
stream's **exposure** (sensitivity) to that factor. The progression of richness:

- **CAPM** uses a single factor, the market, so a stream is summarized by one
  **beta** to the market.
- **Multi-factor models** (e.g. Fama–French and its descendants) add value, size,
  momentum, and more, describing the stream as a vector of betas.
- **Statistical / PCA factors** let the factors themselves be *discovered* from the
  covariance of many streams, rather than named in advance.

The payoff is that factor analysis **commoditizes risk**: it expresses an
arbitrary, idiosyncratic-looking return stream in a *standard basis* of exposures,
so that two strategies can be compared in the same vocabulary ("this one is really
a leveraged momentum bet; that one is short volatility"). It also feeds Part C —
estimating the [covariance matrix](#diversification-and-the-covariance-matrix)
through a handful of factors is far more stable than estimating every pairwise
correlation directly. [Ang](sources.md#ang) and [Grinold &
Kahn](sources.md#grinoldkahn) develop this view.

### Alpha vs. beta

The decomposition draws the most important line in active management. **Beta** is
the part of your return explained by exposure to known factors — *commoditized,
cheap, replicable* risk that anyone can buy through an index or a futures position.
**Alpha** ($\alpha$, the intercept) is what is left after stripping out every
factor bet: genuinely idiosyncratic skill that cannot be obtained by simply
holding a known exposure.

This matters because **you should not pay alpha fees for beta returns**. A strategy
that looks brilliant may, under factor analysis, turn out to be plain market beta
in disguise — its apparent edge is a risk premium you could have rented cheaply.
Characterizing a stream as alpha-plus-betas tells you what is truly novel, what is
redundant with exposures you already hold, and what you are actually being
compensated for.

### What it means for a factor model to work

So far Part B has treated factor analysis as a way to *describe* a stream. But the
decomposition also changes how you *estimate* the quantity Part C needs most — the
covariance between assets — and working through that estimation is the clearest way
to say what it means for a factor model to "work."

Take two assets with returns $r_1$ and $r_2$, and suppose you want their covariance
$\sigma_{12}$. There are two ways to get it.

**Estimate it directly.** Compute the sample covariance straight from the two return
series:

$$ \hat\sigma_{12} = \frac{1}{T-1}\sum_{t=1}^{T}(r_{1,t}-\bar r_1)(r_{2,t}-\bar r_2). $$

This estimator is **unbiased**: whatever the true data-generating process, its
expectation equals the true covariance, and as $T\to\infty$ it converges to the
truth. It assumes nothing about how the assets are related. Its weakness is
**variance** — with a realistic sample length it is noisy, because it leans on those
two series alone and on however many observations you happen to have. Scaled up to
$N$ assets, the full sample covariance matrix has $N(N+1)/2$ free parameters, each
estimated from limited data; the noise compounds, and the matrix becomes
ill-conditioned (even non-invertible) when $T$ is not comfortably larger than $N$.

**Estimate it through the factor model.** Posit $r_i = \alpha_i + \beta_i^\top f +
\varepsilon_i$, with shared factors $f$ (covariance $\Sigma_f$) and idiosyncratic
residuals $\varepsilon_i$ that are *assumed uncorrelated across assets*. Under that
structure the covariance collapses to

$$ \sigma_{12} = \beta_1^\top \Sigma_f\, \beta_2. $$

So you estimate each asset's loadings $\hat\beta_1, \hat\beta_2$ **individually** (one
regression per asset), estimate the factor covariance $\hat\Sigma_f$ **once** — from
the factor history, shared across every asset — and multiply. This estimator has
**much lower variance**: it fits only a handful of loadings per asset plus one
shared, precisely-estimated factor covariance, instead of a free parameter for every
pair, and it pools information across all assets and a long factor history. But it is
**biased**: the assumption that residuals are uncorrelated is never exactly true (two
oil producers share an oil-price risk the market factor misses; loadings drift over
time; the factor set is incomplete), so the model-implied covariance is
systematically off by whatever real co-movement the factors fail to capture.

**This is the bias–variance tradeoff**, in covariance-estimation form. The direct
sample covariance is the **zero-bias, high-variance** estimator; the factor estimate
deliberately accepts a little bias to buy a large reduction in variance. A factor
model **works** precisely when that trade is favorable — when the bias it introduces
is *small relative to the variance it removes*, so the model-implied covariance is
**closer to the truth out of sample** than the raw sample number, even though the
sample number is the one that is technically unbiased. Equivalently, it works when
the residuals really are *mostly* uncorrelated once the common factors are accounted
for: the co-movement the factor estimate throws away is then mostly noise, not
signal. It **fails** when the discarded structure is large — a missing factor, or a
genuine residual correlation big enough that the bias swamps the variance you saved.

This is also why the [robust covariance
fixes](#why-mean-variance-optimization-is-fragile) in Part C exist. **Shrinkage**
(Ledoit–Wolf) interpolates between exactly these two poles — it blends the
unbiased-but-noisy sample covariance with a biased-but-stable structured target
(often a factor or constant-correlation model), choosing the mix that minimizes total
estimation error. The factor model sits at one end of that spectrum and the sample
covariance at the other; the bias–variance tradeoff is the axis running between them.

## Part C — Characterizing how streams combine (reasoning about combining)

### Diversification and the covariance matrix

To reason about *combining* streams you need one more piece of characterization:
how they **co-move**. For a portfolio with weights $w$ over assets with expected
returns $\mu$ and covariance matrix $\Sigma$, the portfolio's mean and variance
are

$$ \mu_p = w^\top \mu, \qquad \sigma_p^2 = w^\top \Sigma w. $$

The second equation is where diversification lives. For two assets,

$$ \sigma_p^2 = w_1^2\sigma_1^2 + w_2^2\sigma_2^2 + 2 w_1 w_2 \rho_{12}\sigma_1\sigma_2, $$

and the cross term shows that portfolio risk depends not just on the individual
volatilities but on the **correlation** $\rho_{12}$. Combine two streams with low
or negative correlation and the portfolio's standard deviation comes in *below* the
weighted average of the parts — you shed risk without giving up the average
return. This risk reduction "for free" is the closest thing to a free lunch in
finance, and the **covariance matrix $\Sigma$ is the object that encodes it**. It
is also, as we will see, the object whose instability does the most damage.

### Markowitz mean-variance optimization

[Markowitz](sources.md#markowitz) mean-variance optimization (MVO) turns the
characterization into an explicit optimization: choose the weights that **minimize
risk for a target return** (or, equivalently, maximize return for a given risk):

$$ \min_{w}\; w^\top \Sigma w \quad \text{subject to} \quad w^\top \mu = \mu_p,
\;\; \textstyle\sum_i w_i = 1. $$

Sweeping the target $\mu_p$ traces out the **efficient frontier** — the set of
portfolios with the best possible return for each level of risk; anything below it
is dominated. Once a risk-free asset exists, the best portfolio of risky assets is
the one where a line from $r_f$ is tangent to the frontier: the **tangency
portfolio**, which is exactly the **maximum-Sharpe** portfolio. Every investor then
holds some mix of the risk-free asset and that one tangency portfolio, tracing the
**Capital Market Line**. This is the clean theoretical answer to "how do I
combine?" — and the rest of this section is about why the clean answer misbehaves.

### Marginal Sharpe: should you add a strategy?

A practical combination question: you already run a portfolio, and a new candidate
strategy lands on your desk. Should you add it? Its **standalone Sharpe is the
wrong test** — what matters is its **marginal** contribution, i.e. how the
*portfolio's* Sharpe changes once the new stream is mixed in. A new stream improves
the portfolio when its own Sharpe beats the Sharpe you already have, **scaled down
by its correlation** to the existing book. Roughly, adding asset $i$ raises the
portfolio Sharpe when

$$ SR_i > \rho_{i,p}\; SR_p, $$

where $\rho_{i,p}$ is the correlation of the candidate with the current portfolio.
The consequence is liberating: a **mediocre but uncorrelated** stream can be more
valuable than an excellent but redundant one. A 0.5-Sharpe strategy that is
*uncorrelated* with everything you hold improves the book; a 1.5-Sharpe strategy
that is 0.95-correlated with what you already run adds almost nothing. This
**marginal contribution to risk** view is how a desk decides what to onboard, and
it is the combination-side analogue of the alpha-vs-beta question — both ask "what
does this stream add that I don't already have?"

### Why mean-variance optimization is fragile

MVO is beautiful in theory and treacherous in practice, because it is an **"error
maximizer."** Its inputs — expected returns $\mu$ and the covariance matrix
$\Sigma$ — are *estimated* with considerable noise, and the optimizer responds to
that noise pathologically: it piles weight onto whatever assets *happen* to look
high-return or low-correlation in-sample, which are precisely the ones whose
estimates are most likely inflated by luck. The result is extreme, unstable,
concentrated weights that often perform terribly out of sample — the same
[overfitting](07-look-forward-bias.md#what-is-data-snooping-or-backtest-overfitting)
disease, now in portfolio-weight form. Expected returns are especially hard to
estimate, so naive MVO weights swing violently with tiny changes in $\mu$.

The practical fixes all inject humility about the estimates:

- **Shrinkage** (e.g. Ledoit–Wolf) pulls the noisy sample covariance toward a
  simple, stable target, taming the wild weights.
- **Risk parity** sidesteps expected-return estimation entirely, sizing positions
  so each contributes *equal risk* — robust precisely because it does not trust
  $\mu$.
- **Constraints** — caps on individual weights, no-shorting, or limits on turnover —
  bluntly prevent the optimizer from chasing noise into extreme positions.

The throughline: trust your characterizations less than the math invites you to,
especially the covariance matrix — whose worst failure is the subject of the final
question.

### When correlations go to one

Every diversification argument in Part C rests on the correlations in $\Sigma$
being **stable**. They are not. In a crisis — 1987, 2008, March 2020 — the
diversifying correlations that justified the portfolio **spike toward 1**: assets
that normally move independently all crash together as investors flee risk
indiscriminately and sell whatever they can. This **tail dependence** means the
correlation matrix you estimated in calm markets describes a different world than
the one you face when it matters, and the risk reduction you counted on
**evaporates exactly when you need it most.**

Here the two covariance estimators from [factor analysis](#what-it-means-for-a-factor-model-to-work)
behave very differently, and the factor-derived one is far better equipped to *model*
this behavior. The **sample covariance** is just a backward-looking average of
realized co-movement: it has no structural handle on "all correlations rise
together," so it cannot represent the regime until it has actually lived through the
crisis, and even then a calm trailing window dilutes the spike. The
**factor-derived covariance** $\Sigma = B\,\Sigma_f\,B^\top + D$ — loadings $B$, a
small factor covariance $\Sigma_f$, and diagonal idiosyncratic variances $D$ — has
the crisis mechanism built in. Correlations going to 1 *is* the systematic factor's
variance swamping idiosyncratic risk: for two assets that both load on a common
factor with variance $\sigma_f^2$,

$$ \operatorname{Corr}(r_1, r_2) = \frac{\beta_1\beta_2\,\sigma_f^2}
{\sqrt{(\beta_1^2\sigma_f^2 + \sigma_{\varepsilon_1}^2)(\beta_2^2\sigma_f^2 + \sigma_{\varepsilon_2}^2)}}
\;\xrightarrow[\;\sigma_f^2 \to \infty\;]{}\; 1. $$

As the common factor's volatility explodes, the idiosyncratic terms $\sigma_\varepsilon^2$
become negligible and the implied correlation between any two assets with same-sign
loadings is driven to 1 — exactly the flight-to-risk-off behavior, where everything
is suddenly dominated by one systematic shock. So a factor model lets you **stress
this regime with a single knob**: scale up the factor variance and watch the whole
correlation matrix migrate toward 1, coherently and across every pair at once,
without re-estimating thousands of pairwise correlations. The sample covariance
offers no such lever.

This is the deepest caveat of the whole section: the mean-variance characterization
is a calm-weather instrument, and its single number for "how these move together"
hides a regime switch. The defensive lessons follow directly — **stress-test**
portfolios under crisis-correlation assumptions rather than historical-average
ones; never trust a single covariance estimate; size for the
[drawdowns](09-prediction-to-actions.md#how-do-you-evaluate-the-strategy-after-execution-costs)
you would suffer if correlations went to 1; and remember that fat tails and crisis
co-movement are the recurring ways that a tidy σ-based story understates real
risk. Characterization makes investments comparable and combinable — but only as
far as the description holds, and in a crisis the description breaks.

---

Previous: **[← Homework](10-homework.md)** · Back to **[Home](index.md)** · Go
deeper with the **[Sources](sources.md)**.
