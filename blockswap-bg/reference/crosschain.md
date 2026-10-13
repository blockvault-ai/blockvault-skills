## Bridge checklist

[ ] **verify native gas on BOTH chains** before quoting — the destination needs gas to claim the bridged funds.

[ ] **Confirm the destination chain is supported** (137=Polygon, 8453=Base, 1=Ethereum, 42161=Arbitrum, 10=Optimism)

[ ] **Warn the user** a bridge is slower and spends gas twice.

## Cross-chain bridge flow

### 1. Estimate

Quote with `sign:false`:

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<src_id>","to_chain":"<dst_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}'
```

### 2. Approve (if `approval_address` is non-null)

Call `approve_token` with:
- **token_address**: String, Required. The `from_token` address from the quote.
- **spender**: String, Required. The `approval_address` from the quote.
- **chain_id**: Number, Required. The source chain id.
- **amount**: String, Optional. Omit to approve unlimited.

### 3. Execute
Re-quote with `sign:true` (same `chain`/`to_chain`). The `transaction_request` is auto-signed+broadcast.

### 4. Track

Poll until the status reaches a final state. Prefix each poll with `sleep` so the bridge has time to settle between checks:

```bash
sleep 15 && curl -s "https://402.blockvault.ai/api/v1/blockswap/status?txHash=<transaction_id>&bridge=<tool>&fromChain=<src>&toChain=<dst>"
```

Repeat while the status is `PENDING`. The response has two legs: `sending` (source chain) and `receiving` (destination chain). Report `DONE` only when the `receiving` leg confirms; `PENDING` means the bridge is still settling.

## Example — bridge USDC from Ethereum to Base

Wallet holds USDC on Ethereum (with ETH gas) and wants it on Base (with ETH gas):

1. `chain: "1"`, `to_chain: "8453"`, `token_in: "USDC"`, `token_out: "USDC"`.
2. Estimate.
3. Approve if `approval_address` present.
4. Execute with `sign:true`.
5. Track until `receiving` leg is `DONE`.

## Example — bridge ETH from Base to Polygon

Wallet holds ETH on Base (with ETH gas) and wants it on Polygon (with POL gas):

1. `chain: "8453"`, `to_chain: "137"`, `token_in: "ETH"`, `token_out: "ETH"`.
2. Estimate.
3. Approve if `approval_address` present.
4. Execute with `sign:true`.
5. Track until `receiving` leg is `DONE`.
