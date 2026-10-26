# Uniswap LP — Endpoints (Background)

Base URL: `https://402.blockvault.ai/api/v1/uniswap`

**CRITICAL — base URL.** The ONLY valid base URL is `https://402.blockvault.ai/api/v1/uniswap`. Never use any other domain or path (e.g. `api.blockvault.net`, `api.blockvault.ai`, or a URL without `/api/v1/`). Copy the exact URL from the curl examples below — do NOT invent or shorten it.

## Discover pools

One call discovers, scores (0-100) and ranks pools across Ethereum/Base/Polygon (cached in Redis).

```bash
curl -sS "https://402.blockvault.ai/api/v1/uniswap/pools/recommend?chainId=<CHAIN_ID>&limit=<N>" -o pools.json
```

`poolReferenceIdentifier` = pool address; `protocol` = V2/V3/V4; `apy` = 7-day fee APY; `risk` = Low/Medium/Higher; `score` 0-100. For V3/V4 pools, `tickBounds` is precomputed server-side — pass it to `prepare` as-is.

## Wallet positions (read-only)

```bash
curl -sS "https://402.blockvault.ai/api/v1/uniswap/lp/positions/<ADDRESS>"
```

Fields: `token0`/`token1` (contract addresses, NOT symbols), `fee`, `tick_lower`/`tick_upper`, `amount0`/`amount1`, `tokens_owed0`/`tokens_owed1` (pending fees), `earned0`/`earned1`, `current_tick`, `in_range`, `token_id` (NFT token ID for V3/V4, or LP token address for V2).

## Open a position (prepare — approvals + create in one call)

```bash
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/prepare" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"protocol":"<V2|V3|V4>","poolReferenceIdentifier":"<POOL_ADDRESS>","token0Address":"<A>","token1Address":"<B>","fee":<FEE>,"independentToken":{"tokenAddress":"<TOKEN_ADDRESS>","amount":"<DECIMAL>"},"simulateTransaction":true}'
```

For V3/V4, add `"rangePct":5` (±5% around current price) unless the pool already provides `tickBounds` — pass `tickBounds` as-is if present.

`prepare` returns a `transactions[]` envelope (approvals first, then the create) plus a `simulation` field. The `bash` tool auto-signs and broadcasts every transaction in order.

## Manage positions

- **Claim fees** (keep the position): `POST /lp/claim_fees` with `tokenId` = `token_id`.
- **Increase**: `POST /lp/increase` with `nftTokenId` = `token_id` and `independentToken`.
- **Decrease / exit**: `POST /lp/decrease` with `nftTokenId` = `token_id` and `liquidityPercentageToDecrease` (100 = full exit).

```bash
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/claim_fees" \
  -H "Content-Type: application/json" \
  -d '{"protocol":"V3","walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"tokenId":"<TOKEN_ID>","collectAsWeth":false,"simulateTransaction":true}'

curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/increase" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"protocol":"V3","token0Address":"<A>","token1Address":"<B>","nftTokenId":"<TOKEN_ID>","independentToken":{"tokenAddress":"<TOKEN_ADDRESS>","amount":"<DECIMAL>"},"simulateTransaction":true}'

curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/decrease" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"protocol":"V3","token0Address":"<A>","token1Address":"<B>","nftTokenId":"<TOKEN_ID>","liquidityPercentageToDecrease":<PERCENT>,"withdrawAsWeth":false,"simulateTransaction":true}'
```

All three return a `transactions[]` envelope that `bash` auto-signs and broadcasts. Verify `simulation` before relying on the result.
