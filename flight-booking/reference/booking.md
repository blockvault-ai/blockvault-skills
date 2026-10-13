# Flight Booking — Booking

## Contents
- Search flights
- Freeze a quote
- Pay and confirm
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

Render the result emitting:

````markdown
```artifact
{"artifact":"<artifact_path>","template":"flight-booking:flight-list","context":{"origin":"<ORIGIN>","destination":"<DEST>","outbound_date":"<YYYY-MM-DD>"}}
```
````

After rendering, add a one-line recommendation (**My pick:** <flight> — <reason tied to the user's needs>), then ask the user which flight they want (by number). Do not re-list the flights in Markdown — the card deck is the list.

**Never print `offerId` values.** They are long opaque tokens — keep them in memory, keyed by the journey number you showed the user. When the user picks "flight 3", use `journeys[2].offers[0].offerId`.

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

## Pay and confirm (x402, on-chain USDC)

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/confirm \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"quote_id":"<quote_id>"}'
```

On success, notify the user with the booking summary (route, dates, amount paid in USDC, `booking_id`).

## Booking history

```bash
curl -s "https://402.blockvault.ai/api/v1/travel/bookings?page=1&page_size=20" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

## Journey fields

The search response returns `journeys[]` (each compacted to the fields the card deck renders) plus `airlines[]` (distinct marketing carriers on the page):

- `journeyKey` — opaque journey identifier.
- `segments[]` — one per leg: `originCode`, `destinationCode`, `departureTime`, `arrivalTime`, `direction`, `duration.minutes`, `carrier.marketingCode`, `carrier.marketingName`, `carrier.marketingLogo`, `carrier.operatingCode`.
- `offers[]` — bookable fares, cheapest first: `offerId`, `pricing.display.total`, `pricing.display.currency`, `terms.refundable`, `fare.family`, `baggage`.
- `totalDuration.minutes` — full journey duration.
- `stops` — total stops (connections + technical stops).
- `airlines[]` — distinct `{code, name}` pairs across the page, for the deck's airline dropdown.

The card deck renders `data.journeys` with the cheapest offer per journey (`offers[0]`).