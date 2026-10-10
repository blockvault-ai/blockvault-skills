# BlockSwap — Endpoints

Base URL: `https://402.blockvault.ai/api/v1/blockswap`

## Contents
- Discover support
- Estimate
- Execute
- Track

## Discover supported chains and tokens

```bash
curl -s "https://402.blockvault.ai/api/v1/blockswap/chains"
curl -s "https://402.blockvault.ai/api/v1/blockswap/tokens?chains=<ids>"
```

`/chains` returns chain ids/keys/names. `/tokens` returns the Li.FI catalog — **always pass `?chains=<ids>`** (comma-separated chain ids from `/chains` ∩ `supported_blockchains`), never call it unfiltered (the full catalog is huge). Use the catalog to confirm the exact `token_in`/`token_out` symbols exist on the target chains.

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

The response's `transaction_request` is auto-signed+broadcast. Note `transaction_id` and `tool` for tracking.

## Track & notify

```bash
curl -s "https://402.blockvault.ai/api/v1/blockswap/status?txHash=<transaction_id>&bridge=<tool>&fromChain=<src>&toChain=<dst>"
```

Report final status (`NOT_FOUND | PENDING | DONE | FAILED`; `DONE` → `substatus` `COMPLETED | PARTIAL | REFUNDED`) and the `token_out` received. For a bridge, `sending`/`receiving` describe the two legs. Never invent a tx hash or amount.