# 2. Futures contracts & expiries

The instrument this wiki targets is the **futures contract**. Unlike a share of
stock, a futures contract is not perpetual — it has a birth, a life, and a death
(expiry), and there are always several contracts on the same underlying trading
at once. That structure shapes everything a backtester must do: which contract to
trade, when to [roll](#what-does-it-mean-to-roll-a-futures-position), and how to
stitch contracts into a continuous series without
[leaking future information](07-look-forward-bias.md#how-can-continuous-futures-construction-leak-information).

The reference throughout is [CME Group's futures education](sources.md#cme).

## What is a futures contract?

A **futures contract** is a standardized, exchange-traded agreement to buy or
sell a specified **underlying** (an index, a commodity, a currency, a bond) at a
predetermined price, with delivery or settlement at a fixed future date. Because
it is *standardized*, every economic term is fixed by the exchange rather than
negotiated per trade:

- **Underlying** — what the contract references (e.g. the S&P 500 index).
- **Expiry** — the date the contract stops trading and settles.
- **Contract multiplier / contract size** — how much underlying one contract
  represents, which converts a price move into dollars (e.g. \$50 × index points
  for the E-mini S&P 500).
- **Tick size** — the minimum price increment, and its dollar value (**tick
  value**).
- **Margin** — the collateral posted to hold a position; futures are leveraged,
  so you control a large notional with a small deposit, marked to market daily.
- **Settlement** — whether the contract settles in **cash** (a payment equal to
  the value difference) or by **physical delivery** of the underlying.
- **Clearing** — the clearinghouse becomes the counterparty to both sides,
  guaranteeing performance and removing bilateral credit risk.

CME states that futures contracts have a limited lifespan, and that expiration and
rollover are central terms of the product ([CME](sources.md#cme)).

## What does expiry mean?

**Expiry** is the end of a contract's life. Two dates matter: the **last trading
day**, after which you can no longer trade the contract, and the **expiration /
settlement date**, when final settlement occurs. On settlement the contract is
resolved either by **cash settlement** (a final mark to an official settlement
price, with cash changing hands) or by **physical delivery** (the short delivers
the actual underlying to the long).

The practical consequence is that **positions must be managed before expiry**. If
you hold a physically-settled contract into delivery you may be obligated to make
or take delivery — rarely what a systematic trader intends. Even for
cash-settled contracts, liquidity dries up as expiry approaches because other
traders have already moved on. So a live or simulated system must either close or
[roll](#what-does-it-mean-to-roll-a-futures-position) the position before the
contract expires. CME's "Understanding Futures Expiration & Contract Roll" lesson
is the canonical treatment ([CME](sources.md#cme)).

## What does it mean to roll a futures position?

To **roll** a position is to maintain continuous exposure across the expiry
boundary: you **close** the position in the expiring (front) contract and
simultaneously **open** an equivalent position in a later (deferred) contract.
For example, a long in the June contract is rolled by selling June and buying
September, leaving you long the same underlying but now in a contract that won't
expire for another quarter.

Rolling has costs and choices. You pay the spread (and possibly market impact)
on two legs, and the two contracts trade at different prices because of the
[term structure](#why-is-there-more-than-one-contract-for-the-same-underlying),
so the roll changes your entry reference. *When* to roll is a policy decision —
some traders roll on a fixed calendar date, others when volume or open interest
migrates to the next contract. This timing decision is also a subtle source of
[leakage](07-look-forward-bias.md#how-can-continuous-futures-construction-leak-information)
if a backtest picks the roll date using information it would not have had in real
time.

## Why is there more than one contract for the same underlying?

Several contracts on the same underlying trade simultaneously because each
**expiry** is a separate instrument, and traders need to express views and hedge
at different horizons. The set of available expiries and their prices is the
**term structure** (the futures curve). Key vocabulary:

- **Contract cycle** — the calendar of available expiries. Equity-index futures
  use a **quarterly cycle** (March, June, September, December).
- **Front (lead/nearby) contract** — the nearest, usually most liquid expiry.
- **Deferred (back) contracts** — later expiries, typically thinner.
- **Contract codes** — the month-and-year symbology (the standard month codes are
  H = March, M = June, U = September, Z = December), so e.g. `ESU5` denotes the
  E-mini S&P 500 expiring September 2025.

The price differences across expiries reflect carry (financing, dividends,
storage), so the same underlying can show an upward- or downward-sloping curve.

## For equity index futures, why is liquidity concentrated in the front or lead contract?

For equity-index futures, almost all trading happens in a single contract at a
time — the **front (lead) contract** — and migrates, nearly all at once, to the
next quarterly contract around the roll. The reason is a coordination effect:
traders want to trade where everyone else is trading (the deepest book and
tightest spread), so liquidity concentrates in whichever contract the market has
collectively designated as "the" contract. Holding most positions in one expiry
also means the whole market needs to roll at roughly the same time.

CME standardizes this: equity-index futures can be rolled at any time, but the
official **equity roll date** is the **Monday before the third Friday** of the
expiration month, and liquidity shifts from the expiring quarterly contract to
the next quarterly contract around that window ([CME equity index roll
dates](sources.md#cme)). For a backtester this means the *active* contract — the
one whose prices you should trade and model — changes on a predictable schedule,
which is the subject of the next question.

## How should a backtester select the active equity index futures contract?

The backtester needs a deterministic rule that, for any historical timestamp,
names the single contract to trade. Common rules, roughly in order of
sophistication:

1. **Fixed calendar / exchange roll date** — switch to the next contract on the
   CME equity roll date (the Monday before the third Friday). Simple, reproducible,
   and matches index conventions.
2. **Volume-based** — switch when the next contract's daily **volume** first
   exceeds the front contract's. Tracks where trading actually is.
3. **Open-interest-based** — switch when **open interest** crosses over, which
   tends to lead volume.
4. **Firm-specific convention** — e.g. "roll N days before expiry," chosen to
   match the desk's live practice.

Whatever rule you choose, it must use **only information available as of the
decision time** — you cannot pick today's active contract using tomorrow's
volume. The data you need (per-contract volume, open interest, and expiry dates)
is exactly what futures datasets expose: QuantConnect's AlgoSeek futures dataset,
for instance, includes price, volume, open interest, and expiry
([AlgoSeek](sources.md#algoseek)). Getting this selection right — and timing the
roll honestly — is what keeps [continuous-contract
construction](07-look-forward-bias.md#how-can-continuous-futures-construction-leak-information)
free of lookahead.

---

Previous: **[← Market structure](01-market-structure.md)** · Next: **[Orders &
execution →](03-orders-and-execution.md)**
