---
name: flight-booking
description: Search and book flights (airfare) via the BlockVault search API, then generate a booking link. Use when the user asks to find flights, book airfare, or plan a trip that includes flights — origin, destination, dates, budget, or a specific airline route. For hotel stays and room reservations use hotel-booking instead.
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

Plan the flight leg of a trip: collect trip details, search outbound and return flights, let the user pick, then generate a booking link. Follow the steps carefully and use the provided tools for each part.

Do not ask the user for trip details directly. Spawn a sub-agent to collect the flight information interactively.

Base URL: `https://402.blockvault.ai`

### Step 1: Gather flight details (sub-agent)

Call `spawn_subagents` with one task to collect the inputs interactively. The sub-agent asks one question at a time (never queue two), using the interactive tools.

Call `spawn_subagents` with:

- **tasks**: Array, Required. One element:
  - **id**: `"flight_gather"`.
  - **objective**: `"Collect the flight details from the user: origin, destination, trip type (round trip or one way), departure date, return date (round trips only), number of adults, max stops, budget, and currency. Then resolve IATA airport codes for both airports via web search."`.
  - **output_format**: `"Markdown list — one '- key: value' per field. Keys: trip_type (round|one_way), origin, destination, outbound_date (YYYY-MM-DD), return_date (YYYY-MM-DD or 'none'), adults, stops (0=Any|1=Nonstop only|2=1 stop or fewer|3=2 stops or fewer), budget (number or 'none'), currency, origin_iata, destination_iata, hl, gl."`.
  - **instructions**: Today's date is **{{DATE}}** — anchor every relative date ("tomorrow", "next Friday", "in 2 weeks") against it and reject past dates. Ask one field at a time in order: trip_type → origin → destination → outbound_date → (return_date if round trip) → adults → max stops → budget → currency. **Wait for the user's answer before asking the next.** Default `stops` to `0` (Any) if cancelled. Validate date format (YYYY-MM-DD, ≥ today) and currency code. After `origin` is confirmed, `web_search` with `query: "IATA code <origin> airport"` and `max_results: 3` to extract the 3-letter primary code; if multiple major airports exist, ask the user to pick. Repeat for `destination`. Never invent IATA codes. Derive `hl` and `gl` (two-letter) from the user's locale.

When the sub-agent completes, use the returned values for all remaining steps.

### Step 2: Search outbound flight

**Round trip:** (1) outbound search, (2) user picks, (3) return search using the chosen option's `departure_token`, (4) user picks return, (5) save the **return** flight's `booking_token` — not the outbound's (which 404s).

**One way:** single search with `"type":"2"`. The chosen flight's `price` is the full one-way total.

> **Round-trip pricing is per complete itinerary.** Every option's `price` in **both** searches is the full round-trip total. Never sum outbound + return — that double-counts. The authoritative total is the **chosen return flight's** `price`.

Always set `type` (`"1"` round, `"2"` one way), `adults`, `stops`, `hl`, `gl`, `currency`. Pass `budget` as `max_price` (integer, no currency prefix). Omit `stops` only when it is `0`. **Send `adults`, `stops`, `max_price` as JSON strings** (e.g. `"adults":"2"`) — the proxy rejects them as numbers. Omitting `adults` silently prices for 1 passenger.

```bash
# Round trip
curl -sS -X POST https://402.blockvault.ai/api/v1/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"q":"flights","engine":"google_flights","hl":"<hl>","gl":"<gl>","extra_params":{"departure_id":"<IATA>","arrival_id":"<IATA>","outbound_date":"<YYYY-MM-DD>","return_date":"<YYYY-MM-DD>","currency":"<CUR>","type":"1","adults":"<adults>","stops":"<stops>","max_price":"<budget_number>"}}'

# One way (omit return_date, set type:"2")
curl -sS -X POST https://402.blockvault.ai/api/v1/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"q":"flights","engine":"google_flights","hl":"<hl>","gl":"<gl>","extra_params":{"departure_id":"<IATA>","arrival_id":"<IATA>","outbound_date":"<YYYY-MM-DD>","currency":"<CUR>","type":"2","adults":"<adults>","stops":"<stops>","max_price":"<budget_number>"}}'
```

If both `best_flights` and `other_flights` come back empty, filters are too tight: drop `max_price`, then relax `stops` to `0`, retry once (≤ 2 searches per category). Render `raw_data.best_flights[]` with the Jinja2 template below (airline, times, duration, stops, price, CO₂ when present). Surface useful signals: overnight markers, often-delayed warnings, codeshare, layovers. Present options to the user; never auto-pick or show a booking link before the user confirms.

**Airline logos:** each `f.flights[]` segment carries `airline_logo` — render it inline (≤ 40px) next to the airline name, as in the template. Guard with `{% if seg.airline_logo %}` so a missing logo never breaks the render.

````jinja
## ✈️ {{ origin }} → {{ destination }}{% if price_insights %} · 🤖 typical price {{ currency }} {{ price_insights.typical_price_range[0] }}–{{ price_insights.typical_price_range[1] }}{% endif %}

{% for f in best_flights[:5] %}
  {%- set hours = f.total_duration // 60 -%}
  {%- set minutes = f.total_duration % 60 -%}
  {%- set stops = f.layovers | length -%}
### {{ loop.index }}. {{ currency }} {{ f.price }} — {{ hours }}h {{ minutes }}m{% if stops == 0 %} · 🟢 Direct{% else %} · 🔁 {{ stops }} stop{{ "s" if stops > 1 }}{% endif %}{% if f.type %} · {{ f.type }}{% endif %}

{%- if f.carbon_emissions %}
  {%- set kg = (f.carbon_emissions.this_flight / 1000) | round(0, "floor") | int -%}
  {%- set typical = (f.carbon_emissions.typical_for_this_route / 1000) | round(0, "floor") | int -%}
  {%- set diff = f.carbon_emissions.difference_percent -%}
> 🌱 {{ kg }} kg CO₂ ({{ "+" if diff > 0 }}{{ diff }}% vs typical {{ typical }} kg)
{% endif %}

{% for seg in f.flights %}
- {% if seg.airline_logo %}<img src="{{ seg.airline_logo }}" width="32" height="32" /> {% endif %}**{{ seg.airline }}** · {{ seg.flight_number }}{% if seg.travel_class %} · 🎟️ {{ seg.travel_class }}{% endif %}{% if seg.overnight %} · 🌙 overnight{% endif %}{% if seg.often_delayed_by_over_30_min %} · ⚠️ often delayed 30m+{% endif %}
  - 🛫 **{{ seg.departure_airport.time }}** — {{ seg.departure_airport.name }} ({{ seg.departure_airport.id }})
  - 🛬 **{{ seg.arrival_airport.time }}** — {{ seg.arrival_airport.name }} ({{ seg.arrival_airport.id }})
  - ⏱ {{ seg.duration // 60 }}h {{ seg.duration % 60 }}m · ✈️ {{ seg.airplane }}{% if seg.legroom %} · 📏 {{ seg.legroom }}{% endif %}
  {%- if seg.plane_and_crew_by %}
  - 🤝 Operated by {{ seg.plane_and_crew_by }}
  {%- endif %}
  {%- if seg.ticket_also_sold_by %}
  - 🏷️ Also sold by: {{ seg.ticket_also_sold_by | join(", ") }}
  {%- endif %}
  {%- if seg.extensions %}
  - 🛎️ {{ seg.extensions | join(" · ") }}
  {%- endif %}
{%- if not loop.last and f.layovers and f.layovers[loop.index0] %}
  {%- set lay = f.layovers[loop.index0] -%}
  - ⏸️ **Layover** {{ lay.duration // 60 }}h {{ lay.duration % 60 }}m at {{ lay.name }} ({{ lay.id }}){% if lay.overnight %} · 🌙 overnight{% endif %}
{%- endif %}
{% endfor %}

---
{% endfor %}
````

Tell the user to "pick a number" and wait for their answer before recording the choice or searching further.

User picks N. Keep in memory:

- **Round trip:** `best_flights[N].departure_token` + a summary of the outbound flight. Do **not** record this `price` as the total — the real total comes from the return search.
- **One way:** `best_flights[N].booking_token`, the flight summary, and `best_flights[N].price` as the final total.

### Step 3: Search return flight (round trip only)

Skip if `return_date` is empty. Otherwise reuse the same base params plus the saved `departure_token`:

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"q":"flights","engine":"google_flights","hl":"<hl>","gl":"<gl>","extra_params":{"departure_id":"<IATA>","arrival_id":"<IATA>","outbound_date":"<YYYY-MM-DD>","return_date":"<YYYY-MM-DD>","currency":"<CUR>","type":"1","adults":"<adults>","departure_token":"<departure_token>"}}'
```

Keep `adults`, `type`, `hl`, `gl`, `currency` identical to the outbound call. Render `best_flights[]` with the same template. **Each option's `price` is the complete round-trip total.** User picks N. Keep the return summary, `best_flights[N].booking_token`, and `best_flights[N].price` as the final round-trip total — never add the outbound price.

Ask **"Ready to book?"** The user can also say "change flights" to revisit the search.

### Step 4: Generate booking link

Only after the user confirms. Verify `booking_token` is present; if not, re-run the search. Never fabricate a token.

Emit the booking link as a single line (no whitespace between query parameters):

```text
https://402.blockvault.ai/api/v1/search/pay?engine=google_flights&departure_id=<departure_id>&arrival_id=<arrival_id>&outbound_date=<outbound_date>[&return_date=<return_date>]&currency=<currency>&hl=<hl>&booking_token=<booking_token>
```

Present a summary: origin → destination, dates, adults, the chosen total price (the per-flight `price`, never summed legs), and the booking link. Note that prices are estimates and the final amount must be verified on the booking site.

### Insufficient credits (402)

On HTTP **402** from any search, the user has no credits. Tell them they can purchase more:

```bash
curl -i -X POST https://402.blockvault.ai/api/v1/inference/credits \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

On HTTP 200, retry the failed search automatically. If the purchase fails, stop and report the error.

### Corrupt or rejected `departure_token` (return search fails)

Stop and reason before retrying:

1. Valid source is **only** `raw_data.best_flights[N].departure_token` of the **outbound** search (`type:"1"`, no `departure_token` in the request). Never read it from `other_flights`, a return search, or a `flights[*]` segment.
2. Pass it **verbatim** — no URL-encode, no trim, no decode, no line breaks.
3. The return call must reuse the **exact same** base params (`departure_id`, `arrival_id`, `outbound_date`, `return_date`, `currency`, `hl`, `gl`, `type:"1"`, `adults`). Do **not** include `stops` or `max_price` — they are encoded in the token; re-sending them invalidates it.
4. On "invalid token" / empty result: re-run the outbound search to get a **fresh** token (they expire), then immediately re-run the return search.
5. Never substitute `booking_token` for `departure_token` or vice versa.

### Corrupt or rejected `booking_token` (404 / "No booking options found")

Stop and reason before retrying:

1. Valid `booking_token` comes only from: **round trip** → the **return** search (the one with `departure_token`); **one way** → the search where `"type":"2"`.
2. Confirm you did **not** copy `departure_token` into `booking_token` — the #1 cause of 404.
3. Confirm `departure_id`, `arrival_id`, `outbound_date` (and `return_date` for round trips) in the booking link match the chosen flight option.
4. If the right token still fails, treat it as expired: re-run the corresponding search and rebuild the link.
5. Never fabricate, truncate, or re-encode tokens.

## Constraints

- **No state file during search.** Keep options, tokens, summaries, totals in memory; only the final booking is written.
- **Never invent dates.** Ask if missing.
- **≤ 2 searches per category.** Reuse memory.
- Use `hl` for the user's language and `gl` for the search country.
- **Always send `adults`** as a string. Omitting it prices for 1 passenger.
- **Set `type` explicitly** — `"1"` round, `"2"` one way. `booking_token` only from the leg with the full itinerary (the `departure_token` call for round trips, or the `type:"2"` call for one way).
- 401 → ask the user to re-authenticate. 402 → run the credits purchase flow.