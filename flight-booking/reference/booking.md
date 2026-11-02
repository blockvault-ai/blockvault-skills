# Flight Booking — Booking

## Contents
- Search flights
- Filters and sorting
- Pay with card (hosted payment page)
- Freeze a quote (USDT)
- Seats and extra bags
- Promo codes
- Booking history
- Journey fields

## Search flights


```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"legs":[{"origin":"<ORIGIN_IATA>","destination":"<DEST_IATA>","date":"<YYYY-MM-DD>","direction":"OUTBOUND","departure_time_after":"<HH:MM>","departure_time_before":"<HH:MM>","arrival_time_after":"<HH:MM>","arrival_time_before":"<HH:MM>"}],"adults":<n>,"children":<n>,"infants":<n>,"currency":"<CUR>","max_stops":<stops>,"max_price":<budget>,"limit":50,"offset":0}' \
  -o <origin>-<destination>-<outbound_date>.json
```

- **Round trip:** two legs — outbound (`direction: "OUTBOUND"`) and return (`direction: "INBOUND"`, origin/destination swapped, `date` = return date). **One way:** a single leg.
- **Departure time window (per leg):** pass `departure_time_after` and/or `departure_time_before` (HH:MM, 24h) on the leg to constrain its departure time. **Omit** a bound when it is `none` — never send an empty string. The API maps these to Nuitee `legs[].filters.departureTimeAfter/Before` server-side.
- **Arrival time window (per leg):** pass `arrival_time_after` and/or `arrival_time_before` (HH:MM, 24h) on the leg to constrain its arrival time. **Omit** a bound when it is `none`. The API maps these to Nuitee `legs[].filters.arrivalTimeAfter/Before` server-side.
- Map `stops` to `max_stops`: `nonstop` → `0`, `1_stop` → `1`, `2_stops` → `2`. **Omit** `max_stops` when `any`.
- Pass `budget` as `max_price` (number, no currency prefix). **Omit** `max_price` when `none`. The API filters journeys to the budget server-side.
- **Paginate** with `limit`/`offset`; the response returns `total` and `has_more`. Never assume the first page is complete — respect `has_more` and fetch more pages when the user wants more options.

## Filters and sorting

Add only what the user asked for:

| User intent | Body field |
|---|---|
| cheapest / fastest / earliest | `"sort_by":"price"` \| `"duration"` \| `"departure"` \| `"arrival"` \| `"stops"`, `"sort_order":"asc"` |
| price range | `"min_price":<n>`, `"max_price":<n>` |
| checked bag / carry-on included | `"includes_checked_bag":true`, `"includes_carry_on_bag":true` |
| flexible ticket | `"changeable_only":true`, `"refundable_only":true` |
| avoid an airline | `"exclude_airlines":["<IATA 2-letter>"]` |
| max travel time | `"max_duration_minutes":<n>` |

The response `highlights` gives the `journeyKey` of the `cheapest`, `fastest` and `fewest_stops` journeys across all results — use it for the recommendation.

Render the result emitting:

````markdown
```artifact
{"artifact":"<artifact_path>","template":"flight-booking:flight-list","context":{"origin":"<ORIGIN>","destination":"<DEST>","outbound_date":"<YYYY-MM-DD>"}}
```
````

After rendering, add a one-line recommendation (**My pick:** <flight> — <reason tied to the user's needs>), then ask the user which flight they want (by number). Do not re-list the flights in Markdown — the card deck is the list.

**Never print `offerId` values.** They are long opaque tokens — keep them in memory, keyed by the journey number you showed the user. When the user picks "flight 3", use `journeys[2].offers[0].offerId`.


## Pay with card (hosted payment page)

Once the user picks a flight, **ask to proceed with card payment**.

1. Spawn the `flight_freeze` sub-agent to collect the contact and passenger
   details

2. Create a checkout session with the chosen journey's `offerId` and the
   collected contact + passenger details:
 
```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/book \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"offer_id":"<offerId>","contact":{"first_name":"<F>","last_name":"<L>","email":"<email>","phone_number":"<phone>","phone_country_code":"<CC>"},"passengers":[{"first_name":"<F>","last_name":"<L>","birthday":"<YYYY-MM-DD>","gender":"<M|F>","nationality":"<CC>","passenger_type":0}]}'
```

Returns `prebook_id`, `transaction_id`, and `payment_url`.

3. Give the user the payment link as a clickable markdown link (the app opens
   external links in the system browser):

```markdown
[Complete payment](<payment_url>)
```

The user completes card payment in the browser; the hosted page finalises the
booking automatically. Tell the user to return to the chat once the page shows
"Flight confirmed".

## Freeze a quote (priced, no charge yet)

1. Spawn the `flight_freeze` sub-agent to collect the contact and passenger details:
  - contact: first name, last name, email, phone number
  - passengers: one entry per traveller (first name, last name, birthday, gender, nationality, passenger type)

2. Call the `POST /travel/flights/quote` endpoint with the chosen journey's `offerId` and the collected details:

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/quote \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"offer_id":"<offerId>","contact":{"first_name":"<F>","last_name":"<L>","email":"<email>","phone_number":"<phone>"},"passengers":[{"first_name":"<F>","last_name":"<L>","birthday":"<YYYY-MM-DD>","gender":"<M|F>","nationality":"<CC>","passenger_type":0}]}'
```

- `passengers[]` must match the `adults`/`children`/`infants` counts from the search. `passenger_type`: 0=adult, 1=child, 2=infant.
- Keep the returned `quote_id` in memory — do not write it to a state file.
- Search with `currency: "USD"` when the user pays in USDT (other currencies return 422).
- Optional `"promo_code":"<CODE>"` (see "Promo codes").
- When the response has `services_available: true`, offer seats and extra bags before confirming.

## Seats and extra bags

1. List what can be added (filter with `category=seat` or `category=baggage`, and `segment_key` per flight):

```bash
curl -s "https://402.blockvault.ai/api/v1/travel/flights/quote/<quote_id>/services?category=baggage" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -o services-<quote_id>.json
```

   Each service has `service_id`, `name`, `price`, `currency`, `segment_key` and `seat` (`number`, `position`, `type`) or `baggage` (`bag_type`, `weight_kg`). Read it with `text_editor` `query`; show only what the user asked for (e.g. window seats, one checked bag).

2. Apply the full selection (it **replaces** the previous one; `[]` clears it):

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/quote/<quote_id>/services \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"selected":[{"passenger_index":0,"service_id":"<service_id>","quantity":1}]}'
```

   Returns the new `amount`/`sell_fiat` and `services_fiat`; confirm charges the new amount. 409 → seat just taken, pick another. 410 → the fare expired, search again.

## Promo codes

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/promo/validate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"code":"<CODE>","kind":"flight","amount":<fare>}'
```

If `valid`, pass `"promo_code"` on the quote; the response shows `original_sell_fiat`, `discount_fiat` and the final `sell_fiat`. 409 on confirm → the code ran out; quote again without it.


## Booking history

```bash
curl -s "https://402.blockvault.ai/api/v1/travel/bookings?page=<PAGE>&page_size=<PAGE_SIZE>" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

## Journey fields

The search response returns `journeys[]` (each compacted to the fields the card deck renders) plus `airlines[]` (distinct marketing carriers on the page):

- `journeyKey` — opaque journey identifier.
- `segments[]` — one per leg: `originCode`, `destinationCode`, `departureTime`, `arrivalTime`, `direction`, `duration.minutes`, `carrier.marketingCode`, `carrier.marketingName`, `carrier.marketingLogo`, `carrier.operatingCode`.
- `offers[]` — bookable fares, cheapest first: `offerId`, `pricing.display.total`, `pricing.display.currency`, `terms.refundable`, `terms.changeable`, `fare.family`, `fare.seatsRemaining`, `baggage`.
- `highlights` — `journeyKey` of the `cheapest`, `fastest` and `fewest_stops` journeys.
- `totalDuration.minutes` — full journey duration.
- `stops` — total stops (connections + technical stops).
- `airlines[]` — distinct `{code, name}` pairs across the page, for the deck's airline dropdown.

The card deck renders `data.journeys` with the cheapest offer per journey (`offers[0]`).