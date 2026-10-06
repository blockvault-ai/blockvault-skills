---
name: hotel-booking
description: Search hotels and book a room (with hotel stay) via the BlockVault travel API, charging USDC on-chain through x402. Use when the user asks to find a hotel, book a room, plan a stay, or reserve accommodation with check-in/check-out dates, a budget, or a city.
metadata:
  tool: bash
  category: travel
  enabled: true
  secrets:
    - key: DELEGATE_JWT
      description: BlockVault delegate API session token, issued via SIWE wallet sign-in.
      siwe:
        provider: delegate
        blockchain: ethereum
---

Today's date: {{DATE}}

# Instructions

Book a hotel stay end to end: collect trip details, search hotel offers (paginated), freeze a priced quote, then charge USDC on-chain via x402 and finalise the booking. Follow the steps carefully and use the provided tools for each part.

Do not ask the user for trip details directly. Spawn a sub-agent to collect them interactively.

Base URL: `https://402.blockvault.ai`

### Step 1: Gather stay details (sub-agent)

Call `spawn_subagents` with one task to collect the inputs interactively. The sub-agent asks one question at a time (never queue two), using the interactive tools.

Call `spawn_subagents` with:

- **tasks**: Array, Required. One element:
  - **id**: `"hotel_gather"`.
  - **objective**: `"Collect hotel stay details from the user: destination, check-in/check-out dates, number of rooms, adults and children per room, and budget and user data related to the step 2. Then resolve the destination to a city name + country code (or a hotel ID / IATA code) via web search."`.
  - **output_format**: `"Markdown list — one '- key: value' per field. Keys: destination, city_name, country_code, hotel_ids, iata_code, checkin (YYYY-MM-DD), checkout (YYYY-MM-DD), occupancies (JSON array of {adults, children:[ages]}), budget (number in USD or 'none'), currency."`.
  - **instructions**: Today's date is **{{DATE}}** — anchor every relative date ("tomorrow", "next weekend") against it and reject past dates. Ask one field at a time in order: destination → checkin → checkout → occupancies (rooms, adults, children ages) → budget. **Wait for each answer before the next.** Validate dates (YYYY-MM-DD, checkout after checkin). After `destination` is confirmed, use `web_search` (query like `"<city> country code ISO 3166"`, max_results 3) to resolve `city_name` + two-letter `country_code`. If the user gives a specific hotel, put its id in `hotel_ids` instead. Derive a reasonable currency (USD default).

The sub-agent must produce exactly one location method plus `checkin`, `checkout`, and `occupancies`. You may pass at most one of `city_name`+`country_code`, `iata_code`, or `hotel_ids`.

### Step 2: Search hotels (paginated)

Call `bash` with a curl to `/travel/search`. It requires `Authorization: Bearer {{DELEGATE_JWT}}`, a location method, `checkin`, `checkout`, and `occupancies`. Paginate with `offset`/`limit`; each response returns `has_more` and `total`.

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"checkin":"<YYYY-MM-DD>","checkout":"<YYYY-MM-DD>","currency":"<CUR>","guest_nationality":"<CC>","occupancies":[{"adults":<n>,"children":[]}],"city_name":"<City>","country_code":"<CC>","limit":50,"offset":0,"max_price":<budget>}'
```

- Use `hotel_ids` if the user named a specific hotel; otherwise `city_name` + `country_code`.
- Pass the user's `budget` as `max_price` (a number, no currency prefix) so the server filters out offers with a total price above the budget. Add `min_price` only if the user gave a price floor. **Omit both** when budget is `none`.
- The server filters server-side: offers without a resolvable price are dropped whenever `min_price`/`max_price` is set. Do not re-filter client-side.
- If `has_more` is true and the user wants more options, repeat with `"offset": <offset + limit>` (keep the same price bounds).
- The response returns `offers` (each with an `offerId`) and `hotels` (metadata). `raw_data` is omitted unless `"include_raw": true` — do not set it.
- Never auto-pick; let the user choose.

**Structure:** `offers[]` are the bookable results (each has `hotelId` and `roomTypes[]`); `hotels[]` is a separate metadata list keyed by `id`. Before rendering, build a lookup `hotel_by_id` mapping each hotel's `id` → its metadata object (for `name`, `main_photo`, `images`, `rating`, `stars`). Each offer's price is the **minimum** `roomTypes[].offerRetailRate.amount` (fall back to `suggestedSellingPrice.amount`); use that as the displayed "from" price.

**Hotel photos:** each hotel's metadata includes a `main_photo` URL. Render it inline (thumbnail ~120px wide) next to each hotel; guard with `{% if %}` so a missing image never breaks the render.

````jinja
## 🏨 {{ city_name }} — {{ checkin }} → {{ checkout }} · {{ occupancy_count }} room(s)

Here are the top options within your budget:

{% for o in offers[:10] %}
{% set hotel = hotel_by_id[o.hotelId] %}
### {{ loop.index }}. {{ hotel.name }}
{% if hotel.main_photo %}<img src="{{ hotel.main_photo }}" width="120" />{% endif %}

| | |
|---|---|
| Price | {{ o.from_price }} {{ currency }} |
{% if hotel.rating %}| Rating | {{ hotel.rating }} |{% endif %}
{% if hotel.stars %}| Stars | {{ hotel.stars }} |{% endif %}

{% endfor %}

Pick a number to see details or book.
````

After rendering, **ask the user** which offer they want (by number). Wait for their answer before freezing a quote. Never assume a choice.

If the search returns a 402 (no credits), stop and tell the user to top up via the credits purchase flow; retry the search after.

### Step 2.5: Show a hotel's photo gallery (optional, on request)

When the user wants to see more photos of a specific hotel, fetch its full gallery and render it as a swipeable carousel. Call `bash` to the hotel detail endpoint:

```bash
curl -s "https://402.blockvault.ai/api/v1/travel/hotels/<hotelId>" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

The response returns `images` (a list of `{url, url_hd, caption}`) and `rooms` (each with a `photos` list). Build a flat list of image URLs — prefer `url_hd` when present, fall back to `url` — and emit a ```carousel fenced block whose body is a JSON array of those URLs. The app renders it as a Swiper carousel natively.

````jinja
## 📸 {{ hotel.name }} — photo gallery

```carousel
{{ image_urls | tojson }}
```
````

- If `images` is empty, fall back to collecting `rooms[].photos[].url` (deduplicated).
- If there are still no images, tell the user no gallery is available and show `main_photo` inline instead.
- Never emit a ```carousel block with an empty array — the renderer drops it.

### Step 3: Freeze a quote (priced, no charge yet)

Picked offer N. Call `bash` to `/travel/account-book/quote` with the guest/holder details. The quote freezes the exact USDC amount before payment.

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/account-book/quote \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"offer_id":"<offerId>","holder":{"first_name":"<F>","last_name":"<L>","email":"<email>"},"guests":[{"occupancy_number":1,"first_name":"<F>","last_name":"<L>","email":"<email>"}]}'
```

Response returns `quote_id` and `amount` (USDC in atomic units, 6 decimals). Keep the `quote_id` in memory — do not write it to a state file.

### Step 4: Pay and confirm (x402, on-chain USDC)

Call `bash` to `/travel/account-book/confirm`. Sending a `quote_id` triggers the x402 middleware: BlockVault signs an EIP-3009 USDC transfer for the frozen amount, the user approves the payment modal, and the booking is finalised on-chain.

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/account-book/confirm \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"quote_id":"<quote_id>"}'
```

On success the response returns `booking_id`, `status` (`CONFIRMED`), the hotel `checkin`/`checkout`, and `price`. Notify the user with the booking summary (hotel, dates, amount paid in USDC, `booking_id`).

On 402 payment-required: the x402 flow was not completed — the user declined or lacked USDC. Do **not** retry confirm with the same quote; report the payment status and offer a fresh quote.

### Step 5: Show booking history (optional)

List the user's persisted bookings (paginated, 1-indexed `page`/`page_size`):

```bash
curl -s "https://402.blockvault.ai/api/v1/travel/bookings?page=1&page_size=20" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

## Constraints

- **No state file during search/quote.** Keep `offerId`, quotes, and tokens in memory; only the confirmed booking is persisted server-side.
- **Never invent dates or prices.** Ask if missing.
- **Pagination**: always respect `has_more`/`total`; never assume the first page is complete.
- **Payment requires explicit user intent** — confirm before calling `/account-book/confirm` (it charges real USDC).
- 401 → ask the user to re-authenticate. 402 on `/search` → credit top-up. 402 on `/confirm` → payment declined/incomplete.