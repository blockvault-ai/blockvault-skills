# BlockSwap — Endpoints

Base URL: `https://402.blockvault.ai`

## Contents
- Discover support
- Estimate
- Execute
- Track

## Discover support (only when unsure)

```bash
curl -s "https://402.blockvault.ai/api/v1/blockswap/chains"
curl -s "https://402.blockvault.ai/api/v1/blockswap/tokens?chains=<ids>"
```

Only quote across chains in BOTH BlockSwap's list AND `supported_blockchains`.

## Estimate (`sign=false` — moves nothing)

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<src_id>","to_chain":"<dst_id>","token_in":"<SYM>","token_out":"<SYM>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}'
```

Present the estimate (send → receive, min received, slippage) and ask the user to confirm. Note `approval_address` if present.

## Execute (`sign=true` — moves funds, confirm first)

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<src_id>","to_chain":"<dst_id>","token_in":"<SYM>","token_out":"<SYM>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":true}'
```

The response's `transaction_request` is auto-signed+broadcast.

## Track & notify

```bash
curl -s "https://402.blockvault.ai/api/v1/blockswap/status?txHash=<transaction_id>&bridge=<tool>&fromChain=<src>&toChain=<dst>"
```

Report final status (`NOT_FOUND | PENDING | DONE | FAILED`; `DONE` → `substatus` `COMPLETED | PARTIAL | REFUNDED`) and the `token_out` received. Never invent a tx hash or amount.