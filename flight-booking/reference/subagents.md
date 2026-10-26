# Flight Booking — Sub-agents

Three sub-agent tasks: `flight_gather` (collect trip details), `flight_freeze` (collect contact + passenger details), and `flight_filter` (answer follow-ups from an artifact).

## flight_gather — collect trip details

Spawn one sub-agent to collect trip details interactively. **Write the `objective` yourself, from the user's message**, so the sub-agent never re-asks for details the user already gave.

Call `spawn_subagents` with one task. The sub-agent asks one question at a time (never queue two), using the interactive tools.

- **tasks**: Array, Required. One element:
  - **id**: `"flight_gather"`.
  - **objective**: <Compose this from the user's intent. Include every detail they already provided (origin, destination, dates, passengers, budget) so the sub-agent only asks for what's missing >.
  - **output_format**: Markdown list — one `- key: value` per field. Keys:

    ```
    - trip_type: <one-way | round-trip>
    - origin: <3-letter IATA code>
    - destination: <3-letter IATA code>
    - outbound_date: <YYYY-MM-DD>
    - return_date: <YYYY-MM-DD, if round-trip>
    - adults: <number>
    - children: <number>
    - infants: <number>
    - max_stops: <number>
    - budget: <number in USD, or 'none'>
    - currency: <three-letter code, USD default>
    - preferred_airline: <airline name, or 'none'>
    ```
  - **instructions**: Today's date is **{{DATE}}** — anchor every relative date ("tomorrow", "next Friday", "in 2 weeks") against it and reject past dates. Ask one field at a time in order: trip_type → origin → destination → outbound_date → (return_date if round trip) adults → children → infants → max stops → budget → currency → preferred airline. **Wait for the user's answer before asking the next.** After `origin` is confirmed, `web_search` with `query: "IATA code <origin> airport"` and `max_results: 3` to extract the 3-letter primary code; if multiple major airports exist, ask the user to pick. Repeat for `destination`. Never invent IATA codes.

## flight_freeze — collect contact + passenger details

Spawn one sub-agent to collect the booking contact and passenger details before freezing a quote.

Call `spawn_subagents` with one task, The sub-agent asks one question at a time (never queue two), using the interactive tools:

- **tasks**: Array, Required. One element:
  - **id**: `"flight_freeze"`.
  - **objective**: `"Collect the booking contact and passenger details for the flight quote."`.
  - **output_format**: Markdown list — one `- key: value` per field. Keys:

    ```
    - contact_first_name: <string>
    - contact_last_name: <string>
    - contact_email: <string>
    - contact_phone_number: <string>
    - contact_phone_country_code: <two-letter country code>
    - passengers: <JSON array of {first_name, last_name, birthday, gender, nationality, passenger_type}>
    ```
  - **instructions**: Ask one field at a time and **wait for each answer before the next**. Collect the contact first (first name, last name, email, phone number), then one passenger entry per traveller — the number of passengers must match the `adults`/`children`/`infants` counts from the search. For each passenger ask: first name, last name, birthday (YYYY-MM-DD), gender (M or F), nationality (two-letter country code), and set `passenger_type` (0=adult, 1=child, 2=infant) from the traveller's age. Validate dates and country codes.

## flight filter — how to answer from an artifact

Spawn one sub-agent to answer follow-up questions from an artifact.

Call `spawn_subagents` with one task:

- **tasks**: Array, Required. One element:
  - **id**: `"flight_filter"`.
  - **objective**: `"Answer this question from the artifact: <user question>. The artifact is at <artifact_path>."`.
  - **preload_skills**: Array, Optional. `["flight-booking"]` — injects the flight-booking instructions directly into the sub-agent system prompt so it knows the artifact schema without a `load_skill` call.
  - **instructions**: Use `text_editor` with `command: "query"` (JMESPath) or `command: "search"` to read only the relevant slice of the artifact — never load the whole file. Return a concise markdown answer.
  - **output_format**: Markdown — a short answer plus the matching journey/offer names:

    ```
    **Answer:** <one or two sentences>

    **Matching journeys:**
    - <journey name> — <offer name>
    - <journey name> — <offer name>
    ```