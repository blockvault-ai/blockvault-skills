# Buy Tokenized Stock — RWA API (opendata)

Base URL: `https://402.blockvault.ai/api/v1/rwa`

## Contents
- List tickers (browse by category)
- Resolve a ticker
- Market status
- Current price

## List tickers

```bash
curl -s "https://402.blockvault.ai/api/v1/rwa/tickers"
```

Returns `{items: [{ticker, symbol, chainId, contractAddress, multiplier, assetType, category, chains}], total}`. One entry per underlying stock (deduplicated across chains, BSC Ondo preferred). `category` is `stock` | `etf` | `other`; `chains` lists every chain the ticker is listed on.

Filters (all optional, combinable):
- `?search=<TERM>` — case-insensitive substring on `ticker`/`symbol`.
- `?category=etf` — filter by category (`stock`, `etf`, `other`).
- `?chain_id=56` — only tickers listed on that chain.
- `?page=1&page_size=50` — pagination (default 50, max 200).

Use this to browse the catalog or answer "what stocks/ETFs can I buy".

## Resolve a ticker

```bash
curl -s "https://402.blockvault.ai/api/v1/rwa/stocks?ticker=<TICKER>"
```

Returns `{items: [{ticker, symbol, chainId, contractAddress, multiplier, name}], total}`. `chainId` is a string ("56" = BSC, "1" = Ethereum).

Pick the Ondo token: prefer `chainId == "56"` (BSC) and `symbol` ending in `on` (e.g. `AAPLon`). Record `symbol`, `chainId`, `contractAddress`.

## Search stocks

When the user names a company but not the exact ticker, search by substring:

```bash
curl -s "https://402.blockvault.ai/api/v1/rwa/stocks?search=<TERM>"
```

`search` is a case-insensitive substring match on `ticker` or `symbol` (e.g. `search=apple` matches `AAPL`/`AAPLon`). Combine with `chain_id` to narrow to BSC: `?search=<TERM>&chain_id=56`. Paginate with `page`/`page_size`.

## Market status

```bash
curl -s "https://402.blockvault.ai/api/v1/rwa/stocks/<TICKER>/market-status"
```

Returns `{ticker, openState, marketStatus, reasonCode, reasonMsg, nextOpenTime, nextCloseTime}`. `openState: false` / `marketStatus: "closed"` means the US market is closed (weekend/holiday) — the reference price is stale and the DEX price may carry a premium.

## Current price

```bash
curl -s "https://402.blockvault.ai/api/v1/rwa/stocks/<TICKER>/price"
```

Returns `{ticker, symbol, chainId, contractAddress, multiplier, tokenPrice, referencePrice, stockPrice, priceChange24h, holders, marketCap}`.

- `tokenPrice` = token price (USD).
- `referencePrice` = per-share price (already computed).
- `stockPrice` = underlying stock price (may be absent when the market is closed).

## Buy quote (shares ↔ USD)

```bash
curl -s "https://402.blockvault.ai/api/v1/rwa/stocks/<TICKER>/quote?usd=<AMOUNT>"
# or
curl -s "https://402.blockvault.ai/api/v1/rwa/stocks/<TICKER>/quote?shares=<N>"
```

Returns `{ticker, symbol, chainId, contractAddress, multiplier, tokenPrice, referencePrice, requestedUsd|requestedShares, tokens, totalUsd, totalShares}`. The server does the multiplier math — use `totalUsd` as the blockswap `amount` (USDC) and `contractAddress` as `token_out`.
