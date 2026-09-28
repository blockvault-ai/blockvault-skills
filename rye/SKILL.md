---
name: rye
description: Browse and purchase real-world products from any merchant (Shopify, Amazon, Best Buy, and thousands more) by URL. Use this whenever the user wants to shop, buy something, compare prices, check shipping/tax for a product page, or place an order. Payments go on-chain from the user's BlockVault wallet in USDC over the x402 protocol — no Rye account or API key is needed.
metadata:
  tool: bash
  category: tools
  enabled: false
  homepage: https://rye.com/docs/api-v2/x402/overview
---

# Rye — Universal Checkout over x402

Interactive shopping assistant powered by [Rye](https://rye.com). Help the user **discover, price-check and purchase products from any merchant** (Shopify, Amazon, Best Buy, Walmart, DTC stores, etc.) directly from chat. Rye handles validation, shipping, tax and order placement; BlockVault pays each request on the fly with a signed USDC transfer.

Detect the language of the user's query and reply in that language. Render results as readable Markdown — never dump raw JSON. Lead with the **product, total price and ETA**, not technical fields.

## How payment works (read this first)

Rye exposes an x402-native endpoint at `https://x402.rye.com`. There is **no API key and no Authorization header**. Each paid call returns `402 Payment Required`; BlockVault's HTTP client **automatically signs an EIP-3009 USDC transfer** for the quoted amount and retries the request with a `PAYMENT-SIGNATURE` header. From the model's point of view this is transparent — just `curl` the endpoint and read the JSON.

| Endpoint | Method | Cost (USDC) | What it does |
|---|---|---|---|
| `/v1/checkout-intents` | POST | $0.02 | Create an intent and start offer retrieval |
| `/v1/checkout-intents/:id` | GET | free | Read intent state + retrieved offer |
| `/v1/checkout-intents/:id/confirm` | POST | purchase total + $0.03 | Confirm and place the order |

The exact amount due is quoted in the `PAYMENT-REQUIRED` header of each `402` response. `GET` calls are free, so polling never costs anything.

**Networks accepted:** Base (`eip155:8453`), Solana (`solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`), Tempo (`eip155:4217`). USDC only. BlockVault picks the wallet that holds enough USDC on a supported chain — the user will be prompted to approve the payment in the wallet modal. Default and recommended: **USDC on Base**.

## API base URL

Production: `https://x402.rye.com`

There is no staging URL on x402 — orders placed here are real and shipped by the merchant. Always confirm with the user before calling `/confirm`.

## The three curl templates

Use these verbatim. The placeholders are the only thing you change.

**1) Create a checkout intent (paid — $0.02):**

```bash
curl -sS -X POST "https://x402.rye.com/v1/checkout-intents" \
  -H "Content-Type: application/json" \
  -d '<JSON_BODY>'
```

`<JSON_BODY>` must contain at least `productUrl`, `quantity` and a complete `buyer` object:

```json
{
  "productUrl": "https://shop.example.com/running-shoes",
  "quantity": 1,
  "buyer": {
    "firstName": "Jane",
    "lastName": "Doe",
    "email": "jane@example.com",
    "phone": "+14155551234",
    "address1": "123 Market St",
    "address2": "",
    "city": "San Francisco",
    "province": "CA",
    "country": "US",
    "postalCode": "94103"
  }
}
```

The response is `{ "id": "<intent-id>", "state": "retrieving_offer" }`.

**2) Poll for the offer (free):**

```bash
curl -sS "https://x402.rye.com/v1/checkout-intents/<INTENT_ID>"
```

Repeat every ~3 s until `state` is `awaiting_confirmation` (offer ready) or `failed`. Hard cap: ~45 minutes per Rye's lifecycle docs, but in practice 5–20 s for retail merchants. Stop early on `failed`.

The successful response contains an `offer` object with `cost.total`, `cost.subtotal`, `cost.tax`, `cost.shipping` (each with `amount`, `currency`, and `amountSubunits` — subunits are integer cents/equivalent), plus shipping options and estimated delivery.

**3) Confirm the intent (paid — purchase total + $0.03):**

```bash
curl -sS -X POST "https://x402.rye.com/v1/checkout-intents/<INTENT_ID>/confirm" \
  -H "Content-Type: application/json" \
  -d '{"paymentMethod":{"type":"x402","network":"base"}}'
```

Use `"network":"base"`, `"solana"` or `"tempo"` to match the wallet that will sign. After this returns `200`, poll `GET /v1/checkout-intents/<INTENT_ID>` until `state` is `completed` (order placed, `orderId` returned) or `failed`. Order placement is async — typical 30 s to a few minutes.

## Checkout Intent state machine

| State | Meaning | Next action |
|---|---|---|
| `retrieving_offer` | Rye is pricing the URL with the merchant | Keep polling |
| `awaiting_confirmation` | Offer is ready (price + shipping + tax) | Show it to the user, confirm, then POST `/confirm` |
| `placing_order` | Payment received, merchant checkout running | Keep polling |
| `completed` | Order placed — `orderId` available | Show receipt to the user |
| `failed` | See `failureReason.message` | Report and stop |

If `state` becomes `failed` before `awaiting_confirmation`, the $0.02 access fee is **not** refunded. If it fails during order placement, the purchase total **is** refunded on-chain to the signing wallet.

## Purchase flow (default — "buy this for me")

How to take the user from a product URL to a completed order, step by step:

### 1. Resolve the product URL

If the user pasted a link, use it. Otherwise ask them for the **direct product page URL** from the merchant (not a search-results page, not a category page). Rye supports Shopify stores, Amazon, Best Buy, Walmart, and many DTC brands.

### 2. Collect buyer details

Ask once for the shipping address and contact info if you don't have them yet. Required fields: `firstName`, `lastName`, `email`, `phone`, `address1`, `city`, `province` (state code for US), `country` (ISO-2), `postalCode`. `address2` is optional.

Remind the user that this information is sent to the merchant for fulfilment.

### 3. Check wallet balance

Call `run_js` with:

- **function**: "get_assets"
- **data**: `{"hasBalance": true}`

Confirm the user has **at least $5 USDC on Base** as a buffer (covers $0.02 + $0.03 fees + a small-ticket purchase). For larger purchases verify USDC ≥ estimated total + $0.05. If insufficient, tell the user and stop.

### 4. Create the intent

`curl POST /v1/checkout-intents` with the JSON body from step 1+2. BlockVault will show the x402 payment modal for **$0.02 USDC on Base** — the user approves it once. Capture the returned `id`.

### 5. Poll for the offer

`curl GET /v1/checkout-intents/<id>` every ~3 s. Stop when `state` is `awaiting_confirmation` or `failed`. Do not poll more than ~20 times before reporting back to the user.

### 6. Show the offer and confirm with the user

Render the offer (see "Rendering" below). The user must explicitly say **yes / confirm** before you call `/confirm`. Never assume confirmation.

### 7. Confirm and pay

`curl POST /v1/checkout-intents/<id>/confirm` with `paymentMethod.network` matching the wallet's chain (`base` by default). BlockVault shows the x402 modal for **purchase total + $0.03** in a single signed transfer. The user approves it.

### 8. Poll until completed

`curl GET /v1/checkout-intents/<id>` every ~5 s until `state` is `completed` (capture `orderId`) or `failed`. Allow up to ~5 minutes.

### 9. Show receipt and save it

Render the final receipt. Then use the `text_editor` tool to create `receipts/rye-<INTENT_ID>.md` containing product, totals, `orderId`, the two on-chain tx hashes (BlockVault returns them after each x402 modal approval), and the merchant ETA.

In your chat reply, emit the receipt path in **backticks** (inline code) so the user can tap it to open the file (the app makes `receipts/…` paths clickable):

```text
Receipt saved to `receipts/rye-<INTENT_ID>.md`
```

## Browse / price-check flow ("how much would this cost me?")

If the user just wants a price quote without buying:

1. Run steps 1–5 of the purchase flow.
2. Render the offer with totals and shipping ETA.
3. Ask if they want to proceed to checkout. If they decline, stop — the intent simply expires after 45 minutes (no further charges).

This is useful for comparing two products: create two intents and present a side-by-side table.

## Rendering

The templates below are **Jinja2** — substitute `{{ var }}` with values from the API response. Drop `{# comments #}` from the final output.

**Offer (after step 5):**

```jinja
## {{ offer.product.title }}

{% if offer.product.image %}![]({{ offer.product.image }}){% endif %}

| | |
|---|---|
| Merchant | {{ offer.merchant.name or offer.merchant.domain }} |
| Quantity | {{ offer.quantity }} |
| Subtotal | {{ offer.cost.subtotal.amount }} {{ offer.cost.subtotal.currency }} |
| Shipping | {{ offer.cost.shipping.amount }} {{ offer.cost.shipping.currency }} ({{ offer.shipping.method or "standard" }}) |
| Tax | {{ offer.cost.tax.amount }} {{ offer.cost.tax.currency }} |
| **Total** | **{{ offer.cost.total.amount }} {{ offer.cost.total.currency }}** |
| Est. delivery | {{ offer.shipping.estimatedDelivery or "n/a" }} |
| Ships to | {{ buyer.city }}, {{ buyer.province }}, {{ buyer.country }} |

x402 fee: $0.03 USDC on top of the total. Confirm to place the order.
```

**Completed receipt (after step 8):**

```jinja
## Order placed — `{{ intent.orderId }}`

| | |
|---|---|
| Product | {{ intent.offer.product.title }} × {{ intent.offer.quantity }} |
| Merchant | {{ intent.offer.merchant.name }} |
| Total paid | {{ intent.offer.cost.total.amount }} {{ intent.offer.cost.total.currency }} |
| Network | USDC on {{ intent.paymentMethod.network }} |
| Ships to | {{ buyer.address1 }}, {{ buyer.city }} |
| ETA | {{ intent.offer.shipping.estimatedDelivery }} |
| Intent | `{{ intent.id }}` |

The merchant will email a tracking link to **{{ buyer.email }}**.
```

**Failure:**

```jinja
## Checkout failed

| | |
|---|---|
| State | {{ intent.state }} |
| Reason | {{ intent.failureReason.message }} |
| Code | `{{ intent.failureReason.code }}` |

{% if intent.failureReason.code == "out_of_stock" %}The item is unavailable right now. Try a different variant or merchant.{% endif %}
{% if intent.failureReason.code == "unsupported_merchant" %}Rye does not support this merchant yet. Try a different product URL.{% endif %}
```

## Failure modes and how to react

| Symptom | Cause | What to do |
|---|---|---|
| `state: "failed"` before `awaiting_confirmation` | Out of stock, unsupported merchant, bad URL | Report the reason. The $0.02 access fee is **not** refunded. |
| `state: "failed"` after `/confirm` | Merchant declined the order | Rye refunds the purchase amount on-chain automatically. Tell the user a refund is coming. |
| HTTP `400` with fresh `PAYMENT-REQUIRED` on retry | Signature expired between calls | BlockVault re-signs automatically — just retry the same curl. |
| HTTP `402` keeps returning | Wallet balance dropped below required total | Top up USDC and retry. Tell the user the exact shortfall. |
| Polling times out at `retrieving_offer` | Slow merchant | Tell the user the offer is still loading; offer to poll again later. |

## Constraints

- **Always pay in USDC on Base** unless the user explicitly asks for Solana or Tempo. Match `paymentMethod.network` to the wallet that will sign.
- **Never call `/confirm` without explicit user confirmation.** The purchase is real and the merchant ships it.
- **Never invent intent ids, order ids or offer fields.** Derive everything from the API responses.
- **Never expose Rye internal identifiers** (intent ids, raw tax codes) in chat unless the user asks — keep them for chaining calls.
- **There is no API key.** Do not invent `Authorization` headers or `X-API-KEY` headers — the wire authentication is the signed `PAYMENT-SIGNATURE`, handled by BlockVault.
- **One product URL per intent.** To buy two different products, create two intents.
- **Buyer address is final after `/confirm`.** Double-check the postal code and country before confirming.
- **Refunds are on-chain, not chat-driven.** If a checkout fails after `/confirm`, the refund lands in the signing wallet automatically — surface that, don't promise more.
