---
name: uniswap-pools
description: Discover and recommend the best Uniswap liquidity pools to invest in, then open and manage LP positions. The agent autonomously finds top pools across the wallet's chains — the user does NOT need to name tokens or blockchains — ranks them by liquidity, fee tier, volume and impermanent-loss risk, and recommends where to provide liquidity. Use when the user wants to invest in pools, provide/add liquidity, earn yield from LP, open/manage an LP position, or claim fees.
metadata:
  tool: bash,sign_transaction
  category: defi
  enabled: true
  homepage: https://developers.uniswap.org/api/trading-api
---

# Uniswap Liquidity Pools

Discover and recommend the best Uniswap liquidity pools to invest in, then open and manage V2/V3/V4 LP positions. Help users earn yield from liquidity provision — all non-custodial through BlockVault.

## Instructions

- **Get the address first.** Call `run_js` → `get_assets` with `{"hasBalance": true}` to obtain the user's real address and holdings before any API call.
- **Discover pools autonomously.** Never ask the user for tokens, chains, fee tiers, or price ranges — resolve them from the wallet and the market yourself.
- **Use `bash` for API calls.** Execute `curl` commands against the BlockVault Uniswap API — never invent responses.
- **Do NOT spawn subagents.** Run the discovery yourself, directly. Subagents loop and duplicate `pool_info` calls.
- **Bound the search.** Query at most ~10 pairs total, then stop and rank. Never retry a pair that already failed.
- Detect the user's language and reply in that language.
- Render results as Markdown tables, never raw JSON.
- Lead with the **recommendation** (best pool + why), not technical fields.

## API endpoints

Base URL: `https://402.blockvault.ai`
Prefix: `/api/v1/uniswap`

```bash
# GET top tokens by TVL on a chain
curl -sS "https://402.blockvault.ai/api/v1/uniswap/tokenlist?sort=tvl&limit=10&chainId=<CHAIN_ID>"

# GET top tokens by 24h volume on a chain
curl -sS "https://402.blockvault.ai/api/v1/uniswap/tokenlist?sort=volume_24h&limit=10&chainId=<CHAIN_ID>"

# GET the wallet's LP positions (read on-chain)
curl -sS "https://402.blockvault.ai/api/v1/uniswap/lp/positions/<ADDRESS>"

# POST pool state (liquidity, fee, price) for a token pair
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/pool_info" \
  -H "Content-Type: application/json" \
  -d '{"protocol":"V3","poolParameters":{"tokenAddressA":"<ADDRESS_A>","tokenAddressB":"<ADDRESS_B>","fee":<FEE>},"chainId":<CHAIN_ID>}'

# POST check LP token approval (pass BOTH tokens of the pair)
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/check_approval" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","protocol":"V3","chainId":<CHAIN_ID>,"lpTokens":[{"tokenAddress":"<TOKEN0_ADDRESS>","amount":"<DECIMAL>"},{"tokenAddress":"<TOKEN1_ADDRESS>","amount":"<DECIMAL>"}],"action":"create"}'

# POST create a V3/V4 LP position (use tickBounds, NOT priceBounds)
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/create" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"protocol":"V3","independentToken":{"tokenAddress":"<TOKEN_ADDRESS>","amount":"<DECIMAL>"},"existingPool":{"token0Address":"<ADDRESS_A>","token1Address":"<ADDRESS_B>","poolReference":"<POOL_ADDRESS>"},"tickBounds":{"tickLower":<TICK_LOWER>,"tickUpper":<TICK_UPPER>},"urgency":"NORMAL","simulateTransaction":true}'

# POST create a full-range V2 position
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/create_classic" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","poolParameters":{"token0Address":"<ADDRESS_A>","token1Address":"<ADDRESS_B>","chainId":<CHAIN_ID>},"independentToken":{"tokenAddress":"<TOKEN_ADDRESS>","amount":"<DECIMAL>"},"includeApprovalSimulation":true,"simulateTransaction":true}'

# POST increase an existing position
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/increase" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"protocol":"V3","token0Address":"<ADDRESS_A>","token1Address":"<ADDRESS_B>","nftTokenId":"<TOKEN_ID>","independentToken":{"tokenAddress":"<TOKEN_ADDRESS>","amount":"<DECIMAL>"},"simulateTransaction":true}'

# POST decrease an existing position
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/decrease" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"protocol":"V3","token0Address":"<ADDRESS_A>","token1Address":"<ADDRESS_B>","nftTokenId":"<TOKEN_ID>","liquidityPercentageToDecrease":<PERCENT>,"withdrawAsWeth":false,"simulateTransaction":true}'

# POST claim accrued fees from a position
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/claim_fees" \
  -H "Content-Type: application/json" \
  -d '{"protocol":"V3","walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"tokenId":"<TOKEN_ID>","collectAsWeth":false,"simulateTransaction":true}'
```

**Notes:**
- Replace `<CHAIN_ID>` with `1` (Ethereum), `137` (Polygon), or `8453` (Base).
- `amount` is a **string in decimal units** (e.g. `"100.5"`), converted to wei server-side.
- `fee` is in hundredths of a basis point: `100` = 0.01%, `500` = 0.05%, `3000` = 0.3%, `10000` = 1%.
- **Use `tickBounds` (integers), NOT `priceBounds`.** The upstream `priceBounds` field is broken and returns `"tickPrice" does not match any of the allowed types`. `tickBounds` takes raw tick integers `{ "tickLower": <int>, "tickUpper": <int> }` and works. Derive ticks from the pool's `currentTick` and `tickSpacing` (both returned by `pool_info`): pick `tickLower`/`tickUpper` as multiples of `tickSpacing` straddling `currentTick` (e.g. currentTick −200·spacing to +200·spacing for a wide range, or ±20·spacing for a tight range).
- **`poolReference` MUST be the `poolReferenceIdentifier` from `pool_info`** for that exact pair + fee tier. Never guess or reuse a pool address from another pair.
- **`walletAddress` MUST be the user's wallet address from `get_assets`** (the `address` field of an asset), never a token address from the `tokenlist` or `pool_info` response.
- Mutating endpoints (`check_approval`, `create`, `create_classic`, `increase`, `decrease`, `claim_fees`) return a signable envelope with an ordered `transactions[]` list (approvals first, then the action), each entry `{ to, data, value, chainId }`. The `bash` tool auto-detects these envelopes and signs+broadcasts each transaction in order. If auto-detection does not fire, sign manually with `run_js` → `sign_transaction` using each returned `to`, `data`, and the matching `blockchain` (resolved from `chainId`).
- **Always set `simulateTransaction: true`** on mutating LP calls. The API simulates the transaction on-chain before returning it. If the simulation fails, the response contains an error like `"Fail with error 'STF'"` (Simulation Transaction Failed) or a `txFailureReason` — **do NOT sign or broadcast in that case**. Only proceed when the response returns a clean `transactions[]` envelope with no simulation error.

## Typed sub-schemas

- `LPAmount` — `{ "tokenAddress": string, "amount": string, "decimals"?: number }` — `amount` is human-readable **decimal** (e.g. `"100.5"`), converted to wei server-side. `decimals` is only needed for tokens outside the curated registry.
- `V2PoolParams` — `{ "token0Address": string, "token1Address": string, "chainId": number }`.
- `NewPoolParams` — `{ "token0Address", "token1Address", "fee" (bps), "tickSpacing", "hooks"? (V4), "initialPrice" }`.
- `ExistingPoolParams` — `{ "token0Address", "token1Address", "poolReference" }` (pool address for V3, pool ID for V4).
- `PriceBounds` — `{ "minPrice": string, "maxPrice": string }` (token1 per token0). **BROKEN upstream — do not use. Use `TickBounds` instead.**
- `TickBounds` — `{ "tickLower": number, "tickUpper": number }`. **Use this.** Raw tick integers, multiples of the pool's `tickSpacing`, straddling `currentTick`.
- `urgency` — optional enum `NORMAL | FAST | URGENT` on mutating LP calls. Default `NORMAL`.
- `simulateTransaction` — optional boolean; set `true` to dry-run and inspect the result before broadcasting.

## Curated token addresses

For forming pairs without a tokenlist round-trip:

| Chain | Symbol | Address |
|---|---|---|
| Ethereum (1) | ETH / WETH | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` |
| Ethereum (1) | USDC | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| Ethereum (1) | USDT | `0xdAC17F958D2ee523a2206206994597C13D831ec7` |
| Ethereum (1) | DAI | `0x6B175474E89094C44Da98b954EedeAC495271d0F` |
| Ethereum (1) | WBTC | `0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599` |
| Base (8453) | WETH | `0x4200000000000000000000000000000000000006` |
| Base (8453) | USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| Base (8453) | DAI | `0x50c5725949A6F0c72E6C4a641F24049A917DB0Cb` |
| Polygon (137) | WETH | `0x7ceB23fD6bC0adD59E62ac25578270cFf1b9f619` |
| Polygon (137) | USDC | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` |
| Polygon (137) | USDT | `0xc2132D05D31c914a87C6611C10748AEb04B58e8F` |
| Polygon (137) | DAI | `0x8f3Cf7ad23Cd3CaDbD9735AFf958023239c6A063` |
| Polygon (137) | WBTC | `0x1BFD67037B42Cf73acF2047067bd4F2C47D9BfD6` |

Native ETH is `0x0000000000000000000000000000000000000000` on every chain.

## Address resolution

Before any action, get the user's address and holdings:

Call `run_js` with:
- **function**: `"get_assets"`
- **data**: `{"hasBalance": true}`

Note each asset's `symbol`, `blockchain`, and `balance`. These holdings are the natural LP candidates (the user already owns them), and they tell you which chains to search.

**The wallet address is the `address` field of an asset** (e.g. `0x…` on the target chain). Use it verbatim as `walletAddress` in every mutating call. Never use a token `address` from the `tokenlist` or `pool_info` response as `walletAddress` — those are token contracts, not the wallet.

Also call `run_js` with:
- **function**: `"supported_blockchains"`
- **data**: `{}`

Note the chains the wallet supports (name + `chainId`). Only chains in this list are usable.

## Discovery flow (recommend pools)

Follow all steps silently. DO NOT OMIT ANY STEP.

1. **Read the wallet** (Address resolution above): get `supported_blockchains` + `get_assets`.

2. **Discover top tokens** on each supported chain that is Ethereum, Base, or Polygon:

   ```bash
   curl -sS "https://402.blockvault.ai/api/v1/uniswap/tokenlist?sort=tvl&limit=10&chainId=<CHAIN_ID>"
   curl -sS "https://402.blockvault.ai/api/v1/uniswap/tokenlist?sort=volume_24h&limit=10&chainId=<CHAIN_ID>"
   ```

   Response: `{ "tokens": [ { "symbol", "name", "address", "chainId", "decimals", "logoURI", ... } ] }`. This gives the highest-liquidity and highest-volume tokens per chain — the building blocks of the best pools.

3. **Build candidate pairs** in this priority order (skip any pair whose tokens do not exist on that chain):

   1. **Stable/stable** (near-zero impermanent loss): USDC/USDT, USDC/DAI, USDT/DAI.
   2. **Stable/blue-chip** (high volume, moderate IL): USDC/ETH, USDC/WETH, USDC/WBTC, USDT/ETH.
   3. **Blue-chip/blue-chip**: ETH/WBTC, WETH/WBTC.
   4. **User's holdings × stablecoin/native**: for each token the user holds with a balance, pair it with USDC and with the chain's native (ETH/WETH).

4. **Fetch pool state** for each pair with the `pool_info` endpoint. V3 requires `fee`, so query the most likely fee tier first and fall back if the result is empty:

   - Stable/stable → try `fee: 100`, then `500`.
   - Stable/blue-chip and blue-chip/blue-chip → try `fee: 500`, then `3000`, then `10000`.

   Response: `{ "pools": [ { "poolReferenceIdentifier", "poolProtocol", "tokenAddressA", "tokenAddressB", "fee", "tickSpacing", "chainId", "tokenDecimalsA", "tokenDecimalsB", "poolLiquidity", "sqrtRatioX96", "currentTick", "protocolFee" } ] }`.

   - `poolReferenceIdentifier` is the pool address (V3) — use it as `poolReference` when creating a position.
   - `poolLiquidity` is the pool's depth (higher = deeper, safer, less slippage).
   - Spot price ≈ `(sqrtRatioX96 / 2^96)²` token1 per token0, adjusted by `10^(tokenDecimalsA - tokenDecimalsB)`. If the exact price is unclear, present liquidity + fee tier + volume instead — the recommendation does not depend on an exact price.
   - An empty `pools` array means that pair/fee does not exist on that chain — try the next fee tier or drop the pair.
   - **A `404` with `"Decimals not available"` means the token is not in Uniswap's registry.** Drop that pair immediately — do NOT retry it with the token order swapped, and do NOT try other fee tiers for it. Move on to the next pair.
   - **Stop after ~10 pairs.** Once you have 3–5 pools with real `poolLiquidity`, stop querying and rank them. Do not keep probing pairs.

5. **Score and rank** each pool 0–100 across three dimensions:

   - **Liquidity depth (0–40)**: the deepest `poolLiquidity` among candidates scores 40; scale the rest proportionally.
   - **Fee income potential (0–30)**: higher fee tier + higher token 24h volume (from step 2) = more fees. A 500-fee stable pair with huge volume scores high; a 10000-fee exotic with tiny volume scores low.
   - **Impermanent-loss safety (0–30)**: stable/stable = 30; stable/blue-chip = 20; blue-chip/blue-chip = 15; volatile/volatile = 5.

   Rank by total score. Keep the top 3–5 pools. Group them by risk profile:

   - **Low risk** — stable/stable pairs (near-zero IL, steady but modest yield).
   - **Medium risk** — stable/blue-chip pairs (higher yield, some IL).
   - **Higher risk** — blue-chip/blue-chip or volatile pairs (highest yield, real IL).

   Prefer pools whose tokens the user already holds (no swap needed to enter). If the user holds only one side of a pair, note that the other side will be bought automatically when the position is created.

6. **Present the recommendation and ask ONE question.** Render the ranked pools with the template in the Rendering section, then ask in plain text which pool and how much. Do not ask about tokens, chains, fee tiers, or price ranges — you have already resolved them. Wait for the user's answer before executing.

## Open position flow

1. Resolve the chosen pool to its `poolReferenceIdentifier` (pool address) and fee tier from the discovery flow.

2. **Verify the user holds BOTH tokens of the pair.** A liquidity position requires both sides — the API computes the dependent token amount and the transaction pulls it from the wallet. If the user only holds one side (e.g. USDC but no WETH), the mint will revert on-chain. Check the balances from `get_assets` (Address resolution). If one side is missing:
   - Tell the user plainly: "This pool needs both USDC and WETH. You have USDC but no WETH."
   - Offer to swap half the held token into the missing side first (use the `blockswap` skill), then open the position. Do NOT call `/lp/create` until both tokens are present.

3. Check approval with the `check_approval` endpoint (`action: "create"`). **Pass BOTH tokens of the pair in `lpTokens`** — a V3 mint pulls both sides from the wallet, so both need approval to the NonfungiblePositionManager. If the response returns approval transactions, they are signed and broadcast automatically before the LP action. Do NOT skip this step: a missing approval on either token makes the mint revert on-chain.

4. Create the position with `simulateTransaction: true`, using **`tickBounds`** (not `priceBounds`). Derive the ticks from the pool's `currentTick` and `tickSpacing` (from `pool_info`): pick `tickLower`/`tickUpper` as multiples of `tickSpacing` straddling `currentTick`. For a stable pair use a tight range (±20·spacing); for a volatile pair use a wide range (±200·spacing). For a stable/stable pair you may instead use a full-range V2 position (`create_classic`).

   - **V3** — *concentrated liquidity*. You provide both tokens and choose a **price range** (`priceBounds` → `tickLower`/`tickUpper`). Your liquidity only earns fees while the price stays in that range; outside it you earn nothing until the price returns. Higher fee rewards at the cost of "out-of-range" risk. Each position is an ERC-721 NFT (`nftTokenId`).
   - **V4** — same concentrated model **plus hooks**. Hooks are contracts that add custom logic (dynamic fees, custom curves). Pools are identified by `(token0, token1, fee, tickSpacing, hooks)` instead of just `(token0, token1, fee)` as in V3. `batchPermitData` covers the batched Permit2 for V4.
   - **V2** (`create_classic`) — the classic *full-range* AMM: liquidity spread across the entire price curve `[0, ∞)`, earning fees at every price but with lower capital efficiency. No NFT, no price range, no hooks. Use for simple full-range exposure, especially stable/stable pairs.

   Default to `existingPool` + a sensible `priceBounds` around the current spot price. Use `newPool` (`NewPoolParams`) only to deploy a brand-new pool.

5. **Check the simulation result before signing.** If the response contains a simulation error (`"Fail with error 'STF'"`, `txFailureReason`, or any error field), do NOT sign or broadcast — tell the user the position would revert and why (e.g. insufficient balance, price moved out of range, approval missing). Only sign and broadcast when the response returns a clean `transactions[]` envelope.

6. Notify the user of the result.

## Manage position flows

**List positions** — call `GET /lp/positions/<ADDRESS>` to read the wallet's positions on-chain across V2, V3 and V4. Each position carries `protocol` (`V2`/`V3`/`V4`), `chain_id`, `token0`, `token1`, `fee`, `tick_lower`, `tick_upper`, `liquidity` (abstract sqrt(x·y) units), `amount0`/`amount1` (the actual token amounts backing the position, in decimal units), `tokens_owed0`/`tokens_owed1` (pending fees, V3, decimal units), `pool`, `current_tick`, `in_range`, and (V2) `reserves0`/`reserves1`. Use this to answer "list my positions" or to resolve the `token_id` for increase/decrease/claim — never ask the user for it.

- **Increase** — call the `increase` endpoint with `nftTokenId` (V3/V4; omit for V2) and the `independentToken` amount. Adds liquidity to a position the user already owns.
- **Decrease / withdraw** — call the `decrease` endpoint with `nftTokenId` (V3/V4; omit for V2) and `liquidityPercentageToDecrease` (integer `1`–`100`). Set `100` to fully exit, or a smaller value to take partial profit while staying in the pool. For V3, uncollected fees are included in the withdrawal automatically — no separate `claim_fees` needed.
- **Claim fees** — call the `claim_fees` endpoint with `tokenId` (V3/V4). Collects fees a position accrued but did not auto-compound. Only needed if you want to claim fees while keeping the position open.

### Withdraw / exit flow

1. Call `GET /lp/positions/<ADDRESS>` to list the wallet's positions and resolve the `token_id` of the position to exit (V3/V4). For V2, identify the position by token pair + wallet instead.
2. Call `check_approval` with `action: "decrease"` (V3/V4 may require approving the NFT to the position manager).
3. Call `decrease` with `simulateTransaction: true` and `liquidityPercentageToDecrease: 100` (or the desired partial amount). Check the simulation result before signing, as in the open flow.
4. Sign and broadcast the returned `transactions[]` envelope. Notify the user of the result.

## Rendering

**Pool recommendation:**

````jinja
## 💧 Best liquidity pools for you

{% for p in pools %}
### {{ loop.index }}. {% if p.logoURI0 %}<img src="{{ p.logoURI0 }}" width="24" height="24" /> {% endif %}{{ p.token0 }}/{{ p.token1 }}{% if p.logoURI1 %} <img src="{{ p.logoURI1 }}" width="24" height="24" />{% endif %} on {{ p.chain }} — {{ p.risk }}
- **Fee tier**: {{ p.fee_pct }}% · **Liquidity**: {{ p.liquidity }} · **Score**: {{ p.score }}/100
- **Why**: {{ p.reason }}
{% endfor %}

**My recommendation**: {{ recommendation }}

Which pool would you like to invest in, and how much? (e.g. "USDC/ETH on Base, 500 USDC")
````

**Token logos:** each pool carries `logoURI0` and `logoURI1` — the `logoURI` from the `tokenlist` response (step 2), mapped by token symbol. Guard every `<img>` with `{% if %}` so a missing logo never breaks the render. Do NOT use a symbol-keyed CDN — use the real `logoURI` from the API.

## Constraints

- Call `get_assets` and `supported_blockchains` to resolve the wallet and verify sufficient funds before any LP action.
- **Verify the user holds BOTH tokens of the pair before `/lp/create`.** A position needs both sides; if one is missing, offer to swap into it first (via `blockswap`) instead of opening a position that will revert.
- Discover pools autonomously — never ask the user for tokens, chains, fee tiers, or price ranges.
- Do NOT spawn subagents; run discovery directly. Query at most ~10 pairs, never retry a failed pair, and stop once you have 3–5 ranked pools.
- Fetch `/lp/pool_info` and call `/lp/check_approval` before creating/increasing/decreasing positions.
- **`check_approval` must include BOTH tokens of the pair in `lpTokens`** — a V3 mint pulls both sides, so both need approval. Missing approval on either token reverts the mint on-chain.
- Always set `simulateTransaction: true` on mutating LP calls and check the result before signing. If the response contains a simulation error (`"Fail with error 'STF'"` or `txFailureReason`), do NOT sign or broadcast — report the failure to the user instead.
- `independentToken` is always `LPAmount { tokenAddress, amount }` with amount in **decimal** (e.g. `"100.5"`). Add `decimals` only for tokens outside the curated registry.
- Use `tickBounds` (integers) for V3/V4 price ranges — `priceBounds` is broken upstream. Derive ticks from `pool_info` (`currentTick` + `tickSpacing`).
- `poolReference` must be the `poolReferenceIdentifier` from `pool_info` for that exact pair + fee tier.
- `walletAddress` must be the user's wallet address from `get_assets`, never a token address.
- `poolParameters` on create-classic is `V2PoolParams`; use `NewPoolParams`/`ExistingPoolParams` for V3/V4 create.
- `nftTokenId` is required for V3/V4 increase/decrease and omitted for V2.
- `liquidityPercentageToDecrease` must be an integer between `1` and `100`.
- Display fee tiers as `fee / 10000` percent (500 → 0.05%).