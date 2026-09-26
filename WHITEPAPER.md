# Moonbag: Splitting the Price of a Stock Token

**Version 0.1 · September 2026 · Robinhood Chain (4663)**

---

## Abstract

Moonbag divides one stock token at a price line for the length of one round. The portion of the
share up to the line belongs to FLOOR. The portion above it belongs to MOON. A holder deposits the
share, keeps FLOOR and receives a premium for the upside sold. A trader buys MOON with a ticket
whose price is the whole of the trader's risk. The share never leaves the vault until the round
settles, so every round is fully collateralised and no position can be liquidated. Ten percent of
every premium passes to a burn vault without an owner, which converts it into $MOONBAG and destroys
it. This paper sets out the mechanism, the settlement rules, the pricing bounds, the token and the
risks.

---

## 1. A share that can only be held

Robinhood Chain carries 149 stock tokens, worth about $135 million on chain in the September 2026
registry snapshot. Each is an ERC-20. Each can be held, sent or sold.

That is the whole repertoire.

A trader who wants the move in NVDA over the next day must buy the entire share and carry every
cent of downside. A holder who would never sell SPY above +2% this week owns a ceiling of real value
and has no market in which to sell it. The two wants fit together exactly. No instrument on the
chain joins them.

The one structured product live on the chain today separates a stock's cash distributions. On a
large company that stream runs near 0.4% a year and is announced months ahead. Little about it
moves, so little about it trades. Price is the part of a stock that moves every hour, and price is
the part Moonbag divides.

---

## 2. The split

A round names one stock, one line K, a time when sales close, a time of settlement and a cap on
deposited shares. Let P be the settlement price.

```
  P <= K   FLOOR receives 1 share          MOON receives 0
  P >  K   FLOOR receives K / P of a share  MOON receives (P - K) / P of a share
```

The two fractions sum to one share in every state of the world. Nothing is borrowed against the
position, so there is nothing to liquidate, and once the price is fixed there is no counterparty to
pursue. The vault already holds everything it will pay.

A worked example makes the shape plain. NVDA stands at $100. A daily round sets the line at +2%,
$102. A MOON ticket on one share costs $0.38.

| Settles at | FLOOR receives | MOON receives | Ticket result |
|---|---|---|---|
| $98 | 1 share, worth $98 | $0 | expires |
| $102 | 1 share, worth $102 | $0 | expires |
| $103 | $102 of shares | $1 | about 2.7x |
| $105 | $102 of shares | $3 | about 8.0x |
| $110 | $102 of shares | $8 | about 21x |

The holder's share of the premium, about $0.34 of the $0.38, does not depend on where the price
lands. The figures are illustrative; live prices come from signed quotes.

A holder whose shares only partly sell is treated fairly by construction. The unsold part returns
whole. The sold part returns at K / P of itself. Premium is shared pro rata by shares deposited.

---

## 3. Rounds and their clocks

Two durations open at launch.

**Daily (1D).** Settles at 20:00 UTC, seven days a week. Sales close one hour before. Settlement
reads a thirty-minute time-weighted average price from the stock's Uniswap v3 pool, with the window
ending exactly at the settlement time.

**Weekly (1W).** Runs from Monday 00:00 UTC to Friday 20:00 UTC. Sales close Thursday 20:00 UTC.
Settlement reads the Chainlink feed: the last print at or before Friday 20:00 UTC.

Each round carries one line. Several lines per round would split a young market into books too
thin to fill. A three-day duration follows once the first two fill with regularity.

---

## 4. Why two sources of truth

The choice of settlement source came from measurement.

The Chainlink RHSPY / USD feed on Robinhood Chain prints on a move of about 0.5% or on a 24-hour
heartbeat. Across its last sixty rounds the median interval was 19.5 hours. From Friday's close to
Monday 00:00 UTC it is silent; over the Labor Day weekend the silence lasted 81 hours.

A weekly round can live with that cadence. Its sales close a full day before settlement, so the
settling print is never visible while tickets are on sale.

A daily round cannot. Its latest print may predate the round itself. Daily rounds therefore settle
on the pool, and the pool never closes. On Saturday and Sunday, while every exchange is shut, daily
rounds keep settling on the price the chain itself discovers.

A time-weighted average can be pushed by anyone with capital to spend. Three defences answer that:
caps sized to pool depth (section 7), a thirty-minute window that makes a push expensive to hold,
and a breaker that halts settlement when a fresh feed print disagrees by more than 15%.

---

## 5. Pricing by signed quote

A daily round must be priceable at any minute of the day. An auction cannot give that. Moonbag
prices MOON by quote.

A quote names a round, an ask in USDG per share-ticket and an expiry. The quote key signs it as
EIP-712 typed data:

```
Quote(uint256 round, uint256 askUsdg, uint64 expiry)
```

Fair value comes from a standard volatility model: time to settlement, distance from spot to the
line, and an annualised volatility per stock. The ask adds a markup of about 10%. Draft figures:

| Round | Line | Fair | Ask | Model chance above the line |
|---|---|---|---|---|
| NVDA 1D, σ 50% | +2% | 0.343% | 0.377% of a share | 22% |
| NVDA 1W, σ 50% | +5% | 1.006% | 1.106% | 23% |
| SPY 1D, σ 16% | +1% | 0.048% | 0.053% | 12% |
| SPY 1W, σ 16% | +2% | 0.227% | 0.250% | 18% |

The contract does not trust the quote key. Before it accepts a sale it reads the pool's spot price
and refuses any ask below the ticket's intrinsic value (spot minus the line), below 0.01% of spot or
above 20% of spot. It refuses a quote for another round, an expired quote, and a quote that claims
to live longer than two minutes. A stolen key can misprice inside that band. It cannot sell the
upside for pennies.

---

## 6. Corporate actions and the multiplier

Robinhood Chain stock tokens apply splits and cash distributions by changing a single on-chain
multiplier. No tokens move and no event fires. The next multiplier and the moment it takes effect
are published on the token contract ahead of time.

A line drawn in one multiplier means nothing in another. Moonbag enforces two rules:

1. A round cannot be created if a scheduled change falls on or before its settlement time.
2. If the multiplier at settlement differs from the one recorded at the round's opening, the round
   is void. Every FLOOR seller receives the full deposit. Every MOON buyer receives the full ticket
   price, fee included.

A round that no one can settle, because a pool's history no longer reaches the window or a feed
never printed, becomes voidable by anyone three days after its settlement time, with the same
refunds.

---

## 7. Caps and safety

Volume follows safety. Every round is capped before it opens.

- Open interest per round is at most 1% of the settlement pool's stock-side liquidity.
- Daily rounds that settle on Saturday or Sunday carry half the weekday cap.
- NVDA settles on its NVDA/USDG pool. SPY settles through SPY/WETH and WETH/USDG. The deepest NVDA
  pool pairs it with a thin token and is left unused.
- On weekdays, a TWAP more than 15% away from a fresh Chainlink print blocks settlement until a
  consistent price exists.
- Pausing only ever stops new rounds. Redeem never closes.

The roles are narrow. The lister, held by the keeper, may list a stock, open rounds, set caps, open
FLOOR deposits to every holder and rotate the quote key. No role may move, freeze or redirect a
user's shares or premium. The treasury address is immutable.

---

## 8. The burn vault

The protocol fee is 10% of every MOON premium. The contract holds it with the round until
settlement, then any caller may sweep it to the burn vault. If the round is void, the fee returns
to the buyer with the rest of the ticket.

The burn vault has no owner, no pause and no withdrawal. Its single public action is `burn()`:

- available once per twenty hours, and only when at least 20 USDG waits;
- spends at most 2% of the WETH/USDG pool's USDG in a single call;
- requires ETH out within 1.5% of that pool's own thirty-minute average;
- pays the caller 0.5% of the ETH;
- buys $MOONBAG on the launchpad curve before graduation, or on its v4 pool after;
- destroys every token bought.

No one decides when to buy back, how much, or whether to bother. The burn is funded by use. With no
premium there is no burn, and no one can switch it off.

---

## 9. The token

| Parameter | Value |
|---|---|
| Name | Moonbag |
| Ticker | $MOONBAG |
| Chain | Robinhood Chain, chain id 4663 |
| Total supply | 1,000,000,000 |
| Team allocation | 0 |
| Distribution | 100% to the launchpad bonding curve, graduating to a DEX pool with liquidity locked |
| Supply changes | Burns only; every burn lowers total supply |

One doctrine governs the token: it buys access and prestige, never a lower price or improved odds.
MOON is priced identically for every wallet. The utilities planned for the second version are the
protocol fee paid in $MOONBAG at 7.5% instead of 10% (burned directly), holder-edition cards, a
fifteen-minute early window on capped rounds, seasonal prizes and a vote on the next tickers and
lines.

| Tier | Holding | Access |
|---|---|---|
| STREET | 0 | Every round, full price, paper mode, standard cards |
| FLOOR CLUB | 1M (0.1% of supply) | Fee in $MOONBAG at 7.5%, holder frame, weekly leaderboard |
| MOON CLUB | 5M (0.5%) | Early window on capped rounds, badge on every card |
| ORBIT | 20M (2%) | Votes on tickers and lines, a name on the Orbit wall |

The product ships first. Daily rounds settle on mainnet, with the treasury as the first FLOOR
seller and each settlement published with its proof, before the curve opens. The curve opens the
same week.

---

## 10. The treasury

Launchpad creator fees from $MOONBAG trading flow to a treasury multisig and are split:

| Share | Use |
|---|---|
| 50% | Buys SPY and NVDA, deposited as FLOOR in daily and weekly rounds |
| 30% | Audit, keepers and infrastructure |
| 20% | Settlement reserve for oracle-failure refunds |

The treasury gains only as a FLOOR seller. It collects the premium any holder would collect and
bears the same capped upside. Its positions and results are published.

Some launchpads on the chain allow the factory owner to reroute a token's creator fees after a
timelock, three days on at least one of them. Moonbag sweeps creator fees to the multisig daily, so
the most that could ever be redirected is the next unclaimed balance.

---

## 11. Scale, stated honestly

Suppose the treasury deposits $50,000 of NVDA into daily rounds and every MOON sells at the ask.
Traders pay about $188 a day: about $170 to the FLOOR side and about $19 to the burn.

The number is small by design. Caps precede volume. Another protocol token on the chain did about
$1.3 million of daily volume and paid its creator 130.7 ETH of launchpad fees in 22 days. Creator
fees can carry a treasury. Moonbag's will depend on Moonbag's own volume.

---

## 12. Roadmap

| Phase | Scope |
|---|---|
| 1. MVP | Contracts rehearsed on a mainnet fork: vault, quotes, settlement, burn vault. Keeper, caps, web app with paper mode, documentation and a plain statement of risks. Audit booked, test suite public |
| 2. Launch | Mainnet with SPY and NVDA, daily and weekly rounds, treasury-only FLOOR. Public settlement page from the first round. $MOONBAG fair launch the same week |
| 3. Growth | FLOOR deposits open to every holder, auto-roll, three-day rounds, TSLA and QQQ when caps allow. Seasons and holder tiers |
| 4. Ecosystem | Widget and API for wallets and extensions, holder votes on tickers and lines, rounds for every liquid stock token with a feed |

At the time of writing the contracts pass 21 unit tests and 5 fork tests against the real stock
tokens, pools, feeds and launchpad. They are not yet deployed to mainnet.

---

## 13. Risks

A MOON ticket can expire worthless, and most will. A +2% line is crossed in a day perhaps one time
in five. The most a buyer can lose is the ticket.

A FLOOR seller gives up every dollar above the line for the round joined. If NVDA rises 10%, the
seller keeps the line's worth and the premium.

Settlement prices come from public sources that can lag, run thin or be wrong. The caps, bounds,
breaker and void rules exist because of that, and they reduce the risk without removing it.

Stock tokens are issued by third parties, are not available in every country and carry their own
terms. Smart contracts can hold defects that tests did not find; an external audit precedes any
public FLOOR deposit. $MOONBAG is a utility token for access within the product and confers no
ownership, rights or claims.

---

<sub>Moonbag is experimental software on Robinhood Chain. Nothing in this document is financial, investment, legal or tax advice. Digital assets are volatile. Never risk funds you cannot afford to lose.</sub>
