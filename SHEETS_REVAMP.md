# Vaadi Cabin — Revamped Sheet Structure

Full cabin: **2 rooms** (upper = **Sky View Cabin**, lower = **Forest View Cabin**) or **Vaadi - Entire Villa** (both). Bookings can be any of these three.

---

## Comma-separated headers (copy into row 1 of each sheet)

### 1. Guests
```
Guest ID,Name,Phone,Email,Notes,Created At,Last Used At
```

### 2. Rooms
```
Room ID,Name,Type,Default Rate,Status,Notes
```
*Type: `single_room` | `full_villa`*

Seed rows:
- `R001,Sky View Cabin,single_room,6999,active,Upper room`
- `R002,Forest View Cabin,single_room,6999,active,Lower room`
- `R003,Vaadi - Entire Villa,full_villa,21000,active,Both rooms`

### 3. Reservations (main stay bookings)
```
Reservation ID,Guest ID,Guest Name,Check-in,Booking Type,Nights,Booking Source,Display Price/Night,Total Discount,Final Amount,Tax (5%),Airbnb Fee,Net Received,Payout Date,Booking Date,Tip,Payee,Status,Notes,Created On,Last Updated
```
*Check-in = reservation/arrival date. Booking Type = Sky View Cabin | Forest View Cabin | Vaadi - Entire Villa. Booking Source = Airbnb | Direct | WhatsApp | Instagram.*

### 4. Experiences (trek, cab, bonfire, etc.)
```
Txn ID,Reservation ID,Date,Guest Name,Experience Type,No. of People,Price per Person,Total Amount,Cost,Net Profit,Payment Mode,Notes,Created At
```
*Experience Type: Trek | Forest Walk | Meditation | Camping | Custom Tour | Bonfire | Cab Service | Mattress | Horse/Mule service*

### 5. Food (per guest / stay)
```
Txn ID,Reservation ID,Date,Guest,Stay Nights,People,Rate per Person per Day,Base Food Revenue,Food Cost,Special Request Charge,Total Food Revenue,Food Profit,Payee,Notes,Created At
```

### 6. Payments (payments against reservations or general)
```
Txn ID,Reservation ID,Date,Amount,Method,Reference,Payee,Notes,Created At
```
*Method: cash | upi | bank_transfer | advance | refund*

### 7. Expenses (operational)
```
Txn ID,Date,Category,Sub-category,Description,Amount,Paid From,Created At
```
*Category: Groceries | Cleaning | Staff | Utilities | Internet | Maintenance | Marketing | Transport | Supplies | Misc | Electricity*

### 8. AuditLog
```
Timestamp,Actor,Action,Entity Type,Entity ID,Details
```

### 9. Dashboard (detailed metrics — run refreshDashboard() to fill values)
```
Section,Metric,Value,Period,Updated At
```
*Sections: Summary (totals, revenue, profit, outstanding), Month (this month), By Type (Sky View / Forest View / Entire Villa count & revenue), Experiences (by type), Expenses (by category).*

---

## Sheet summary

| Sheet         | Purpose |
|---------------|---------|
| **Guests**    | One row per guest; reused across reservations. |
| **Rooms**     | 3 products: Sky View Cabin, Forest View Cabin, Vaadi - Entire Villa. |
| **Reservations** | One row per stay: guest, booking type (room), nights, source, pricing, net received, payee. |
| **Experiences**  | Cab, trek, bonfire, mattress, etc.: revenue, cost, net profit. |
| **Food**      | Food revenue per guest/stay: rate, people, revenue, cost, profit, payee. |
| **Payments**  | Payment transactions (method, reference, payee) linked to reservation or standalone. |
| **Expenses**  | Operational spend: electricity, internet, groceries, staff, maintenance. |
| **AuditLog**  | Who did what, when. |
| **Dashboard** | Key metrics (reservations count, revenue by source, experience/food profit, expenses, net). |

Running `setupSheets()` in the script will create these 9 tabs and seed Rooms + Dashboard labels.
