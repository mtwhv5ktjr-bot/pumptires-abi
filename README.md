# pump.tires ABI (PulseChain) — reconstructed

The pump.tires launchpad on PulseChain runs on an **unverified** bonding-curve contract. This repo
publishes a reconstructed ABI for it, the evidence behind every name, and the method used, so anyone
can read the pad's events, quote the curve, or trade it from their own code.

Live explorer and no-fee desk built on this ABI: **https://wick-garage.vercel.app**

## The contracts

The same contract has been deployed three times by `0x128fbE627cC5c2f595cE626517709Ad76Fe22443`.
All three have the identical 38-selector dispatcher and 5 events, so the ABI applies to each.

| Core | Address | Live | Graduates into |
|---|---|---|---|
| v1 | `0xcf6402cdedff50fe334471d0fdd33014e40e828c` | Jan 2025 – May 2026 (dead) | PulseX V2 |
| v1.5 | `0x7f5fde490ca966a333a1c117d17118bdca79ea7b` | Feb – Jun 2025 (dead) | PulseX V1 |
| **v2** | `0x6538a83a81d855b965983161af6a83e616d16fd5` | **Jun 2025 → now** | PulseX V1 |

Each token the core creates is a plain ERC20 with `launch()` / `launched()` on top (`pumptires-token.abi.json`).

## Files

- `pumptires-core.abi.json` — the core: curve trading, quotes, per-token state, constants, events.
- `pumptires-token.abi.json` — the per-launch ERC20.

## How the curve works

Constant product on virtual reserves. With `sold` tokens gone:

```
vTOK  = 1,200,000,000 − sold
vPLS  = K / vTOK            (K = 1.2e9 × 1e8, whole units)
price = vPLS / vTOK         → 0.0833 PLS at launch, 0.75 PLS at graduation
```

| Constant | Value |
|---|---|
| TOTAL_SUPPLY | 1,000,000,000 |
| TOKENS_FOR_SALE | 820,000,000 |
| LAUNCH_THRESHOLD | 800,000,000 **tokens sold** (not a PLS amount) |
| INITIAL_VIRTUAL_TOKEN / INITIAL_VIRTUAL_PLS | 1.2e9 / 1e8 |
| FEE_NUMERATOR / FEE_DENOMINATOR | 100 / 10000 = 1% on every buy, sell and the launch, paid to `0x…0369` (burn) |

Graduation: the buy that crosses 800M sold continues in the same transaction — create the PulseX pair,
wrap ~200M PLS, move it plus every unsold token into the pair, mint LP to the core, **burn the LP**, call
`launch()` on the token, renounce it, emit `TokenLaunched`.

## Functions

```
createToken(string name, string symbol) payable              // msg.value = creator's first buy
buyTokenExactIn(address token, uint256 minTokensOut) payable
buyTokenExactOut(address token, uint256 tokensOut) payable
sellToken(address token, uint256 tokensIn, uint256 minPlsOut) // approve the core first
burnToken(address token, uint256 amount)
getBuyAmountOut(token, plsIn)    → (tokensOut, fee)
getBuyAmountIn(token, tokensOut) → (plsIn, fee)
getSellAmountOut(token, tokensIn)→ (plsOut, fee)
getCurrentPrice(token)           → PLS per token, 18 dec
tokens(token) → creator · tokensSold(token) · plsReceived(token) · totalBurned(token)
```

## Events

| Event | topic0 |
|---|---|
| `TokenCreated(token, creator, name, symbol, initialPrice, timestamp)` | `0xc12630d7…1cce` |
| `TokenPurchased(token, buyer, plsIn, tokensOut, fee, priceBefore, priceAfter, tokensSold, timestamp)` | `0xce05a639…814e` |
| `TokenSold(token, seller, tokensIn, plsOut, fee, priceBefore, priceAfter, tokensSold, timestamp)` | `0x2067389f…493b` |
| `TokenBurned(token, burner, amount, totalBurned, timestamp)` | `0x4e7e33fe…4319` |
| `TokenLaunched(token, pair, launchFee, plsLiquidity, tokenLiquidity, timestamp)` | `0x98ffb71d…c1ed` |

## How it was reconstructed, and how sure it is

1. Pulled the runtime bytecode and extracted every `PUSH4` selector in the dispatcher (38).
2. Resolved selectors and event topics against the OpenChain signature database (20 of 22 core
   functions, 4 of 5 events). The ERC20 selectors belong to the embedded token creation code.
3. Named the two misses by hashing candidate names until they matched: `INITIAL_VIRTUAL_PLS()` →
   `0x40a28a7e`, `TokenLaunched(address,address,uint256,uint256,uint256,uint256)` → `0x98ffb71d…`.
   Both are now submitted to OpenChain and 4byte.
4. Decoded real transactions (buy exact-in, buy exact-out, sell, burn, create, graduation) and
   cross-checked every field: fees are exactly 1%, each event's `priceAfter` equals `getCurrentPrice`,
   `tokensSold` and `totalBurned` in events equal the views, the launch fee is 1% of the raise.
5. Compared all three deployments' selector sets (identical) and read their constructor args.

Parameter **names** are inferred, not published source. Treat them as high-confidence. Nothing here
is affiliated with or endorsed by pump.tires.

## Quick use (ethers v6)

```js
import { Contract } from "ethers";
import abi from "./pumptires-core.abi.json" with { type: "json" };
const core = new Contract("0x6538a83a81d855b965983161af6a83e616d16fd5", abi, provider);
const [tokensOut, fee] = await core.getBuyAmountOut(token, plsIn);
// buy:  core.buyTokenExactIn(token, minTokensOut, { value: plsIn })
// sell: approve the core on the token, then core.sellToken(token, tokensIn, minPlsOut)
```

## License

CC0 1.0 — public domain. Use it, fork it, no attribution required.
