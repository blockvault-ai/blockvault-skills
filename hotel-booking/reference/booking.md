# Hotel Booking — Rooms

## Contents
- Search rooms
- Choose a payment method
- Pay with card (hosted payment page)
- Freeze a quote (USDT)
- Booking history
- Room fields

## Search rooms for a hotel
Once the user has chosen a hotel, search for its available rooms call:

  ```bash
  curl -sS -X POST https://402.blockvault.ai/api/v1/travel/search \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer {{DELEGATE_JWT}}" \
    -d '{"checkin":"<YYYY-MM-DD>","checkout":"<YYYY-MM-DD>","currency":"<CUR>","guest_nationality":"<CC>","occupancies":[{"adults":<n>,"children":[]}],"hotel_ids":["<hotelId>"],"limit":<limit>,"offset":0,"max_price":<budget>}' \
    -o <hotelId>-<checkin>-<checkout>.json
  ```
  
  - `hotel_ids` is a JSON array of hotel IDs (from the hotel list). **Never print them** — keep them in memory keyed by the hotel number shown to the user.
  - `limit` is the maximum number of rooms to return (default 100, max 1000). Paginate with `offset`/`limit`; each response returns `has_more` and `total`.
  - `max_price` is the user's budget in the chosen currency;

Render the result emitting:

````markdown
  ```artifact
  {"artifact":"<artifact_path>","template":"hotel-booking:room-list","context":{"currency":"<CUR>"}}
  ```
````

After rendering, **ask the user** which room they want (by number). Wait for their answer.

## Pay with card (hosted payment page)

Once the user picks a room, **ask to proceed with card payment**.

1. Create a checkout session with the chosen room's `offerId`:

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/travel/book \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"offer_id":"<room.offerId>"}'
```

Returns `prebook_id`, `transaction_id`, and `payment_url`.

2. Give the user the payment link as a clickable markdown link (the app opens
   external links in the system browser):

```markdown
[Complete payment](<payment_url>)
```

The user completes card payment in the browser; the hosted page then collects
the lead guest's name/email and finalises the booking automatically. Tell the
user to return to the chat once the page shows "Booking confirmed".

## Freeze a quote (priced, no charge yet)

1. Spawn the `hotel_freeze` sub-agent to  collect the guest's:
  - first name
  - last name
  - email
  
2. Call the `POST /travel/account-book/quote` endpoint with the chosen room's `offerId` and the guest's details:
  ```bash
  curl -sS -X POST https://402.blockvault.ai/api/v1/travel/account-book/quote \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer {{DELEGATE_JWT}}" \
    -d '{"offer_id":"<room.offerId>","holder":{"first_name":"<F>","last_name":"<L>","email":"<email>"},"guests":[{"occupancy_number":1,"first_name":"<F>","last_name":"<L>","email":"<email>"}]}'
  ```

Returns `quote_id` and `amount` (USDT in atomic units, 6 decimals).

## Booking history

```bash
curl -sS "https://402.blockvault.ai/api/v1/travel/bookings?page=<page>&page_size=<page_size>" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

