---
name: blockswap
description: Swaps, bridges, and transfers tokens via BlockSwap (Li.FI aggregation). Use when the user asks to swap tokens (same-chain or cross-chain), bridge between chains, convert a token for a DeFi action (e.g. into LP gas or the other side of a pair), or top up native gas on a chain. Plans routes as a single aggregator quote and handles native-gas funding.
metadata:
  category: defi
  enabled: true
---


## Reference files

Discover the full reference for this skill in the `reference` folder. Each file is a self-contained reference for a specific topic. Load it with `run_js` → `read_skill_reference` (see below).

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"blockswap","file":"<name>"}`

To list all available files, omit `file`:

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"blockswap"}`

Determine which reference files to load based on the workflow step:
- is related to swapping/bridging → load `endpoints` + `fields`
- is related to cross-chain bridging → load `crosschain` + `routing` + `endpoints`
- is related to multi-hop routing → load `routing`
- do not have native gas on a chain → load `gas` (funding cases)

## Workflow

### Step 1: Read the wallet & and supported chains to swap/bridge on

Call `run_js` with:
- **function**: "supported_blockchains"
- **data**: `{}`

Call `run_js` with:
- **function**: "get_assets"
- **data**: `{ "hasBalance": true }`

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"blockswap","file":"endpoints"}`

Use the endpoint `/tokens?chains=<ids>` to confirm the `token_in`/`token_out` symbols exist on the source/destination chains. 
Stop if the user has no assets on the source chain or the destination chain is not in `supported_blockchains`.

Checklist:

- **Wallet address** — the `address` field of an asset on the target chain.
- **Native gas** — the `symbol` of the native token on each chain (ETH, MATIC, BNB, etc.) and its `balance`. 
- **Assets** — the `symbol`, `blockchain`, and `balance` of each asset with a non-zero balance. Use these to confirm the `token_in`/`token_out` symbols exist on the source/destination chains. Stop if the user has no assets on the source chain or the destination chain is not in `supported_blockchains`.

### Step 2: Verify gas.

Identify the native gas token on the source chain (and destination chain for a bridge) and check its balance. If the source chain has no native gas, fund it first:

Call `run_js` with:
  - **function**: "read_skill_reference"
  - **data**: `{"skill":"blockswap","file":"gas"}` 

Stop if no route exists to fund gas.

- **Gas present** → continue to Step 3.
- **Gas missing or zero** → load the gas reference and follow its funding cases BEFORE quoting:


**Do NOT quote until gas is present** — the quote will fail if the source chain has no gas, and a bridge will fail if the destination chain has no gas.

### Step 3: Quote
Load the endpoints reference if not already loaded.

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"blockswap","file":"endpoints"}`

When `token_in` is the native gas token, the swap/bridge itself consumes gas on the source chain. Never quote the full balance — leave a buffer for the transaction fee.

1. Call `run_js` with:
   - **function**: "estimate_fee"
   - **data**: `{"symbol":"<native>","blockchain":"<chain>"}`
2. Quote `amount` = full balance **minus** the fee buffer (and a small safety margin).

Quote the swap with `sign:false` and render the estimate as a card — do NOT read the JSON and do NOT write a Markdown list.

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<src_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}' \
  -o quote.json
```

Emit this exact artifact fence so the app renders the estimate (send → receive, min received, approval):

````markdown
  ```artifact
  {"artifact":"<artifact_path>","template":"blockswap:quote","context":{"token_in":"<SYMBOL>","token_out":"<SYMBOL>","from_chain":"<src>","to_chain":"<dst>"}}
  ```
````

Then ask the user to confirm before approving/executing. 
Stop if no route exists.


### Step 5: Approve (only if `approval_address` is non-null)

Ask the user to confirm, then call `run_js` with:
- **function**: "approve_token"
- **data**: `{"token":"<from_token>","spender":"<approval_address>","blockchain":"<src_chain_name>"}`

`token` is the `from_token` address from the quote; `spender` is `approval_address`; `blockchain` is the source chain name. 
Omit `amount` to approve unlimited. Wait for it to succeed before Step 6.

### Step 6: Execute (`sign=true` — moves funds, confirm first)

After the user confirms, re-quote with `sign:true` (see `reference/endpoints.md`). 
The response's `transaction_request` is auto-signed+broadcast. Note 

### Step 7: Track & notify (feedback loop)

`curl` the `/status` endpoint (see `reference/endpoints.md`). Report final status (`NOT_FOUND | PENDING | DONE | FAILED`; `DONE` → `substatus` `COMPLETED | PARTIAL | REFUNDED`) and the `token_out` received. Never invent a tx hash or amount.

##Cross Chain Bridge
Load the cross-chain reference if not already loaded.

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"blockswap","file":"crosschain"}`
