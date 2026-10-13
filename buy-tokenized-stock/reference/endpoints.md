# Buy Tokenized Stock — BlockSwap endpoints

Base URL: `https://402.blockvault.ai/api/v1/blockswap`

## Contents
- Discover tokens
- Estimate
- Execute
- Track

## Discover tokens

```bash
curl -s "https://402.blockvault.ai/api/v1/blockswap/tokens?chains=56"
```

Confirm the funding stablecoin (USDC) exists on BSC. The stock token is passed by `contractAddress` (unambiguous), so this is mainly to confirm the stablecoin.

## Estimate (`sign=false` — moves nothing)

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"56","token_in":"USDC","token_out":"<contractAddress>","amount":"<usd>","slippage":0.005,"swapper":"<addr>","sign":false}' \
  -o quote.json
```

- `chain` = "56" (BSC). `token_in` = "USDC" (or the stablecoin the user holds). `token_out` = the stock token's `contractAddress`. `amount` = USD amount (decimal, exact-input). `swapper` = the wallet address.
- Present the estimate (send → receive, min received, approval) and ask the user to confirm.

Emit this artifact fence so the app renders the estimate:

````markdown
  ```artifact
  {"artifact":"<artifact_path>","template":"blockswap:quote","context":{"token_in":"USDC","token_out":"<SYMBOL>","from_chain":"56","to_chain":"56"}}
  ```
````

## Execute (`sign=true` — moves funds, confirm first)

Re-quote with `sign:true` (same fields):

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"56","token_in":"USDC","token_out":"<contractAddress>","amount":"<usd>","slippage":0.005,"swapper":"<addr>","sign":true}'
```

The response's `transaction_request` is auto-signed and broadcast. Note `transaction_id` and `tool` for tracking.

## Track

```bash
curl -s "https://402.blockvault.ai/api/v1/blockswap/status?txHash=<transaction_id>&bridge=<tool>&fromChain=56&toChain=56"
```

Report final status (`NOT_FOUND | PENDING | DONE | FAILED`; `DONE` → `substatus` `COMPLETED | PARTIAL | REFUNDED`) and the `token_out` received. Never invent a tx hash or amount.
