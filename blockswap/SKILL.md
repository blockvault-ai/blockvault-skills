---
name: blockswap
description: Swaps, bridges, and transfers tokens via BlockSwap (Li.FI aggregation). Use when the user asks to swap tokens (same-chain or cross-chain), bridge between chains, convert a token for a DeFi action (e.g. into LP gas or the other side of a pair), or top up native gas on a chain. Plans routes as a single aggregator quote and handles native-gas funding.
metadata:
  category: defi
  enabled: true
---

# BlockSwap (same-chain swap & cross-chain transfer)

One `/quote` endpoint covers both same-chain swaps and cross-chain bridges. Li.FI (the aggregator behind BlockSwap) routes cross-chain internally — express the goal as ONE quote, never as a chain of manual swaps.

Base URL: `https://402.blockvault.ai`

## Golden rules

1. **ONE quote = the whole goal.** Quote `token_in → token_out` directly (set `to_chain` for a bridge). Li.FI finds the route. Never hand-chain intermediate swaps: `USDC→ETH→WETH` is a longer, costlier path than a single `USDC→WETH` quote.
2. **Gas before funds.** A token balance is not enough — every transaction needs native gas on its source chain. Verify gas in `get_assets` before quoting; fund it first if missing (see Gas management).
3. **Estimate → approve → sign.** `sign=false` = estimate (moves nothing). If it returns `approval_address`, approve that spender first. `sign=true` = the only step that moves funds — confirm with the user before calling it.
4. **Read the wallet once**, then act. `supported_blockchains` + `get_assets` gives you chains, balances, wallet address, and gas in one pass.

## Reference files

Read on demand via `run_js` → `read_skill_reference` (`{skill: "blockswap", file: "<name>"}`):

- `endpoints` — all curl commands (chains, tokens, quote, status).
- `fields` — quote field table.

## Wallet & gas

**Read the wallet:**
- `run_js` → `supported_blockchains` → chains with a derived address (only these are usable).
- `run_js` → `get_assets` `{"hasBalance": true}` → balances + the wallet `address` per chain.

**Native gas token per chain:** Ethereum/Base/Arbitrum/Optimism = ETH, Polygon = POL, BSC = BNB.

**If the source chain has no native gas:**
- Has some gas + holds a non-native token → same-chain swap a slice of it into gas to top up.
- **Zero gas → a same-chain swap is impossible** (you cannot pay its own fee). Fund from another chain that HAS gas: bridge the native token (or one that lands as gas) from the funded chain to the target chain, re-check `get_assets`, then proceed.
- No chain has gas → tell the user and stop.

Bridges spend gas on BOTH ends (send + possibly claim). Keep native gas on the destination too if a follow-up step needs it.

## Workflow

### Step 1: Read the wallet

Call `run_js` with:
- **function**: "supported_blockchains"
- **data**: `{}`

Call `run_js` with:
- **function**: "get_assets"
- **data**: `{"hasBalance": true}`

Note chains, balances, the wallet `address` per chain, and native gas. Verify the source chain has native gas before quoting.

### Step 2: Estimate (`sign=false` — moves nothing)

`curl` the `/quote` endpoint with `sign:false` (see `reference/endpoints.md`). Present the estimate (send → receive, min received, slippage) and ask the user to confirm. Note `approval_address` if present.

Render token logos with a symbol-keyed CDN (≤ 28px, guard every `<img>` with `{% if %}`): `https://cryptocurrencyliveprices.com/img/{{ symbol | lower }}.png`. Omit the image if the logo is not known to exist.

### Step 3: Approve (only if `approval_address` is non-null)

Ask the user to confirm, then call `run_js` with:
- **function**: "approve_token"
- **data**: `{"token":"<from_token>","spender":"<approval_address>","blockchain":"<src_chain_name>"}`

Wait for it to succeed before Step 4.

### Step 4: Execute (`sign=true` — moves funds, confirm first)

After the user confirms, re-quote with `sign:true` (see `reference/endpoints.md`). The response's `transaction_request` is auto-signed+broadcast.

### Step 5: Track & notify (feedback loop)

`curl` the `/status` endpoint (see `reference/endpoints.md`). Report final status (`NOT_FOUND | PENDING | DONE | FAILED`; `DONE` → `substatus` `COMPLETED | PARTIAL | REFUNDED`) and the `token_out` received. Never invent a tx hash or amount.