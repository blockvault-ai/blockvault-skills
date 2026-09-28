---
name: pair-market-analyst
description: Deep-dive analysis of a specific trading pair (e.g., BTC, ETH, SOL). Use this when the user asks to analyze a specific coin, token, or pair — NOT for general market scanning or discovering trending tokens.
metadata:
  tool: get_historical_price
  category: market
  enabled: true
---

# Crypto Market Analyst

Perform a comprehensive, multi-layered analysis of a **specific trading pair** by combining historical OHLCV price data with chained web research. Generate a rich markdown report with interactive ECharts candlestick visualization showing support/resistance levels.

This skill is for single-asset deep dives. For market-wide scanning and rankings, use the `alpha-detector` skill instead.

## Instructions

Execute all steps silently. Do NOT output internal thoughts. No exceptions. Do not omit any steps.

### Step 1: Determine parameters

Map the asset to a trading pair by appending USDT: BTC→BTCUSDT, ETH→ETHUSDT, SOL→SOLUSDT, BNB→BNBUSDT, XRP→XRPUSDT, DOGE→DOGEUSDT, ADA→ADAUSDT, AVAX→AVAXUSDT, LINK→LINKUSDT, DOT→DOTUSDT.

Choose interval and limit based on the user's timeframe. **Unless the user explicitly asks for a different period, ALWAYS use the default: interval "1d", limit 7.**

- **DEFAULT** (no timeframe mentioned, "analyze BTC", "last 2 weeks") → interval: "1d", limit: 14
- "today" → interval: "1h", limit: 24
- "last 2 weeks" → interval: "1d", limit: 14
- "last month" → interval: "1d", limit: 30

### Step 2: Fetch price data

Call `run_js` with:

- **function**: "get_historical_price"
- **data**: JSON string with:
  - **pair**: String, Required. Trading pair (e.g., "BTCUSDT").
  - **interval**: String, Required. Candle interval: "1h", "4h", "1d", "1w".
  - **limit**: Number, Optional. Number of candles (default: 7, max: 50).

Store the returned OHLCV candles for chart generation and analysis. The tool returns JSON with `{ symbol, interval, from, to, candles: [{d, o, h, l, c, v}] }`. Use `d` as the xAxis label, and map each candle to echarts format `[o, c, l, h]`.

### Step 3: Chained web research

Call the `web_search` tool repeatedly in a chained loop. Start with a broad query about the asset's recent news. Read the results, identify what you learned and what gaps remain, then formulate the next query to fill those gaps.

Call `web_search` with:

- **query**: String, Required. The search query.
- **backend**: String, Optional. Use `"news"` for current events and market news.
- **limit**: Number, Optional. Max results per search (default: 5, max: 10).

Keep searching until you can confidently cover: recent events, market sentiment, technical outlook, macro/regulatory context, institutional signals, and key catalysts/risks. Minimum 5 searches, maximum 8. Never repeat a query. Stop only when you have enough context for a comprehensive analysis.

### Step 4: Deep analysis

Combine price data and all research findings for a multi-dimensional analysis:

**Technical Analysis (from OHLCV data):**

- **Trend direction & strength**: Identify using candle body sizes, wick lengths, and consecutive direction
- **Key levels** (identify ALL that apply from the OHLCV data):
  - **S1 (primary support)**: Strongest bounce — the low where price reversed upward most decisively (long lower wicks, high volume)
  - **S2 (secondary support)**: Next lower level where lows cluster or wicks repeatedly touch
  - **S3 (breakdown level)**: Period low or next psychological round number below S2
  - **R1 (primary resistance)**: Strongest rejection — the high where price reversed downward (long upper wicks, selling pressure)
  - **R2 (secondary resistance)**: Next higher level with repeated highs or wick rejections
  - **R3 (breakout target)**: Period high or next psychological round number above R2
  - **Pivot**: The midpoint where price spent the most time consolidating (acts as both S and R)
- **How to find levels**: Scan all candle lows/highs. Group nearby values (within 1% of each other) into clusters. The cluster with the most touches is the strongest level. Round numbers ($70,000, $75,000) that align with clusters get higher confidence.
- **Volatility**: Average candle range as % of price; compare early vs late period volatility
- **Volume profile**: If volume data available, note distribution anomalies
- **Candle patterns**: Identify notable patterns (doji, engulfing, hammer, shooting star)
- **Key price zones**: Consolidation areas, breakout levels, gap fills

**Fundamental Analysis (from web research):**

- **Sentiment score**: Aggregate sentiment across all sources (Strong Bullish / Bullish / Neutral / Bearish / Strong Bearish)
- **Narrative clusters**: Group news into themes (regulatory, adoption, technology, competition)
- **Catalyst timeline**: Upcoming events with dates that could move price
- **Risk factors**: Identified threats (regulatory, technical, competitive)
- **Institutional flow signals**: Any whale/institutional activity mentioned
- **On-chain metrics**: If found in research (TVL changes, active addresses, exchange flows)

**Sentiment Gauge (for the gauge chart):**

Based on ALL your analysis above (technical trend, news sentiment, institutional signals, risk factors), assign a single `sentiment_score` from 0 to 100 representing the overall market sentiment for this asset right now. Also assign a `sentiment_label` (e.g. "Extreme Fear", "Fear", "Neutral", "Greed", "Extreme Greed"). This is YOUR conclusion — not a fixed formula. A strong downtrend with negative news might be 15 (Extreme Fear), while bullish breakout with positive catalysts might be 82 (Extreme Greed).

### Step 5: Respond to the user

Present the full analysis directly in your response using the template structure below. The template is **Jinja2** — substitute `{{ var }}` with concrete values, expand `{% for %}` over items, and resolve `{% if %}` blocks. Drop `{# comments #}` from the final output.

Output the complete analysis as your message (markdown rendered in chat):

````jinja
# {{ pair | upper }} Market Report

**Period**: {{ period_from }} to {{ period_to }} ({{ interval }})
**Generated**: {{ current_date }}

## Price Chart

```echarts
{
  "backgroundColor": "transparent",
  "animation": true,
  "tooltip": {
    "trigger": "axis",
    "axisPointer": { "type": "cross" }
  },
  "grid": { "left": "10%", "right": "10%", "bottom": "2%", "top": "2%" },
  "xAxis": {
    "type": "category",
    "data": {{ candle_labels_json }},
    "axisLabel": { "color": "#999", "fontSize": 9, "rotate": 45 },
    "axisLine": { "lineStyle": { "color": "#333" } }
  },
  "yAxis": {
    "type": "value",
    "scale": true,
    "axisLabel": { "color": "#999", "fontSize": 10 },
    "splitLine": { "lineStyle": { "color": "#222" } }
  },
  "series": [
    {
      "type": "candlestick",
      "data": {{ candle_ohlc_json }},
      "itemStyle": {
        "color": "#00ff94",
        "color0": "#ff4d4d",
        "borderColor": "#00ff94",
        "borderColor0": "#ff4d4d"
      },
      "markLine": {
        "silent": true,
        "symbol": "none",
        "data": [
{% for s in support_levels %}
          {
            "yAxis": {{ s.price }},
            "lineStyle": { "color": "#4fc3f7", "type": "dashed", "width": {{ 2 if loop.index == 1 else 1 }} },
            "label": { "show": true, "position": "insideStartTop", "formatter": "{{ s.label }}", "color": "#4fc3f7", "fontSize": 9 }
          },
{% endfor %}
{% for r in resistance_levels %}
          {
            "yAxis": {{ r.price }},
            "lineStyle": { "color": "#ff7043", "type": "dashed", "width": {{ 2 if loop.index == 1 else 1 }} },
            "label": { "show": true, "position": "insideStartTop", "formatter": "{{ r.label }}", "color": "#ff7043", "fontSize": 9 }
          }{{ "," if not loop.last or pivot_price else "" }}
{% endfor %}
{% if pivot_price %}
          ,{
            "yAxis": {{ pivot_price }},
            "lineStyle": { "color": "#ffd740", "type": "dotted", "width": 1 },
            "label": { "show": true, "position": "insideStartTop", "formatter": "P {{ pivot_price }}", "color": "#ffd740", "fontSize": 9 }
          }
{% endif %}
        ]
      }
    }
  ]
}
```

## Price Summary

| Metric | Value |
|--------|-------|
| Trend | {{ trend }} |
| Period Low | ${{ low }} |
| Period High | ${{ high }} |
| Current Price | ${{ last_close }} |
| Change | {{ change_pct }}% |
| Volatility (avg range) | {{ volatility_pct }}% |

## Sentiment Gauge

```echarts
{
  "backgroundColor": "transparent",
  "grid": { "left": "10%", "right": "10%", "bottom": "2%", "top": "2%" },
  "series": [{
    "type": "gauge",
    "min": 0,
    "max": 100,
    "startAngle": 180,
    "endAngle": 0,
    "data": [{ "value": {{ sentiment_score }}, "name": "{{ sentiment_label }}" }],
    "axisLine": {
      "lineStyle": {
        "width": 18,
        "color": [
          [0.2, "#ff4d4d"],
          [0.4, "#ff7043"],
          [0.6, "#ffd740"],
          [0.8, "#66bb6a"],
          [1, "#00ff94"]
        ]
      }
    },
    "pointer": { "width": 4, "length": "60%", "itemStyle": { "color": "#fff" } },
    "axisTick": { "show": false },
    "splitLine": { "show": false },
    "axisLabel": { "show": false },
    "detail": { "formatter": "{value}", "fontSize": 20, "color": "#fff", "offsetCenter": [0, "20%"] },
    "title": { "fontSize": 12, "color": "#999", "offsetCenter": [0, "45%"] }
  }]
}
```

## Technical Analysis

{{ technical_analysis }}

## Research Findings

### Breaking News

{% for n in breaking_news %}
- **{{ n.title }}**: {{ n.summary }}
{% endfor %}

### Market Analysis & Predictions

{% for n in analysis_items %}
- **{{ n.source }}**: {{ n.insight }}
{% endfor %}

### Macro & Regulatory

{% for n in macro_items %}
- **{{ n.title }}**: {{ n.summary }}
{% endfor %}

## Sentiment Analysis

| Dimension | Rating |
|-----------|--------|
| News Sentiment | {{ news_sentiment }} |
| Technical Bias | {{ technical_bias }} |
| Institutional Interest | {{ institutional_interest }} |
| Overall | **{{ overall_sentiment }}** |

## Catalysts & Risk Factors

### Upcoming Catalysts

{% for c in catalysts %}
- {{ c.date }}: {{ c.event }} — *{{ c.expected_impact }}*
{% endfor %}

### Risk Factors

{% for r in risks %}
- **{{ r.category }}**: {{ r.description }}
{% endfor %}

## Outlook

{{ outlook }}
````

### Step 6: Save report (only if requested)

Only if the user explicitly asks to save, export, or generate a report file, save it using the `text_editor` tool:

**file path**: `"reports/crypto-market-analysis-{{ pair }}-{{ current_date }}.md"`

Save the same content you already presented in Step 5. Confirm to the user that the report was saved, and emit its path in **backticks** (inline code) so they can tap it to open the file (the app makes `reports/…` paths clickable):

```text
Report saved to `reports/crypto-market-analysis-{{ pair }}-{{ current_date }}.md`
```

## ECharts Data Formatting Rules

When building the `echarts` JSON block:

1. **candle_labels_json**: A JSON array of unique label strings for each candle. For daily intervals use `"MM-DD"` format (e.g. `["05-27", "05-28", "05-29"]`). For sub-daily intervals (1h, 4h), use `"MM-DD HH:mm"` format. Every label must be unique.
2. **candle_ohlc_json**: A JSON array of arrays `[open, close, low, high]` — this is ECharts candlestick format (NOT OHLC order). Must have exactly the same length as `candle_labels_json`.
3. **support_levels** / **resistance_levels**: Arrays of `{ price, label }` objects. `price` is a raw number. `label` is a short string like `"S1 67000"`, `"R2 74000"`. Include 2-3 support levels (S1, S2, S3) and 2-3 resistance levels (R1, R2, R3). Optionally include `pivot_price` for the consolidation midpoint.
4. Do NOT set `min`/`max` on yAxis — `scale: true` auto-fits to the data range.
5. In the final JSON output, remove all Jinja loop/if syntax — expand the markLine data array into concrete entries. No trailing commas.
5. All numbers must be raw (no quotes, no $ signs) in the JSON.
6. Verify that `candle_labels_json.length === candle_ohlc_json.length` before outputting.
7. Do NOT add separate `line` series for support/resistance — use `markLine` inside the candlestick series.

## Constraints

- Supported pairs: any pair available on Bybit spot (append USDT if not specified).
- For timeframes, use "4h" interval with limit 42 for "last week" or default, "1h" with limit 24 for "today", "1d" with limit 30 for "last month".
- Respond with the full analysis directly in chat. Do NOT save a file unless the user explicitly asks.
- Always perform at least 3 web searches. Chain them — let each result inform the next query.
- If news search fails or returns no results, still present the analysis with price data only and note the section as unavailable.
- The echarts JSON must be valid — test mentally that all arrays have the same length as candle_dates.
- Do not invent price data. All numbers must come from the tool response.
