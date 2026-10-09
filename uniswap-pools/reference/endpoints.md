# Uniswap LP — Endpoints

Base URL: `https://402.blockvault.ai` · Prefix: `/api/v1/uniswap`

## Contents
- Discover pools
- Wallet positions
- Check approval
- Create V3/V4
- Create V2 (classic)
- Increase / decrease / claim

## Discover pools

One call discovers, scores (0-100) and ranks pools across Ethereum/Base/Polygon (cached in Redis).

```bash
curl -sS "https://402.blockvault.ai/api/v1/uniswap/pools/recommend?chainId=<CHAIN_ID>&limit=<N>" -o pools.json
```

`poolReferenceIdentifier` = pool address; `protocol` = V2/V3/V4; `apy` = 7-day fee APY; `risk` = Low/Medium/Higher; `score` 0-100.

## Wallet positions (read-only)

```bash
curl -sS "https://402.blockvault.ai/api/v1/uniswap/lp/positions/<ADDRESS>"
```

Fields: `token0/token1`, `fee`, `tick_lower/upper`, `amount0/amount1`, `tokens_owed0/1` (pending fees), `earned0/1`, `current_tick`, `in_range`. Resolve `token_id` from here.

## Check approval

**Create/increase** — approve BOTH tokens of the pair (with amounts):

```bash
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/check_approval" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","protocol":"V3","chainId":<CHAIN_ID>,"lpTokens":[{"tokenAddress":"<TOKEN0_ADDRESS>","amount":"<DECIMAL>"},{"tokenAddress":"<TOKEN1_ADDRESS>","amount":"<DECIMAL>"}],"action":"create"}'
```

**Decrease** — approve the NFT instead (no `lpTokens`):

```bash
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/check_approval" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","protocol":"V3","chainId":<CHAIN_ID>,"nftTokenId":"<TOKEN_ID>","action":"decrease"}'
```

## Create V3/V4 (tickBounds, NOT priceBounds)

```bash
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/create" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"protocol":"V3","independentToken":{"tokenAddress":"<TOKEN_ADDRESS>","amount":"<DECIMAL>"},"existingPool":{"token0Address":"<A>","token1Address":"<B>","poolReference":"<POOL_ADDRESS>"},"tickBounds":{"tickLower":<T>,"tickUpper":<T>},"urgency":"NORMAL","simulateTransaction":true}'
```

## Create full-range V2

```bash
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/create_classic" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","poolParameters":{"token0Address":"<A>","token1Address":"<B>","chainId":<CHAIN_ID>},"independentToken":{"tokenAddress":"<TOKEN_ADDRESS>","amount":"<DECIMAL>"},"includeApprovalSimulation":true,"simulateTransaction":true}'
```

## Increase / decrease / claim

```bash
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/increase" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"protocol":"V3","token0Address":"<A>","token1Address":"<B>","nftTokenId":"<TOKEN_ID>","independentToken":{"tokenAddress":"<TOKEN_ADDRESS>","amount":"<DECIMAL>"},"simulateTransaction":true}'
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/decrease" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"protocol":"V3","token0Address":"<A>","token1Address":"<B>","nftTokenId":"<TOKEN_ID>","liquidityPercentageToDecrease":<PERCENT>,"withdrawAsWeth":false,"simulateTransaction":true}'
curl -sS -X POST "https://402.blockvault.ai/api/v1/uniswap/lp/claim_fees" \
  -H "Content-Type: application/json" \
  -d '{"protocol":"V3","walletAddress":"<ADDRESS>","chainId":<CHAIN_ID>,"tokenId":"<TOKEN_ID>","collectAsWeth":false,"simulateTransaction":true}'
```