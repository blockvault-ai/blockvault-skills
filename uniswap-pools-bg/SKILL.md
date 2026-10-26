---
name: uniswap-pools-bg
description: Background version of uniswap-pools. Autonomously discovers and ranks the best Uniswap liquidity pools, presents the top pools to the user via notify_user, opens V2/V3/V4 LP positions, and manages existing positions (increase/decrease/claim/exit) on the agent's ephemeral wallet — no user interaction. Use when the user wants the agent to invest in or manage Uniswap LP positions on its own.
metadata:
  category: defi
  enabled: true
  background: true
---

# Uniswap Liquidity Pools (Background)

Discover and rank pools, then open and manage V2/V3/V4 LP positions autonomously on the agent's ephemeral wallet. 
All endpoints are public and free at `https://402.blockvault.ai/api/v1/uniswap`. Transactions returned by mutating endpoints are auto-signed and broadcast by the `bash` tool — do NOT sign them manually.

**CRITICAL — base URL.** The ONLY valid base URL is `https://402.blockvault.ai/api/v1/uniswap`. Never use any other domain or path (e.g. `api.blockvault.net`, `api.blockvault.ai`, or a URL without `/api/v1/`). Copy the exact URL from the curl examples below — do NOT invent or shorten it.

## Reference files

Load reference files on demand with `read_skill_reference`:

Call `read_skill_reference` with:

- **skill**: String, Required. `"uniswap-pools-bg"`.
- **file**: String, Optional. The reference basename. Omit to list available files.

- Discover pools / open / manage positions → load `endpoints`.
- Extract top pools with JMESPath → load `pools`.

### Step 1: Read the agent wallet

Call `get_balances` with:

- **chain_id**: Number, Optional. Omit (or `0`) to read ALL chains.

Note from the result:
- **Wallet address** — the top-level `address` field.
- **Chains** — the `chainId` of each chain with a non-zero balance (supported: 137=Polygon, 8453=Base, 1=Ethereum, 42161=Arbitrum, 10=Optimism).
- **Native gas** — the entry in `balances` whose `contractAddress` is `null`. Its `symbol` is the native token (ETH/POL/etc.) and `balance` is the gas balance. Its `hasGasForTx` (computed from the live gas price, with `estimatedTxFee` and `affordableTxs`) is the ONLY test for native gas: `true` means the chain can pay LP transactions, even if the balance looks small. Never estimate fees yourself.
- **Tokens** — every other entry: `symbol`, `contractAddress`, `decimals`, and `balance` of each ERC-20.

Gas check:
- **No supported chain has `hasGasForTx: true`** → nothing can be signed and gas can only come from outside. Do not load `blockswap-bg`, do not notify: call `end_task` with status `"blocked"`, reason `"no native gas on any supported chain"`, and a summary listing what the user must fund. Stop.
- **The chain you need lacks gas but another chain has it** → fund it with `blockswap-bg` (gas reference, Case 2) before any transaction.

**Check open positions.** Call `bash` with:

- **command**: String, Required. A curl to list the wallet's open LP positions.

```bash
curl -sS "https://402.blockvault.ai/api/v1/uniswap/lp/positions/<ADDRESS>"
```

Use `<ADDRESS>` = the wallet address from `get_balances`. Each position has `token0`/`token1` (contract addresses, NOT symbols), `fee`, `amount0`/`amount1`, `tokens_owed0`/`tokens_owed1` (pending fees), `earned0`/`earned1`, `current_tick`, `in_range`, and `token_id` (NFT token ID for V3/V4, or LP token address for V2).

- **No open positions** → continue to Step 2 (discover pools and open a new one).
- **Open positions exist** → manage the risk based on the invested amount, decide if open another position increase, decrease or exit the open positions.

### Step 2: Discover pools

Load the `endpoints` and `pools` references first.

Call `bash` with:

- **command**: String, Required. A curl to the pool recommendation API, writing the result to an artifact.

```bash
curl -sS "https://402.blockvault.ai/api/v1/uniswap/pools/recommend?chainId=<CHAIN_ID>&limit=50" -o pools.json
```

Use `chainId` = the chain where the wallet holds the most value. The response is written to `artifacts/pools.json` (the `bash` tool returns `artifact_path`).

### Step 3: Present the top pools to the user

Extract the top 5 pools with JMESPath. Call `text_editor` with:

- **command**: String, Required. `"query"`.
- **path**: String, Required. `"artifacts/pools.json"`.
- **expression**: String, Required. `"pools[:5].[token0, token1, apy, score, risk]"`.

Then call `notify_user` with:

- **title**: String, Required. E.g. `"Top Uniswap pools"` (max 50 chars).
- **message**: String, Required. Plain text list of the top pools (token0/token1, APY, score, risk). No markdown.
- **level**: String, Optional. `"info"`.

### Step 4: Pick the best pool

Extract the full field set of the top-ranked pool. Call `text_editor` with:

- **command**: String, Required. `"query"`.
- **path**: String, Required. `"artifacts/pools.json"`.
- **expression**: String, Required. `"pools[0].[token0, token1, token0Address, token1Address, token0Decimals, token1Decimals, protocol, fee, poolReferenceIdentifier, tickBounds, apy, risk, score]"`.

Pick the best pool autonomously: highest `score`, then highest `apy`, preferring `risk` = `"Low risk"`/`"Medium risk"` (the field values are `"Low risk"`, `"Medium risk"`, `"Higher risk"`). Do NOT ask the user.

### Step 5: Fund both sides

**5a. Resolve the pool.** From the chosen pool object, note `token0Address`, `token1Address`, `token0Decimals`, `token1Decimals`, `poolReferenceIdentifier`, `fee`, and `protocol`.

**5b. Decide the amounts.** Decide autonomously how much to invest in USD (do NOT ask the user). The `independentToken.amount` is in decimal units of that token (not USD) — for a stablecoin side, "X USD" = `"X"`. The API computes the other side's amount from the pool ratio; it does NOT auto-swap.

**5c. Fund what's missing.** Check which of the two tokens the wallet holds:

- **Both tokens present** → done, continue to Step 6.
- **One token missing** → swap half of the *investment amount* into the missing token with `blockswap-bg`, then continue to Step 6.
- **Neither token present** → fund both sides with `blockswap-bg`, then continue to Step 6.
- **Not enough native gas** (`hasGasForTx: false` on the pool's chain) → fund gas first with `blockswap-bg`.

To fund or swap, load the `blockswap-bg` skill first:

Call `load_skill` with:

- **skill_names**: String, Required. `"blockswap-bg"`.

Then follow its instructions to swap/bridge the missing side or top up native gas. Do not proceed to Step 6 until both sides are funded and the wallet has enough gas.

### Step 6: Open the position

Call `bash` with a single `prepare` call (checks approvals AND builds the create transaction):

```bash
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/prepare" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"protocol":"<V2|V3|V4>","poolReferenceIdentifier":"<POOL_ADDRESS>","token0Address":"<A>","token1Address":"<B>","fee":<FEE>,"independentToken":{"tokenAddress":"<TOKEN_ADDRESS>","amount":"<DECIMAL>"},"simulateTransaction":true}'
```

- `protocol` = the pool's `protocol`. For V3/V4, add `"rangePct":5` (±5% around current price) unless the pool already provides `tickBounds` — pass `tickBounds` as-is if present.
- `independentToken.amount` is in decimal units of that token (not USD). For a stablecoin side, "X USD" = "X".
- The response returns a `transactions[]` envelope (approvals first, then the create) plus a `simulation` field. The `bash` tool auto-signs and broadcasts every transaction in order.

Before relying on the result, check `simulation`:
- `simulation.success == false` → do NOT sign; call `end_task` with status `"failed"` and `simulation.error` as the reason.
- `transactions[]` empty → no calldata to sign; call `end_task` with status `"failed"`.

### Step 7: Manage existing positions

To review current positions, call `bash` with:

```bash
curl -sS "https://402.blockvault.ai/api/v1/uniswap/lp/positions/<ADDRESS>"
```

Each position has `token0`/`token1` (contract addresses, NOT symbols), `fee`, `amount0`/`amount1`, `tokens_owed0`/`tokens_owed1` (pending fees), `earned0`/`earned1`, `current_tick`, `in_range`, and `token_id` (NFT token ID for V3/V4, or LP token address for V2).

- **Claim fees** (keep the position): `POST /lp/claim_fees` with `tokenId` = `token_id`. Claim only when `tokens_owed0`/`tokens_owed1` are worth clearly more than the fee (at least 3× the native entry's `estimatedTxFee`); skip dust. Claim each position at most once per run.
- **Increase**: `POST /lp/increase` with `nftTokenId` = `token_id` and `independentToken`.
- **Decrease / exit**: `POST /lp/decrease` with `nftTokenId` = `token_id` and `liquidityPercentageToDecrease` (100 = full exit).

All three return a `transactions[]` envelope that `bash` auto-signs and broadcasts. Verify `simulation` before relying on the result.

If any of them fails with `errorCode` `INSUFFICIENT_GAS`, stop all transactions on that chain: the remaining positions would fail the same way. Do not swap for gas on that chain (it needs gas too).

### Step 7b: Confirm transactions

The `bash` result lists every broadcast transaction with its `status` (`confirmed`, `reverted`, `unconfirmed`) and `hash`. For any `unconfirmed` entry, call `check_transaction` with:

- **chain_id**: Number, Required. The transaction's `chainId`.
- **tx_hash**: String, Required. The transaction's `hash`.
- **wait_seconds**: Number, Optional. `60`.

Never verify a Uniswap transaction with the BlockSwap `/status` endpoint or `sleep && curl` — it only knows bridges and answers 404. Never resend a transaction that is `unconfirmed` or `pending`.

### Step 8: Notify the user

Call `notify_user` with:

- **title**: String, Required. E.g. `"LP position opened"` (max 50 chars).
- **message**: String, Required. Plain text summary (max 1024 chars). No markdown.
- **level**: String, Optional. `"info"` (default), `"warning"`, or `"critical"`.

Report the pool (`token0`/`token1`), protocol, amount invested, and the outcome (opened / increased / decreased / claimed / exited). Skip this notification when nothing changed since the last run (see Recent Runs).

### Step 9: End the run

Call `end_task` with:

- **status**: String, Required. `"done"` (actions completed), `"no_action"` (nothing to do), `"blocked"` (missing gas or funds) or `"failed"` (non-retryable error).
- **summary**: String, Required. Pool, protocol, amounts, positions touched and outcome.
- **reason**: String, Optional. Required for `blocked`/`failed`: the error that stopped you.

When a step fails with a non-retryable error, skip Step 8 and end here with `blocked` or `failed`.

## Constraints

- Only use the BlockVault Uniswap and BlockSwap APIs via `bash` — never invent data.
- Never sign transactions manually — the `bash` tool auto-signs any `transactions[]` envelope or `transaction_request` in a response.
- Always verify `simulation.success` before treating a position as opened.
- Do NOT ask the user for confirmation — this is the background version; act autonomously and report via `notify_user`.
- Do NOT save reports to files (except the `-o` artifact from `bash`, which is expected).
