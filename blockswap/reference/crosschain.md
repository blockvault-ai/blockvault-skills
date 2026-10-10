

## Bridge checklist

[ ] **verify native gas on BOTH chains** before quoting — the destination needs gas to claim the bridged funds.

[ ] **Confirm the destination chain is in `supported_blockchains`**

[ ] **Warn the user** a bridge is slower and spends gas twice.


## Cross-chain bridge flow

### 1. Estimate

Quote with `sign:false` and render the estimate as a card — do NOT read the JSON and do NOT write a Markdown list.

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<src_id>","to_chain":"<dst_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}' \
  -o quote-<src_id>-<dst_id>.json
```

Emit this exact artifact fence so the app renders the estimate (send → receive, min received, approval):

````markdown
  ```artifact
  {"artifact":"<artifact_path>","template":"blockswap:quote","context":{"token_in":"<SYMBOL>","token_out":"<SYMBOL>","from_chain":"<src>","to_chain":"<dst>"}}
  ```
````

Then ask the user to confirm before approving/executing.

### 2. Approve (if `approval_address` is non-null)

Ask the user to confirm, then call `run_js` with:
- **function**: "approve_token"
- **data**: `{"token":"<from_token>","spender":"<approval_address>","blockchain":"<src_chain_name>"}`

### 3. Execute
After the user confirms, Re-quote with `sign:true` (same `chain`/`to_chain`). The `transaction_request` is auto-signed+broadcast.

### 4. Track

```bash
curl -s "https://402.blockvault.ai/api/v1/blockswap/status?txHash=<transaction_id>&bridge=<tool>&fromChain=<src>&toChain=<dst>"
```

The response has two legs: `sending` (source chain) and `receiving` (destination chain). Report `DONE` only when the `receiving` leg confirms; `PENDING` means the bridge is still settling.

## Example — bridge USDC from Ethereum to Base

User holds USDC on Ethereum (with ETH gas) and wants it on Base (with ETH gas):

1. `chain: "1"`, `to_chain: "8453"`, `token_in: "USDC"`, `token_out: "USDC"`.
2. Estimate → present → confirm.
3. Approve if `approval_address` present.
4. Execute with `sign:true`.
5. Track until `receiving` leg is `DONE`.

## Example — bridge ETH from Base to Polygon

User holds ETH on Base (with ETH gas) and wants it on Polygon (with MATIC gas):

1. `chain: "8453"`, `to_chain: "137"`, `token_in: "ETH"`, `token_out: "ETH"`.
2. Estimate → present → confirm.
3. Approve if `approval_address` present.
4. Execute with `sign:true`.
5. Track until `receiving` leg is `DONE`.