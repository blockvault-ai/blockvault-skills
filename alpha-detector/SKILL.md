---
name: alpha-detector
description: Scan the entire market for opportunities using rankings (trending, alpha, top search, social hype, smart money, meme). Use this when the user asks about market trends, what's hot, which tokens are trending, or wants to discover new opportunities — NOT for analyzing a specific coin or pair.
metadata:
  tool: get_historical_prices
  category: market
  enabled: true
---

## Instructions

Follow all steps in order. Do NOT skip any step. Do NOT output internal thoughts.
Detect the language of the user's query and write everything (report included) in that language.

### Step 1: Fetch ranked token list

Call `bash` with a curl to the BlockVault ranking API. Pick the endpoint based on user intent (default: trending):

| Intent | Endpoint |
|--------|----------|
| trending (default) | `/api/v1/ranking/trending` |
| alpha | `/api/v1/ranking/alpha` |
| most searched | `/api/v1/ranking/top-search` |
| social hype | `/api/v1/ranking/social-hype` |
| smart money / whales | `/api/v1/ranking/smart-money` |
| meme | `/api/v1/ranking/meme` |

```shell
curl -s 'https://api.blockvault.ai/api/v1/ranking/trending?limit=10'
```

Optional query params: `limit` (default 10), `chain_id` (`1`=ETH, `56`=BSC, `8453`=Base, `CT_501`=SOL), `period` (smart-money only: `5m`,`1h`,`4h`,`24h`).

Response: `{tokens: [{rank, symbol, price, price_change, market_cap, volume, holders, chain_id, contract_address, ...}], total}`. Extra fields vary: `liquidity`, `inflow`+`traders`, `social_hype`+`sentiment`, `score`.

Take top 5-10 tokens. Map each symbol → trading pair by appending USDT (e.g. BTC → BTCUSDT).

If the API fails, fall back to: BTC, ETH, SOL, BNB, XRP.

### Step 2: Fetch OHLCV price data

Call `run_js` once:
- **function**: `"get_historical_prices"`
- **data**: `{"symbols": ["BTCUSDT", ...], "interval": "1d", "limit": 7}`

Use `"1h"` interval only if user asks for intraday. Wait for result.

### Step 3: Web search for context

Use `web_search` (2-3 calls) to find crypto news related to the top movers from Steps 1-2. Start broad ("crypto market trends 2026"), then narrow to specific assets showing unusual action.

### Step 4: Analyze and rank

For each asset, cross-reference rank data + OHLCV + news:

- **Price action**: trend direction, % change, breakout above/below prior range, wick patterns
- **Volume**: increasing/decreasing trend, spikes vs average
- **Catalyst**: news sentiment (bullish/bearish/neutral), catalyst strength, narrative alignment

Rate each asset:
- **Strong Alpha**: 3+ confirming signals (e.g. breakout + volume spike + bullish news)
- **Moderate Alpha**: 2 confirming signals
- **Watchlist**: 1 interesting signal, insufficient confirmation
- **No Signal**: nothing notable

Rank by conviction, not just % change.

### Step 5: Save report

Call `text_editor` to save:

**path**: `"reports/alpha-scan-{{ current_date }}.md"`

The template below is **Jinja2** — substitute `{{ var }}` with concrete values, expand `{% for %}` over the assets/watchlist, and resolve `{% if %}` blocks. Drop `{# comments #}` from the final output.

```jinja
# Alpha Scan Report

**Date**: {{ current_date }} | **Source**: BlockVault {{ endpoint }} | **Chain**: {{ chain | default("All") }} | **Assets**: {{ asset_count }} | **Interval**: {{ interval }}

## Top Opportunities

{% for a in opportunities %}
### {{ loop.index }}. {{ a.symbol | upper }} — {{ a.rating }}  {# Strong Alpha / Moderate Alpha #}
- **Price**: ${{ a.price }} ({{ a.change_24h }}% 24h, {{ a.change_7d }}% 7d)
- **Market Cap**: ${{ a.market_cap }} | **Volume**: ${{ a.volume }} | **Holders**: {{ a.holders }} | **Chain**: {{ a.chain }}
- **Signal**: {{ a.signal }}
- **Catalyst**: {{ a.catalyst }}
- **Conviction**: {{ a.conviction }}  {# High / Medium / Low #}

{% endfor %}

## Watchlist

{% for w in watchlist %}
- **{{ w.symbol | upper }}**: {{ w.reason }}
{% endfor %}

## Market Sentiment

- **Overall**: {{ sentiment_overall }}  {# Bullish / Bearish / Neutral / Mixed #}
- **Narrative**: {{ dominant_narrative }}
- **Risk**: {{ risk_level }}  {# Low / Medium / High #}

## Summary Table

| Asset | Chain | Price | 24h% | 7d% | Volume | MCap | Sentiment | Rating |
|-------|-------|-------|------|-----|--------|------|-----------|--------|
{% for a in summary_rows %}
| {{ a.symbol }} | {{ a.chain }} | ${{ a.price }} | {{ a.change_24h }}% | {{ a.change_7d }}% | ${{ a.volume }} | ${{ a.market_cap }} | {{ a.sentiment }} | {{ a.rating }} |
{% endfor %}
```

### Step 6: Respond

Brief summary (3-4 sentences): strongest opportunity, overall sentiment, assets to watch. Do not paste the full report. Emit the saved report path in **backticks** (inline code) so the user can tap it to open the file (the app makes `reports/…` paths clickable):

```text
Report saved to `reports/alpha-scan-{{ current_date }}.md`
```
