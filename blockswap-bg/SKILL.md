---
name: blockswap-bg
description: Background version of blockswap. Swaps, bridges, and transfers tokens via BlockSwap (Li.FI aggregation) on the agent's ephemeral wallet — no user interaction, results delivered via notify_user. Use when a background skill needs to swap tokens (same-chain or cross-chain), bridge between chains, convert a token for a DeFi action (e.g. into LP gas or the other side of a pair), or top up native gas on a chain.
metadata:
  category: defi
  enabled: true
  background: true
---

**CRITICAL — base URL.** The ONLY valid base URL is `https://402.blockvault.ai/api/v1/blockswap`. Never use any other domain or path (e.g. `api.blockvault.net`, `api.blockvault.ai`, or a URL without `/api/v1/`). Copy the exact URL from the curl examples below — do NOT invent or shorten it.

## Reference files

Discover the full reference for this skill in the `reference` folder. Each file is a self-contained reference for a specific topic. Load it with `read_skill_reference` (see below).

Call `read_skill_reference` with:
- **skill**: String, Required. `"blockswap-bg"`.
- **file**: String, Required. `"<name>"`.

To list all available files, omit `file`:

Call `read_skill_reference` with:
- **skill**: String, Required. `"blockswap-bg"`.

Determine which reference files to load based on the workflow step:
- is related to swapping/bridging → load `endpoints`
- is related to cross-chain bridging → load `crosschain` + `routing` + `endpoints`
- is related to multi-hop routing → load `routing`
- do not have native gas on a chain → load `gas` (funding cases)

## Workflow

### Step 1: Read the wallet & supported chains to swap/bridge on

Call `get_balances` with:
- **chain_id**: Number, Optional. Omit (or `0`) to read ALL chains.

Call `read_skill_reference` with:
- **skill**: String, Required. `"blockswap-bg"`.
- **file**: String, Required. `"endpoints"`.

Use the endpoint `/tokens?chains=<ids>` to confirm the `token_in`/`token_out` symbols exist on the source/destination chains.
Stop if the wallet has no assets on the source chain or the destination chain is not supported (supported chains: 137=Polygon, 8453=Base, 1=Ethereum, 42161=Arbitrum, 10=Optimism).

Checklist:

- **Wallet address**:
    - The top-level `address` field from `get_balances`.
- **Native gas**:
    - The entry in `balances` whose `contractAddress` is `null`. Its `symbol` is the native token and its `balance` is the gas balance.
    - Its `hasGasForTx` says whether the chain can pay a transaction (computed from the live gas price). Trust it; never estimate fees yourself.
- **Assets**:
    - The `symbol`, `chainId`, and `balance` of each entry with a non-zero balance.
    - Use these to confirm the `token_in`/`token_out` symbols exist on the source/destination chains.
    - Stop if the wallet has no assets on the source chain or the destination chain is not supported.
- **Chain id** — the `chainId` of the source and destination chains, from `get_balances`.

### Step 2: Verify gas.

Identify the native gas token on the source chain (and destination chain for a bridge) and check its balance. If the source chain has no native gas, fund it first:

Call `read_skill_reference` with:
- **skill**: String, Required. `"blockswap-bg"`.
- **file**: String, Required. `"gas"`.

If no chain can fund gas (Case 3 of the gas reference), call `end_task` with status `"blocked"` and stop.

- **`hasGasForTx: true`** → gas is present, continue to Step 3 (even if the balance looks small).
- **`hasGasForTx: false` or entry missing** → load the gas reference and follow its funding cases BEFORE quoting.

**Do NOT quote until gas is present** — the quote will fail if the source chain has no gas, and a bridge will fail if the destination chain has no gas.

### Step 3: Quote
Load the endpoints reference if not already loaded.

Call `read_skill_reference` with:
- **skill**: String, Required. `"blockswap-bg"`.
- **file**: String, Required. `"endpoints"`.

When `token_in` is the native gas token, the swap/bridge itself consumes gas on the source chain. Never quote the full balance — leave a buffer for the transaction fee.

`amount` is in **decimal units** of `token_in` (e.g. `"0.5"` for 0.5 USDC), never base units, and never more than the `token_in` balance from `get_balances`.

Quote the swap with `sign:false`:

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<src_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}'
```

If no route exists, call `end_task` with status `"failed"`.

**Never re-quote a failed quote with other amounts.** A failure with `errorCode` `INSUFFICIENT_GAS` (`insufficient funds for intrinsic transaction cost`), `INSUFFICIENT_TOKEN` (`TRANSFER_FROM_FAILED`, `STF`) or `REVERTED` will fail again at any amount: call `end_task` with status `"blocked"` (gas or funds) or `"failed"`.

### Step 5: Approve (only if `approval_address` is non-null)

Call `approve_token` with:
- **token_address**: String, Required. The `from_token` address from the quote.
- **spender**: String, Required. The `approval_address` from the quote.
- **chain_id**: Number, Required. The source chain id.
- **amount**: String, Required. The exact amount in base units: the quote's `from_amount`. Unlimited approvals are rejected.

Wait for it to succeed before Step 6.

### Step 6: Execute (`sign=true` — moves funds)

Re-quote with `sign:true` (see `reference/endpoints.md`).
The response's `transaction_request` is auto-signed+broadcast.

### Step 7: Track & notify (feedback loop)

**Same-chain swap:** confirm it with `check_transaction` (`chain_id` = source chain, `tx_hash` = the broadcast `hash`, `wait_seconds` = `60`). Do NOT use `/status` for same-chain swaps — it answers 404.

**Cross-chain bridge only:** confirm the source transaction with `check_transaction` first, then poll the `/status` endpoint (see `reference/endpoints.md`) until it reaches a final state. Prefix each poll with `sleep` so the bridge has time to settle between checks:

```bash
sleep 15 && curl -s "https://402.blockvault.ai/api/v1/blockswap/status?txHash=<transaction_id>&bridge=<tool>&fromChain=<src>&toChain=<dst>"
```

Repeat while the status is `PENDING`, then report the final status (`NOT_FOUND | PENDING | DONE | FAILED`; `DONE` → `substatus` `COMPLETED | PARTIAL | REFUNDED`) and the `token_out` received. Never invent a tx hash or amount.

Call `notify_user` with:
- **title**: String, Required. E.g. `"Swap complete"` (max 50 chars).
- **message**: String, Required. Plain text summary (max 1024 chars). No markdown.
- **level**: String, Optional. `"info"` (default), `"warning"`, or `"critical"`.

## Cross Chain Bridge
Load the cross-chain reference if not already loaded.

Call `read_skill_reference` with:
- **skill**: String, Required. `"blockswap-bg"`.
- **file**: String, Required. `"crosschain"`.
