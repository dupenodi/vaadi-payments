# Script critical analysis — enums, status, columns, recalculation

## 1. Enum / naming confusion

### Booking status
- **`BOOKING_STATUSES`** is defined as `['active', 'checked_out', 'cancelled', 'reserved']`.
- **Bug:** New reservations are created with status **`'active'`** (hardcoded in `toolAddBooking` line ~942). For a new booking that hasn’t arrived yet, status should be **`'reserved'`**. Check-in should set status to `'active'`, check-out to `'checked_out'`.
- **Fix:** Default status for new reservations → `'reserved'`. Keep `check_in_booking` setting `'active'` and `check_out_booking` setting `'checked_out'`.

### Room ID vs Reservation ID
- **Rooms** sheet uses IDs `R001`, `R002`, `R003` (Room ID).
- **Reservations** use `nextId(sheet, 'Reservation ID', 'R')` → first reservation is also `R001`.
- Same prefix `R` in different sheets can confuse users (“R001” = room or reservation?). Consider using **`RV001`** (or similar) for reservations, or **`RM001`** for rooms, to avoid ambiguity in conversation and exports.

### Experience type vs charge category
- **Tool `add_charge`** uses **`CHARGE_CATEGORIES`**: `food, vehicle, trek, activity, laundry, heater, bonfire, equipment, other`.
- **Experiences** sheet column is **“Experience Type”** with values like: Cab service, Trek, Custom Tour, Bonfire, Mattress, Horse/mule service.
- **`normaliseCategory`** maps “cab” → `vehicle`, “trek” → `trek`, “bonfire” → `bonfire`. So the sheet stores `vehicle` not “Cab service”, and dashboard/reports show “vehicle”. That splits naming: internal (`vehicle`) vs display (“Cab service”).
- **Recommendation:** Either (a) use one set of labels everywhere (e.g. “Cab service”, “Trek”, “Bonfire” in both tool schema and sheet), or (b) keep internal categories but add a display-name map for UI/reports. Today the tool description says “vehicle” but the sheet/data often use “Cab service”.

### Booking source
- **`BOOKING_SOURCES`** = `['Airbnb', 'Direct']`. Old data has “Airbnb, Direct” (combined). Code uses `source.indexOf('airbnb') !== -1`, so combined values still work. No change needed, but worth documenting that combined sources are allowed.

---

## 2. New reservation → status “reserved”

- **Current:** Every new reservation is created with `'active'`.
- **Expected:** New reservation = **`reserved`** until guest checks in; **check_in_booking** sets **`active`**; **check_out_booking** sets **`checked_out`**.
- **Fix:** In `toolAddBooking`, set status to `'reserved'` (or use a constant/default from `BOOKING_STATUSES`) instead of `'active'`.

---

## 3. Edit booking — correct columns and recalculation

### Discount / final amount / tax
- **Current:** `toolEditBooking` updates **Total Discount** and **Final Amount** only if the user passes them. It does **not** recompute **Final Amount** from (Display Price/Night × Nights − Total Discount) when only discount (or price or nights) is changed.
- **Expected:** “Add this discount to this reservation” should:
  1. Update **Total Discount** (and optionally **Display Price/Night** / **Nights** if provided).
  2. Recompute **Final Amount** = (Display Price/Night × Nights) − Total Discount (when any of these three change).
  3. Recompute **Tax (5%)**, **Airbnb Fee**, **Net Received** from the new Final Amount when source is Airbnb (and when switching to Direct, zero them).
- **Fix:** In `toolEditBooking`, when **total_discount**, **display_price_per_night**, or **nights** are updated, derive:
  - `finalAmount = (display_price_per_night * nights) - total_discount`
  then set **Final Amount** and run the existing tax/fee/net logic (same as today when `final_amount` or `booking_source` change).

### Column consistency
- Tool params use **snake_case** (e.g. `total_discount`, `final_amount`, `display_price_per_night`). Sheet columns use **Title Case** (“Total Discount”, “Final Amount”, “Display Price/Night”). The mapping in `toolEditBooking` is correct. No bug, but keep this in mind when adding new fields.

---

## 4. Add payment — due amount and Net Received

### Due is wrong
- **Current:** `toolAddPayment` uses `due = Number(booking['Due']) || 0`. The **Reservations** sheet has no **Due** column; due is derived as **Final Amount − Net Received**.
- **Effect:** `booking['Due']` is always undefined → `due` is always 0 → the check “payment exceeds due” never triggers, and remaining_due in the response is wrong.
- **Fix:** Compute due from the reservation row:
  - `due = Math.max(0, (Number(booking['Final Amount']) || 0) - (Number(booking['Net Received']) || 0))`.

### Net Received not updated
- **Current:** Adding a payment only appends a row to the **Payments** sheet. The reservation’s **Net Received** is not updated.
- **Effect:** “Due” is computed from reservation only (Final Amount − Net Received), so recording a new payment doesn’t reduce the reservation’s due unless we change Net Received.
- **Fix:** When recording a payment (non-refund), increase the reservation’s **Net Received** by the payment amount (and for refund, decrease it). So: after appending the payment row, `updateBookingRow(booking['Reservation ID'], { 'Net Received': currentNetReceived + amount })` (and subtract for refund).

---

## 5. recalcBooking and Payments

- **Current:** `recalcBooking` only reads **Final Amount** and **Net Received** from the reservation and returns `{ paid: netRecv, due: finalAmt - netRecv }`. It does **not** sum the **Payments** sheet.
- **Design choice:** If we keep “Net Received” as the single source of truth and update it whenever we add a payment (see above), then `recalcBooking` stays correct. If we ever want “paid” = sum(Payments), we’d need to sum the Payments sheet and optionally sync that to Net Received. For now, “update Net Received when adding a payment” is the minimal fix and keeps one source of truth.

---

## 6. edit_transaction (experience) — Price per Person

- When editing an experience’s **amount** (Total Amount), the code updates **Total Amount** and **Net Profit** but not **Price per Person** or **No. of People**. So Total Amount can become inconsistent with (Price per Person × No. of People).
- **Optional improvement:** When amount is updated, set **Price per Person** = Total Amount / No. of People (if No. of People > 0). Low priority.

---

## 7. Tool descriptions and schema

- **add_booking:** Description says “optional discount, tax, fee, net_received”. Correct. Consider mentioning that new bookings are created with status **reserved**.
- **edit_booking:** Mention that updating discount/price/nights will recalculate final amount and (for Airbnb) tax, fee, net received.
- **add_payment:** Schema says “booking_id like B001” but IDs are now R001 (Reservation ID). Update description to “Reservation ID e.g. R001 or guest name”.
- **add_charge:** Same: “booking_id like B001” → “Reservation ID e.g. R001 or guest name”. Category list could be aligned with Experience Type (e.g. Cab service, Trek, Bonfire) for consistency.

---

## 8. Summary of fixes to implement

| # | Issue | Fix |
|---|--------|-----|
| 1 | New reservation status | Default to `'reserved'` in `toolAddBooking`. |
| 2 | Edit booking recalc | When total_discount / display_price_per_night / nights change, recompute Final Amount and then tax/fee/net. |
| 3 | add_payment due | Compute due as Final Amount − Net Received from reservation row. |
| 4 | add_payment Net Received | After adding a payment row, update reservation’s Net Received by +amount (or −amount for refund). |
| 5 | Experience type enum | Optionally align CHARGE_CATEGORIES / tool description with “Cab service”, “Trek”, “Bonfire”, etc., or add display-name mapping. |
| 6 | Tool descriptions | Use “Reservation ID (e.g. R001)” and mention “reserved” and recalc behaviour where relevant. |

Implementing 1–4 is essential; 5–6 improve consistency and UX.
