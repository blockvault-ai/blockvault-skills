# Hotel Booking — Sub-agents

Two sub-agent tasks: `hotel_gather` (collect stay details) and `hotel_filter` (answer follow-ups from an artifact).

## hotel_gather — collect stay details

Spawn one sub-agent to collect trip details interactively. **Write the `objective` yourself, from the user's message**, so the sub-agent never re-asks for details the user already gave.

Call `spawn_subagents` with one task. The sub-agent asks one question at a time (never queue two), using the interactive tools.

- **tasks**: Array, Required. One element:
  - **id**: `"hotel_gather"`.
  - **objective**: Compose this from the user's intent. Include every detail they already provided (destination, check-in/check-out dates, rooms, adults/children, budget) so the sub-agent only asks for what's missing..
  - **output_format**: `"Markdown list — one '- key: value' per field. Keys: destination, city_name, country_code, hotel_ids, iata_code, checkin (YYYY-MM-DD), checkout (YYYY-MM-DD), occupancies (JSON array of {adults, children:[ages]}), budget (number in USD or 'none'), currency."`.
  - **instructions**: Today's date is **{{DATE}}** — anchor every relative date ("tomorrow", "next weekend") against it and reject past dates. Ask one field at a time in order: destination → checkin → checkout → occupancies (rooms, adults, children ages) → budget. **Wait for each answer before the next.** Skip any field the user already provided. Validate dates (YYYY-MM-DD, checkout after checkin). After `destination` is confirmed, use `web_search` (query like `"<city> country code ISO 3166"`, max_results 3) to resolve `city_name` + two-letter `country_code`. If the user gives a specific hotel, put its id in `hotel_ids` instead. Derive a reasonable currency (USD default).

The sub-agent must produce exactly one location method plus `checkin`, `checkout`, and `occupancies`. You may pass at most one of `city_name`+`country_code`, `iata_code`, or `hotel_ids`.

## hotel_filter — answer follow-ups from an artifact

When the user asks a follow-up about rendered results ("which hotel has a pool?", "show only 4-star hotels"), do **not** re-read the whole artifact in the main conversation. Spawn a sub-agent to answer from the artifact.

Call `spawn_subagents` with one task:

- **tasks**: Array, Required. One element:
  - **id**: `"hotel_filter"`.
  - **objective**: `"Answer this question from the artifact: <user question>. The artifact is at <artifact_path>."`.
  - **instructions**: Use `text_editor` with `command: "query"` (JMESPath) or `command: "search"` to read only the relevant slice of the artifact — never load the whole file. Return a concise markdown answer.
  - **output_format**: `"Markdown — a short answer plus the matching hotel/room names."`.

