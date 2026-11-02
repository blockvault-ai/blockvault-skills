# Hotel Booking — Sub-agents

Three sub-agent tasks: `hotel_gather` (collect stay details), `hotel_freeze` (collect guest details before a quote) and `hotel_filter` (answer follow-ups from an artifact).

## hotel_gather — collect stay details

Spawn one sub-agent to collect trip details interactively. **Write the `objective` yourself, from the user's message**, so the sub-agent never re-asks for details the user already gave.

Call `spawn_subagents` with one task. The sub-agent asks one question at a time (never queue two), using the interactive tools.

- **tasks**: Array, Required. One element:
  - **id**: `"hotel_gather"`.
  - **objective**: Compose this from the user's intent. Include every detail they already provided (destination, check-in/check-out dates, rooms, adults/children, budget) so the sub-agent only asks for what's missing..
  - **output_format**: Markdown list — one `- key: value` per field. Keys:

    ```
    - destination: <free text>
    - city_name: <resolved city>
    - country_code: <two-letter ISO 3166>
    - hotel_ids: <comma-separated ids, if a specific hotel was given>
    - iata_code: <3-letter airport code, if used>
    - near: <landmark, district or address to stay close to, or 'none'>
    - amenities: <comma-separated must-have amenities (pool, parking, pets…), or 'none'>
    - board: <breakfast | half board | full board | all inclusive | room only | 'any'>
    - checkin: <YYYY-MM-DD>
    - checkout: <YYYY-MM-DD>
    - occupancies: <JSON array of {adults, children:[ages]}>
    - budget: <number in USD, or 'none'>
    - currency: <three-letter code, USD default>
    ```
  - **instructions**: Ask in the user's language. Today's date is **{{DATE}}** — anchor every relative date ("tomorrow", "next weekend") against it and reject past dates. Ask one field at a time in order: destination → checkin → checkout → occupancies (rooms, adults, children ages) → budget. **Wait for each answer before the next.** Skip any field the user already provided. Validate dates (YYYY-MM-DD, checkout after checkin). After `destination` is confirmed, use `web_search` (query like `"<city> country code ISO 3166"`, max_results 3) to resolve `city_name` + two-letter `country_code`. If the user gives a specific hotel, put its id in `hotel_ids` instead. Only fill `near`, `amenities` and `board` when the user mentioned them — never ask for them. Derive a reasonable currency (USD default; always USD when the user will pay in USDT).

## hotel_freeze — collect guest details

Spawn **exactly once** after the user picked a room and a payment method, before `POST /travel/account-book/quote`. **Never ask for guest details in the main conversation.** Write the `objective` yourself and include any detail the user already gave (name, email) so the sub-agent only asks for what's missing. Reuse its result for every retry of the quote — never spawn it again in the same booking.

The sub-agent collects **one person**: the lead guest, who is also the booking holder. Build both `holder` and `guests[0]` (`occupancy_number: 1`) from it. Only for more than one room, add a second task per extra room.

Call `spawn_subagents` with one task:

- **tasks**: Array, Required. One element:
  - **id**: `"hotel_freeze"`.
  - **objective**: `"Collect the lead guest for <hotel name>, <room name>, <checkin>–<checkout>. Ask in <user language>. Already known: <details or 'nothing'>."`.
  - **output_format**: Markdown list — one `- key: value` per field:

    ```
    - first_name: <text>
    - last_name: <text>
    - email: <email>
    - phone: <E.164 phone, or 'none'>
    ```
  - **instructions**: Ask in the user's language (the one named in the objective). Ask one field at a time, in order: first name → last name → email → phone (optional, accept 'skip' / 'no'). **Wait for each answer before the next.** Ask each field only once and skip any field already in the objective. Validate the email has `@` and a domain. Never ask for other travellers, nationality, passport or birth date.

## hotel_filter — answer follow-ups from an artifact

When the user asks a follow-up about rendered results ("which hotel has a pool?", "show only 4-star hotels"), do **not** re-read the whole artifact in the main conversation. Spawn a sub-agent to answer from the artifact.

Make sure the sub-agent's `objective` includes the user's question and the artifact path. 

Call `spawn_subagents` with one task:

- **tasks**: Array, Required. One element:
  - **id**: `"hotel_filter"`.
  - **objective**: `"Answer this question from the artifact: <user question>. The artifact is at <artifact_path>."`.
  - **preload_skills**: Array, Optional. `["hotel-booking"]` — injects the hotel-booking instructions directly into the sub-agent system prompt.
  - **instructions**: Use `text_editor` with `command: "query"` (JMESPath) to read only the relevant slice of the artifact — never load the whole file. use the bash tool to make calls when there is not available artifact.
  - **output_format**: Markdown — a short answer plus the matching hotel/room names:

    ```
    **Answer:** <one or two sentences>

    **Matching hotels:**
    - <hotel name> — <room name>
    - <hotel name> — <room name>
    ```
