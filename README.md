<div align="center">

# moonbag-treatise

**The written case for Moonbag: the mechanism, the math, and the ways it can fail.**

![Docs](https://img.shields.io/badge/docs-whitepaper%20%2B%20threat%20model-184A4E?style=flat-square)
![Method](https://img.shields.io/badge/threat%20model-STRIDE-C1F741?style=flat-square&labelColor=184A4E)
![Chain](https://img.shields.io/badge/chain-Robinhood%20Chain-184A4E?style=flat-square)
![Version](https://img.shields.io/badge/version-0.1-F3EEDC?style=flat-square&labelColor=0A0B0D)

[Website](https://moonbag.fi) · [Docs](https://moonbag.fi/docs) · [X](https://x.com/moonbagfi)

</div>

---

## Contents

| Document | What it covers |
|---|---|
| [WHITEPAPER.md](WHITEPAPER.md) | The split, daily and weekly rounds, why two settlement sources, signed quotes and their on-chain bounds, the multiplier rule, caps, the ownerless burn vault, $MOONBAG, the treasury, honest scale, roadmap and risks |
| [THREAT_MODEL.md](THREAT_MODEL.md) | Assets, trust boundaries, STRIDE per boundary, protocol abuse cases (self-dealing, weekend pushes, burn front-running, fee reroutes, impostors), known gaps, disclosure |

## Moonbag in one screen

Moonbag cuts one Robinhood Chain stock token at a price line for one round.

| Side | Keeps | Pays or receives |
|---|---|---|
| **FLOOR** | Everything up to the line | Receives 90% of every MOON premium, pro rata by shares |
| **MOON** | Everything above the line | Pays a ticket; the ticket is the maximum loss |
| **Burn vault** | Nothing | Receives 10% of every premium and burns $MOONBAG with it |

```
                 the line (K)
  0 ─────────────────┼────────────────────► price at settlement (P)
     FLOOR keeps     │   MOON takes (P - K) / P
     the whole share │   FLOOR takes K / P
```

| Round | Settles | Source |
|---|---|---|
| 1D | 20:00 UTC, every day including weekends | 30-minute DEX TWAP ending at settlement |
| 1W | Friday 20:00 UTC | Last Chainlink print at or before settlement |

## Reading order

1. **Whitepaper, sections 1 to 3** for what the product is.
2. **Whitepaper, sections 4 to 8** for why the rules are what they are.
3. **Threat model** before depositing anything or integrating the API.

## Related repositories

| Repository | Role |
|---|---|
| [moonbag-substrate](../moonbag-substrate) | Contracts and C4 system design |
| [moonbag-integrations](../moonbag-integrations) | Public API reference and TypeScript SDK |
| [moonbag-terminal](../moonbag-terminal) | The web app |
| [moonbag-keeper](../moonbag-keeper) | Round clock, quote signer, indexer and runbook |

---

<div align="center">
<sub>Moonbag is experimental software on Robinhood Chain. Nothing here is financial, investment, legal or tax advice. MOON tickets can expire worthless; the most a buyer can lose is the ticket. FLOOR holders give up all upside above the line for the round they join. $MOONBAG confers no ownership, rights or claims.</sub>
</div>
