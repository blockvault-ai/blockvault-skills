---
name: flight-booking
description: Search and book flights (airfare) via the BlockVault travel API (Nuitee/LiteAPI), charging USDC on-chain through x402. Use when the user asks to find flights, book airfare, or plan a trip that includes flights — origin, destination, dates, budget, or a specific airline route. For hotel stays and room reservations use hotel-booking instead.
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

# Flight Booking

Interactive flight assistant. Search flights, freeze a priced quote, and book airfare end to end — charging USDC on-chain via x402. Render results through `render/*.html` templates — never dump raw JSON.

Detect the language of the user's query and reply in that language.

## Instructions

- **Render, don't re-list.** After a search with `-o`, emit the ```artifact fence for the matching template. Do NOT re-list the results in Markdown — the card deck IS the list.
- **Always recommend.** After the ```artifact fence, add a one-line recommendation: **My pick:** <flight> — <reason tied to the user's needs>.
- **Ask when the intent is unclear.** If origin, destination, or dates are missing, ask ONE clarifying question before searching — do not guess.
- **Never auto-pick** — let the user choose a flight by number before quoting.
- **Confirm before paying** — `/flights/confirm` charges real USDC. Never call it without explicit user intent.

## Gathering trip details

Do not ask the user for trip details directly. Spawn a sub-agent to collect them interactively (one question at a time, never queue two).

Call `spawn_subagents` with:

- **tasks**: Array, Required. One element:
  - **id**: `"flight_gather"`.
  - **objective**: `"Collect the flight details from the user: origin, destination, trip type (round trip or one way), departure date, return date (round trips only), preferred departure time window for the outbound leg, preferred departure time window for the return leg (round trips only), number of adults, children, infants, max stops, budget, and currency. Then resolve IATA airport codes for both airports via web search."`.
  - **output_format**: `"Markdown list — one '- key: value' per field. Keys: trip_type (round|one_way), origin, destination, outbound_date (YYYY-MM-DD), return_date (YYYY-MM-DD or 'none'), outbound_departure_time_after (HH:MM or 'none'), outbound_departure_time_before (HH:MM or 'none'), return_departure_time_after (HH:MM or 'none'), return_departure_time_before (HH:MM or 'none'), adults, children, infants, stops (any|nonstop|1_stop|2_stops), budget (number or 'none'), currency, origin_iata, destination_iata."`.
  - **instructions**: Today's date is **{{DATE}}** — anchor every relative date ("tomorrow", "next Friday", "in 2 weeks") against it and reject past dates. Ask one field at a time in order: trip_type → origin → destination → outbound_date → (return_date if round trip) → outbound departure time window → (return departure time window if round trip) → adults → children → infants → max stops → budget → currency. **Wait for the user's answer before asking the next.** Default `stops` to `any` if cancelled. For each departure time window, ask "what time of day do you want to depart?" and accept either a free HH:MM range ("between 06:00 and 14:00") or a single bound ("after 18:00", "before 12:00"); set the missing bound to `none`. Default both bounds to `none` (any time) if the user has no preference. Validate time format (HH:MM, 24h) and date format (YYYY-MM-DD, ≥ today) and currency code. After `origin` is confirmed, `web_search` with `query: "IATA code <origin> airport"` and `max_results: 3` to extract the 3-letter primary code; if multiple major airports exist, ask the user to pick. Repeat for `destination`. Never invent IATA codes.

When the sub-agent completes, use the returned values for all remaining steps.

## API

Base URL: `https://402.blockvault.ai`. The secret placeholder `{{DELEGATE_JWT}}` is resolved automatically — write it verbatim in the `Authorization` header, never echo it back to the user.

### Search flights

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"legs":[{"origin":"<ORIGIN_IATA>","destination":"<DEST_IATA>","date":"<YYYY-MM-DD>","direction":"OUTBOUND","departure_time_after":"<HH:MM>","departure_time_before":"<HH:MM>"}],"adults":<n>,"children":<n>,"infants":<n>,"currency":"<CUR>","max_stops":<stops>,"max_price":<budget>,"limit":50,"offset":0}' \
  -o <origin>-<destination>-<outbound_date>.json
```

- **Round trip:** two legs — outbound (`direction: "OUTBOUND"`) and return (`direction: "INBOUND"`, origin/destination swapped, `date` = return date). **One way:** a single leg.
- **Departure time window (per leg):** pass `departure_time_after` and/or `departure_time_before` (HH:MM, 24h) on the leg to constrain its departure time. **Omit** a bound when it is `none` — never send an empty string. The API maps these to Nuitee `legs[].filters.departureTimeAfter/Before` server-side.
- Map `stops` to `max_stops`: `nonstop` → `0`, `1_stop` → `1`, `2_stops` → `2`. **Omit** `max_stops` when `any`.
- Pass `budget` as `max_price` (number, no currency prefix). **Omit** `max_price` when `none`. The API filters journeys to the budget server-side.
- **Paginate** with `limit`/`offset`; the response returns `total` and `has_more`. Never assume the first page is complete — respect `has_more` and fetch more pages when the user wants more options.

### Freeze a quote (priced, no charge yet)

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/quote \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"offer_id":"<offerId>","contact":{"first_name":"<F>","last_name":"<L>","email":"<email>","phone_number":"<phone>"},"passengers":[{"first_name":"<F>","last_name":"<L>","birthday":"<YYYY-MM-DD>","gender":"<M|F>","nationality":"<CC>","passenger_type":0}]}'
```

- `passengers[]` must match the `adults`/`children`/`infants` counts from the search. `passenger_type`: 0=adult, 1=child, 2=infant.
- Keep the returned `quote_id` in memory — do not write it to a state file.

### Pay and confirm (x402, on-chain USDC)

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/confirm \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"quote_id":"<quote_id>"}'
```

On success, notify the user with the booking summary (route, dates, amount paid in USDC, `booking_id`).

### Booking history

```bash
curl -s "https://402.blockvault.ai/api/v1/travel/bookings?page=1&page_size=20" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

## Rendering

**Flight list** (search results):

Call the search endpoint with `-o <name>.json`. The `bash` result includes `artifact_path`. **Do NOT read the JSON.** Emit an ```artifact fence with template `flight-booking:flight-list` (renders `data.journeys` as a swipeable card deck, cheapest offer per journey, with search/sort/filter toolbar).

````markdown
```artifact
{"artifact":"<artifact_path>","template":"flight-booking:flight-list","context":{"origin":"<ORIGIN>","destination":"<DEST>","outbound_date":"<YYYY-MM-DD>"}}
```
````

After rendering, add a one-line recommendation, then ask the user which flight they want (by number).

**Never print `offerId` values.** They are long opaque tokens — keep them in memory, keyed by the journey number you showed the user. When the user picks "flight 3", use `journeys[2].offers[0].offerId`.

## Follow-up questions

When the user asks a follow-up about the rendered results ("which flight is nonstop?", "show only morning departures"), do **not** re-read the whole artifact. Spawn a sub-agent to answer from the artifact:

Call `spawn_subagents` with one task:

- **tasks**: Array, Required. One element:
  - **id**: `"flight_filter"`.
  - **objective**: `"Answer this question from the artifact: <user question>. The artifact is at <artifact_path>."`.
  - **instructions**: Use `text_editor` with `command: "query"` (JMESPath) or `command: "search"` to read only the relevant slice of the artifact — never load the whole file. Return a concise markdown answer.
  - **output_format**: `"Markdown — a short answer plus the matching journey/offer names."`.

**Time-of-day follow-ups re-search, not filter.** When the user asks to narrow by departure time ("only mornings", "after 6pm", "between 10:00 and 14:00"), the artifact only holds the journeys already downloaded — filtering it would hide valid options. Instead, re-run the search with the same legs plus the requested `departure_time_after`/`departure_time_before` on the relevant leg(s), then render the new artifact. Only use the `flight_filter` sub-agent for questions answerable from the already-downloaded results (stops, duration, price, carrier, baggage).

## Constraints

- **No state file during search/quote.** Keep `offerId`, quotes, and tokens in memory; only the confirmed booking is persisted server-side.
- **Never invent dates or prices.** Ask if missing.
- **Payment requires explicit user intent** — confirm before calling `/flights/confirm` (it charges real USDC).
- 401 → ask the user to re-authenticate. 402 on `/search` → credit top-up. 402 on `/confirm` → payment declined/incomplete.
