---
name: blockswap
description: Execute same-chain swaps and cross-chain transfers via BlockSwap (Li.FI aggregation). Discovers supported chains and tokens, returns a quote (with sign=false showing only the estimate, or sign=true returning the transaction to sign), and tracks transfer status by tx hash. Use when the user asks to swap tokens (same chain OR cross-chain), bridge, or transfer assets between chains. This is the canonical swap/bridge skill — prefer it over uniswap for cross-chain transfers.
metadata:
  category: defi
  enabled: true
---

### Instructions
Before any swap or transfer, get the balances of the agent wallet and verify the source tokens are available.
Discover the supported chains/tokens, get a signable quote, sign+broadcast the returned transaction, then track its status. Finally, notify the user of the result.

If there are insufficient funds, or the swap fails, notify the user with the error message.
Do not create a swap or transfer without first checking the balances and confirming the tokens/chains are supported.


# BlockSwap (same-chain swap & cross-chain transfer)

Execute swaps and cross-chain transfers via the BlockVault BlockSwap API (Li.FI aggregation).
Base URL: `https://402.blockvault.ai`

A single `/quote` endpoint covers **both** same-chain swaps and cross-chain transfers: omit `to_chain` (or set it equal to `chain`) for a same-chain swap, or set `to_chain` to a different chain to bridge.

Two modes, driven by the `sign` field:

- **`sign=false`** (default) — returns only the **estimate**: what you give (`from_amount_decimal`) and what you receive (`to_amount_decimal`/`to_amount_min`). It does **not** return a `transaction_request`, so it never triggers the metatransaction sign flow. Use this to present the rate to the user before executing.
- **`sign=true`** — additionally returns the **signing flow**: `transaction_request` (EVM `to`/`data`/`value`/`chainId`), `transaction_id`, `approval_address` and `execution_duration`. The `bash` tool auto-detects metatransaction responses (containing `to`, `data`, `chainId`) and signs+broadcasts them via WDK.

Always quote with `sign=false` first to show the user the estimate, then re-quote with `sign=true` only when executing.

### Get the wallet address and balances

First, learn which blockchains the wallet actually supports. Call `run_js` with:
- **function**: "supported_blockchains"
- **data**: `{}`

The response lists every blockchain on which the wallet has a derived address (independent of token balances). **Only blockchains in this list are usable for a swap/bridge.** Note the list of available chains.

Then get the wallet address and source-chain funds. Call `run_js` with:
- **function**: "get_assets"
- **data**: `{"hasBalance": true}`

Each asset carries its `blockchain` (chain name) and `address` (the wallet address on that chain). Use these to obtain the `swapper` address for the source chain and to verify sufficient funds for the swap/transfer plus gas.


### Discover supported chains and tokens

Call `bash` to list the chains and tokens that **BlockSwap** supports. **This is a read-only discovery step** so you can map the user's intent ("swap USDC for WETH", "bridge USDC on Base to POL on Polygon") to concrete symbols and chain ids before quoting.

```bash
# supported chains (id, key, name, chain_type)
curl -s "https://402.blockvault.ai/api/v1/blockswap/chains"

# tokens available on the supported chains (Li.FI catalog)
curl -s "https://402.blockvault.ai/api/v1/blockswap/tokens"

# optionally filter tokens by chain ids (comma-separated, e.g. '1,8453')
curl -s "https://402.blockvault.ai/api/v1/blockswap/tokens?chains=<chain_ids>"
```

**Intersect the two lists before acting:** only swap/bridge across chains that appear in **both** `/blockswap/chains` (BlockSwap-supported) **and** the `supported_blockchains` response (wallet-supported). If the source or destination chain is not in both lists, do not attempt the swap — notify the user and stop.


### Get an estimate quote (`sign=false`)

Return an estimate for either a same-chain swap or a cross-chain transfer. **This step does not move funds and does not trigger signing** — it returns the amounts so you can present the rate to the user.

Set `chain` and `to_chain` to **chain ids from `/blockswap/chains`**, restricted to ids present in both `/blockswap/chains` and the `supported_blockchains` response. Never quote a swap on a chain the wallet does not support.

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<source_chain_id>","to_chain":"<dest_chain_id>","token_in":"<symbol>","token_out":"<symbol>","amount":"<decimal>","slippage":<fraction>,"swapper":"<address>","sign":false}'
```

The estimate response populates `from_amount_decimal`, `to_amount_decimal`, and `to_amount_min` (plus routing metadata like `from_chain`/`to_chain`). The signing-flow fields (`transaction_request`, `transaction_id`, `execution_duration`) are omitted, **but `approval_address` is ALWAYS returned** when the route requires an ERC20 approval — the spender you must approve BEFORE signing the swap.

Render the estimate to the user with the Jinja2 template below, then **ask in plain text** whether they want to proceed (and note any required token approval). Wait for their answer before calling `sign=true`.

**Token logos:** render a small logo (≤ 28px) next to each token using a public, symbol-keyed CDN. Use `{{ token_in }}`/`{{ token_out }}` lowercased as the filename, and guard every `<img>` with `{% if %}` so a missing logo never breaks the render:

- Preferred: `https://cryptocurrencyliveprices.com/img/{{ symbol | lower }}.png` (e.g. `bitcoin.png`, `ethereum.png`).
- Fallback: omit the image entirely if a logo is not known to exist — never point at a guessed or broken URL.

````jinja
## 🤝 {{ "Bridge" if is_cross_chain else "Swap" }} estimate

| | |
|---|---|
| You send | <img src="https://cryptocurrencyliveprices.com/img/{{ (token_in | lower) }}.png" width="24" height="24" /> `{{ from_amount_decimal }} {{ token_in }}` on {{ from_chain }} |
| You receive | ≈ <img src="https://cryptocurrencyliveprices.com/img/{{ (token_out | lower) }}.png" width="24" height="24" /> `{{ to_amount_decimal }} {{ token_out }}` on {{ to_chain }} (min `{{ to_amount_min }}`) |
| Slippage | {{ (slippage * 100) | round(2) }}% |
{% if approval_address %}
| Approval required | pre-approve `{{ token_in }}` for `{{ approval_address }}` |
{% endif %}

Reply **confirm** to execute, or **cancel** to stop.
````


### Approve the token if required

If the estimate response includes a non-null `approval_address`, you **must** approve it first, before the `sign=true` quote. Without this approval the cross-chain swap (or same-chain swap) will fail on-chain.

**Requires explicit user confirmation first.** Ask the user in plain text — show the token being approved and the spender (`approval_address`) — and do NOT call `approve_token` until the user confirms.

Only after confirmation, call `run_js` with:
- **function**: "approve_token"
- **data**: `{"token":"<from_token>","spender":"<approval_address>","blockchain":"<source_chain_name>"}`

- `token`: the `from_token` address from the quote (the ERC20 input token).
- `spender`: the `approval_address` from the quote.
- `blockchain`: the source chain **name** matching the quote (e.g. `ethereum`, `base`, `polygon`).
- `amount`: optional — omit to approve unlimited (`MAX_UINT256`).

`approve_token` builds, signs (the user approves via the transaction confirm modal), and broadcasts the `approve(spender, amount)` transaction. Wait for it to succeed before proceeding.


### Get a signable quote (`sign=true`)

**Requires explicit user confirmation first.** Before calling with `sign=true`, show the user the estimate in plain readable Markdown (asset pair, amount, `from_amount_decimal` → `to_amount_decimal`, slippage, chain) and ask for confirmation. Do NOT call `sign=true` until the user confirms.

Re-quote with `sign=true` **only after confirmation**. The response additionally returns `transaction_request` (`to`/`data`/`value`/`chainId`) plus `transaction_id` and `execution_duration`. The `transaction_request` is detected as a metatransaction and signed+broadcast via the agent wallet.

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<source_chain_id>","to_chain":"<dest_chain_id>","token_in":"<symbol>","token_out":"<symbol>","amount":"<decimal>","slippage":<fraction>,"swapper":"<address>","sign":true}'
```

Note the `transaction_id` and `tool` from the response so you can track status. The approval for the input token must already be in place (see the previous step).


#### Quote request fields

- `chain` (required): source **chain id as a string** (from `/blockswap/chains`), e.g. `"1"` for Ethereum, `"137"` for Polygon, `"8453"` for Base. A chain name is also accepted, but prefer the id.
- `to_chain` (optional): destination **chain id as a string**. **Omit (or set equal to `chain`) for a same-chain swap**; set to a different chain id to bridge.
- `token_in` (required): input token symbol (e.g. `USDC`) or address.
- `token_out` (required): output token symbol (e.g. `WETH`) or address.
- `amount` (required): input amount in decimal (e.g. `"100.5"`). Always refers to the input token (Li.FI is exact-input).
- `slippage` (optional): slippage tolerance as a **fraction** (0.005 = 0.5%). Omit for auto.
- `swapper` (required): wallet address from `get_assets`.
- `sign` (optional, default `false`): return only the estimate (`false`) or also the signing flow (`true`).


### Track transfer status

Poll status by tx hash to report progress to the user. Use the `transaction_id` returned by the quote as the `txHash`, and pass `tool`, `fromChain`, `toChain` from the quote as optional hints to speed up the lookup.

```bash
curl -s "https://402.blockvault.ai/api/v1/blockswap/status?txHash=<tx_hash>&bridge=<tool>&fromChain=<from_chain>&toChain=<to_chain>"
```

- `txHash` (required): the transaction hash (or the quote's `transaction_id`).
- `bridge` (optional): the quote's `tool` (bridge/DEX used).
- `fromChain` / `toChain` (optional): source/destination chain ids.

The response `status` is `NOT_FOUND | PENDING | DONE | FAILED`; when `DONE`, `substatus` refines it to `COMPLETED | PARTIAL | REFUNDED`.


### Present the result to the user

Reply to the user using well-structured Markdown. Show the transfer status and report the final `token_out` received. Never invent a tx hash or amount.

## Constraints

- Confirm the source and destination tokens/chains appear in `/blockswap/tokens` and `/blockswap/chains` before quoting.
- Confirm the source and destination chains are present in the `supported_blockchains` response. **Never swap or bridge across a blockchain the wallet does not hold.**
- Use `supported_blockchains` first to know the usable chains, then `get_assets` to resolve the wallet address and verify source-chain funds.
- **Always pass `chain` and `to_chain` as string chain ids** (from `/blockswap/chains`, quoted as JSON strings), never as unquoted numbers.
- Always quote with `sign=false` first to show the estimate; only re-quote with `sign=true` after the user explicitly confirms.
- If the `sign=false` estimate returns a non-null `approval_address`, ask the user for confirmation, then call `approve_token` and wait for it to succeed **before** the `sign=true` quote.
- Same-chain swap: omit `to_chain`. Cross-chain: set `to_chain` to the destination chain id.
- Report status via `GET /blockswap/status`.
- Default slippage: 0.5% (0.005 as a fraction). Max 5% (0.05) unless the user explicitly requests more.