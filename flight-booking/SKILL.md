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

# Instructions

Book a flight end to end: collect trip details, search flight offers (legs-based), freeze a priced quote, then charge USDC on-chain via x402 and finalise the booking. Follow the steps carefully and use the provided tools for each part.

Do not ask the user for trip details directly. Spawn a sub-agent to collect them interactively.

Base URL: `https://402.blockvault.ai`

### Step 1: Gather flight details (sub-agent)

Call `spawn_subagents` with one task to collect the inputs interactively. The sub-agent asks one question at a time (never queue two), using the interactive tools.

Call `spawn_subagents` with:

- **tasks**: Array, Required. One element:
  - **id**: `"flight_gather"`.
  - **objective**: `"Collect the flight details from the user: origin, destination, trip type (round trip or one way), departure date, return date (round trips only), number of adults, max stops, budget, and currency. Then resolve IATA airport codes for both airports via web search."`.
  - **output_format**: `"Markdown list — one '- key: value' per field. Keys: trip_type (round|one_way), origin, destination, outbound_date (YYYY-MM-DD), return_date (YYYY-MM-DD or 'none'), adults, stops (0=Any|1=Nonstop only|2=1 stop or fewer), budget (number or 'none'), currency, origin_iata, destination_iata."`.
  - **instructions**: Today's date is **{{DATE}}** — anchor every relative date ("tomorrow", "next Friday", "in 2 weeks") against it and reject past dates. Ask one field at a time in order: trip_type → origin → destination → outbound_date → (return_date if round trip) → adults → max stops → budget → currency. **Wait for the user's answer before asking the next.** Default `stops` to `0` (Any) if cancelled. Validate date format (YYYY-MM-DD, ≥ today) and currency code. After `origin` is confirmed, `web_search` with `query: "IATA code <origin> airport"` and `max_results: 3` to extract the 3-letter primary code; if multiple major airports exist, ask the user to pick. Repeat for `destination`. Never invent IATA codes.

When the sub-agent completes, use the returned values for all remaining steps.

### Step 2: Search flights (legs-based)

Call `bash` with a curl to `/travel/flights/search`. It requires `Authorization: Bearer {{DELEGATE_JWT}}`, a `legs` array, `adults`, and `currency`.

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"legs":[{"origin":"<ORIGIN_IATA>","destination":"<DEST_IATA>","date":"<YYYY-MM-DD>","direction":"OUTBOUND"}],"adults":<n>,"currency":"<CUR>","max_stops":<stops>,"max_price":<budget>}' \
  -o <origin>-<destination>-<outbound_date>.json
```

- **Round trip:** send two legs — outbound (`direction: "OUTBOUND"`) and return (`direction: "INBOUND"`, origin/destination swapped, `date` = return date).
- **One way:** a single leg.
- Pass the user's `budget` as `max_price` (a number, no currency prefix). **Omit** `max_price` when budget is `none`. Omit `max_stops` when it is `0`.
- The response returns `journeys[]` (each with `segments[]` and `offers[]`). Each offer carries an `offerId` and `pricing.display.total` (the full price). Render the cheapest offer per journey.
- Never auto-pick; let the user choose.

**Structure:** `journeys[]` are the bookable results. Each journey has `segments[]` (with `originCode`, `destinationCode`, `departureTime`, `arrivalTime`, `carrier.marketingName`, `carrier.marketingLogo`, `duration.minutes`) and `offers[]` (each with `offerId`, `pricing.display.total`, `pricing.display.currency`, `terms.refundable`, `baggage`). Use the **cheapest** offer's `pricing.display.total` as the displayed price.

**Airline logos:** each segment's `carrier.marketingLogo` is a URL — the template renders it inline (≤ 40px) next to the airline name.

The `bash` result includes `artifact_path` (from `-o`). **Do NOT read the JSON.** Emit an ```artifact fence with template `flight-booking:flight-list` (renders `data.journeys` as a swipeable card deck, cheapest offer per journey).

````markdown
```artifact
{"artifact":"<artifact_path>","template":"flight-booking:flight-list","context":{"origin":"<ORIGIN>","destination":"<DEST>","outbound_date":"<YYYY-MM-DD>"}}
```
````

After rendering, **ask the user** which offer they want (by number). Wait for their answer before freezing a quote. Never assume a choice.

If the search returns a 402 (no credits), stop and tell the user to top up via the credits purchase flow; retry the search after.

### Step 3: Freeze a quote (priced, no charge yet)

Picked offer N. Call `bash` to `/travel/flights/quote` with the contact and passenger details. The quote freezes the exact USDC amount before payment.

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/quote \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"offer_id":"<offerId>","contact":{"first_name":"<F>","last_name":"<L>","email":"<email>","phone_number":"<phone>"},"passengers":[{"first_name":"<F>","last_name":"<L>","birthday":"<YYYY-MM-DD>","gender":"<M|F>","nationality":"<CC>","passenger_type":0}]}'
```

- `passengers[]` must match the `adults`/`children`/`infants` counts from the search. `passenger_type`: 0=adult, 1=child, 2=infant.
- Response returns `quote_id` and `amount` (USDC in atomic units, 6 decimals). Keep the `quote_id` in memory — do not write it to a state file.

### Step 4: Pay and confirm (x402, on-chain USDC)

Call `bash` to `/travel/flights/confirm`. Sending a `quote_id` triggers the x402 middleware: BlockVault signs an EIP-3009 USDC transfer for the frozen amount, the user approves the payment modal, and the booking is finalised on-chain.

```bash
curl -s -X POST https://402.blockvault.ai/api/v1/travel/flights/confirm \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"quote_id":"<quote_id>"}'
```

On success the response returns `booking_id`, `status` (`CONFIRMED`), and the itinerary summary. Notify the user with the booking summary (route, dates, amount paid in USDC, `booking_id`).

On 402 payment-required: the x402 flow was not completed — the user declined or lacked USDC. Do **not** retry confirm with the same quote; report the payment status and offer a fresh quote.

### Step 5: Show booking history (optional)

List the user's persisted bookings (paginated, 1-indexed `page`/`page_size`):

```bash
curl -s "https://402.blockvault.ai/api/v1/travel/bookings?page=1&page_size=20" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

## Follow-up questions (sub-agent reader)

When the user asks a follow-up about the rendered results ("which flight is nonstop?", "show only morning departures"), do **not** re-read the whole artifact in the main conversation. Spawn a sub-agent to answer from the artifact:

Call `spawn_subagents` with one task:

- **tasks**: Array, Required. One element:
  - **id**: `"flight_filter"`.
  - **objective**: `"Answer this question from the artifact: <user question>. The artifact is at <artifact_path>."`.
  - **instructions**: Use `text_editor` with `command: "query"` (JMESPath) or `command: "search"` to read only the relevant slice of the artifact — never load the whole file. Return a concise markdown answer.
  - **output_format**: `"Markdown — a short answer plus the matching journey/offer names."`.

## Constraints

- **No state file during search/quote.** Keep `offerId`, quotes, and tokens in memory; only the confirmed booking is persisted server-side.
- **Never invent dates or prices.** Ask if missing.
- **Payment requires explicit user intent** — confirm before calling `/flights/confirm` (it charges real USDC).
- 401 → ask the user to re-authenticate. 402 on `/search` → credit top-up. 402 on `/confirm` → payment declined/incomplete.
