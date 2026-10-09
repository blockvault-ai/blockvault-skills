---
name: uniswap-pools
description: Discovers and ranks the best Uniswap liquidity pools, then opens and manages V2/V3/V4 LP positions. Use when the user wants to provide/add liquidity, earn yield from LP, open/manage a position, claim fees, or invest in pools. Finds top pools autonomously (no token/chain needed from the user) and guarantees both sides of the pair are funded before minting.
metadata:
  category: defi
  enabled: true
---

# Uniswap Liquidity Pools

Discover and rank pools, then open and manage V2/V3/V4 LP positions — non-custodial through BlockVault.

Base URL: `https://402.blockvault.ai` · Prefix: `/api/v1/uniswap`

## Golden rules

1. **Discovery is ONE call.** `GET /pools/recommend` already discovers, scores (0-100) and ranks pools across Ethereum/Base/Polygon (cached in Redis). Never probe `tokenlist`/`pool_info` yourself; never spawn subagents for discovery.
2. **Fund both sides before ANY LP call.** A position needs BOTH tokens. Missing side → `add_asset` (register if unlisted) → `blockswap` into it → re-check `get_assets`. Never call `check_approval`/`create` with a missing side — the mint reverts.
3. **Estimate → approve → create.** `check_approval` (both tokens) → `create` with `simulateTransaction:true`. Only sign/broadcast a clean `transactions[]` envelope.
4. **`tickBounds`, never `priceBounds`** — the upstream `priceBounds` field is broken. Derive ticks from the pool's `currentTick`/`tickSpacing`.

## Reference files

Read on demand via `run_js` → `read_skill_reference` (`{skill: "uniswap-pools", file: "<name>"}`):

- `endpoints` — all curl commands (discover, positions, check_approval, create, create_classic, increase, decrease, claim_fees).
- `protocols` — V2/V3/V4 table and routing by `protocol`.
- `fields` — field reference (chainId, amount, fee, tickBounds, poolReference, …).

## Workflow

### Step 1: Read the wallet

Call `run_js` with:
- **function**: "supported_blockchains"
- **data**: `{}`

Note the usable chains (only these can hold a position).

Call `run_js` with:
- **function**: "get_assets"
- **data**: `{"hasBalance": true}`

Note each asset's `symbol`, `blockchain`, and `balance`, plus the wallet `address` (the `address` field of an asset on the target chain). Use it verbatim as `walletAddress` — never a token address from `/pools/recommend`.

### Step 2: Discover pools

`curl .../pools/recommend -o pools.json` (one call, with `-o` so the `bash` result returns `artifact_path`). See `reference/endpoints.md` for the exact command. `poolReferenceIdentifier` = pool address; `protocol` = V2/V3/V4; `apy` = 7-day fee APY (lead with it); `risk` = Low/Medium/Higher; `score` 0-100.

**Render as a card deck — do NOT read the JSON and do NOT write a Markdown list.** Emit this exact ```artifact fence, then a one-line **My pick:** + ONE question (which pool, how much). Never ask for tokens/chains/fees/ranges — they're resolved.

````markdown
```artifact
{"artifact":"<artifact_path>","template":"uniswap-pools:pool-list"}
```
````

### Step 3: Confirm the protocol with the user

Read `protocol` from the pool object (V2/V3/V4). **Ask the user which protocol they want**. Present the trade-off in one line (V3/V4 = concentrated, higher APY, out-of-range risk; V2 = full-range, simpler, no range risk). If the pool is V2-only, skip the question and use V2.

### Step 4: Fund both sides

Resolve `poolReferenceIdentifier` + fee tier + **`protocol`** from the pool object. If one token is missing from `get_assets`, register it with `add_asset` (only if unlisted), then `blockswap` into it, then re-check `get_assets`. Stop if no route.

Call `run_js` with:
- **function**: "add_asset"
- **data**: `{"symbol":"<TOKEN_SYMBOL>","address":"<token0Address|token1Address>","decimals":<token0Decimals|token1Decimals>,"blockchain":"<chain>"}`

`symbol`, `address` and `decimals` all come from the pool object (`token0Address`/`token0Decimals` or `token1Address`/`token1Decimals`). `blockchain` is the pool's `chain` name.

### Step 5: Check approval

`check_approval` with `action:"create"`, **both tokens in `lpTokens`**. `protocol` must match the pool's `protocol`. See `reference/endpoints.md`.

### Step 6: Open the position

Open based on `protocol` (see `reference/protocols.md`):
- `V2` → `create_classic` (`poolParameters`, no `tickBounds`).
- `V3`/`V4` → `create` (`existingPool` + `tickBounds`).

**Token order (critical):** `token0Address`/`token1Address` from `/pools/recommend` are already in the pool's canonical order. Pass them to `existingPool`/`poolParameters` **exactly as returned** — never swap or reorder them, or the mint computes wrong amounts.

**Amount (critical):** `independentToken.amount` is in **decimal units, never wei**. For "X USD", provide the stablecoin side (USDT/USDC/DAI) with `amount: "X"` — 1 stablecoin = 1 USD. If the pool has no stablecoin, convert USD to the token amount using the current price. Never pass wei or a tiny fraction; the server converts decimal → wei.

Always `simulateTransaction:true`.

### Step 7: Verify simulation (feedback loop)

- `STF`/`txFailureReason` → do NOT sign; report why and stop.
- `transactions[]` empty → the wallet has no approval to spend the tokens. Sign the approval transactions from `check_approval` first, then re-run `create`. Do NOT sign an empty envelope.
- Clean `transactions[]` → sign + notify.

## Manage positions

- **List** — `GET /lp/positions/<ADDRESS>` (V2/V3/V4). Resolve `token_id` from here.
- **Increase** — `increase` + `nftTokenId` (omit for V2) + `independentToken`.
- **Decrease/withdraw** — `decrease` + `nftTokenId` + `liquidityPercentageToDecrease` (1-100; 100 = full exit). V3 fees auto-included; no separate `claim_fees`.
- **Claim fees** — `claim_fees` + `tokenId` (only to claim while keeping the position).

### Withdraw / exit

1. `GET /lp/positions/<ADDRESS>` → resolve `token_id`.
2. `check_approval` `action:"decrease"` (V3/V4 NFT approval).
3. `decrease` `simulateTransaction:true`, `liquidityPercentageToDecrease:100` → check simulation → sign.

## Rendering

After the search, emit the ```artifact fence (Step 2) so the app renders `data.pools` as a swipeable card deck. **Do NOT re-list pools in Markdown — the card deck IS the list.** Token logos come from the API's `logoURI0`/`logoURI1` — never a symbol-keyed CDN.