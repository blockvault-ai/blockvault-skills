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

Help the user find and choose the best flight, then book it end to end: search journeys (paginated), freeze a priced quote, and charge USDC on-chain via x402 to finalise the booking.

## Instructions

- **Help the user choose.** Compare options, explain tradeoffs (price, duration, stops, carrier, refundability), and give a clear recommendation — don't just dump a list.
- **Use `bash` for API calls.** Execute `curl` commands to interact with the travel API — never invent responses.
- Detect the user's language and reply in that language.
- Render results as Markdown tables, never raw JSON.
- **Never auto-pick** — let the user choose, but always recommend the best fit for their stated needs.
- **Round trip = one journey.** Nuitee returns a round trip as a single journey (outbound + inbound segments, one `offerId`). The user picks one combined journey — never two separate one-way bookings.
- **Airline + schedule are filters, not search terms.** The card deck filters by airline client-side (Nuitee has no airline search filter); departure/arrival time windows are passed per leg on the search request.

## Reference files

Discover the full reference for this skill in the `reference` folder. Each file is a self-contained reference for a specific topic. Load it with `run_js` → `read_skill_reference` (see below).

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"flight-booking","file":"<name>"}`

To list all available files, omit `file`:

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"flight-booking"}`

Determine which reference files to load based on the workflow step:
- is related to searching flights, freezing a quote, or paying → load `booking`
- is related to collecting trip details or answering follow-ups → load `subagents`

## Workflow

### Step 1: Gather trip details (sub-agent)

Do not ask the user for trip details directly. Load the `subagents` reference and spawn the `flight_gather` sub-agent to collect them interactively — but **write the `objective` yourself, from the user's message**, so the sub-agent never re-asks for details the user already gave.

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"flight-booking","file":"subagents"}`

### Step 2: Search flights

Based on the user's origin, destination, dates, and budget, search flights. Load the `booking` reference and follow its instructions.

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"flight-booking","file":"booking"}`

Render the result emitting:
  ````markdown
    ```artifact
    {"artifact":"<artifact_path>","template":"flight-booking:flight-list","context":{"origin":"<ORIGIN>","destination":"<DEST>","outbound_date":"<YYYY-MM-DD>"}}
    ```
  ````

After rendering, **ask the user** which flight they want (by number). Do not re-list the flights in Markdown — the card deck is the list. Wait for their answer before quoting.

### Step 3: Freeze a quote and pay

Once the user chooses a flight, load the `booking` reference (if not already loaded) and follow its instructions to freeze a quote and confirm the booking.

## Follow-up questions (sub-agent reader)

When the user asks a follow-up about the rendered results ("which flight is nonstop?", "show only morning departures"), do **not** re-read the whole artifact in the main conversation. Load the `subagents` reference and spawn the `flight_filter` sub-agent to answer from the artifact.

## Constraints

- **No state file during search/quote.** Keep `offerId`, quotes, and tokens in memory; only the confirmed booking is persisted server-side.
- **Never invent dates or prices.** Ask if missing.
- **Pagination**: always respect `has_more`/`total`; never assume the first page is complete.
- **Payment requires explicit user intent** — confirm before calling `/flights/confirm` (it charges real USDC).
- 401 → ask the user to re-authenticate. 402 on `/search` → credit top-up. 402 on `/confirm` → payment declined/incomplete.
