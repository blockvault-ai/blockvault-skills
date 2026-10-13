# BlockSwap — Endpoints

Base URL: `https://402.blockvault.ai/api/v1/blockswap`

## Contents
- Discover support
- Estimate
- Execute
- Track

## Discover supported chains and tokens

Call this endpoint to discover the supported chains. 
  ```bash
  curl -s "https://402.blockvault.ai/api/v1/blockswap/chains"
  ```
  `/chains` returns chain ids/keys/names.

Call this endpoint to discover the supported tokens on those chains:

  ```bash
    curl -s "https://402.blockvault.ai/api/v1/blockswap/tokens?chains=<ids>"
  ```
  - Always pass `?chains=<ids>` (comma-separated chain ids from `/chains` ∩ `supported_blockchains`)
  - `tokens` returns the list of available tokens on the requested chains.
  - the response contains `address`, `decimals`, `symbol`, `chainId`, and `logo_uri` for each token. Use `logo_uri` as the `image` when registering a token with `add_asset`.


## Estimate (`sign=false` — moves nothing)

Same-chain swap: omit `to_chain` (or set it equal to `chain`). Cross-chain bridge: set `to_chain` to a different chain id.

```bash
# same-chain swap
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<src_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}'

# cross-chain bridge
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<src_id>","to_chain":"<dst_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}'
```

Present the estimate (send → receive, min received, slippage) and ask the user to confirm. Note `approval_address` if present. For a bridge, warn the user it spends gas on BOTH chains.

## Execute (`sign=true` — moves funds, confirm first)

Re-quote with `sign:true` (same `chain`/`to_chain` as the estimate):

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<src_id>","to_chain":"<dst_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":true}'
```

The response's `transaction_request` is auto-signed+broadcast. 
Note `transaction_id` and `tool` for tracking. Do not use -o to save the JSON — it contains sensitive data.

## Track & notify

Poll until the status reaches a final state. Prefix each poll with `sleep` so the bridge has time to settle between checks:

```bash
sleep 15 && curl -s "https://402.blockvault.ai/api/v1/blockswap/status?txHash=<transaction_id>&bridge=<tool>&fromChain=<src>&toChain=<dst>"
```

Repeat while the status is `PENDING`. Report final status (`NOT_FOUND | PENDING | DONE | FAILED`; `DONE` → `substatus` `COMPLETED | PARTIAL | REFUNDED`) and the `token_out` received. For a bridge, `sending`/`receiving` describe the two legs. Never invent a tx hash or amount.