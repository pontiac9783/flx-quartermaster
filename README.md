# FLX Quartermaster

Inventory, point-of-sale, member tabs and ledger for **Punishers LEMC — Finger Lakes**.
Installable web app (PWA) for Android, iPhone/iPad and PC. Free to run.

**App:** https://pontiac9783.github.io/flx-quartermaster/

---

## Features

| Area | What it does |
|---|---|
| **Sell** | Tap-to-sell tiles with pictures, cart, per-line price override, payment method, buyer name. Out-of-stock items hidden by default. |
| **Shipping** | Optional per-item shipping rate. **Add shipping** in the cart (or **Add shipping to a past sale** / Ledger → **Ship** afterwards) suggests an editable amount and ship-to. Not allowed on Club Account (tab) sales. Recorded as a separate `shipping` ledger line with the same Sale ID, kept out of revenue/profit, included in takings. One shipping line per sale (void to change). |
| **Cart holds** | Items in a cart are held for that device so two people can't sell the same last unit. Holds renew while the app is open and expire 10 minutes after it's closed; completing or clearing the sale releases them. |
| **Tabs** | Club Account (member tab) charges, balances, partial/full payments, per-member history. |
| **Events** | Admins start/end an event (e.g. a rally) and set rider/passenger fees. While running, sales, shipping, tab charges and tab payments are tagged; optional event-only prices (devices re-price carts automatically if prices change; stale checkouts are rejected); opening/closing cash count with over/short. |
| **Event money** | Anyone can record rider/passenger fees, 50/50 drawings (auto half payout), basket drawing tickets, cash raffle (tickets + prize), other income and out-of-pocket costs (paid from cash box, card, or personally → reimbursement list). Included in the cash box check and the event net (merch profit + income − payouts − costs). |
| **Bar** | Tablet-first Bar screen: tap drinks to build a round, then tap **Cash** or a member's name to charge it — one write per round. Member/guest pricing, comps with reason, quick items, per-drink ✕ before charging and **↺ undo** of the last round. Bar shifts have their own cash box (open/close count, over/short) and can be counted toward a running event. |
| **Receiving** | One **📦 Receive** screen: low-stock shortcuts, type-and-Enter search, qty steppers, unit cost per line, one note (supplier / who paid) — saved as a single batch. Spirits are entered in bottles. |
| **Spirits** | Bottle size + pour size → pours per bottle. Stock is held in pours; receive in bottles, count as bottles + quarters. Mixed drinks deduct pours from their liquor. |
| **Item details** | Tap an item name (Stock, Reports → Top sellers): picture, price/cost/margin, on hand & held, days of stock left, sales this month / 30 days / all time, profit, 12-week trend, per-size breakdown, recent activity. |
| **Stock** | Items with category, size/variant, auto SKU, price, cost, reorder level, picture, description. Receive, physical count, loss/damage, comp. |
| **Sale pricing** | Per item or bulk (whole product / category). Regular price is kept; ending a sale restores it. |
| **Ledger** | Append-only record of every sale, receipt, adjustment, tab payment. Mistakes are **voided**, never deleted. |
| **Reports** | Revenue, COGS, gross profit, takings by payment method (incl. tab payments), discounts, top sellers, reorder list, shrinkage. |
| **Settings** | Club/chapter name, payment methods, categories (rename/merge/delete updates items), users & roles. |

### Roles

| | User | Admin |
|---|:---:|:---:|
| Sell (incl. Club Account) | ✅ | ✅ |
| Tabs — view & record payments | ✅ | ✅ |
| Stock — view on hand, prices, costs, stock value | ✅ | ✅ |
| Reports (incl. COGS, profit) | ✅ | ✅ |
| Add/edit items, receive, count, write-off, sale pricing | ❌ | ✅ |
| Ledger, void | ❌ | ✅ |

A third role, **Bar tablet**, sees only Bar, Tabs, Stock and device settings — no costs, no Sell/Ledger/Reports. Intended for a shared tablet account left signed in at the bar; the shift records who is behind the bar.
| Settings, categories, user management | ❌ | ✅ |

Roles are enforced server-side: all changes go through the API and are checked against the user's role. Users have read-only access to stock and reports for transparency; nobody except the owner needs access to the Sheet itself. The Sheet owner is always admin.

---

## Architecture

```
Phone / Tablet / PC (installed PWA)
        │  Google Sign-In (ID token)
        ▼
GitHub Pages  ── index.html, sw.js, manifest, icons, config.js
        │  HTTPS POST  { fn, arg, idToken }
        ▼
Google Apps Script "App API" (Execute as: owner, Access: Anyone)
        │  verifies token → checks allow-list & role → runs action
        ▼
Google Sheet  ── Items · Ledger · Settings · Images
```

- **No server or database to host.** The Google Sheet is the database.
- **Access control:** a request must carry a valid Google ID token for this app's Client ID **and** the email must be on the `Settings` allow-list. The Apps Script URL is public, but useless without both.
- **Members do not need access to the Sheet.**
- **Offline:** the app opens offline (cached shell) but cannot load or save data until reconnected.

---

## Repository files

| File | Purpose |
|---|---|
| `index.html` | The entire app (UI + logic) |
| `config.js` | **Only file with settings** — API URL and Client ID |
| `sw.js` | Service worker (offline shell + updates). Bump `VERSION` on every release. |
| `manifest.webmanifest` | App name, colors, icons |
| `icon-192.png`, `icon-512.png`, `icon-maskable-512.png`, `apple-touch-icon.png`, `favicon.png` | App icons |
| `.nojekyll` | Serve files as-is on GitHub Pages |

The backend (`Code.gs`, `appsscript.json`) lives in the Apps Script project bound to the Sheet, not in this repo.

---

## Setup (one-time)

### 1. Google Sheet + Apps Script
1. Create a Google Sheet → **Extensions → Apps Script**.
2. Paste `Code.gs`. Enable **Project Settings → Show "appsscript.json"** and paste `appsscript.json`.
3. Run **`setup`** once and approve permissions. Creates `Items`, `Ledger`, `Settings`, `Images`.

### 2. Google Cloud (sign-in)
1. [console.cloud.google.com](https://console.cloud.google.com) → new project.
2. **Google Auth Platform → Branding:** app name + support email.
3. **Audience:** External, leave in **Testing**, add every member under **Test users** (limit 100).
4. **Clients → Create client → Web application**
   - Authorized JavaScript origins: `https://<github-username>.github.io`
   - Copy the **Client ID**. The client secret is not used.

### 3. Apps Script API deployment
1. In `Code.gs`, set `OAUTH_CLIENT_ID` to the Client ID. Save.
2. **Deploy → New deployment → Web app**
   - Execute as: **Me**
   - Who has access: **Anyone**
3. Copy the `/exec` URL.

### 4. GitHub Pages
1. Upload the repository files.
2. **Settings → Pages →** Deploy from branch `main` / `(root)`.
3. Edit `config.js`:
   ```js
   window.QM_CONFIG = {
     API_URL: 'https://script.google.com/macros/s/XXXX/exec',
     CLIENT_ID: 'XXXX.apps.googleusercontent.com'
   };
   ```

---

## Installing the app

Open the app link **in a real browser**, not inside Messenger/Facebook/Instagram (Google blocks sign-in there; the app detects this and shows an **Open in Chrome** button).

| Device | Steps |
|---|---|
| Android | Chrome → sign in → **Install app** button (or ⋮ → **Install app** / **Add to Home screen**) |
| iPhone / iPad | Safari → **Share → Add to Home Screen** |
| PC | Chrome / Edge → install icon in the address bar |

---

## Adding a member

Both steps are required:
1. **App → Settings → Users → Add** (email + role).
2. **Google Cloud → Google Auth Platform → Audience → Test users → Add users.**

Send them the app link and install steps. To remove access, remove them in **Settings → Users** (immediate).

---

## Updating

| Change | Steps |
|---|---|
| `index.html` / app files | Upload to GitHub → bump `VERSION` in `sw.js` (e.g. `qm-v20` → `qm-v21`) → commit. The app detects the new version on next open and reloads itself ("App updated"). |
| `Code.gs` | Save → **Deploy → Manage deployments →** select the API deployment → ✏️ → **Version: New version** → Deploy. URL does not change. |
| New columns / sheets | Run **`setup`** again after pasting `Code.gs`. Safe to re-run; it never deletes data. |

> ⚠️ Never **Archive** the API deployment — archived deployments stop working and cannot be restored. A new deployment means a new URL in `config.js`.

---

## Data model (Google Sheet)

**Items**

| A | B | C | D | E | F | G | H | I | J | K | L | M |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ItemID | Name | Category | Size | SKU | Price | UnitCost | ReorderAt | Active | Created | OnHand* | StockValue* | SalePrice |

| N | O | P | Q | R | S | T | U | V | W | X | Y |
|---|---|---|---|---|---|---|---|---|---|---|---|
| OnSale | Description | ImageID | ShipPrice | GuestPrice | Section | MemberOnly | BottleSize | PourOz | PoursPerBottle | PoursFromItem | PoursPerDrink |

\* Formula columns — do not edit.

**Ledger** (append-only)

| A | B | C | D | E | F | G | H | I | J |
|---|---|---|---|---|---|---|---|---|---|
| EntryID | Timestamp | Type | SaleID | ItemID | ItemName | Category | Qty | Unit | Total |

| K | L | M | N | O | P | Q | R | S | T | U | V | W |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| UnitCost | Payment | RecordedBy | Party | Note | VoidsEntry | Voided | ListPrice | PriceType | OnTab | EventID | ShiftID | Section |

Types: `sale`, `shipping`, `receive`, `adjust`, `loss`, `comp`, `payment` (tab payment), `income` / `expense` (event money; kind in `Category`: riders, passengers, fifty, basket, raffle, otherinc, payout, expense), `void`.
Stock on hand = sum of `Qty` for non-voided entries.

**Settings**

| Cell / range | Contents |
|---|---|
| `B1` | Club name |
| `B2` | Chapter |
| `A4↓` | Merch categories |
| `B4↓` | Payment methods |
| `C4↓` | Payment methods that require a member name (tabs) |
| `D4↓` | Allowed Google emails |
| `E4↓` | Role (`admin` / `user` / `bar`) |
| `F4↓` | Bar categories (separate list from merch) |
| `G4↓` | Members (roster for tabs and the bar) |
| `H4↓` | Member name linked to each login (same row as the email in `D`) |

**Events** — `EventID`, `Name`, `StartedAt`, `EndedAt`, `StartedBy`, `EndedBy`, `OpeningCash`, `ClosingCash`, `Notes`, `PricesJSON` (event price per item ID), `RiderFee`, `PassengerFee`. Only one event can run at a time. Price precedence: event price → sale price → regular.

**Shifts** — `ShiftID`, `OpenedAt`, `ClosedAt`, `Bartender`, `OpenedBy`, `ClosedBy`, `OpeningCash`, `ClosingCash`, `EventID`, `Notes`. One shift open at a time; the bar cash box is separate from merch and event cash.

**Images** — one row per picture (`ImageID`, base64 thumbnail ≤ ~45 KB, created, uploaded by).

> Don't hand-edit `Ledger` rows. Use **Void** in the app so history stays intact.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Blank page after Google sign-in on phone | Opened inside Messenger/Facebook. Use **Open in Chrome** / open the link in Chrome or Safari. |
| "Access blocked" on Google screen | Email not in **Google Cloud → Audience → Test users**. |
| "… is not authorized for the Quartermaster app" | Email not in **Settings → Users** (or typo). The message shows the signed-in email. |
| "Server rejected sign-in: Server is missing OAUTH_CLIENT_ID" | Client ID not saved in `Code.gs`, or the API deployment wasn't updated to a **New version**. |
| "Admins only" | Signed-in account has the `user` role. |
| App still shows old version | Confirm `sw.js` `VERSION` was bumped and the Pages build finished; close/reopen the app. |
| Sign-in button does nothing | Allow pop-ups for the site. |
| Server errors | Apps Script → **Executions** → latest `doPost` shows the error. |

---

## Limits

- Google Auth **Testing** mode: max 100 lifetime test users.
- Pictures are small thumbnails stored in the Sheet (not full-size photos).
- The app loads the full ledger on open; comfortable to ~10–20k ledger rows. Archive old years to a separate tab beyond that.
- Apps Script and Sheets quotas far exceed chapter-store volume.
