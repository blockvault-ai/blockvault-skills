---
name: hotel-booking
description: Search hotels and book a room (with hotel stay) via the BlockVault travel API (Nuitee/LiteAPI). Pay by card through a hosted payment page, or charge USDC on-chain through x402. Use when the user asks to find a hotel, book a room, plan a stay, or reserve accommodation with check-in/check-out dates, a budget, or a city.
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

# Hotel Booking

Help the user find and choose the best hotel stay, then book it end to end: search offers (paginated), freeze a priced quote, and charge USDC on-chain via x402 to finalise the booking. Card payment is also available through a hosted payment page.

## Instructions

- **Help the user choose.** Compare options, explain tradeoffs (price, rating, location, refundability), and give a clear recommendation — don't just dump a list.
- **Use `bash` for API calls.** Execute `curl` commands to interact with the travel API — never invent responses.
- Detect the user's language and reply in that language.
- Render results as Markdown tables, never raw JSON.
- **Never auto-pick** — let the user choose, but always recommend the best fit for their stated needs.

## Reference files

Discover the full reference for this skill in the `reference` folder. Each file is a self-contained reference for a specific topic. Load it with `run_js` → `read_skill_reference` (see below).

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"hotel-booking","file":"<name>"}`

To list all available files, omit `file`:

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"hotel-booking"}`

Determine which reference files to load based on the workflow step:
- is related to searching rooms, freezing a quote, or paying → load `booking`
- is related to collecting stay details or answering follow-ups → load `subagents`

## Workflow

### Step 1: Gather stay details (sub-agent)

Do not ask the user for trip details directly. Load the `subagents` reference and spawn the `hotel_gather` sub-agent to collect them interactively — but **write the `objective` yourself, from the user's message**, so the sub-agent never re-asks for details the user already gave.

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"hotel-booking","file":"subagents"}`

### Step 2: Discover hotels

Based on the user's destination, check-in/check-out dates, and budget, discover hotels with:

  ```bash
  curl -sS "https://402.blockvault.ai/api/v1/travel/hotels?city_name=<City>&country_code=<CC>&limit=50&offset=0" \
    -H "Authorization: Bearer {{DELEGATE_JWT}}" \
    -o <city>-<checkin>-<checkout>.json
  ```

Render the result emitting:
  ````markdown
    ```artifact
    {"artifact":"<artifact_path>","template":"hotel-booking:hotel-list","context":{"city_name":"<City>","checkin":"<YYYY-MM-DD>","checkout":"<YYYY-MM-DD>"}}
    ```
  ````

After rendering, **ask the user** which hotel they want (by number). Do not re-list the hotels in Markdown — the card deck is the list. Wait for their answer before showing rooms.

Once the user chooses a hotel, proceed to Step 3.

### Step 3: Fetch the chosen hotel's rooms:

Load the `rooms` reference if not already loaded and follow its instructions.

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"hotel-booking","file":"booking"}`

### Step 4: Book the chosen room

Once the user picks a room, **ask how they want to pay** (card or USDC), then
follow the matching section of the `booking` reference:

- **Card** → `POST /travel/book` → give the returned `payment_url` to the user
  as a clickable markdown link. The hosted page collects the guest and
  finalises the booking.
- **USDC** → freeze a quote (`/account-book/quote`) then confirm
  (`/account-book/confirm`, charges on-chain via x402).

## Follow-up questions (sub-agent reader)

When the user asks a follow-up about the rendered results, do not re-read the whole artifact in the main conversation. Spawn the `hotel_filter` sub-agent to answer from the artifact, filter the result to answer the user's question, and return a concise Markdown answer.

To look up prices, ratings, or amenities, the sub-agent may call the `/travel/hotels` -> `/travel/search` endpoints with the relevant hotel IDs.


## Constraints

- **No state file during search/quote.** Keep `offerId`, quotes, and tokens in memory; only the confirmed booking is persisted server-side.
- **Never invent dates or prices.** Ask if missing.
- **Pagination**: always respect `has_more`/`total`; never assume the first page is complete.
- **Payment requires explicit user intent** — confirm before calling `/account-book/confirm` (it charges real USDC) or opening the card payment page.
- 401 → ask the user to re-authenticate. 402 on `/search` → credit top-up. 402 on `/confirm` → payment declined/incomplete.