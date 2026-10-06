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

# Hotel Booking

Help the user find and choose the best hotel stay, then book it end to end: search offers (paginated), freeze a priced quote, and charge USDC on-chain via x402 to finalise the booking.

## Instructions

- **Help the user choose.** Compare options, explain tradeoffs (price, rating, location, refundability), and give a clear recommendation — don't just dump a list.
- **Use `bash` for API calls.** Execute `curl` commands to interact with the travel API — never invent responses.
- Detect the user's language and reply in that language.
- Render results as Markdown tables, never raw JSON.
- **Never auto-pick** — let the user choose, but always recommend the best fit for their stated needs.

Base URL: `https://402.blockvault.ai`

## API endpoints

The secret placeholder `{{DELEGATE_JWT}}` is resolved automatically — write it verbatim in the `Authorization` header. Never echo it back to the user.

### Discover hotels (lightweight, no rates)

```bash
curl -sS "https://402.blockvault.ai/api/v1/travel/hotels?city_name=<City>&country_code=<CC>&limit=50&offset=0" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

### Search rooms for a hotel

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/travel/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"checkin":"<YYYY-MM-DD>","checkout":"<YYYY-MM-DD>","currency":"<CUR>","guest_nationality":"<CC>","occupancies":[{"adults":<n>,"children":[]}],"hotel_ids":["<hotelId>"],"limit":100,"offset":0,"max_price":<budget>}'
```

### Hotel photo gallery

```bash
curl -sS "https://402.blockvault.ai/api/v1/travel/hotels/<hotelId>" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

### Freeze a quote (priced, no charge yet)

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/travel/account-book/quote \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"offer_id":"<room.offerId>","holder":{"first_name":"<F>","last_name":"<L>","email":"<email>"},"guests":[{"occupancy_number":1,"first_name":"<F>","last_name":"<L>","email":"<email>"}]}'
```

### Pay and confirm (x402, on-chain USDC)

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/travel/account-book/confirm \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"quote_id":"<quote_id>"}'
```

### Booking history

```bash
curl -sS "https://402.blockvault.ai/api/v1/travel/bookings?page=1&page_size=20" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

## Step 1: Gather stay details (sub-agent)

Do not ask the user for trip details directly. Spawn a sub-agent to collect them interactively — but **write the `objective` yourself, from the user's message**, so the sub-agent never re-asks for details the user already gave.

Call `spawn_subagents` with one task. The sub-agent asks one question at a time (never queue two), using the interactive tools.

- **tasks**: Array, Required. One element:
  - **id**: `"hotel_gather"`.
  - **objective**: Compose this from the user's intent. Include every detail they already provided (destination, check-in/check-out dates, rooms, adults/children, budget) so the sub-agent only asks for what's missing. Example — user said "hotel in Madrid, 3 nights from July 1, 2 adults, under $200/night": `"Collect the remaining hotel stay details. The user already gave: destination Madrid, check-in 2026-07-01, 2 adults, budget $200/night. Ask only for what's missing: checkout date, number of rooms, children ages. Then resolve Madrid to city_name + country_code via web search."`.
  - **output_format**: `"Markdown list — one '- key: value' per field. Keys: destination, city_name, country_code, hotel_ids, iata_code, checkin (YYYY-MM-DD), checkout (YYYY-MM-DD), occupancies (JSON array of {adults, children:[ages]}), budget (number in USD or 'none'), currency."`.
  - **instructions**: Today's date is **{{DATE}}** — anchor every relative date ("tomorrow", "next weekend") against it and reject past dates. Ask one field at a time in order: destination → checkin → checkout → occupancies (rooms, adults, children ages) → budget. **Wait for each answer before the next.** Skip any field the user already provided. Validate dates (YYYY-MM-DD, checkout after checkin). After `destination` is confirmed, use `web_search` (query like `"<city> country code ISO 3166"`, max_results 3) to resolve `city_name` + two-letter `country_code`. If the user gives a specific hotel, put its id in `hotel_ids` instead. Derive a reasonable currency (USD default).

The sub-agent must produce exactly one location method plus `checkin`, `checkout`, and `occupancies`. You may pass at most one of `city_name`+`country_code`, `iata_code`, or `hotel_ids`.

## Step 2: Discover hotels (lightweight)

Call `bash` with `GET /travel/hotels`. It returns a compact hotel list (no rates) — `hotels[]` with `id`, `name`, `rating`, `stars`, `address`, `city`, `country`, `latitude`, `longitude`, `distance`, `main_photo`. Paginate with `offset`/`limit`; each response returns `has_more` and `total`.

- Search by city (`city_name` + `country_code`) or geolocation (`latitude` + `longitude` + `distance`).
- This endpoint never returns room rates — it is cheap and token-light. Use it to browse hotels, then fetch rooms for the chosen hotel.

## Rendering — Step A: hotels only

Render the hotel list. Show one line per hotel (name, photo, rating, stars, distance). No rooms here.

````jinja
## 🏨 {{ city_name }} — {{ checkin }} → {{ checkout }} · {{ occupancy_count }} room(s)

Here are the top options:

{% for h in hotels[:10] %}
### {{ loop.index }}. {{ h.name }}
{% if h.main_photo %}<img src="{{ h.main_photo }}" width="120" />{% endif %}

{% if h.rating %}Rating: {{ h.rating }}{% endif %}
{% if h.stars %} · {{ h.stars }} stars{% endif %}
{% if h.distance %} · {{ h.distance }} km away{% endif %}

{% endfor %}

**My pick:** <hotel name> — <one-line reason tied to the user's needs>.

Pick a hotel number to see its rooms.
````

After rendering, **ask the user** which hotel they want (by number). Wait for their answer before showing rooms.

## Step 2.1: Fetch the chosen hotel's rooms (hotel-scoped search)

When the user picks a hotel, search its rooms with `POST /travel/search` scoped to that hotel (`hotel_ids`):

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/travel/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"checkin":"<YYYY-MM-DD>","checkout":"<YYYY-MM-DD>","currency":"<CUR>","guest_nationality":"<CC>","occupancies":[{"adults":<n>,"children":[]}],"hotel_ids":["<hotelId>"],"limit":100,"offset":0}'
```

The response returns `offers[]` (one per hotel) with `rooms[]` — each room has a short `offerId`, `price`, `name`, `board`, `refundable`, `bedType`, `maxOccupancy`, `photos`, and `amenities`.

Then compute the hotel's price range from the response: `min = min(r.price for r in rooms)`, `max = max(r.price for r in rooms)`. **Ask the user for a min and max within those bounds** before rendering — a hotel can return hundreds of room rates. Ask one question: "Rooms at <hotel> range from $<min> to $<max>. What's your price range? (e.g. $100–$200, or 'all')".

Then filter `rooms` to the range and **cap the list** — never render more than ~15 rooms. If more match, show the cheapest 15 and say so.

````jinja
## 🏨 {{ hotel.name }} — rooms ({{ min_price }}–{{ max_price }} {{ currency }})

{% for r in rooms[:15] %}
- **{{ loop.index }}. {{ r.name or 'Room' }}** — {{ r.price }} {{ currency }}{% if r.board %} · {{ r.board }}{% endif %}{% if r.refundable %} · refundable{% endif %}
{% endfor %}

{% if rooms | length > 15 %}…and {{ rooms | length - 15 }} more — narrow the price range to see them.{% endif %}

Pick a room number to see details or book.
````

- Filter `rooms` client-side by the user's min/max before rendering (the server's `min_price`/`max_price` filter hotels, not individual rooms).
- If the range yields nothing, widen it or show the nearest 3 rooms and say so.
- After rendering, **ask the user** which room they want (by number). Wait for their answer.

## Step 2.2: Show room details (photos + what's included)

When the user picks a room, show its details before booking: photos and amenities. Each room carries `photos` (list of `{url, urlHd}`) and `amenities` (list of strings), plus `bedType` and `maxOccupancy` when present.

````jinja
## 🏨 {{ hotel.name }} — {{ room.name }}

{% if room.photos %}<img src="{{ room.photos[0].urlHd or room.photos[0].url }}" width="240" />{% endif %}

| | |
|---|---|
| Price | {{ room.price }} {{ currency }} |
{% if room.board %}| Board | {{ room.board }} |{% endif %}
{% if room.bedType %}| Bed | {{ room.bedType }} |{% endif %}
{% if room.maxOccupancy %}| Sleeps | {{ room.maxOccupancy }} |{% endif %}
{% if room.refundable %}| Refundable | Yes |{% endif %}

{% if room.amenities %}**Included:** {{ room.amenities | join(', ') }}{% endif %}

Book this room?
````

- If `photos` is empty, fall back to the hotel's `main_photo`.
- If `amenities` is empty, omit the line.
- Confirm before freezing a quote.

**Never print `offerId` values.** They are short opaque tokens — keep them in memory, keyed by the hotel/room number you showed the user. When the user picks "hotel 2, room 1", look up that room's `offerId` from the search response and use it in the quote call.

If the search returns a 402 (no credits), stop and tell the user to top up via the credits purchase flow; retry the search after.

## Step 2.5: Show a hotel's photo gallery (optional, on request)

When the user wants to see more photos of a specific hotel, fetch its full gallery and render it as a swipeable carousel. Call `bash` to the hotel detail endpoint.

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

## Step 3: Freeze a quote (priced, no charge yet)

The user picked a hotel and a specific room. Use that room's `offerId` (from `offers[].rooms[].offerId`) as the `offer_id`. Call `bash` to `/travel/account-book/quote` with the guest/holder details. The quote freezes the exact USDC amount before payment.

Response returns `quote_id` and `amount` (USDC in atomic units, 6 decimals). Keep the `quote_id` in memory — do not write it to a state file.

## Step 4: Pay and confirm (x402, on-chain USDC)

Call `bash` to `/travel/account-book/confirm`. Sending a `quote_id` triggers the x402 middleware: BlockVault signs an EIP-3009 USDC transfer for the frozen amount, the user approves the payment modal, and the booking is finalised on-chain.

On success the response returns `booking_id`, `status` (`CONFIRMED`), the hotel `checkin`/`checkout`, and `price`. Notify the user with the booking summary (hotel, dates, amount paid in USDC, `booking_id`).

On 402 payment-required: the x402 flow was not completed — the user declined or lacked USDC. Do **not** retry confirm with the same quote; report the payment status and offer a fresh quote.

## Step 5: Show booking history (optional)

List the user's persisted bookings (paginated, 1-indexed `page`/`page_size`) via the booking history endpoint.

## Constraints

- **No state file during search/quote.** Keep `offerId`, quotes, and tokens in memory; only the confirmed booking is persisted server-side.
- **Never invent dates or prices.** Ask if missing.
- **Pagination**: always respect `has_more`/`total`; never assume the first page is complete.
- **Payment requires explicit user intent** — confirm before calling `/account-book/confirm` (it charges real USDC).
- 401 → ask the user to re-authenticate. 402 on `/search` → credit top-up. 402 on `/confirm` → payment declined/incomplete.