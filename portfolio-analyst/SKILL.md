---
name: portfolio-analyst
description: Comprehensive portfolio analysis with asset distribution, risk assessment, diversification score, and performance insights.
metadata:
  tool: get_assets
  category: wallet
  enabled: true
---

# Portfolio Analyst

Analyze the user's entire portfolio across all blockchains. Combine asset holdings, current prices, and historical data to generate a detailed portfolio report with distribution analysis, risk scoring, and actionable insights.

## Instructions

Execute all steps silently. Do NOT output internal thoughts. No exceptions. Do not omit any steps.

### Step 1: Get portfolio assets

Call `run_js` with:

- **function**: "get_assets"
- **data**: JSON string with:
  - **hasBalance**: Boolean, Required. Set to `true`.

This returns all assets with a positive balance across all blockchains.

### Step 2: Get prices for all assets

Call `run_js` with:

- **function**: "get_price"
- **data**: JSON string with:
  - **portfolio**: Boolean, Required. Set to `true`.

This returns current prices and USD values for all assets with balance.

### Step 3: Get historical data for top holdings

For each of the top 5 assets by USD value, fetch 7-day price history by calling `run_js` with:

- **function**: "get_historical_price"
- **data**: JSON string with:
  - **pair**: String, Required. Asset pair (e.g., "BTCUSDT").
  - **interval**: String, Required. "1d".
  - **limit**: Number, Required. 7.

Map asset symbols to trading pairs: BTC→BTCUSDT, ETH→ETHUSDT, SOL→SOLUSDT, BNB→BNBUSDT, XRP→XRPUSDT, POL→POLUSDT.

Skip assets that don't have a USDT trading pair (e.g., small tokens, NFTs). Only fetch history for assets that map to supported pairs.

### Step 4: Analyze the portfolio

Perform the following analysis using all collected data:

**A. Distribution Analysis:**

- Calculate each asset's % of total portfolio value.
- Group by category:
  - **Blue chips**: BTC, ETH
  - **Large caps**: SOL, BNB, XRP, POL
  - **Stablecoins**: USDT, USDC, DAI
  - **DeFi tokens**: LINK, ARB, OP, UNI
  - **Other**: Everything else
- Group by blockchain: Bitcoin, Ethereum, Polygon, BSC, Base, Arbitrum, Optimism, Solana, Counterparty, Cosmos.

**B. Concentration Risk:**

- **Top asset %**: What percentage does the #1 holding represent?
  - \>60% = High concentration risk
  - 30-60% = Moderate concentration
  - <30% = Well distributed
- **Top 3 concentration**: Combined % of top 3 holdings.
- **Stablecoin ratio**: % of portfolio in stablecoins.
  - \>50% = Very defensive
  - 20-50% = Balanced
  - <20% = Aggressive / Growth-oriented

**C. Diversification Score (1-100):**

Calculate based on:

- Asset count (more = higher score, diminishing returns past 10)
- Category spread (points for each category represented)
- Blockchain spread (points for each chain with holdings)
- Concentration penalty (deduct points if top asset > 50%)

Formula guidance:

- 5+ assets across 3+ categories and 3+ chains, no asset > 40% → score 70-100
- 3-5 assets across 2+ categories → score 40-69
- 1-2 assets or single chain → score 1-39

**D. Performance Insights (from historical data):**

For top holdings with price history:

- 7-day % change per asset.
- Which assets gained vs lost value this week.
- Best and worst performer.
- Estimated 7-day portfolio value change.

**E. Risk Assessment:**

Combine all signals into a risk profile:

- **Conservative**: High stablecoin ratio, blue chips dominant, low concentration.
- **Moderate**: Mix of blue chips and altcoins, some stablecoins.
- **Aggressive**: High altcoin exposure, low stablecoins, high concentration.

### Step 5: Save the portfolio report

Build a markdown report and save it using the `text_editor` tool directly. The template below is **Jinja2** — substitute `{{ var }}` with concrete values, expand `{% for %}` loops over the holdings/categories/insights, and resolve `{% if %}` blocks. Drop `{# comments #}` from the final output.

**file path**: `"reports/portfolio-analysis-{{ current_date }}.md"`

**Report template**:

```jinja
# Portfolio Analysis Report

**Date**: {{ current_date }}
**Total Value**: ${{ total_usd_value }}
**Assets**: {{ asset_count }} across {{ chain_count }} blockchains

## Holdings Overview

| # | Asset | Blockchain | Balance | Price | Value | % of Portfolio |
|---|-------|-----------|---------|-------|-------|----------------|
{% for h in holdings %}
| {{ loop.index }} | {{ h.symbol }} | {{ h.chain }} | {{ h.balance }} | ${{ h.price }} | ${{ h.value }} | {{ h.pct }}% |
{% endfor %}

## Distribution

### By Category

| Category | Value | % |
|----------|-------|---|
{% for c in categories %}  {# Blue Chips, Large Caps, Stablecoins, DeFi Tokens, Other #}
| {{ c.name }} | ${{ c.value }} | {{ c.pct }}% |
{% endfor %}

### By Blockchain

| Blockchain | Value | % |
|-----------|-------|---|
{% for b in blockchains %}
| {{ b.name }} | ${{ b.value }} | {{ b.pct }}% |
{% endfor %}

## Risk Profile

- **Profile**: {{ risk_profile }}  {# Conservative / Moderate / Aggressive #}
- **Diversification Score**: {{ diversification_score }}/100
- **Top Asset Concentration**: {{ top_asset_symbol }} at {{ top_asset_pct }}%
- **Stablecoin Ratio**: {{ stablecoin_ratio }}%

### Score Breakdown

- Asset diversity: {{ asset_diversity_count }} assets (+{{ asset_diversity_pts }} pts)
- Category spread: {{ category_count }} categories (+{{ category_pts }} pts)
- Chain spread: {{ chain_spread_count }} chains (+{{ chain_pts }} pts)
- Concentration penalty: {{ concentration_penalty }} pts

## 7-Day Performance

{% if performance %}
| Asset | 7d Change | Impact on Portfolio |
|-------|-----------|--------------------|
{% for p in performance %}
| {{ p.symbol }} | {{ p.change_pct }}% | ${{ p.impact_usd }} |
{% endfor %}

- **Best performer**: {{ best_performer.symbol }} ({{ best_performer.pct }}%)
- **Worst performer**: {{ worst_performer.symbol }} ({{ worst_performer.pct }}%)
- **Estimated 7d change**: ${{ portfolio_change_usd }} ({{ portfolio_change_pct }}%)
{% else %}
Historical data unavailable.
{% endif %}

## Insights & Suggestions

{% for insight in insights %}  {# 3-5 actionable insights #}
- {{ insight }}
{% endfor %}
```

### Step 6: Respond

Tell the user the portfolio analysis is complete, report was saved, and provide a brief verbal summary (3-4 sentences max):

- State total portfolio value.
- Mention the risk profile and diversification score.
- Highlight the most important insight or suggestion.

Emit the saved report path in **backticks** (inline code) so the user can tap it to open the file (the app makes `reports/…` paths clickable):

```text
Report saved to `reports/portfolio-analysis-{{ current_date }}.md`
```

Do not output the raw data or the full report content.

## Constraints

- Always fetch assets with `hasBalance: true` first — this is the foundation of the analysis.
- Always fetch prices with `portfolio: true` to get USD values.
- Historical data is optional — if it fails, still generate the report with distribution and risk analysis and note "Historical data unavailable" in the Performance section.
- Do not output raw data or the full report content. Analyze and present insights.
- Always save the report before responding.
- Relay exact balance and price numbers from tool responses. Do NOT invent, round, or modify any value.
- If the portfolio has only 1 asset, still generate a complete report with appropriate risk warnings about zero diversification.
- Categorization should be based on the asset lists provided. Unknown tokens go in "Other".
