# What to do in Google Sheets (max capabilities)

The script creates and manages all tabs for you. You only do these steps once.

---

## 1. Create or open a Google Sheet

- Go to [sheets.google.com](https://sheets.google.com) and create a **new blank spreadsheet**, or open an existing one you want to use.
- If you use an existing sheet, the script will **add new tabs** and will **not** delete your other tabs. Only tabs named exactly as below are created/overwritten when you run `setupSheets()`.

---

## 2. Get the Sheet ID

- Open the spreadsheet.
- Look at the URL. It looks like:
  `https://docs.google.com/spreadsheets/d/ **SHEET_ID_IS_HERE** /edit`
- Copy the long string between `/d/` and `/edit`. That is your **Sheet ID**.

---

## 3. In Apps Script: set Script Properties

- Open your Apps Script project (script.google.com) that contains the Cabin Manager script.
- Go to **Project Settings** (gear icon) → **Script Properties**.
- Add or edit:
  - **`SHEET_ID`** = the Sheet ID you copied (no spaces, no quotes).
  - **`ANTHROPIC_API_KEY`** = your API key from console.anthropic.com (if not already set).
- Save.

---

## 4. Run setup once

- In the Apps Script editor, open the script file.
- Select the function **`setupSheets`** in the dropdown at the top.
- Click **Run** (▶).
- The first time, authorize the app when Google asks (view/manage spreadsheets, etc.).
- When it finishes, your spreadsheet will have **6 tabs** (see below). You do **not** need to create or edit columns manually.

---

## 5. Deploy as web app (if not already)

- In Apps Script: **Deploy** → **New deployment** → type **Web app**.
- **Execute as:** Me  
- **Who has access:** Anyone (or “Anyone with Google account” if you prefer)
- Deploy, then copy the **Web app URL** into `index.html` as `BACKEND_URL`.

---

## Tabs the script creates (you don’t create these by hand)

| Tab name    | Purpose |
|------------|---------|
| **Guests** | One row per guest (ID, name, phone, email, notes, created, last used). The assistant reuses guests when the same name/phone books again. |
| **Rooms**  | One row per room/cabin (ID, name, default rate, status). One default room “Main Cabin” is created. Add more rooms here if you have multiple units. |
| **Bookings** | One row per stay. Links to Guest ID and Room ID. Includes dates, rate, amounts, paid/due, status, and optional check-in/check-out timestamps. |
| **Charges** | One row per extra charge (food, vehicle, etc.). Linked to Booking ID. |
| **Payments** | One row per payment (cash, UPI, etc.). Linked to Booking ID. |
| **AuditLog** | One row per important action (create/update/delete). Timestamp, action, entity type, ID, and details for debugging and compliance. |

---

## After setup

- You can **browse and edit** the sheet as usual; the assistant reads and writes these tabs via the script.
- To **add more rooms**: add a row in the **Rooms** tab (Room ID like R002, Name, Default Rate, Status, Notes). Or ask the assistant to “add a room” if you add that tool.
- To **reset** and recreate all 6 tabs from scratch, run **`setupSheets()`** again. This **overwrites** the tabs named above and **deletes their data**; other tabs in the same spreadsheet are untouched.

---

## Summary checklist

- [ ] Create or open a Google Sheet  
- [ ] Copy Sheet ID from the URL  
- [ ] In Apps Script: set Script Property `SHEET_ID` (and `ANTHROPIC_API_KEY`)  
- [ ] Run **`setupSheets()`** once  
- [ ] Deploy as web app and set `BACKEND_URL` in index.html  

You do **not** need to create tabs or columns manually; the script does it.
