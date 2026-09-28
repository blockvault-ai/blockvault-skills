---
name: uniswap-pools
description: Manage Uniswap liquidity positions (LP) on Base, Ethereum, or Polygon. Discover pool info, create/increase/decrease V2/V3/V4 positions, and claim accrued fees using the agent wallet. Use when the user asks to provide liquidity, add/remove liquidity, open/manage an LP position, or claim fees earned from providing liquidity.
metadata:
  category: defi
  enabled: true
---

### Instructions

This is a **conversational** skill: discover the user's intent, present pool state, and act only after the user picks the pool and action.

1. Get the wallet address and balances (`get_assets`).
2. Discover the pool state (`/lp/pool_info`) and render it — spot price, fee tier, liquidity, and APY when available.
3. **Ask the user** which action they want (`create`, `increase`, `decrease`, or `claim_fees`) plus the amount and (for V3/V4) the price range.
4. Check approval, then execute the requested position change. Finally, notify the user of the result.

If there are insufficient funds, or an LP action fails, notify the user with the error message.
Do not open LP positions without first checking the balances.


# Uniswap Liquidity Pools (Background)

Manage Uniswap V2/V3/V4 liquidity positions via the BlockVault Uniswap API.
Base URL: `https://402.blockvault.ai`

The `bash` tool auto-detects metatransaction responses (containing `to`, `data`, `chainId`) and signs+broadcasts them via WDK.

## Typed sub-schemas

- `LPAmount` — `{ "tokenAddress": string, "amount": string, "decimals"?: number }` — `amount` is human-readable **decimal** (e.g. `"100.5"`), converted to wei server-side. `decimals` is only needed for tokens outside the curated registry.
- `V2PoolParams` — `{ "token0Address": string, "token1Address": string, "chainId": number }`.
- `NewPoolParams` — `{ "token0Address", "token1Address", "fee" (bps), "tickSpacing", "hooks"? (V4), "initialPrice" }`.
- `ExistingPoolParams` — `{ "token0Address", "token1Address", "poolReference" }` (pool address for V3, pool ID for V4).
- `PriceBounds` — `{ "minPrice": string, "maxPrice": string }` (token1 per token0).
- `TickBounds` — `{ "tickLower": number, "tickUpper": number }`.
- `urgency` — optional enum `NORMAL | FAST | URGENT` on mutating LP calls. Default `NORMAL`.
- `simulateTransaction` — optional boolean; set `true` to dry-run and inspect the result before broadcasting.


### Get the wallet address and balances

Call `run_js` with:
- **function**: "get_assets"
- **data**: `{"hasBalance": true}`

Pick the asset on the target chain and note its `address`. Verify sufficient funds for the LP action.


### Discover pool info

Fetch pool state (liquidity, fee, price) before creating a position. `protocol` is required.

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/pool_info" \
  -H "Content-Type: application/json" \
  -d '{"protocol":"<V2|V3|V4>","poolParameters":{"tokenAddressA":"<address>","tokenAddressB":"<address>","fee":<fee_bps>,"tickSpacing":<tick_spacing>,"hookAddress":"<hook_address>"},"poolReferences":[{"protocol":"<V2|V3|V4>","chainId":<chainId>,"referenceIdentifier":"<pool_address_or_id>"}],"chainId":<chainId>,"pageSize":20,"currentPage":1}'
```

- `protocol`: `V2`, `V3`, or `V4` (required)
- `poolParameters` (optional): `PoolLookupParams` `{ tokenAddressA, tokenAddressB, fee?, tickSpacing?, hookAddress? }` — lookup by token pair
- `poolReferences` (optional): array of `PoolReference` `{ protocol, chainId, referenceIdentifier }` (pool address for V3, pool ID for V4, pair address for V2)
- `chainId` / `pageSize` / `currentPage`: optional

**What it returns & why:** reads the live pool state — current liquidity, active fee tier, and spot price — so you can quote a realistic position before committing. Use it whenever the user asks "what's the APY/price on this pool" or right before opening a position to pick the right fee tier and price range.

Present the pool state to the user with the Jinja2 template below, then **ask in plain text** which action and position they want before calling any mutating endpoint.

**Token logos:** render a small logo (≤ 24px) next to each token using the pool's quoted token symbols (`token0`/`token1`) via a public, symbol-keyed CDN. Use `https://cryptocurrencyliveprices.com/img/{{ symbol | lower }}.png` and guard with `{% if %}` so a missing logo never breaks the render. For the pair heading, show logo `token0` + logo `token1` inline.

````jinja
## 💧 Uniswap {{ protocol }} pool — <img src="https://cryptocurrencyliveprices.com/img/{{ (token0 | lower) }}.png" width="24" height="24" /> {{ token0 }}/{{ token1 }} <img src="https://cryptocurrencyliveprices.com/img/{{ (token1 | lower) }}.png" width="24" height="24" />

| | |
|---|---|
| Spot price | `{{ spot_price }}` {{ token1 }} per {{ token0 }} |
| Fee tier | {{ fee_bps / 100 }}% |
| Liquidity | ${{ liquidity }} |
{% if apy %}| Est. APY | {{ apy | round(2) }}% |{% endif %}

What would you like to do? `create` a new position, `increase` an existing one, `decrease`/exit, or `claim_fees` — plus the amount{% if protocol in ["V3","V4"] %} and the price range (min–max){% endif %}.
````


### Check approval before adding liquidity

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/check_approval" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<address>","protocol":"<V2|V3|V4>","chainId":<chainId>,"lpTokens":[{"tokenAddress":"<token_address>","amount":"<decimal>"}],"action":"<create|increase|decrease|migrate>"}'
```

- `action` (required): `create`, `increase`, `decrease`, or `migrate`

If approval is needed, the transaction is automatically approved before the LP action.


### Create a V3/V4 LP position

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/create" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<address>","chainId":<chainId>,"protocol":"<V3|V4>","independentToken":{"tokenAddress":"<address>","amount":"<decimal>"},"existingPool":{"token0Address":"<address>","token1Address":"<address>","poolReference":"<pool_address_or_id>"},"priceBounds":{"minPrice":"<price>","maxPrice":"<price>"},"tickBounds":{"tickLower":<int>,"tickUpper":<int>},"batchPermitData":<permit_data_object>,"nativeTokenBalance":"<decimal>","urgency":"NORMAL"}'
```

- `protocol`: `V3` or `V4` (required)
- `independentToken` (required): `LPAmount` `{ tokenAddress, amount }` — amount in decimal (e.g. `"100.5"`), converted to wei server-side
- Use `existingPool` (`ExistingPoolParams`) to join an existing pool, or `newPool` (`NewPoolParams`) to create a new one
- `batchPermitData`: optional batch permit data for V4 positions
- `signature`: optional signed permit
- `nativeTokenBalance`: optional wallet native token balance (for native wrapping)

**V3 vs V4 — how they work:**

- **V3** — *concentrated liquidity*. You provide both tokens and choose a **price range** (`priceBounds` → `tickLower`/`tickUpper`). Your liquidity only earns fees while the price stays in that range; outside it you earn nothing until the price returns. Higher fee rewards at the cost of "out-of-range" risk. Each position is an ERC-721 NFT (`nftTokenId`).
- **V4** — same concentrated model **plus hooks**. Hooks are contracts that add custom logic (dynamic fees, custom curves). Pools are identified by `(token0, token1, fee, tickSpacing, hooks)` instead of just `(token0, token1, fee)` as in V3. `batchPermitData` covers the batched Permit2 for V4.

**What it returns & why:** builds the signed calldata that mints the position. Use `existingPool` to join a pool that already exists; use `newPool` (with `fee`, `tickSpacing`, `initialPrice`, optional `hooks`) to deploy a fresh pool. Choose V3 for plain concentrated ranges; choose V4 if you need hook features. Default to `existingPool` + a sensible `priceBounds` around the current spot price.


### Create a full-range V2 (classic) position

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/create_classic" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<address>","poolParameters":{"token0Address":"<address>","token1Address":"<address>","chainId":<chainId>},"independentToken":{"tokenAddress":"<address>","amount":"<decimal>"},"includeApprovalSimulation":true}'
```

- `poolParameters` (required): `V2PoolParams` `{ token0Address, token1Address, chainId }`
- `independentToken` (required): `LPAmount` `{ tokenAddress, amount }`
- `includeApprovalSimulation`: optional boolean — include approval simulation in response

**V2 vs V3/V4:** V2 is the classic *full-range* AMM — your liquidity is spread across the entire price curve `[0, ∞)`, so it earns fees at every price but with lower capital efficiency. No NFT, no price range, no hooks. Use V2 (create_classic) only for simple full-range exposure or when the pair has no V3/V4 pool yet.


### Increase an existing position

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/increase" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<address>","chainId":<chainId>,"protocol":"<V2|V3|V4>","token0Address":"<address>","token1Address":"<address>","nftTokenId":"<tokenId>","independentToken":{"tokenAddress":"<address>","amount":"<decimal>"},"v4BatchPermitData":<permit_data_object>}'
```

- `nftTokenId`: NFT token ID (V3/V4; **omit for V2**)
- `independentToken` (required): `LPAmount`
- `v4BatchPermitData`: optional batch permit data for V4 positions
- `signature`: optional signed permit

**What it does & why:** adds liquidity to a position you already own, identified by `nftTokenId` (for V3/V4) or by the token pair + wallet (for V2). Call it when the user wants to "add more" to an existing position rather than open a new one.


### Decrease an existing position

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/decrease" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<address>","chainId":<chainId>,"protocol":"<V2|V3|V4>","token0Address":"<address>","token1Address":"<address>","nftTokenId":"<tokenId>","liquidityPercentageToDecrease":<percent>,"withdrawAsWeth":false}'
```

- `liquidityPercentageToDecrease`: Integer `1`–`100`
- `nftTokenId`: NFT token ID (V3/V4; **omit for V2**)
- `withdrawAsWeth`: optional boolean to withdraw native as WETH

**What it does & why:** removes a percentage of your liquidity. Set `liquidityPercentageToDecrease=100` to fully exit, or a smaller value to take partial profit while staying in the pool. Call it when the user wants to "pull some liquidity out" or cash out part of a position.


### Claim accrued fees from a position

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/claim_fees" \
  -H "Content-Type: application/json" \
  -d '{"protocol":"<V3|V4>","walletAddress":"<address>","chainId":<chainId>,"tokenId":"<nftTokenId>","collectAsWeth":false}'
```

- `protocol`: `V3` or `V4` (required)
- `tokenId` (required): NFT token ID identifying the position
- `collectAsWeth`: optional boolean to collect native as WETH

**What it does & why:** collects the fees a position accrued but did not auto-compound. Call it when the user asks "how much have I earned" and wants to claim those fees to their wallet.


### Present the result to the user

Reply to the user using well-structured Markdown.

## Constraints

- Call `get_assets` to resolve the wallet address and verify sufficient funds before any LP action.
- Fetch `/lp/pool_info` and call `/lp/check_approval` before creating/increasing/decreasing positions.
- `independentToken` is always `LPAmount { tokenAddress, amount }` with amount in **decimal** (e.g. `"100.5"`). Add `decimals` only for tokens outside the curated registry.
- `poolParameters` on create-classic is `V2PoolParams`; use `NewPoolParams`/`ExistingPoolParams` for V3/V4 create.
- `nftTokenId` is required for V3/V4 increase/decrease and omitted for V2.
- `liquidityPercentageToDecrease` must be an integer between `1` and `100`.