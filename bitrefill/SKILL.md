---
name: bitrefill
description: Browse, search and purchase gift cards, mobile top-ups, eSIMs and bill payments on Bitrefill, consult invoices and orders, and answer support questions from the Bitrefill help center. Payments go on-chain from the user's BlockVault wallet as an exact USDT transfer on Polygon to the invoice address (one wallet approval). Use this whenever the user wants to explore Bitrefill products, see prices, buy something, look up a previous purchase, or get Bitrefill help.
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

Interactive Bitrefill assistant. Use the available endpoints to **browse, search, inspect, purchase, and review** products and invoices on behalf of the user, and to **answer support questions** from the Bitrefill help center. Render results through the `render/*.html` templates — never dump raw JSON or TOON, and never summarize a rendered list into Markdown.

Detect the language of the user's query and reply in that language.

## Instructions

- **Render, don't re-list.** After a read tool call with `-o`, emit the ```artifact fence for the matching template. Do NOT re-list the results in Markdown — the card deck IS the list.
- **Always recommend.** After the ```artifact fence, add a one-line recommendation in plain text: **My pick:** <product/package> — <reason tied to the user's stated needs or preferences>. Never emit the fence without a recommendation.
- **Ask when the intent is unclear.** If the user's request is ambiguous (no product, brand, category, or country named), ask ONE clarifying question before searching — do not guess and dump a generic list.
- **Never auto-pick** — let the user choose a product by number before buying.

## API

All calls hit the **same MCP JSON-RPC endpoint**: `POST https://api.bitrefill.com/mcp`.

The secret placeholder `{{BITREFILL_API_KEY}}` is resolved automatically at execution time — write it verbatim in the Authorization header, never echo it back to the user.

### The one curl template

Copy this template **as-is**. The ONLY thing you change between calls is the `<TOOL_NAME>`, `<ARGUMENTS>` and the `-o <name>.json` output filename. Headers, URL and everything else are fixed.

```bash
curl -sS -X POST https://api.bitrefill.com/mcp \
  -H "Authorization: Bearer {{BITREFILL_API_KEY}}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"<TOOL_NAME>","arguments":<ARGUMENTS>}}' \
  -o <name>.json
```

`<TOOL_NAME>` is one of the 10 tools below; `<ARGUMENTS>` is its JSON arguments object.

Never split `-H` from its value. Never put an `-H` between `-d` and the URL. The URL is the LAST argument before `-o`.

### Response format

Bitrefill answers with one Server-Sent Events frame:

```
event: message
data: {"result":{"content":[{"type":"text","text":"<payload>"}]},"jsonrpc":"2.0","id":1}
```

Read tools (`search-products`, `get-product-details`, `list-invoices`, `get-invoice-by-id`, `update-order`, and the three help tools) return **TOON** — Token-Oriented Object Notation, a compact indented key/value text meant for language models (reads like indented `key: value` lines, ~40% fewer tokens than JSON). `buy-products` returns plain **JSON** so payment fields parse exactly, plus an `agent_instructions` string that tells you the next step. If you see `error` instead of `result`, follow the error-handling section below.

**When you add `-o <name>.json`, the `bash` tool decodes the SSE frame and TOON payload to JSON automatically** and writes that JSON to the artifact. The `artifact_path` in the result points at the JSON, ready for a render template. Do NOT read the JSON yourself — emit the ```artifact fence instead.

### Rate limit

30 requests per minute per bearer token. Over the limit you get HTTP 429 with `status: "rate_limit_reached"`. Stop and back off for the rest of the minute before retrying.

### Tools (10)

For every call, only `<TOOL_NAME>` and `<ARGUMENTS>` change. Pick a tool below and drop its `name` + `arguments` into the template above.

**`search-products`** — list or search gift cards / eSIMs / refills.
Args: `intent` (required — the user's goal, e.g. "buy a gift card for a friend"), `query` (optional keyword filter, default `*` = all), `country` (ISO-2, default `US`), `category` (optional — exact slug from the API: `gifts`, `esim`, `refill`, `food`, `streaming`, `games`, `travel`, `accommodation`, `online-marketplaces`, `department-stores`, `electronics`, `bill`, …), `product_type` (`giftcard` | `esim`), `page` (default 1), `per_page` (default 25, max 250).

**Search by `query`, not `category`.** `query` matches brand/name/keywords and returns broad, useful results. `category` is a narrow exact-match filter that often returns very few products — only use it when the user explicitly names a category. To browse everything in a country, omit `query` (or use `query: "*"`) and omit `category`.

```json
{"name": "search-products", "arguments": {"intent": "buy an Amazon gift card", "query": "amazon", "country": "ES", "per_page": 20}}
{"name": "search-products", "arguments": {"intent": "browse gift cards", "query": "*", "country": "ES", "per_page": 20}}
{"name": "search-products", "arguments": {"intent": "find a Netflix gift card", "query": "netflix", "country": "US", "per_page": 10}}
```

**`get-product-details`** — full product info: `packages[]` (each with `package_value`), `range` (`min` / `max` / `step`), `recipient_type`, `payment_methods` (grouped as `address_based` / `link_only` / `balance`), and `prepayment` (if any).
Args: `product_id` (required slug from search), `language` (default `en`), `currency` (`BTC` | `USD` | `EUR` | `USDC` | `USDT` | `ETH` | `SOL` | `GBP` | `AUD` | `CAD` | `INR` | `BRL`; default `BTC`, use `USDT` for BlockVault).

```json
{"name": "get-product-details", "arguments": {"product_id": "amazon_com-usa", "currency": "USDT"}}
```

**`buy-products`** — create the invoice. See variants below. Returns `invoice_id`, `invoice_access_token`, `payment_link`, `x402_payment_url`, `payment_info` (crypto address + amount), `expiration_minutes`, and `agent_instructions`.
Args:

- `cart_items` — array (max 15). Each item: `product_id` + `package_id` set to the **`package_value`** string (e.g. `"50"` or `"5GB, 30 Days"` — NOT the full `slug<&>value`). Optional per-item: `refill_input` (phone/account/email when `recipient_type` requires it), `bill_payment_id` (for prepayment products), `gift` (`{recipient_name, recipient_email, sender_name, message?, theme?, send_date?}`).
- `payment_method` — for BlockVault use **`usdt_polygon`** (USDT is the default; Bitrefill offers no USDT on Arbitrum or Base). Other crypto: `bitcoin`, `lightning`, `ethereum`, `solana`, `usdc_polygon`, `usdc_base`, `usdc_erc20`, `usdt_erc20`, `usdt_trc20`, `usdc_arbitrum`, `usdc_solana`, `eth_base`, `litecoin`. Only use another method if the user explicitly asks for it. Account methods `balance` / `cashback` require a Bitrefill account balance — the user has none, so never use them.
- `return_payment_link` — controls which payment channels the response carries. Default `true` = returns `payment_info` (address + amount) plus `payment_link` and `x402_payment_url`. `false` = returns only `payment_info`. BlockVault pays from `payment_info`, so either value works; keep the default.
- `balance_currency` — only with `payment_method:"balance"` (`EUR` | `USD` | `XBT`).

```json
{"name": "buy-products", "arguments": {"cart_items": [{"product_id": "amazon_com-usa", "package_id": "50"}], "payment_method": "usdt_polygon", "return_payment_link": true}}
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

Always: `"payment_method": "usdt_polygon"`. The invoice response carries `payment_info` with the exact `address`, `altcoinPrice` (USDT amount, decimal string) and `contractAddress` (USDT on Polygon) — that is what BlockVault pays.

> The cart-item denomination field is documented two ways: the "eCommerce MCP" page calls it `package_id`, the current Partner guide calls it `package_value`. Both mean *"set this to the `package_value` string"*. Prefer `package_value`; the `tools/list` schema is the source of truth — if a call fails with `VALIDATION_ERROR`, inspect that name and adjust.

### Redemption info (from `get-invoice-by-id.orders[i].redemption_info` once the invoice is `complete` and `redemption_available` is true)

**gift_card**: `code`, optional `pin`, `link`, `instructions`
**phone_refill**: usually empty (credit is applied directly to the number)
**esim**: `esim_install_link` (LPA activation), `pin`
**bill_payment**: provider receipt fields, `instructions`

Never mask the redemption code — the user paid for it.

## Rendering

Render results through the `render/*.html` templates below. Every read tool call adds `-o <name>.json` so the response is saved to an artifact; the `bash` tool decodes the TOON/SSE payload to JSON automatically, so the template consumes it directly. Never dump raw JSON or TOON to the user.

Ignore the `price` field entirely — it is an internal Bitrefill unit. The user-facing denomination is `value` + `currency`. The actual USDT cost appears in the invoice as `payment_info.amount`. Never expose product slugs / ids to the user; keep them internal for chaining tool calls and refer to products by row number or display name.

**Product list** (search / browse results):

Call `search-products` with `-o bitrefill-products.json`. The `bash` result includes `artifact_path`. **Do NOT read the JSON.** Emit an ```artifact fence with template `bitrefill:product-list` (renders `data.products` as a swipeable card deck).

````markdown
```artifact
{"artifact":"<artifact_path>","template":"bitrefill:product-list"}
```
````

After rendering, add a one-line recommendation below the fence: **My pick:** <product name> — <reason tied to the user's stated needs or preferences>. Then ask the user to pick a number to see details or buy.

**Product detail**:

Call `get-product-details` with `-o bitrefill-product.json`. The `bash` result includes `artifact_path`. **Do NOT read the JSON.** Emit an ```artifact fence with template `bitrefill:product-detail` (renders `data` as a detail card with packages and reviews).

````markdown
```artifact
{"artifact":"<artifact_path>","template":"bitrefill:product-detail"}
```
````

After rendering, recommend the best package for the user's needs (e.g. "For a €50 top-up, pick package 1 — €56.19 in USDT"). Then ask which package they want.

**Invoice list**:

Call `list-invoices` with `-o bitrefill-invoices.json`. The `bash` result includes `artifact_path`. **Do NOT read the JSON.** Emit an ```artifact fence with template `bitrefill:invoice-list` (renders `data.invoices` as a swipeable card deck).

````markdown
```artifact
{"artifact":"<artifact_path>","template":"bitrefill:invoice-list"}
```
````

**Invoice detail**:

Call `get-invoice-by-id` with `-o bitrefill-invoice.json`. The `bash` result includes `artifact_path`. **Do NOT read the JSON.** Emit an ```artifact fence with template `bitrefill:invoice-detail` (renders `data` as a detail card).

````markdown
```artifact
{"artifact":"<artifact_path>","template":"bitrefill:invoice-detail"}
```
````

**Order redemption** (read from `inv.orders[i].redemption_info` after `get-invoice-by-id` returns `status:"complete"`):

Call `get-invoice-by-id` with `-o bitrefill-redemption.json`. The `bash` result includes `artifact_path`. **Do NOT read the JSON.** Emit an ```artifact fence with template `bitrefill:redemption` (renders `data.orders` — one card per delivered order with its redemption code).

````markdown
```artifact
{"artifact":"<artifact_path>","template":"bitrefill:redemption"}
```
````

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

Confirm USDT on `polygon` has enough for the purchase. If not, tell the user and stop.

### Resolve product

Search or browse using `search-products` (always pass `intent`) if needed, then call `get-product-details` with `currency:"USDT"` to read `packages[]` and pick the exact `package_value` to use as `package_id`. Do NOT invent ids — they must come from a previous API response. If the product has a `prepayment` block, run the `submit-prepayment-step` loop until you get a `bill_payment_id`.

### Create invoice

Call `buy-products` with `payment_method:"usdt_polygon"`. Extract from `response`: `invoice_id`, `invoice_access_token`, `payment_info.address`, `payment_info.altcoinPrice` (the exact USDT amount, e.g. `"2.53"`), `payment_info.contractAddress`, and `expiration_minutes`.

### Confirm with the user

Show the product, denomination, the exact USDT amount (`payment_info.altcoinPrice`), and chain (Polygon). Wait for an explicit "yes" / "confirm". Do NOT proceed without it.

### Pay the invoice

Call `run_js` with:

- **function**: String, Required. Set to `"transfer"`.
- **data**: Object, Required. `{"symbol": "USDT", "amount": "<payment_info.altcoinPrice>", "toAddress": "<payment_info.address>", "blockchain": "polygon"}`.

Copy `amount` and `toAddress` verbatim from `payment_info` — never round, recompute, or add fees. The user approves the transfer once in the wallet modal; the result carries `txHash`. If the user rejects, stop and do not retry.

**Never `curl` the `x402_payment_url` for MCP invoices.** Bitrefill's `/x402/invoice/pay` does not resolve the price of invoices created by `buy-products` and asks for its 1000-unit route cap instead of the invoice amount. Paying it would authorize far more than the invoice.

Pay before `expiration_minutes` elapses; if the invoice expired, recreate it with `buy-products` first.

### Poll invoice status

Call `get-invoice-by-id` with `invoice_id` + `invoice_access_token` every ~10–30 s (not faster) until the status is `complete`. Poll up to ~10 times before reporting back. If it transitions to `blocked` / `denied` / `payment_error` / `failed` / `expired`, report it and stop. Intermediate statuses: `unpaid`, `payment_detected`, `payment_confirmed`, `pending`.

### Redeem

The successful `get-invoice-by-id` response contains `orders[].redemption_info` (with `redemption_available: true`). Show the redemption code, link and / or PIN to the user. Never mask the code.

### Save receipt

Use the `text_editor` tool to create `receipts/bitrefill-<INVOICE_ID>.md` containing product name, totals, the on-chain `txHash` returned by the `transfer` call, and redemption codes.

Then, in your chat reply, emit the receipt path in **backticks** (inline code) so the user can tap it to open the file — the app makes `receipts/…`, `reports/…`, `trips/…`, `artifacts/…` paths clickable:

```text
Receipt saved to `receipts/bitrefill-<INVOICE_ID>.md`
```

Never invent the `INVOICE_ID` — use the one returned by `buy-products`.

## Constraints

- Always `payment_method: "usdt_polygon"` and pay with `run_js` → `transfer` using `payment_info.address` + `payment_info.altcoinPrice`. Never `"balance"` / `"cashback"` (the user has no Bitrefill account balance).
- Never call `x402_payment_url` — it overcharges MCP invoices (1000-unit cap).
- In `cart_items[].package_id` use the `package_value` string, NOT the full `slug<&>value`.
- Never invent product / invoice / order ids — derive them from API responses.
- Never retry a failed payment without explicit user confirmation, and never retry `PAYMENT_UNCERTAIN` at all — poll instead.
- Respect the 30 requests/minute rate limit; back off a full minute on HTTP 429.
- Never echo `{{BITREFILL_API_KEY}}` to the user.
- The OAuth access token is audience-bound to `https://api.bitrefill.com/mcp`. The legacy REST `/v2/...` endpoints will reject it with `invalid_token` — always use the MCP JSON-RPC endpoint above.
