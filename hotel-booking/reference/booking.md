# Hotel Booking — Rooms

## Contents
- Discover by place or coordinates
- Filter by amenities
- Compare prices
- Search rooms
- Pay with card (hosted payment page)
- Freeze a quote (USDT)
- Promo codes
- Add-ons
- Booking history
- Room fields

## Discover by place or coordinates

1. Resolve a landmark, district or address to a `place_id`:

  ```bash
  curl -sS "https://402.blockvault.ai/api/v1/travel/places?q=<url-encoded text>&language=<lang>" \
    -H "Authorization: Bearer {{DELEGATE_JWT}}"
  ```

  Returns `places[]` with `place_id`, `name`, `address`, `types`. If several match, ask the user which one (show `name` + `address`).

2. List hotels around it, nearest first:

  ```bash
  curl -sS "https://402.blockvault.ai/api/v1/travel/hotels?place_id=<place_id>&sort=distance&limit=50&offset=0" \
    -H "Authorization: Bearer {{DELEGATE_JWT}}" \
    -o <place>-<checkin>-<checkout>.json
  ```

  A bare `place_id` covers the whole place region (a landmark returns the whole city), so for "near X" always add `radius_km` (1–2 for walking distance). With coordinates instead (e.g. the user's location): `?latitude=<lat>&longitude=<lng>&radius_km=<km>&sort=distance`. Every hotel gets `distance_km`. Optional filters: `min_rating` (0–10), `star_rating=4&star_rating=5`, `hotel_name`.

## Priced area search

When the user gave dates and wants a budget, the cheapest option or a specific area, price all hotels in one call (no separate discovery or merge):

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/travel/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"checkin":"<YYYY-MM-DD>","checkout":"<YYYY-MM-DD>","currency":"USD","guest_nationality":"<CC>","occupancies":[{"adults":<n>,"children":[]}],"place_id":"<place_id>","radius_m":<metres>,"sort":"price_asc","max_price":<total budget>,"limit":50,"offset":0}' \
  -o <place>-<checkin>-<checkout>-priced.json
```

- Location: `"place_id"` + `"radius_m"` (1000–2000 for "near"), or `"latitude"`/`"longitude"` + `"radius_m"`, or `"city_name"` + `"country_code"`.
- `sort`: `price_asc` (cheapest first), `price_desc`, `distance` (nearest first).
- `max_price` is the total stay budget; omit when there is none. Add `facility_ids`, `board_types`, `min_rating` when the user asked.
- Empty `offers` come with `empty_reason`: `over_budget` (+ `cheapest_available`, the cheapest total found) or `no_availability`. Never remove the user's budget without asking.
- Render it with the same `hotel-booking:hotel-list` template (context `{"currency":"USD"}`); each card shows `from_price`, per-night price and `distance_km`. Keep `hotels[i].id` for the room search in the next step.

## Filter by amenities

```bash
curl -sS "https://402.blockvault.ai/api/v1/travel/facilities?lang=<lang>" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -o facilities-<lang>.json
```

Find the matching ids with `text_editor` `query` (e.g. `facilities[?contains(name, 'Pool')]`) — never print the whole catalogue. Pass them as `facility_ids=<id>&facility_ids=<id>` on `GET /travel/hotels`, or `"facility_ids":[<id>]` on `POST /travel/search`; add `strict_facilities=true` to require all of them.

## Compare prices

Price up to 200 discovered hotels in one call (cheapest bookable rate each):

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/travel/hotels/min-rates \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"checkin":"<YYYY-MM-DD>","checkout":"<YYYY-MM-DD>","currency":"<CUR>","guest_nationality":"<CC>","occupancies":[{"adults":<n>,"children":[]}],"hotel_ids":["<id>","<id>"],"max_price":<budget>,"sort":"price_asc"}'
```

Returns `rates[]` with `hotel_id`, `price`, `price_per_night`, `public_price` (public OTA price — mention the saving when `price` is lower) and a bookable `offer_id`. Omit `max_price` when there is no budget.

## Search rooms for a hotel
Once the user has chosen a hotel, search for its available rooms call:

  ```bash
  curl -sS -X POST https://402.blockvault.ai/api/v1/travel/search \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer {{DELEGATE_JWT}}" \
    -d '{"checkin":"<YYYY-MM-DD>","checkout":"<YYYY-MM-DD>","currency":"<CUR>","guest_nationality":"<CC>","occupancies":[{"adults":<n>,"children":[]}],"hotel_ids":["<hotelId>"],"limit":<limit>,"offset":0,"max_price":<budget>}' \
    -o <hotelId>-<checkin>-<checkout>.json
  ```
  
  - `hotel_ids` is a JSON array of hotel IDs (from the hotel list). **Never print them** — keep them in memory keyed by the hotel number shown to the user.
  - `limit` is the maximum number of rooms to return (default 100, max 1000). Paginate with `offset`/`limit`; each response returns `has_more` and `total`.
  - `max_price` is the user's budget in the chosen currency;
  - Optional: `"board_types":["BI"]` (BI breakfast, HB half board, FB full board, AI all inclusive, RO room only), `"refundable_rates_only":true`, `"sort":"price_asc"`.
  - Area search without picking a hotel first is also possible: replace `hotel_ids` with `"place_id":"<id>"` or `"latitude":<lat>,"longitude":<lng>,"radius_m":<m>` (+ `"sort":"distance"` or `"price_asc"`). Offers then carry `from_price`, `from_price_per_night`, `distance_km`, `has_perks` and `max_discount_percent` per hotel.

Render the result emitting:

````markdown
  ```artifact
  {"artifact":"<artifact_path>","template":"hotel-booking:room-list","context":{"currency":"<CUR>"}}
  ```
````

After rendering, **ask the user** which room they want (by number). Wait for their answer.

## Pay with card (hosted payment page)

Once the user picks a room, **ask to proceed with card payment**.

1. Create a checkout session with the chosen room's `offerId`:

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/travel/book \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"offer_id":"<room.offerId>"}'
```

Returns `prebook_id`, `transaction_id`, and `payment_url`.

2. Give the user the payment link as a clickable markdown link (the app opens
   external links in the system browser):

```markdown
[Complete payment](<payment_url>)
```

The user completes card payment in the browser; the hosted page then collects
the lead guest's name/email and finalises the booking automatically. Tell the
user to return to the chat once the page shows "Booking confirmed".

## Freeze a quote (priced, no charge yet)

1. Spawn the `hotel_freeze` sub-agent (see the `subagents` reference) to collect the holder and one lead guest per room (first name, last name, email). Do not ask for them in the main conversation and do not quote before they are known.

2. Call the `POST /travel/account-book/quote` endpoint with the chosen room's `offerId` and the guest's details:
  ```bash
  curl -sS -X POST https://402.blockvault.ai/api/v1/travel/account-book/quote \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer {{DELEGATE_JWT}}" \
    -d '{"offer_id":"<room.offerId>","holder":{"first_name":"<F>","last_name":"<L>","email":"<email>"},"guests":[{"occupancy_number":1,"first_name":"<F>","last_name":"<L>","email":"<email>"}]}'
  ```

`guests` holds one entry per room (`occupancy_number` 1..n, matching `occupancies`), not one per traveller. Add `"promo_code"` / `"addons"` only when the user asked for them.

Returns `quote_id` and `amount` (USDT in atomic units, 6 decimals), plus `sell_fiat` (USD to pay), and when applicable `original_sell_fiat`, `discount_fiat`, `promo_code`, `addons_total_fiat` and `addons`. Always show the user the final `sell_fiat` (and the saving) before confirming.

## Promo codes

Check a code before quoting (does not consume it):

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/travel/promo/validate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"code":"<CODE>","kind":"hotel","amount":<room price>}'
```

If `valid` is true, add `"promo_code":"<CODE>"` to the quote body. If false, tell the user the `reason`. The discount never takes the price below cost, so the quote may apply less than the preview.

## Add-ons

Optional extras bundled into the USDT quote (USD, **non-refundable**):

- **Uber ride credit** — `{"type":"uber","value":<10..100, step 10>}`.
- **eSIM data plan** — list plans for the destination country first:

  ```bash
  curl -sS "https://402.blockvault.ai/api/v1/travel/addons/esim/<CC>" \
    -H "Authorization: Bearer {{DELEGATE_JWT}}"
  ```

  Then `{"type":"esim","package_id":<id>,"country_code":"<CC>","start_date":"<checkin - 1 day>","end_date":"<checkout + 1 day>"}`. The price comes from the catalogue.

Add them to the quote body as `"addons":[...]`. After confirming, the response `addons[]` contains the Uber voucher link (`voucher_code`) and the eSIM QR (`esim[].qr_code`) — share them with the user.

## Booking history

```bash
curl -sS "https://402.blockvault.ai/api/v1/travel/bookings?page=<page>&page_size=<page_size>" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

`GET /travel/bookings/<booking_id>` returns the detail, including `extras.addons` (Uber link, eSIM QR) and `discount_fiat`.

## Room fields

Each room in `offers[0].rooms[]`: `offerId`, `price`, `price_per_night`, `original_price` + `discount_percent` (strikethrough price when the rate is discounted), `public_price`, `name`, `board`, `board_type`, `refundable`, `perks[]` (`name`, `amount` value) + `perks_value`, `promotions[]`, `photos`, `amenities`, `bedTypes`.

