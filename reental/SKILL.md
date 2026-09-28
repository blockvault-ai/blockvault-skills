---
name: reental
description: Supply liquidity to or borrow from Reental lending pools, and manage (withdraw/repay) those positions. Fetches reserves with APY and the user's current position before acting. Use when the user asks to lend, supply, deposit, borrow, withdraw, repay, or earn yield via Reental.
metadata:
  category: defi
  enabled: true
---

### Instructions

This is a **conversational** skill. First **identify the user's intent**, then guide them to a concrete action — never execute a mutating endpoint without an explicit user pick.

#### Step 0: Parse user intent

Determine the action from the user's message:

| Intent | Action | Keywords |
|---|---|---|
| **SUPPLY** | `/supply` | lend, supply, deposit, earn yield, earn, APY, put to work |
| **BORROW** | `/borrow` | borrow, loan, take out, leverage |
| **WITHDRAW** | `/withdraw` | withdraw, redeem, take out, pull out, exit |
| **REPAY** | `/repay` | repay, pay back, close loan, settle |
| **POSITION** | `/user/<wallet>` | my positions, my position, what do I have, balance, supplied, borrowed |
| **RESERVES** | `/reserves` | rates, pools, what can I lend, list, earn on |

Read-only intents (POSITION, RESERVES) never require a pick — just fetch and render. Mutating intents (SUPPLY, BORROW, WITHDRAW, REPAY) require the user to confirm **asset** and **amount** before any call.

#### Step 1: Get the wallet address and balances

Call `run_js` with `function: "get_assets"`, `data: {"hasBalance": true}`. Note the wallet `address` for the target chain and verify sufficient funds first.

#### Step 2: Fetch the data the intent needs

- SUPPLY / BORROW → `/reserves` to list assets and APYs.
- WITHDRAW / REPAY → `/user/<wallet>` plus `/reserves`, so the user can pick from what they actually hold/owe.
- POSITION → `/user/<wallet>` only.

#### Step 3: Present options and ask

Render the relevant table/chart, then ask the user for asset + action + amount using `ask_choice` (asset/action) and `ask_input` (amount). Wait for their answer before any mutating endpoint.

#### Step 4: Execute and notify

Call the chosen endpoint, then notify the user of the result via `notify_user`.

If there are insufficient funds, or an action fails, notify the user with the error message.
Do not supply or borrow without first checking the reserves and the user's balances.


# Reental Lending Pools

Supply and borrow assets through the Reental lending protocol via the BlockVault API.
Base URL: `https://402.blockvault.ai`

Every mutating endpoint (`/supply`, `/withdraw`, `/borrow`, `/repay`) returns a **signable envelope** with an ordered `transactions[]` list, each entry `{ to, data, value, chainId, from }`. The `bash` tool auto-detects these `transactions[]` envelopes — it signs and broadcasts each transaction in order via the wallet (blockchain resolved from `chainId`, e.g. `137` = polygon). If auto-detection does not fire (e.g. an unexpected response shape), sign manually with `run_js` → `sign_transaction` using the returned `to`, `data`, and `blockchain: "polygon"`.

## Typed request fields

- `asset` — underlying asset contract address (0x-prefixed).
- `amount` — human-readable **decimal** (e.g. `"100.5"`), converted to base units server-side.
- `decimals` — optional; only needed when the asset is not resolvable on-chain.
- `on_behalf_of` — wallet address the position is credited to (usually the user's own address).
- `interest_rate_mode` — `2` = variable (default), `1` = stable.
- `referral_code` — optional `uint16` referral code (supply/borrow only).
- `to` — optional recipient of withdrawn funds (withdraw only; defaults to `on_behalf_of`).


### Get the wallet address and balances

Call `run_js` with:
- **function**: "get_assets"
- **data**: `{"hasBalance": true}`

Note the wallet `address` for the target chain and verify sufficient funds for the action.


### List lending reserves

Fetch all reserves with supply/borrow APY, liquidity, and USD price:

```bash
curl -s "https://402.blockvault.ai/api/v1/reental/reserves"
```

Each reserve returns `underlying_asset`, `symbol`, `decimals`, `supply_apy`, `borrow_apy`, `price_in_usd`, and USD-normalized amounts (`available_liquidity_usd`, `total_supplied_usd`, `total_borrowed_usd`). APY values are decimal (e.g. `0.1412` = 14.12%). The list is already cleaned server-side — inactive (0% APY, no liquidity) reserves are omitted from the response.


### Present the reserve options (Jinja2)

Sort the reserves by `supply_apy` descending (best yield first), then render a readable options table to the user using the Jinja2 template below. Substitute `{{ var }}` with concrete values, expand the `{% for %}` loop over the sorted reserves, and drop `{# comments #}` from the final output. Convert each APY to a percentage by multiplying by 100 (round to 2 decimals).

````jinja
## Reental lending reserves

Here are the current pools, ordered by best supply APY:

| # | Asset | Supply APY | Borrow APY | Available liquidity | Price |
|---|-------|-----------|------------|---------------------|-------|
{% for r in reserves_sorted %}
| **{{ loop.index }}** | {{ r.symbol }} | {{ (r.supply_apy * 100) | round(2) }}% | {{ (r.borrow_apy * 100) | round(2) }}% | ${{ r.available_liquidity_usd | round(2) }} | ${{ r.price_in_usd }} |
{% endfor %}

Tell me the **number** of the asset and what you want to do — `supply`, `borrow`, `withdraw`, or `repay` — plus the amount.
````

After rendering, **ask the user** which reserve and action they want, and the amount. Prefer `ask_choice` with the sorted asset symbols (plus the action) and `ask_input` (type `number`) for the amount. Wait for their answer before calling any mutating endpoint. Never pick an asset or amount for the user.


### Market summary

Optional — aggregate metrics across all reserves:

```bash
curl -s "https://402.blockvault.ai/api/v1/reental/market"
```

Returns `total_liquidity`, `total_borrowed`, `total_value_locked`, and `reserve_count`.

Render the market composition as an ECharts donut chart (the app renders ```echarts fenced blocks natively). Use the Jinja2 + ECharts template below — substitute values and drop `{# comments #}`:

````jinja
## Reental market overview

```echarts
{
  "backgroundColor": "transparent",
  "tooltip": { "trigger": "item", "formatter": "{b}: ${c} ({d}%)" },
  "legend": { "bottom": 0, "textStyle": { "color": "#999" } },
  "series": [{
    "type": "pie",
    "radius": ["45%", "70%"],
    "itemStyle": { "borderColor": "#0e1916", "borderWidth": 2 },
    "label": { "color": "#ccc" },
    "data": [
      { "value": {{ total_liquidity }}, "name": "Liquidity" },
      { "value": {{ total_borrowed }}, "name": "Borrowed" }
    ]
  }]
}
```

| Metric | Value |
|--------|-------|
| Total liquidity | ${{ total_liquidity }} |
| Total borrowed | ${{ total_borrowed }} |
| TVL | ${{ total_value_locked }} |
| Reserves | {{ reserve_count }} |
````

Optionally add a bar chart of supply APY per reserve (from `/reserves`) to help the user compare yield at a glance:

````jinja
```echarts
{
  "backgroundColor": "transparent",
  "grid": { "left": 60, "right": 20, "top": 20, "bottom": 30 },
  "tooltip": { "trigger": "axis" },
  "xAxis": { "type": "category", "data": {{ reserve_symbols_json }} },
  "yAxis": { "type": "value", "name": "APY %" },
  "series": [{
    "type": "bar",
    "data": {{ reserve_apys_json }},
    "itemStyle": { "color": "#00ff94" }
  }]
}
```
````

Convert decimal APY to percent (× 100) before building the chart data.


### Check the user's current position

Fetch supplied and borrowed positions across all reserves:

```bash
curl -s "https://402.blockvault.ai/api/v1/reental/user/<wallet_address>"
```

Returns `positions` with `underlying_asset`, `symbol`, `decimals`, and human-readable `supplied_human` / `borrowed_human` (in tokens, already divided by `decimals` — use these, not the raw `supplied`/`borrowed` which are base units).

Render the position with an ECharts grouped bar (supplied vs borrowed per asset) plus a summary table, using the template below:

````jinja
## Your Reental position

```echarts
{
  "backgroundColor": "transparent",
  "tooltip": { "trigger": "axis" },
  "legend": { "data": ["Supplied", "Borrowed"], "textStyle": { "color": "#999" } },
  "grid": { "left": 50, "right": 20, "top": 40, "bottom": 30 },
  "xAxis": { "type": "category", "data": {{ position_symbols_json }} },
  "yAxis": { "type": "value" },
  "series": [
    { "name": "Supplied", "type": "bar", "data": {{ position_supplied_json }}, "itemStyle": { "color": "#00ff94" } },
    { "name": "Borrowed", "type": "bar", "data": {{ position_borrowed_json }}, "itemStyle": { "color": "#ff4d4d" } }
  ]
}
```

| Asset | Supplied | Borrowed |
|-------|----------|----------|
{% for p in positions %}
| {{ p.symbol }} | {{ p.supplied_human }} | {{ p.borrowed_human }} |
{% endfor %}
````

Only include reserves where `supplied` or `borrowed` is non-zero. If the position is empty, say "You have no open Reental positions" and skip the chart.


### Supply liquidity

Deposit an asset to earn supply APY:

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/reental/supply" \
  -H "Content-Type: application/json" \
  -d '{"asset":"<asset_address>","amount":"<decimal>","on_behalf_of":"<address>","referral_code":0}'
```

- `asset` (required): underlying asset address
- `amount` (required): decimal amount (e.g. `"100.5"`)
- `on_behalf_of` (required): wallet to credit
- `referral_code`: optional uint16


### Withdraw liquidity

Withdraw a supplied asset:

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/reental/withdraw" \
  -H "Content-Type: application/json" \
  -d '{"asset":"<asset_address>","amount":"<decimal>","on_behalf_of":"<address>","to":"<recipient_address>"}'
```

- `asset` (required), `amount` (required), `on_behalf_of` (required)
- `to`: optional recipient (defaults to `on_behalf_of`)


### Borrow

Borrow an asset against the supplied collateral:

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/reental/borrow" \
  -H "Content-Type: application/json" \
  -d '{"asset":"<asset_address>","amount":"<decimal>","on_behalf_of":"<address>","interest_rate_mode":2,"referral_code":0}'
```

- `asset` (required), `amount` (required), `on_behalf_of` (required)
- `interest_rate_mode`: `2` variable (default) or `1` stable
- `referral_code`: optional uint16


### Repay

Repay a borrowed asset:

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/reental/repay" \
  -H "Content-Type: application/json" \
  -d '{"asset":"<asset_address>","amount":"<decimal>","on_behalf_of":"<address>","interest_rate_mode":2}'
```

- `asset` (required), `amount` (required), `on_behalf_of` (required)
- `interest_rate_mode`: `2` variable (default) or `1` stable


### notify_user

Send the result to the user.

- **title**: String, Required. Max 50 chars.
- **message**: String, Required. Plain text, max 1024 chars. No markdown.

## Constraints

- Call `get_assets` to resolve the wallet address and verify sufficient funds before any action.
- List `/reserves` and check `/user/{wallet_address}` before supplying or borrowing.
- `amount` is always human-readable decimal; `asset` is the underlying token contract address.
- Default `interest_rate_mode` is `2` (variable).
- Borrowing is collateralized — verify the user has supplied collateral before borrowing.
- Always call `notify_user` as the final action.