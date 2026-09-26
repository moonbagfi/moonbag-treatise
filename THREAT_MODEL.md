# Moonbag Threat Model

**Method:** STRIDE per trust boundary, followed by protocol-specific abuse cases.
**Scope:** `MoonbagRounds`, `BurnVault`, the keeper worker (lister, quote signer, indexer), the
public API and the terminal web app.
**Out of scope:** the stock token issuer, the Uniswap and Chainlink contracts themselves, and the
user's own wallet.

---

## 1. Assets

| Asset | Where it lives | Value at risk |
|---|---|---|
| Deposited stock shares | `MoonbagRounds` | Every FLOOR position, up to each round's cap |
| Held premium (90%) | `MoonbagRounds` | Owed to FLOOR sellers at redeem |
| Held fee (10%) | `MoonbagRounds`, then `BurnVault` | Burn budget |
| Keeper key | Worker env, 0600 | Network fee ETH only |
| Quote key | Worker env, 0600 | No funds; pricing authority inside on-chain bounds |
| History store credentials | Worker and API env, 0600 | Off-chain history and the paper game |
| Settlement correctness | Pools, feeds, contract | Every payout of every round |

## 2. Trust boundaries

```
 Internet ──► Terminal + Public API ──(private network)──► Keeper worker ──► Robinhood Chain
    │                 │                                         │
    │                 └──► History store (private network) ◄────┘
    │
    └──► User wallet ────────────── signed tx ─────────────────────────────► Robinhood Chain
```

Only two paths move value: a user's own signed transaction, and the keeper's transactions, whose
powers the contract limits. Neither the terminal nor the API holds a key.

---

## 3. STRIDE

### Spoofing

| Threat | Mitigation |
|---|---|
| A forged MOON quote | EIP-712 signature recovered on chain against `quoteSigner`; malleable signatures rejected |
| A quote replayed on another round | `Quote.round` must equal the round being bought |
| A quote replayed later | `expiry` checked against the block; quotes claiming more than 120 s of life are refused |
| A quote replayed on another chain or deployment | Domain separator binds chain id and contract address |
| A fake Moonbag site or token | Addresses published only on moonbag.fi/docs and @moonbagfi; the terminal reads addresses from its own runtime config |

### Tampering

| Threat | Mitigation |
|---|---|
| Pushing the pool TWAP near 20:00 UTC | 30-minute window ending at `settleAt`; per-round cap at 1% of pool depth; weekend caps halved; 15% breaker against a fresh feed print |
| Moving spot to shift the quote bounds | Bounds only restrict the ask (intrinsic value, 0.01% to 20% of spot). A pushed spot can block a sale; it cannot make a sale pay the buyer more than the round's settlement allows |
| A split or cash distribution mid-round | `createRound` refuses a scheduled change before `settleAt`; `settle` voids when the multiplier moved |
| Changing a settled result | `settle` runs once; state moves OPEN to SETTLED or VOID and never back |
| Tampering with indexed history | History is a copy. Round state, positions and payouts are always read from the contract |

### Repudiation

| Threat | Mitigation |
|---|---|
| Disputing a settlement | `Settled` event records price, `moonFrac`, source and the feed round or print time; the settlement page links the tx |
| Disputing a burn | `Burn` event records caller, USDG in, ETH out, tip and tokens burned |
| Disputing a quote | Each `MoonBought` event carries the signed `askUsdg` |

### Information disclosure

| Threat | Mitigation |
|---|---|
| Server secrets in the browser | No secret is ever `NEXT_PUBLIC_*`; built chunks are scanned before release |
| Keys in images or repositories | Keys enter only through runtime env files (0600); `.env*` and key files are ignored by git |
| Wallet balances | Positions are public chain data; the API stores nothing about a wallet beyond indexed events |

### Denial of service

| Threat | Mitigation |
|---|---|
| Quote signer offline | Buying pauses; redeem, settle and sweep are open to anyone and need no worker |
| Keeper offline | Nobody needs the keeper to settle or redeem. A round nobody settles is voidable by anyone after 3 days, with full refunds |
| Pool history too short for the window | Settlement reverts; `voidStale` refunds after 3 days |
| Feed silent past 7 days on a weekly round | Settlement reverts; `voidStale` refunds after 3 days |
| API flooding | Per-client limits (quotes: 120 a minute; paper tickets: 30 a minute) and edge rate limits |

### Elevation of privilege

| Threat | Mitigation |
|---|---|
| Keeper key stolen | Can open rounds, set caps, rotate the quote signer. Cannot move, freeze or redirect shares or premium, cannot stop a redeem |
| Quote key stolen | Can sign asks inside the band. Rotated by the lister with `setQuoteSigner` |
| Web container compromised | Holds no signing key; quotes come from the worker over the private network; containers run non-root with read-only filesystems |
| Burn vault takeover | No owner exists. `bind` works exactly once |

---

## 4. Protocol abuse cases

### 4.1 Self-dealing for leaderboard rank

A wallet deposits FLOOR and buys its own MOON. FLOOR plus MOON always equals the share, so the
wallet gains nothing but the fee it paid. Self-fills are excluded from every board, prizes need
real rounds and a minimum premium, and per-wallet caps apply.

### 4.2 Weekend thin-pool push

Weekend pools are thinner. Weekend daily caps are half the weekday cap, which halves what a push
can win against the cost of holding a pushed price for thirty minutes.

### 4.3 Stale-feed window on weekly rounds

Weekly sales close Thursday 20:00 UTC, a full day before the Friday settlement, so the settling
print cannot be known while tickets sell. The contract accepts only a print at or before
`settleAt` and no older than seven days.

### 4.4 Burn front-running

`burn()` is public and predictable. The ETH leg must come within 1.5% of the pool's own thirty-minute
average, and each burn spends at most 2% of the pool's USDG, so a sandwich has little room. The
$MOONBAG leg currently passes a zero minimum; a tighter bound is a known gap.

### 4.5 Launchpad fee reroute

Some launchpads let the factory owner redirect creator fees after a timelock. Creator fees are
swept to the treasury multisig daily, limiting exposure to one unclaimed balance.

### 4.6 Phishing and impostor tokens

Before launch, any token called $MOONBAG is an impostor. After launch, the only valid addresses are
the ones on moonbag.fi/docs and pinned by @moonbagfi. The team never sends direct messages about
presales, allocations or whitelists.

---

## 5. Known gaps

| Gap | Current state | Plan |
|---|---|---|
| Volatility input | A constant per stock (NVDA 50%, SPY 16%) | 30-day realised volatility plus a margin |
| Burn token leg | `minTokensOut = 0` | Derive a floor from the curve or pool state |
| External audit | Booked, not complete | Report published before any public FLOOR deposit |

---

## 6. Operating practices

- Fresh keys only. Well-known development accounts are swept on this chain and on forks of it.
- Every fork rehearsal runs with a chain id other than 4663, from an archive endpoint.
- Contracts are immutable. A change means a new deployment, preceded by green unit and fork tests,
  positive bytecode size margins, re-exported ABIs and green worker checks.
- Worker and web containers run non-root, read-only, with all Linux capabilities dropped and no
  new privileges.
- Alerts on every outbound treasury transfer and every keeper transaction are part of the launch
  checklist, so an unexpected move is seen in minutes.

---

## 7. Reporting a vulnerability

Report privately to the team through a direct message to @moonbagfi on X, asking for a secure
channel. Do not open a public issue for a vulnerability. Include the affected contract or service,
the steps to reproduce, and the impact you observed.
