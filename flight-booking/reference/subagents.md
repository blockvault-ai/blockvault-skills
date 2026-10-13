# Flight Booking — Sub-agents

Three sub-agent tasks: `flight_gather` (collect trip details), `flight_freeze` (collect contact + passenger details), and `flight_filter` (answer follow-ups from an artifact).

## flight_gather — collect trip details

Spawn one sub-agent to collect trip details interactively. **Write the `objective` yourself, from the user's message**, so the sub-agent never re-asks for details the user already gave.

Call `spawn_subagents` with one task. The sub-agent asks one question at a time (never queue two), using the interactive tools.

- **tasks**: Array, Required. One element:
  - **id**: `"flight_gather"`.
  - **objective**: Compose this from the user's intent. Include every detail they already provided (origin, destination, dates, passengers, budget) so the sub-agent only asks for what's missing.
  - **output_format**: `"Markdown list — one '- key: value' per field. Keys: trip_type (round|one_way), origin, destination, outbound_date (YYYY-MM-DD), return_date (YYYY-MM-DD or 'none'), outbound_departure_time_after (HH:MM or 'none'), outbound_departure_time_before (HH:MM or 'none'), outbound_arrival_time_after (HH:MM or 'none'), outbound_arrival_time_before (HH:MM or 'none'), return_departure_time_after (HH:MM or 'none'), return_departure_time_before (HH:MM or 'none'), return_arrival_time_after (HH:MM or 'none'), return_arrival_time_before (HH:MM or 'none'), adults, children, infants, stops (any|nonstop|1_stop|2_stops), budget (number or 'none'), currency, preferred_airline (name or 'none'), origin_iata, destination_iata."`.
  - **instructions**: Today's date is **{{DATE}}** — anchor every relative date ("tomorrow", "next Friday", "in 2 weeks") against it and reject past dates. Ask one field at a time in order: trip_type → origin → destination → outbound_date → (return_date if round trip) → outbound departure time window → outbound arrival time window → (return departure time window if round trip) → (return arrival time window if round trip) → adults → children → infants → max stops → budget → currency → preferred airline. **Wait for the user's answer before asking the next.** Default `stops` to `any` if cancelled. For each departure/arrival time window, ask "what time of day do you want to depart/arrive?" and accept either a free HH:MM range ("between 06:00 and 14:00") or a single bound ("after 18:00", "before 12:00"); set the missing bound to `none`. Default both bounds to `none` (any time) if the user has no preference. For the preferred airline, accept a name ("Iberia", "Delta") or `none`; do not resolve its IATA code here — the card deck filters by carrier client-side. Validate time format (HH:MM, 24h) and date format (YYYY-MM-DD, ≥ today) and currency code. After `origin` is confirmed, `web_search` with `query: "IATA code <origin> airport"` and `max_results: 3` to extract the 3-letter primary code; if multiple major airports exist, ask the user to pick. Repeat for `destination`. Never invent IATA codes.

## flight_freeze — collect contact + passenger details

Spawn one sub-agent to collect the booking contact and passenger details before freezing a quote.

Call `spawn_subagents` with one task:

- **tasks**: Array, Required. One element:
  - **id**: `"flight_freeze"`.
  - **objective**: `"Collect the booking contact and passenger details for the flight quote."`.
  - **output_format**: `"Markdown list — one '- key: value' per field. Keys: contact_first_name, contact_last_name, contact_email, contact_phone_number, passengers (JSON array of {first_name, last_name, birthday (YYYY-MM-DD), gender (M|F), nationality (CC), passenger_type (0=adult,1=child,2=infant)})."`.
  - **instructions**: Ask one field at a time and **wait for each answer before the next**. Collect the contact first (first name, last name, email, phone number), then one passenger entry per traveller — the number of passengers must match the `adults`/`children`/`infants` counts from the search. For each passenger ask: first name, last name, birthday (YYYY-MM-DD), gender (M or F), nationality (two-letter country code), and set `passenger_type` (0=adult, 1=child, 2=infant) from the traveller's age. Validate dates and country codes.

## flight_filter — answer follow-ups from an artifact

When the user asks a follow-up about rendered results ("which flight is nonstop?", "show only morning departures"), do **not** re-read the whole artifact in the main conversation. Spawn a sub-agent to answer from the artifact.

Call `spawn_subagents` with one task:

- **tasks**: Array, Required. One element:
  - **id**: `"flight_filter"`.
  - **objective**: `"Answer this question from the artifact: <user question>. The artifact is at <artifact_path>."`.
  - **instructions**: Use `text_editor` with `command: "query"` (JMESPath) or `command: "search"` to read only the relevant slice of the artifact — never load the whole file. Return a concise markdown answer.
  - **output_format**: `"Markdown — a short answer plus the matching journey/offer names."`.

**Time-of-day follow-ups re-search, not filter.** When the user asks to narrow by departure time ("only mornings", "after 6pm", "between 10:00 and 14:00"), the artifact only holds the journeys already downloaded — filtering it would hide valid options. Instead, re-run the search with the same legs plus the requested `departure_time_after`/`departure_time_before` on the relevant leg(s), then render the new artifact. Only use the `flight_filter` sub-agent for questions answerable from the already-downloaded results (stops, duration, price, carrier, baggage).