---
name: buy-tokenized-stock
description: Buy tokenized US stocks (Ondo Finance) on-chain — resolve a ticker, check US market hours, and swap USDC for the tokenized stock via BlockSwap (Li.FI). Use when the user asks to buy, purchase, or invest in a tokenized stock (e.g. "buy $100 of Apple", "compra acciones de Tesla"). For research/analysis only use stock-market-analyst.
metadata:
  tool: bash
  category: rwa
  enabled: true
---

Today's date: {{DATE}}

# Buy Tokenized Stock

Buy tokenized US stocks (Ondo Finance) on BSC by swapping USDC for the stock token through BlockSwap (Li.FI). Resolves the ticker to its on-chain token, gates on US market hours, and executes the swap end to end.

## Instructions

- **Help the user choose.** Confirm the ticker, the USD amount, and the funding stablecoin before quoting.
- **Use `bash` for API calls** (curl) — never invent responses.
- Detect the user's language and reply in that language.
- Render results as Markdown, never raw JSON.
- **Never auto-execute** — always confirm the quote with the user before approving/executing (it moves real funds).


## Reference files

Discover the full reference for this skill in the `reference` folder. Load a file with `run_js` → `read_skill_reference`.

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"buy-tokenized-stock","file":"<name>"}`

To list all files, omit `file`.

Determine which reference to load by step:
- resolving the ticker, searching, checking market hours, or getting the buy quote → load `rwa-api`
- quoting, executing, or tracking the swap → load `endpoints`

## Workflow

### Step 1: Resolve the ticker

Load `rwa-api` and follow its "Resolve a ticker" section to map the ticker to its on-chain token (`symbol`, `chainId`, `contractAddress`, `multiplier`).

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"buy-tokenized-stock","file":"rwa-api"}`

### Step 2: Check market hours

Load `rwa-api` (if not already loaded) and follow its "Market status" section. If the market is closed, warn the user the reference price is stale and the DEX price may carry a premium — ask them to confirm before continuing.

### Step 3: Get the buy quote

Load `rwa-api` and follow its "Buy quote" section to convert the user's intent (USD amount OR number of shares) into the exact USDC spend and token count. The server does the multiplier math — use `totalUsd` as the blockswap `amount` and `contractAddress` as `token_out`. Show the user the token price, reference price, and the resulting USDC spend.

### Step 4: Verify the wallet

Call `run_js` with:
- **function**: "supported_blockchains"
- **data**: `{}`

Call `run_js` with:
- **function**: "get_assets"
- **data**: `{ "hasBalance": true }`

Confirm BSC (chainId 56) is in `supported_blockchains`, and find:
- **Funding stablecoin** — USDC on BSC (fall back to USDT/BUSD if the user holds those instead).
- **Native gas** — BNB on BSC (needed to pay the swap fee).
- **Wallet address** — the `address` field of an asset on BSC.

Stop if BSC is not supported or the user has no stablecoin + BNB on BSC.

### Step 5: Quote (estimate)

Load `endpoints` and follow its "Estimate" section to quote the swap with `sign:false` (stablecoin → the stock token). Render the estimate as a card and ask the user to confirm.

### Step 6: Approve (if needed)

If the quote's `approval_address` is non-null, ask the user to confirm, then call `run_js` with:
- **function**: "approve_token"
- **data**: `{"token":"<from_token>","spender":"<approval_address>","blockchain":"bsc"}`

### Step 7: Execute

After the user confirms, load `endpoints` and follow its "Execute" section to re-quote with `sign:true`. The `transaction_request` is auto-signed and broadcast.

### Step 8: Track

Load `endpoints` and follow its "Track" section to poll `/status` until `DONE`. Report the final status and the tokens received. Never invent a tx hash or amount.

## Constraints

- **No state file.** Keep the resolved token, quote, and tx id in memory.
- **Never invent prices or amounts.** Ask if missing.
- **Payment requires explicit user intent** — confirm before `sign:true` (it moves real USDC).
- **Market closed** → warn, don't silently proceed.
- This skill is ONLY for tokenized US stocks (Ondo Finance). Do NOT use it for crypto tokens like BTC, ETH, SOL.
