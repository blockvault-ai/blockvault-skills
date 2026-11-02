---
name: hotel-booking
description: Search hotels and book a room (with hotel stay) via the BlockVault travel API (Nuitee/LiteAPI). Pay by card through a hosted payment page, or charge USDT on-chain (Arbitrum) through x402. Use when the user asks to find a hotel, book a room, plan a stay, or reserve accommodation with check-in/check-out dates, a budget, or a city.
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

Help the user find and choose the best hotel stay, then book it end to end: search offers (paginated), freeze a priced quote, and charge USDT on-chain (Arbitrum) via x402 to finalise the booking. Card payment is also available through a hosted payment page.

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
- is related to collecting stay or guest details, or answering follow-ups → load `subagents`

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

- **Budget, "cheapest" or "near a landmark/district"** → do NOT use the call above. Use one priced area search instead: `POST /travel/search` with the location (`place_id` + `radius_m`, coordinates, or city), `sort` and `max_price`. For "near"/"close to" use `radius_m: 1500`; widen to 3000 only if fewer than 5 hotels come back. Its artifact renders directly with `hotel-booking:hotel-list` and already shows price, per-night price and distance. See the `booking` reference, section "Priced area search".
- **Near a landmark, address or district** ("near the Sagrada Familia") → resolve it with `GET /travel/places?q=<text>` first (the parameter is `q`). See "Discover by place or coordinates".
- **Amenities** ("with pool", "pet friendly") → look up ids with `GET /travel/facilities` and pass `facility_ids`. See "Filter by amenities".
- **Empty result** — read `empty_reason` (the server already retried once):
  - `over_budget` → tell the user the cheapest option costs `cheapest_available` and ask whether to raise the budget. Never drop `max_price` on your own.
  - `no_availability` → widen `radius_m` to 3000 once (keep `max_price`); if still empty, tell the user and suggest other dates.
- **Never build or merge JSON files yourself** to render cards — always render the artifact written by `curl -o`.

Render the result emitting:
  ````markdown
    ```artifact
    {"artifact":"<artifact_path>","template":"hotel-booking:hotel-list","context":{"city_name":"<City>","checkin":"<YYYY-MM-DD>","checkout":"<YYYY-MM-DD>"}}
    ```
  ````

After rendering, **ask the user** which hotel they want (by number). Do not re-list the hotels in Markdown — the card deck is the list. Wait for their answer before showing rooms.

Once the user chooses a hotel, proceed to Step 3.

### Step 3: Fetch the chosen hotel's rooms

There is no `/travel/rooms` endpoint. Rooms come from `POST /travel/search` scoped to the chosen hotel id (`hotels[<n-1>].id` of the rendered list), reusing the same dates, occupancy and currency:

  ```bash
  curl -sS -X POST https://402.blockvault.ai/api/v1/travel/search \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer {{DELEGATE_JWT}}" \
    -d '{"checkin":"<YYYY-MM-DD>","checkout":"<YYYY-MM-DD>","currency":"<CUR>","guest_nationality":"<CC>","occupancies":[{"adults":<n>,"children":[]}],"hotel_ids":["<hotelId>"],"max_price":<total budget>,"sort":"price_asc","limit":100,"offset":0}' \
    -o <hotelId>-<checkin>-<checkout>.json
  ```

If the user gave a budget in Step 2, pass the same total as `max_price` (per-night budget × nights); drop the field only when there was no budget. The response holds one cheapest rate per distinct room option (up to 30, `room_count` is the total).

Render it with `hotel-booking:room-list` (context `{"currency":"<CUR>"}`), then ask which room. Room number `n` is `offers[0].rooms[<n-1>]` of that artifact — read its `offerId` with one `text_editor` query (`offers[0].rooms[<n-1>].offerId`). For optional filters (board, refundable, budget) and room fields, load the `booking` reference:

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"hotel-booking","file":"booking"}`

### Step 4: Book the chosen room

Once the user picks a room, **ask how they want to pay** (card or USDT), then
follow the matching section of the `booking` reference:

- **Card** → `POST /travel/book` → give the returned `payment_url` to the user
  as a clickable markdown link. The hosted page collects the guest and
  finalises the booking.
- **USDT** →
  1. Ask once whether the user has a **promo code** and offer the optional
     **add-ons** (Uber ride credit, eSIM data plan) — skip what the user already
     declined. Both are described in the `booking` reference ("Promo codes",
     "Add-ons"). Highlight perks and discounts the room already includes
     (`perks`, `discount_percent`).
  2. **Spawn the `hotel_freeze` sub-agent once** (`subagents` reference) to
     collect the lead guest; use it as both `holder` and `guests[0]`. Never ask
     for names or emails in the main conversation, and never call
     `/account-book/quote` without them.
  3. **410 on `/quote`** (offer expired) → re-run the Step 3 room search, pick
     the room with the same `name` and `board`, tell the user the new price and
     quote again with the same guest details (do not ask the user to choose
     again unless the room is gone).
  4. Freeze the quote. `holder` and `guests` are top-level fields (never nest
     them under another key):

     ```bash
     curl -sS -X POST https://402.blockvault.ai/api/v1/travel/account-book/quote \
       -H "Content-Type: application/json" \
       -H "Authorization: Bearer {{DELEGATE_JWT}}" \
       -d '{"offer_id":"<room.offerId>","holder":{"first_name":"<F>","last_name":"<L>","email":"<email>"},"guests":[{"occupancy_number":1,"first_name":"<F>","last_name":"<L>","email":"<email>"}]}' \
       -o quote-<offerId>.json
     ```

     Add `"promo_code"` / `"addons"` only when the user asked for them.
  5. Show `sell_fiat` (and `discount_fiat` when present), then confirm
     (`/account-book/confirm`, charges on-chain via x402) only after an explicit
     "yes".

## Follow-up questions (sub-agent reader)

When the user asks a follow-up about the rendered results, do not re-read the whole artifact in the main conversation. Spawn the `hotel_filter` sub-agent to answer from the artifact, filter the result to answer the user's question, and return a concise Markdown answer.

To look up prices, ratings, or amenities, the sub-agent may call the `/travel/hotels` -> `/travel/search` endpoints with the relevant hotel IDs.


## Constraints
**USDT quotes require USD prices** — search with `currency: "USD"` when the user will pay in USDT.
- **Add-ons are non-refundable** — say so before adding them.
- 401 → ask the user to re-authenticate. 402 on `/search` → credit top-up. 402 on `/confirm` → payment declined/incomplete. 409 on `/confirm` → the promo code ran out; quote again without it. 422 on `/quote` → read `detail` (invalid promo, price below cost, non-USD, bad add-on) and fix the request
- **No state file during search/quote.** Keep `offerId`, quotes, and tokens in memory; only the confirmed booking is persisted server-side.
- **Never invent dates or prices.** Ask if missing.
- **Pagination**: always respect `has_more`/`total`; never assume the first page is complete.
- **Payment requires explicit user intent** — confirm before calling `/account-book/confirm` (it charges real USDT) or opening the card payment page.
- 401 → ask the user to re-authenticate. 402 on `/search` → credit top-up. 402 on `/confirm` → payment declined/incomplete.