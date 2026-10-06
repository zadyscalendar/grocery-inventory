# Grocery register and owner manager

Two offline screens that share one data format:

| File | What it is |
|---|---|
| `checkout_terminal.html` | The register the cashier and customer see |
| `owner_manager.html` | The owner's back-office screen |
| `sw.js`, `*.webmanifest`, `icon-*.png` | Make both screens installable and available offline |
| `inventory_db.sample.json` | Practice data: 14 items, 4 promotions, 3 customers, 2 device profiles |

No frameworks and no CDNs. Everything is stored in the browser on the computer that runs it. Selling needs no
internet. Only the optional Wi-Fi sync uses it, for a moment, so computers can find each other.

## Run it

Keep all the files in one folder and serve that folder from the same computer, then open the pages at a
`localhost` address. For example, with Python installed:

    python -m http.server 8000

- Manager: `http://localhost:8000/owner_manager.html`
- Register: `http://localhost:8000/checkout_terminal.html`

Opening the files by double-click also works for a quick look, but browsers only allow the offline cache
and app install from `localhost` (or HTTPS). Once a page has loaded from `localhost` it keeps working with
the server stopped. Use Chrome or Edge.

## First run

1. Open the manager and choose a 4-digit PIN.
2. Add items, customers, promotions and card devices, or press **Import a sync file** and pick
   `inventory_db.sample.json` to practise.
3. Press **Compile & Export Sync File** and enter the PIN. The browser saves `inventory_db.json`.
4. On the register computer, drag that file onto the checkout screen and enter the PIN.

If the manager and the register run in the same browser on one computer, they already share live data and
step 3 and 4 are not needed.

## Wi-Fi sync between computers

With this on, registers send every sale to the owner's computer (the host) and pick up price, promotion,
account and setting changes on their own. There is no database server and nothing to install.

**Set up, once**

1. On the owner's computer: owner manager, **Settings**, switch on **Wi-Fi sync**. Keep that page open; it
   keeps hosting while locked.
2. On each register: open the checkout page. A new register asks for its host and lists the hosts it can
   see by name. Pick yours. (If none appears, type the host code shown in the owner's Settings.)
3. On the owner's computer a request appears with a 4-digit code. If the register shows the same code,
   press **Allow**. The register then loads the shop's data by itself, with no file to carry.

To change a register's host later, press **Alt+Shift+H** on it and enter the owner PIN.

**How it decides what is right**

- Every owner change is timestamped, and the newest timestamp wins on every computer.
- Sales are never overwritten. Each has an ID; the host applies each one once to stock and balances.
- If the host is off or the Wi-Fi drops, registers keep selling and show how many sales are waiting.
  They send them when the host is back.
- If the host computer is replaced, set a PIN on the new one, switch on Wi-Fi sync, and link the
  registers to it. It starts empty and takes the newest copy from the first register that joins.
- A timestamp more than ten minutes in the future (a wrong clock) is ignored.

**What it relies on**

- All computers on the same Wi-Fi or wired network, with "client isolation" off on the router.
- Internet on that network at the moment a page opens or reconnects. Two free public services are used
  only to make the introduction: `ntfy.sh` carries a short "how to reach me" note each way, and
  `api.ipify.org` tells a new register which internet connection it is on so it can list nearby hosts.
  No prices, sales or customer details pass through either. If they are unreachable, computers already
  connected carry on, new connections wait, and the file method below still works.
- Chrome or Edge on every computer.

A register should be linked before it starts trading. Sales made on a never-linked register are sent to
the host as history when it joins, but their stock changes are not; pull that register's file into the
manager first if it has traded.

## Keeping two computers in step with a file

The register changes stock counts, customer balances and the sales list. To bring those back:

1. On the register press **Alt+Shift+E**, enter the PIN, and save the file it exports.
2. In the manager press **Import a sync file** and choose **Pull register activity**. Stock, balances and
   sales come in; your prices, promotions, devices and settings stay.
3. Make your changes, export, and drop the new file on the register.

Do the pull before you export, otherwise the register's stock and balances are overwritten by older
numbers (the register warns you when that is about to happen, and when a file is older than what it
already has). This loop is built for one register per manager; use Wi-Fi sync for more.

## Register keys

| Key | Action |
|---|---|
| F2 / F3 / F4 | Pay cash / Card / Charge to Account |
| Alt+Shift+E | Owner: export this register's data (PIN) |
| Alt+Shift+D | Owner: change which card device this register uses (PIN) |
| Alt+Shift+H | Owner: choose or change this register's Wi-Fi sync host (PIN) |

There are no visible admin controls on the register. Typing always goes to the scanner box unless the
phone field or a dialog is open.

## Receipt printing with no clicks

Browsers show a print dialog unless told otherwise. For true zero-click printing:

1. Make the receipt printer the default printer on the register computer.
2. Start Chrome with kiosk printing, for example:

       chrome --kiosk --kiosk-printing http://localhost:8000/checkout_terminal.html

Without the flag the first print shows the dialog; pick the receipt printer and the browser remembers it.
Receipts are laid out for 80 mm paper. The owner's **Receipt printing** switch turns all printing off.

## Card devices

The owner registers each device under **Payment devices**. The first card sale on a register asks which
device that register uses and remembers the answer.

For a sale the register sends the amount and waits up to two minutes:

    { "type": "sale", "amount": "24.18", "amount_cents": 2418,
      "currency": "USD", "reference": "SALE-ID", "station": "REG-AB12" }

- **LAN address** such as `110.12.0.45:8080/v2/sale`: sent as an HTTP POST. A `ws://` address uses a
  WebSocket instead.
- **Serial port**: written as one line; the port is chosen once on the register (Chrome or Edge only).
- The sale completes only when the reply's `status` is `approved` (also `success`, `ok`, `paid`).
  `declined`, `failed`, `cancelled`, anything unrecognised, or no answer leaves the sale open and offers
  **Send again** or **Manual Standalone Card Device Capture**.

Clover, Verifone, Ingenico, PAX and Square terminals each speak their maker's own protocol and need
pairing. They will not answer this message directly. Connecting one takes a small bridge program on the
LAN that receives the message above, drives the terminal with the maker's SDK, replies with the status,
and allows cross-origin requests (CORS). Until then, switch on **Manual Standalone Card Device Capture**
and key amounts on the device.

`sim://approve` and `sim://decline` are practice addresses that charge nothing; receipts say "Test mode".
Delete the practice profile before trading.

## How promotions are applied

- **Sale price** on an item always applies.
- **Percentage** and **dollar** promotions are coupons: they apply once the cashier scans or types the code.
  A dollar coupon takes that amount off each matching unit, or once off the whole order when it applies to
  all. A promotion whose code is an item's own barcode applies whenever that item is sold.
- **Clearance** is always on, is a percentage off the regular price, and coupons skip clearance items.
- Tax (`settings.tax_rate_percent`) is charged on items marked as taxed, after discounts.

## Customer credit

Accounts are looked up by phone number. **Charge to Account** adds the sale to the balance and, for a
first unpaid charge, sets the due date to today plus the owner's payment window. An account past its due
date with a balance is frozen automatically: the sale is locked until the balance is collected at the
register (or the customer is removed from the sale and pays another way).

## Things to know

- The PIN keeps staff out of owner functions. It is not encryption: a 4-digit PIN can be guessed by
  someone who has the exported file, and that file holds customer names, phone numbers and balances in
  plain text. Treat exports like the cash drawer.
- Clearing the browser's site data erases the shop's data on that computer. Export regularly and keep the
  files as backups.
- Wi-Fi sync was tested with a stand-in for the public introduction service, on one machine. Try it on
  your own network with two computers before relying on it.
- After replacing any of these files with a newer version, change `CACHE` in `sw.js` (for example
  `grocer-pos-v3`) so browsers pick up the update.
