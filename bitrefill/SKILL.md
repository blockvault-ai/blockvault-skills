---
name: bitrefill
description: Browse, search and purchase gift cards, mobile top-ups, eSIMs and bill payments on Bitrefill, consult invoices and orders, and answer support questions from the Bitrefill help center. Payments go on-chain from the user's BlockVault wallet in USDC on Base via x402 (auto-signed, no manual transfer). Use this whenever the user wants to explore Bitrefill products, see prices, buy something, look up a previous purchase, or get Bitrefill help.
metadata:
  tool: bash
  category: tools
  enabled: true
  homepage: https://www.bitrefill.com/account/developers
  secrets:
    - key: BITREFILL_API_KEY
      description: "Sign in with your Bitrefill account."
      oauth:
        issuer: https://api.bitrefill.com
        scope: mcp
        resource: https://api.bitrefill.com/mcp
---

# Bitrefill

Interactive Bitrefill assistant. Use the available endpoints to **browse, search, inspect, purchase, and review** products and invoices on behalf of the user, and to **answer support questions** from the Bitrefill help center. Render results as readable Markdown so the user can navigate them, never dump raw JSON.

Detect the language of the user's query and reply in that language.

## API

All calls hit the **same MCP JSON-RPC endpoint**: `POST https://api.bitrefill.com/mcp`.

The secret placeholder `{{BITREFILL_API_KEY}}` is resolved automatically at execution time — write it verbatim in the Authorization header, never echo it back to the user.

### The one curl template

Copy this template **as-is**. The ONLY thing you change between calls is the `<TOOL_NAME>` and `<ARGUMENTS>` placeholders in the `-d` body. Headers, URL and everything else are fixed.

```bash
curl -sS -X POST https://api.bitrefill.com/mcp \
  -H "Authorization: Bearer {{BITREFILL_API_KEY}}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"<TOOL_NAME>","arguments":<ARGUMENTS>}}'
```

`<TOOL_NAME>` is one of the 10 tools below; `<ARGUMENTS>` is its JSON arguments object.

Never split `-H` from its value. Never put an `-H` between `-d` and the URL. The URL is the LAST argument.

### Response format

Bitrefill answers with one Server-Sent Events frame:

```
event: message
data: {"result":{"content":[{"type":"text","text":"<payload>"}]},"jsonrpc":"2.0","id":1}
```

Take the JSON on the `data:` line, then read `result.content[0].text`. Read tools (`search-products`, `get-product-details`, `list-invoices`, `get-invoice-by-id`, `update-order`, and the three help tools) return **TOON** — Token-Oriented Object Notation, a compact indented key/value text meant for language models (reads like indented `key: value` lines, ~40% fewer tokens than JSON). Read it directly. `buy-products` returns plain **JSON** so payment fields parse exactly, plus an `agent_instructions` string that tells you the next step. If you see `error` instead of `result`, follow the error-handling section below.

### Rate limit

30 requests per minute per bearer token. Over the limit you get HTTP 429 with `status: "rate_limit_reached"`. Stop and back off for the rest of the minute before retrying.

### Tools (10)

For every call, only `<TOOL_NAME>` and `<ARGUMENTS>` change. Pick a tool below and drop its `name` + `arguments` into the template above.

**`search-products`** — list or search gift cards / eSIMs / refills.
Args: `intent` (required — the user's goal, e.g. "buy a gift card for a friend"), `query` (optional keyword filter, default `*` = all), `country` (ISO-2, default `US`), `category` (enum: `gift-cards`, `esim`, `refill`, `flights`, `accommodation`, `car-rental`, `streaming`, `games`, `groceries`, `travel`, `bill`, …), `product_type` (`giftcard` | `esim`), `page` (default 1), `per_page` (default 25, max 250).

```json
{"name": "search-products", "arguments": {"intent": "buy an Amazon gift card", "query": "amazon", "country": "US", "per_page": 10}}
{"name": "search-products", "arguments": {"intent": "find accommodation", "country": "ES", "category": "accommodation", "per_page": 20}}
```

**`get-product-details`** — full product info: `packages[]` (each with `package_value`), `range` (`min` / `max` / `step`), `recipient_type`, `payment_methods` (grouped as `address_based` / `link_only` / `balance`), and `prepayment` (if any).
Args: `product_id` (required slug from search), `language` (default `en`), `currency` (`BTC` | `USD` | `EUR` | `USDC` | `USDT` | `ETH` | `SOL` | `GBP` | `AUD` | `CAD` | `INR` | `BRL`; default `BTC`, use `USDC` for BlockVault).

```json
{"name": "get-product-details", "arguments": {"product_id": "amazon_com-usa", "currency": "USDC"}}
```

**`buy-products`** — create the invoice. See variants below. Returns `invoice_id`, `invoice_access_token`, `payment_link`, `x402_payment_url`, `payment_info` (crypto address + amount), `expiration_minutes`, and `agent_instructions`.
Args:

- `cart_items` — array (max 15). Each item: `product_id` + `package_id` set to the **`package_value`** string (e.g. `"50"` or `"5GB, 30 Days"` — NOT the full `slug<&>value`). Optional per-item: `refill_input` (phone/account/email when `recipient_type` requires it), `bill_payment_id` (for prepayment products), `gift` (`{recipient_name, recipient_email, sender_name, message?, theme?, send_date?}`).
- `payment_method` — for BlockVault use **`usdc_base`**. Other crypto: `bitcoin`, `lightning`, `ethereum`, `solana`, `usdc_polygon`, `usdt_polygon`, `usdc_erc20`, `usdt_erc20`, `usdt_trc20`, `usdc_arbitrum`, `usdc_solana`, `eth_base`, `litecoin`. Account methods `balance` / `cashback` require a Bitrefill account balance — the user has none, so never use them.
- `return_payment_link` — controls which payment channels the response carries. Default `true` = returns `payment_link` **and** `x402_payment_url` (the two pieces needed for x402 wallets). `false` = returns only crypto `payment_info` (address + amount) for a manual transfer, and **omits `x402_payment_url`/`payment_link`**. **For the BlockVault x402 flow always use the default (`true`) — do NOT set `false`, or there will be no `x402_payment_url` to call.**
- `balance_currency` — only with `payment_method:"balance"` (`EUR` | `USD` | `XBT`).

```json
{"name": "buy-products", "arguments": {"cart_items": [{"product_id": "amazon_com-usa", "package_id": "50"}], "payment_method": "usdc_base", "return_payment_link": true}}
```

**`submit-prepayment-step`** — only for products whose `get-product-details` response contains a `prepayment` block (e.g. prepaid Visa, utility bills). Loop until the response has `step:"final"` and a `bill_payment_id`, then pass that id into `buy-products.cart_items[].bill_payment_id`. Requires a logged-in user.
Args: `product_id`, `step_number` (start at 1), `form_data` (object keyed by field id), `bill_payment_id` (echoed back from step 1 onwards).

```json
{"name": "submit-prepayment-step", "arguments": {"product_id": "<slug>", "step_number": 1, "form_data": {"first_name": "Alice"}}}
```

**`get-invoice-by-id`** — invoice status, payment info, embedded `orders[]` with `redemption_info` once delivered.
Args: `invoice_id` (UUID with dashes), `invoice_access_token` (returned by `buy-products`).

```json
{"name": "get-invoice-by-id", "arguments": {"invoice_id": "<uuid>", "invoice_access_token": "<token>"}}
```

**`list-invoices`** — paginated invoice history for the authenticated user. Only returns paid/confirmed invoices — unpaid invoices are not included.
Args: `limit` (1–50, default 25), `start`, `after` / `before` (ISO 8601), `include_orders` (default `true`).

```json
{"name": "list-invoices", "arguments": {"limit": 20}}
```

**`update-order`** — order housekeeping. `order_id` is the **24-char hex** id from `invoice.orders[].id` (not the invoice UUID). Requires a delivered order and at least one of `remaining_amount` / `is_archived`.
Args: `order_id`, `remaining_amount?`, `is_archived?`.

```json
{"name": "update-order", "arguments": {"order_id": "<24-char hex>", "is_archived": true}}
```

**`search_help_articles`** — find Bitrefill help-center articles by keyword.
Args: `query`.

```json
{"name": "search_help_articles", "arguments": {"query": "refund"}}
```

**`get_help_article`** — fetch the full content of one help article.
Args: `article_id` (from a search result).

```json
{"name": "get_help_article", "arguments": {"article_id": "<id>"}}
```

**`list_help_articles`** — list available help articles. No arguments.

```json
{"name": "list_help_articles", "arguments": {}}
```

### Invoice creation variants

`buy-products.cart_items[i]` shape per recipient_type:

**Fixed-denomination gift card**: `package_id` is the `package_value` (just the amount), not the full `slug<&>value`.

```json
{"product_id": "amazon_com-usa", "package_id": "50"}
```

**Range / custom value**: products with a `range` accept any numeric value between `range.min` and `range.max`, in multiples of `range.step`, as `package_id`.

```json
{"product_id": "amazon_com-usa", "package_id": "37.5"}
```

**Phone top-up** (`recipient_type:"phone_number"`): add `refill_input` in international (E.164) format.

```json
{"product_id": "att-usa-topup", "package_id": "25", "refill_input": "14155551234"}
```

**eSIM**: `package_id` is the plan string from `packages[].package_value`.

```json
{"product_id": "esim-europe", "package_id": "5GB, 30 Days"}
```

**Bill payment**: complete the `submit-prepayment-step` loop first, then include the resulting `bill_payment_id`.

```json
{"product_id": "<bill slug>", "package_id": "<amount>", "bill_payment_id": "<from prepayment loop>"}
```

Always: `"payment_method": "usdc_base"` and `"return_payment_link": true` (the default). The invoice response then carries `x402_payment_url` for settling in USDC on Base via BlockVault's x402 client; `payment_info` holds the exact `address` and `amount` if a manual transfer is ever needed.

> The cart-item denomination field is documented two ways: the "eCommerce MCP" page calls it `package_id`, the current Partner guide calls it `package_value`. Both mean *"set this to the `package_value` string"*. Prefer `package_value`; the `tools/list` schema is the source of truth — if a call fails with `VALIDATION_ERROR`, inspect that name and adjust.

### Redemption info (from `get-invoice-by-id.orders[i].redemption_info` once the invoice is `complete` and `redemption_available` is true)

**gift_card**: `code`, optional `pin`, `link`, `instructions`
**phone_refill**: usually empty (credit is applied directly to the number)
**esim**: `esim_install_link` (LPA activation), `pin`
**bill_payment**: provider receipt fields, `instructions`

Never mask the redemption code — the user paid for it.

## Rendering

The templates below are **Jinja2** — substitute `{{ var }}` placeholders with concrete values from the API response, expand `{% for %}` loops, and resolve `{% if %}` blocks. Drop `{# comments #}` from the final output.

Ignore the `price` field entirely — it is an internal Bitrefill unit. The user-facing denomination is `value` + `currency`. The actual USDC cost appears in the invoice as `payment_info.amount`. Never expose product slugs / ids to the user; keep them internal for chaining tool calls and refer to products by row number or display name.

**Product list** (search / browse results):

```jinja
{# one-line intro paraphrasing the user intent #}

| # | Product | Country | Currency |
|---|---------|---------|----------|
{% for p in data[:10] %}
| **{{ loop.index }}** | **{{ p.name }}** | {{ p.country_name or p.country_code }} | {{ p.currency }} |
{% endfor %}

Pick a number to see details or buy.
```

**Product detail**:

```jinja
## {{ product.name }}

{{ product.description | truncate(200) if product.description }}

| | |
|---|---|
| Country | {{ product.country_name }} |
| Currency | {{ product.currency }} |
{% if product.range %}| Range | {{ product.currency }} {{ product.range.min }} – {{ product.range.max }} |{% endif %}
{% if product.packages %}| Packages | {{ product.packages | map(attribute='value') | join(' · ') }} {{ product.currency }} |{% endif %}
{% if product.recipient_type == 'phone_number' %}| Requires | Phone number |{% endif %}
```

**Invoice list**:

```jinja
| # | Invoice | Created | Status | Total |
|---|---------|---------|--------|-------|
{% for inv in data %}
| {{ loop.index }} | `{{ inv.id }}` | {{ inv.created_time | date }} | {{ inv.status }} | {{ inv.payment_info.amount }} {{ inv.payment_info.currency }} |
{% endfor %}
```

**Invoice detail**:

```jinja
## Invoice `{{ inv.id }}`

| | |
|---|---|
| Status | {{ inv.status }} |
| Total | {{ inv.payment_info.amount }} {{ inv.payment_info.currency }} |
| Method | {{ inv.payment_info.method }} |
{% if inv.payment_info.address %}| Address | `{{ inv.payment_info.address[:8] }}…{{ inv.payment_info.address[-6:] }}` |{% endif %}
{% if inv.expiration_minutes %}| Expires in | {{ inv.expiration_minutes }} min |{% endif %}

## Orders

| Order | Product | Amount |
|-------|---------|--------|
{% for o in inv.orders %}
| `{{ o.id }}` | {{ o.product_name }} | {{ o.value }} {{ o.currency }} |
{% endfor %}
```

**Order redemption** (read from `inv.orders[i].redemption_info` after `get-invoice-by-id` returns `status:"complete"`):

```jinja
## {{ order.product_name }} — {{ order.value }} {{ order.currency }}

| | |
|---|---|
{% if order.redemption_info.code %}| Code | `{{ order.redemption_info.code }}` |{% endif %}
{% if order.redemption_info.pin %}| PIN | `{{ order.redemption_info.pin }}` |{% endif %}
{% if order.redemption_info.link %}| Link | {{ order.redemption_info.link }} |{% endif %}
{% if order.redemption_info.esim_install_link %}| eSIM activation | {{ order.redemption_info.esim_install_link }} |{% endif %}

{% if order.redemption_info.instructions %}{{ order.redemption_info.instructions }}{% endif %}
```

After rendering a list, ask the user to pick a number to drill down or proceed to purchase.

## Invoice statuses

`get-invoice-by-id` reports `status` (or `invoice_status`) through this lifecycle:

| Status | Meaning |
|---|---|
| `unpaid` | Waiting for payment |
| `payment_detected` | Payment seen on-chain, awaiting confirmation |
| `payment_confirmed` | Payment confirmed |
| `pending` | Processing the order |
| `complete` | Delivered — redemption codes available |

Error statuses — report to the user and do **not** retry the payment:

| Status | Meaning |
|---|---|
| `blocked` | Compliance review; the agent cannot resolve it |
| `denied` | Payment denied |
| `payment_error` | Payment failed |
| `failed` / `expired` | Order/invoice failed or timed out |

## Errors

Tool errors come back as `isError: true` with a JSON body `{ error, code, details }`. `details.status` narrows the cause.

| `code` | Meaning | Action |
|---|---|---|
| `VALIDATION_ERROR` | Arguments don't match the tool schema | Fix the arguments |
| `INVALID_INPUT` | Well-formed but wrong for this purchase (`details.status` says why) | Fix and retry once |
| `RESOURCE_NOT_FOUND` | Unknown product/invoice/order, or missing access token | Check ids; `details.suggestions` may list similar slugs |
| `PERMISSION_DENIED` | Tool needs a logged-in user | Tell the user; move on |
| `SERVICE_UNAVAILABLE` | Search/quote/payment session failed temporarily | Retry with back-off |
| `PAYMENT_UNCERTAIN` | A balance payment may already have started | **Never retry** — poll `get-invoice-by-id` instead |
| `INTERNAL_ERROR` | Unexpected failure on Bitrefill's side | Report it and stop |

Common `details.status` values: `product_not_found`, `product_not_available`, `invalid_package`, `number_missing`, `payment_method_not_accepted`, `invalid_payment_method`, `purchase_limit_reached`, `balance_too_low`, `bill_payment_id_missing`, `coupon_invalid`, `verification_required`, `access_denied`, `user_not_registered`, `missing_fiat_payment_info`, `invalid_billing_country`.

HTTP-level errors are separate: 400 for a bad `Accept` header, 401 for authentication (re-run OAuth), 405 for GET/DELETE, 429 for the rate limit.

## Purchase flow

How to pay an invoice | buy a product on Bitrefill, step by step:

### Check balance

Call `run_js` with:

- **function**: "get_assets"
- **data**: `{"hasBalance": true}`

Confirm USDC on `base` has enough for the purchase. If not, tell the user and stop.

### Resolve product

Search or browse using `search-products` (always pass `intent`) if needed, then call `get-product-details` with `currency:"USDC"` to read `packages[]` and pick the exact `package_value` to use as `package_id`. Do NOT invent ids — they must come from a previous API response. If the product has a `prepayment` block, run the `submit-prepayment-step` loop until you get a `bill_payment_id`.

### Create invoice

Call `buy-products` with `payment_method:"usdc_base"` and `return_payment_link: true` (the default — do NOT set `false`, which drops `x402_payment_url` from the response). Extract from the response: `invoice_id`, `invoice_access_token`, `x402_payment_url`, `payment_info` (address + amount), and `expiration_minutes`. If `x402_payment_url` is missing when you expected it, you almost certainly sent `return_payment_link: false` — recreate the invoice with the default.

### Confirm with the user

Show the product, denomination, USDC amount, and chain (Base). Wait for an explicit "yes" / "confirm". Do NOT proceed without it.

### Pay via x402

`curl` the **`x402_payment_url` returned by `buy-products`** — do NOT rebuild it, take the exact value from the response (it points to `https://api.bitrefill.com/x402/invoice/pay`). The first call returns HTTP 402 with a `PAYMENT-REQUIRED` header; BlockVault intercepts it, signs the USDC transfer on Base and retries with the `PAYMENT-SIGNATURE` header — the user approves it once in the wallet modal:

```bash
curl -sS -X POST "<x402_payment_url>" \
  -H "Content-Type: application/json" \
  -d '{"invoice_id": "<invoice_id>"}'
```

x402 applies only to invoices created with a USDC method (`usdc_base`, `usdc_arbitrum`, `usdc_polygon`, `usdc_solana`) and younger than 15 minutes (the price lock). On the automatic retry, HTTP 200 means the payment is accepted and being settled. If the invoice is older than 15 minutes, the x402 call fails — recreate the invoice with `buy-products` first.

### Poll invoice status

Call `get-invoice-by-id` with `invoice_id` + `invoice_access_token` every ~10–30 s (not faster) until the status is `complete`. Poll up to ~10 times before reporting back. If it transitions to `blocked` / `denied` / `payment_error` / `failed` / `expired`, report it and stop. Intermediate statuses: `unpaid`, `payment_detected`, `payment_confirmed`, `pending`.

### Redeem

The successful `get-invoice-by-id` response contains `orders[].redemption_info` (with `redemption_available: true`). Show the redemption code, link and / or PIN to the user. Never mask the code.

### Save receipt

Use the `text_editor` tool to create `receipts/bitrefill-<INVOICE_ID>.md` containing product name, totals, the on-chain tx hash BlockVault returns after the x402 modal approval, and redemption codes.

Then, in your chat reply, emit the receipt path in **backticks** (inline code) so the user can tap it to open the file — the app makes `receipts/…`, `reports/…`, `trips/…`, `artifacts/…` paths clickable:

```text
Receipt saved to `receipts/bitrefill-<INVOICE_ID>.md`
```

Never invent the `INVOICE_ID` — use the one returned by `buy-products`.

## Constraints

- Always `payment_method: "usdc_base"` + `return_payment_link: true` (the default — needed so the response includes `x402_payment_url`). Never `"balance"` / `"cashback"` (the user has no Bitrefill account balance).
- In `cart_items[].package_id` use the `package_value` string, NOT the full `slug<&>value`.
- Never invent product / invoice / order ids — derive them from API responses.
- Never retry a failed payment without explicit user confirmation, and never retry `PAYMENT_UNCERTAIN` at all — poll instead.
- Respect the 30 requests/minute rate limit; back off a full minute on HTTP 429.
- Never echo `{{BITREFILL_API_KEY}}` to the user.
- The OAuth access token is audience-bound to `https://api.bitrefill.com/mcp`. The legacy REST `/v2/...` endpoints will reject it with `invalid_token` — always use the MCP JSON-RPC endpoint above.
