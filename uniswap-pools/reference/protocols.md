# Uniswap LP — Protocol versions

Each pool in `/pools/recommend` has a `protocol` field that decides which endpoint you use and which fields you send — never guess it.

| Version | What it is | Endpoint to open | Position ID | Range |
|---|---|---|---|---|
| **V2** | Classic full-range AMM (2020). Liquidity spread over the whole price curve `[0, ∞)`. No NFT, no range choice. Capital-inefficient but zero range risk. | `create_classic` | none (by pair) | full range (fixed) |
| **V3** | Concentrated liquidity (2021). You pick a price range; fees only accrue while price stays in it. Higher efficiency, "out-of-range" risk. Position = ERC-721 NFT. | `create` | `nftTokenId` | you choose (`tickBounds`) |
| **V4** | Concentrated + hooks (2023). Same as V3 but pools support hook contracts (dynamic fees, custom curves). Pool id = `(token0, token1, fee, tickSpacing, hooks)`. | `create` | `nftTokenId` | you choose (`tickBounds`) |

Uniswap **V1** is obsolete and not supported — ignore it.

## Routing by protocol

- `protocol: "V2"` → open with `create_classic` (`poolParameters` = `{ token0Address, token1Address, chainId }`). No `tickBounds`, no `nftTokenId`.
- `protocol: "V3"` or `"V4"` → open with `create` (`existingPool` = `{ token0Address, token1Address, poolReference }` + `tickBounds`). Position is an NFT, so `increase`/`decrease`/`claim_fees` need `nftTokenId`.

## V2 vs V3 for a stable/stable pair

For stablecoins the price barely moves, so a V3 tight range earns more fees than a V2 full-range position. Either is valid — present the trade-off and let the user choose; never assume V3.