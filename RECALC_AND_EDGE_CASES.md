# Recalculation logic & edge cases

## Airbnb alignment (from your screenshots)

- **Guest paid:** Nightly rate × nights → Subtotal; + occupancy taxes → Total. We don’t track guest-side total.
- **Host payout:**  
  - **Gross room fee** = display price × nights (before discount).  
  - **Nightly rate adjustment** = discount (negative).  
  - **Subtotal (our “Final Amount”)** = Gross − discount.  
  - **Tax (5%)** and **Host service fee (15.5%)** are on this subtotal.  
  - **Net received** = Subtotal − Tax − Fee.

So: **Final Amount = (Display price/night × Nights) − Total discount**; **Tax = 5% of Final**; **Fee = 15.5% of Final**; **Net = Final − Tax − Fee**. Our script does this; only rounding may differ (we use whole rupees).

---

## Edge cases (recalculation & validation)

### Add booking

| # | Scenario | Current behaviour | Fix / note |
|---|----------|-------------------|------------|
| 1 | check_out before or equal to check_in | nightsBetween gives 1 (clamped) | Validate and return error so user fixes dates. |
| 2 | price_per_night 0 | Final = 0 or −discount → 0; tax/fee 0 | OK. |
| 3 | Both final_amount and (price, nights, discount) provided | final_amount wins; discount/price/nights ignored for amount | By design; agent can pass final_amount from Airbnb. |
| 4 | Only price and nights, no discount | Final = price×nights; tax/fee on that | OK. |
| 5 | Discount > price×nights | Final = max(0, …) = 0 | OK. |
| 6 | Source Direct/WhatsApp/Instagram | No tax/fee; net_received = finalAmount (or provided) | OK. |
| 7 | Source Airbnb | Tax 5%, fee 15.5%, net = final − tax − fee | OK. |
| 8 | All three (tax, fee, net_received) provided for Airbnb | We use them; no recalc | OK. |
| 9 | nights from check_in/check_out (e.g. Apr 15–28) | 13 nights | OK (we use Math.round on days). |
| 10 | Missing booking_date | We use check_in | OK. |

### Edit booking

| # | Scenario | Current behaviour | Fix / note |
|---|----------|-------------------|------------|
| 11 | Only total_discount changed | Derive Final = price×nights − discount; recalc tax/fee/net for Airbnb | OK. |
| 12 | Only display_price_per_night changed | Same derivation and recalc | OK. |
| 13 | Only nights changed | Same | OK. |
| 14 | nights set to 0 | derivedFinal = max(0, 0−discount); can write Nights=0 to sheet | Clamp nights ≥ 1 when deriving and when updating. |
| 15 | Only booking_source changed Airbnb → Direct | Tax=0, Fee=0, Net=Final | OK. |
| 16 | Only booking_source changed Direct → Airbnb | Recalc tax, fee, net from current Final | OK. |
| 17 | Only status/notes changed | No amount recalc | OK. |
| 18 | Only final_amount set (no discount/price/nights) | We set Final; if Airbnb, recalc tax/fee/net | OK. |
| 19 | final_amount + total_discount both set | Derived Final overwrites (discount/price/nights path); final_amount ignored for derivation | By design. |
| 20 | Manual tax/fee override (user passes tax, fee) | We write them; net = final − tax − fee if net_received not passed | OK. |
| 21 | Discount cleared (total_discount = 0) | Derive Final = price×nights; recalc | OK. |
| 22 | Price per night 0 on edit | derivedFinal = 0; tax/fee 0 | OK. |

### Payments & due

| # | Scenario | Current behaviour | Fix / note |
|---|----------|-------------------|------------|
| 23 | add_payment: amount > due | Error “exceeds due” | OK. |
| 24 | add_payment: amount = due | Net Received += amount; due becomes 0 | OK. |
| 25 | add_payment: refund | Net Received −= amount; clamped to ≥ 0 | OK. |
| 26 | add_payment: refund > net received | newNetReceived = 0 | OK. |
| 27 | get_booking / due | due = Final Amount − Net Received | OK. |
| 28 | Multiple payments | Each add_payment increases Net Received | OK. |

### List / filter

| # | Scenario | Current behaviour | Fix / note |
|---|----------|-------------------|------------|
| 29 | list_bookings default status = active | Reserved bookings not shown | Prompt can say “use status=reserved or all for upcoming”. |
| 30 | list_bookings status=all | All statuses | OK. |
| 31 | get_summary for cancelled booking | Allowed | OK. |

### Rounding

| # | Scenario | Current behaviour | Fix / note |
|---|----------|-------------------|------------|
| 32 | Tax/fee rounding | Math.round(finalAmount * 0.05) etc. (whole rupees) | Airbnb may show 2 decimals; we store integers. Optional: round to 2 decimals. |
| 33 | Final amount | Math.round(price×nights − discount) | OK. |
| 34 | Net = final − tax − fee | Can be 1 rupee off if tax+fee rounded separately | Acceptable. |

### Data integrity

| # | Scenario | Current behaviour | Fix / note |
|---|----------|-------------------|------------|
| 35 | edit_booking: change guest_name only | Guest Name updated; Guest ID unchanged | Bigger: optionally resolve Guest ID from name. |
| 36 | delete_booking | Deletes reservation + experiences, food, payments for that reservation | OK. |
| 37 | findBooking by guest name | Returns first match | Multiple bookings same name: agent should use ID. |
| 38 | Confirmation code (e.g. HMZZ5MHZKX) | Not stored | Bigger: add column or Notes. |

### Experiences / food / expenses

| # | Scenario | Current behaviour | Fix / note |
|---|----------|-------------------|------------|
| 39 | add_charge without booking | Error | OK. |
| 40 | add_food for non-existent reservation | findBooking fails | OK. |
| 41 | edit_transaction: experience amount | Total Amount + Net Profit updated; Price per Person not back-filled | Optional: recalc price = total/people. |

### Dashboard / report

| # | Scenario | Current behaviour | Fix / note |
|---|----------|-------------------|------------|
| 42 | Direct revenue | Non-Airbnb (Direct, WhatsApp, Instagram) | OK. |
| 43 | get_report date range | Filter by check-in and transaction dates | OK. |

---

## Quick fixes applied in script

1. **add_booking:** If check_out ≤ check_in, return error so user corrects dates.
2. **edit_booking:** When deriving Final Amount, use nights = Math.max(1, nights) so we never use 0 nights; when writing a.nights to sheet, clamp to ≥ 1 so sheet never has 0 nights.

---

## Bigger changes (plan)

1. **Airbnb screenshot / PDF parsing**  
   - Add a tool (e.g. `parse_airbnb_booking`) that accepts image or pasted text.  
   - Extract: guest name, check-in, checkout, nights, confirmation code, gross/subtotal, discount, tax, fee, net payout, booking date.  
   - Call add_booking (and optionally edit) with those fields so the user can paste a screenshot and get one created/updated.

2. **Confirmation code**  
   - Add column “Confirmation Code” (or “Airbnb Code”) to Reservations.  
   - add_booking / edit_booking accept confirmation_code; store in sheet.  
   - Helps match sheet rows to Airbnb and avoid duplicates.

3. **Guest ID on name change**  
   - When edit_booking updates guest_name, optionally lookup or create guest and set Guest ID so Reservations stays linked to Guests.

4. **Two-phase booking from Airbnb**  
   - “Guest paid” total (with guest-side tax) vs “Host payout” (our Net Received).  
   - Optional columns: Guest Paid Total, Occupancy Tax (guest-side) for reporting only; no change to recalc.

5. **Rounding to 2 decimals**  
   - Store tax/fee/net with 2 decimals if you want exact match to Airbnb; currently we use whole rupees.

6. **list_bookings default**  
   - Option to default status to “reserved” or “all” so “list upcoming” shows reserved without passing status.

7. **Validation helpers**  
   - Small validators: nights ≥ 1, discount ≥ 0, final_amount ≥ 0, dates parseable; return clear errors from add_booking / edit_booking.

Implementing the two quick fixes in the script next.
