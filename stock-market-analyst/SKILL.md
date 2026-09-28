---
name: stock-market-analyst
description: Research tokenized US stocks — browse available tickers, analyze fundamentals, and generate comprehensive reports with web research.
metadata:
  tool: tokenized_securities
  category: rwa
  enabled: true
---

# Stock Market Analyst

Research tokenized US stocks (Ondo Finance) on Ethereum and BSC. Opens an interactive stock selector, fetches on-chain and fundamental data, performs web research, and saves a comprehensive analysis report.

## Instructions

Execute all steps silently. No internal thoughts. Do not omit any Step. Do no make suggestions outside of the defined steps.
If the query is ambiguous, use the (Step 1) stock selector to get the specific ticker before proceeding. Always use the exact ticker returned by the selector for all subsequent steps. Do NOT proceed without a ticker. Do not ask the ticker. open the selector, and use the returned ticker for all subsequent steps.

## Important Concept

Each token represents `multiplier` shares of the underlying stock, NOT exactly 1 share. The multiplier starts at 1.0 and grows as dividends accumulate.

```
referencePrice = tokenPrice ÷ sharesMultiplier
```

A small premium/discount (±0.1%) between token and stock price is normal.

### Step 1: Open stock selector

Open the selector modal. Use the returned ticker for all subsequent steps.

Call `run_js` with:

- **function**: "tokenized_securities"
- **data**: `{"command": "browse"}`

Returns a ticker (e.g. "AAPL"). This is the stock to analyze in all following steps. Do NOT proceed until you have this ticker.

### Step 2: Fetch stock data

Call `run_js` with:

- **function**: "tokenized_securities"
- **data**: `{"command": "analyze", "ticker": "<ticker>"}`

Returns company info, on-chain metrics, stock fundamentals, chainId, and contract address. Extract all relevant data for the report.

### Step 3: Web research

Use the `web_search` tool.
Chain 2-4 calls, each building on previous results.
Each call result seeds the next query.
Start broad, then let the findings guide you deeper each subsequent call.
Each call must differ.
If a call returns poor results, reformulate the search.

### Final Step: Generate report

Synthesize everything: API data from Step 2, web research from Step 3, and the user's original question. Look for relationships between the fundamentals, recent news, and market sentiment. Save the report with the `text_editor` tool. The template below is **Jinja2** — substitute `{{ var }}` placeholders with concrete values, expand `{% for %}` loops over your collected data, and resolve `{% if %}` blocks. Drop `{# comments #}` from the final output.

**file path**: `"reports/stock-analysis-{{ ticker }}-{{ current_date }}.md"`

**Report template:**

```jinja
# {{ ticker | upper }} — Tokenized Stock Report

**Company:** {{ company_name }}
**Industry:** {{ industry }}
**Generated:** {{ current_date }}

## Token Identity

- **Chain:** {{ chain_name }} ({{ chain_id }})
- **Contract:** {{ contract_address }}
- **Multiplier:** {{ multiplier }}

## Price Summary

| Metric | Value |
|--------|-------|
| Token Price | ${{ token_price }} |
| Reference Price (÷ multiplier) | ${{ reference_price }} |
| Stock Price | {% if stock_price %}${{ stock_price }}{% else %}Market closed{% endif %} |
| 24h Change | {{ change_24h }}% |
| 52-Week Range | ${{ low_52w }} – ${{ high_52w }} |

## Fundamentals

| Metric | Value |
|--------|-------|
| P/E Ratio | {{ pe_ratio }} |
| Dividend Yield | {{ dividend_yield }}% |
| Market Cap | ${{ market_cap }} |
| On-Chain Holders | {{ holders }} |

## Company Overview

{{ company_overview }}  {# 2 sentences about company description and concept tags #}

## Recent Analysis & News

{% for finding in news_findings %}
- **{{ finding.topic }}**: {{ finding.insight }}
  [source]({{ finding.url }})
{% endfor %}

## Outlook

{{ outlook }}  {# Insights combining fundamentals, news sentiment, and notable catalysts/risks #}

## Observations

### Near-Term Performance

{{ near_term_performance }}  {# Observations on near-term potential based on fundamentals, technicals and recent catalysts #}

### Key Factors to Watch

{% for factor in key_factors %}  {# up to 5 factors #}
- **{{ factor.name }}**: {{ factor.impact }}
{% endfor %}

### Suggested Actions

{{ suggested_actions }}  {# Concrete action/watchlist ideas: entry points, levels to monitor, upcoming events #}
```

Save the report. Tell the user a brief summary (2-3 sentences) and that the report was saved. Emit the saved report path in **backticks** (inline code) so the user can tap it to open the file (the app makes `reports/…` paths clickable):

```text
Report saved to `reports/stock-analysis-{{ ticker }}-{{ current_date }}.md`
```

Do NOT output the full raw report.

## Constraints

- Always save the report, Do not omit this step.
- This skill is ONLY for tokenized US stocks (Ondo Finance). Do NOT use it for crypto tokens like BTC, ETH, SOL.
- Always relay exact values from the API. Do NOT invent or modify numbers.
- Report formatting should be clean and organized, with clear sections and tables where appropriate.
- When showing prices, always clarify whether it's the token price or the stock price.
- Web searches must use the current year and company-specific terms for relevant results.
- If some tool returns an error or fails, explain to the user that part of the data is unavailable but still generate a report with the information you do have.
