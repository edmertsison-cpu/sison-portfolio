# Sison Family Portfolio — Requirements & Specification

| | |
|---|---|
| **Product** | Sison Family Portfolio |
| **Type** | Installable Progressive Web App (PWA), mobile-first |
| **Live URL** | https://edmertsison-cpu.github.io/sison-portfolio/ |
| **Repository** | `edmertsison-cpu/sison-portfolio` (public, code only) |
| **Current version** | v11 (2026-10-22) — see `what-changed.md` |
| **Owner** | Tep Sison |
| **Users** | Tep and Jeng (household) |
| **Status** | In use. Credit card tracking live. Other account and investment details pending requirements. |

> **Privacy rule for this repo:** No holdings, amounts, account numbers, or backups may ever be committed. The repo holds only the app's code. Portfolio data lives on each device and in the owner's Google Drive.

---

## 1. Purpose

Give the household one place to see the **total value of all family assets**, split into **Philippines** and **Global**, with any kind of asset (stocks, ETFs, mutual funds, MP2, PERA, VUL, bonds, gold, crypto, cash, real estate, and others). The app tracks how that total changes over time and keeps an audit trail of every edit.

### 1.1 Goals
- G1. See gross portfolio value at a glance on a phone, in PHP, with a USD equivalent.
- G2. Accept any asset type with no fixed list.
- G3. Keep a tamper-resistant history of every change, plus a net-worth-over-time chart.
- G4. Keep two phones in sync through Google Drive.
- G5. Keep data private: nothing in the public repo, and the app locked behind phone biometrics.
- G6. Track credit card cut-offs, due dates, amounts, and payments so no due date is missed.

### 1.2 Non-goals (current scope)
- Live market prices or brokerage integrations (prices are entered by hand).
- Cost basis, gain/loss, dividends, or yield tracking.
- Liabilities and net worth after loans. Credit card balances are tracked but not subtracted from Gross Portfolio.
- Multi-user accounts with separate permissions.
- Native App Store or Play Store apps.

---

## 2. Platform & Architecture

| Layer | Choice |
|---|---|
| Front end | Single `index.html` with vanilla JS, inline CSS, and inline SVG charts. No framework or build step. |
| Hosting | GitHub Pages (public repo, `main` branch, root folder) |
| Offline / install | `manifest.webmanifest` + `sw.js` service worker |
| Local storage | Browser `localStorage`, per device |
| Cloud sync | Google Drive REST API v3, OAuth 2.0 implicit flow, scope `drive.file` |
| App lock | WebAuthn platform authenticator (device passkey): Face ID, fingerprint, or passcode |
| Fonts | Google Fonts: Montserrat (display), Instrument Sans (body), IBM Plex Mono (numbers) |

### 2.1 Files

| File | Purpose |
|---|---|
| `index.html` | The whole app: markup, styles, logic |
| `sw.js` | Offline cache. `VERSION` (`portfolio-vN`) must be bumped on every release. |
| `manifest.webmanifest` | Install metadata: name, icons, colors, standalone display |
| `config.js` | `window.GOOGLE_CLIENT_ID`, the OAuth Web client ID (not a secret) |
| `icon-192.png`, `icon-512.png`, `icon-maskable-512.png`, `apple-touch-icon.png` | App icons (family logo emblem) |
| `logo.png` | Logo shown in the header, menu, and lock screen |
| `what-changed.md` | Release notes, newest first |
| `SPEC.md` | This document |
| `.gitignore` | Blocks `*backup*.json` from being committed |

### 2.2 Service worker caching rules
- Page navigations are **network first**, with the cache as an offline fallback.
- `config.js` is **network first**, so a changed client ID applies immediately.
- Other same-origin files and Google Fonts are **cache first**.
- Google APIs (`googleapis.com`, `accounts.google.com`) are **never cached**.
- Activating a new `VERSION` deletes old caches.

---

## 3. Data Model

### 3.1 Holding
```json
{
  "id": "a<base36>",          // unique, generated
  "region": "PH | GLOBAL",
  "type": "free text",          // e.g. Stock, ETF, MP2, Gold. Empty means "Other"
  "name": "code or name",       // required
  "desc": "optional note",
  "qty": 1200 | null,           // positions (shares/units); null = lump-sum asset
  "price": 15.38,               // per-unit price, or the total amount when qty is null
  "currency": "PHP | USD"
}
```
**Value rule:** `value_PHP = (qty != null ? qty × price : price) × (currency = USD ? fx : 1)`

### 3.2 Credit card
```json
{
  "id": "c<base36>",
  "name": "BDO Visa Tep",       // nickname, letters and spaces only; card numbers are never stored
  "cutoff": 15 | null,            // cut-off day of the month, 1–31 (months shorter than this use their last day)
  "limit": 150000 | null,         // credit limit, PHP
  "dueDate": "YYYY-MM-DD" | "",   // statement due date
  "dueAmt": 24500.5 | null,       // statement due amount, PHP
  "paidOn": "YYYY-MM-DD" | "",    // paid / settled on
  "paidType": "full | minimum | \"\""  // whole amount, minimum, or not yet paid
}
```

### 3.3 Settings
```json
{ "fx": 62.63, "asOf": "YYYY-MM-DD" }
```

### 3.4 History
```json
{
  "log": [ {
    "id": "h<base36>", "at": "ISO timestamp",
    "action": "start | add | edit | delete | settings | restore-backup | restore-version | keep-drive | keep-local | card-add | card-edit | card-delete",
    "asset": { "name", "type", "region", "desc" },          // add / edit / delete
    "card": { "name" },                                     // card-add / card-edit / card-delete
    "txn": { "add": 0, "sub": 0, "unit": "units | amount", "cur": "PHP | USD" }, // edit via Added / Sold
    "changes": [ { "f": "price", "a": 15.38, "b": 16.00 } ], // edit / settings
    "value": 0,                                             // add / delete
    "before": 0, "after": 0,                                // portfolio total in PHP
    "prev": { "holdings": [], "settings": {} },             // restores and sync choices
    "from": "YYYY-MM-DD", "count": 0
  } ],
  "snaps": {
    "YYYY-MM-DD": { "at", "holdings", "settings", "total", "ph", "gl", "count" }
  }
}
```

### 3.5 Storage keys (per device)

| Key | Contents |
|---|---|
| `sison-portfolio-v1` | `{ holdings, settings, cards, savedAt }` |
| `sison-portfolio-history` | `{ log, snaps }` |
| `sison-portfolio-drive` | Drive connection: token, expiry, fileId, version, lastSync |
| `sison-portfolio-lock` | `{ enabled, credId, since }` |
| `sison-portfolio-page` | Last page opened |

### 3.6 Drive file
- Name: **`Sison Family Portfolio data.json`** (in My Drive)
- Body: `{ app: "sison-portfolio", version: 3, holdings, settings, cards, history, savedAt }`
- Backup files use the same shape, so a backup can be restored into any device.

---

## 4. Functional Requirements

### 4.1 Dashboard
| ID | Requirement |
|---|---|
| FR-D0 | All titles, section headings, and header buttons use Title Case (e.g. *By Asset Type*, *Add Asset*). **SISON FAMILY PORTFOLIO** and **GROSS PORTFOLIO** are always all caps. |
| FR-D1 | Header shows the logo, a **GROSS PORTFOLIO** title, and the as-of date (e.g. *October 11, 2026*). |
| FR-D2 | Summary shows the gross total in PHP, the USD equivalent, the position count, and Philippines / Global subtotals with % and a split bar (PH gold, Global green). |
| FR-D3 | A donut chart shows allocation by asset type, with a legend and percentages. |
| FR-D4 | **By asset type**: one row per type with total value, position count, and % of total. Each full-width bar is split by that type's PH (gold) vs Global (green) share, so a type held only in one region is a single color. |
| FR-D5 | **Philippine assets** and **Global assets** sections list each asset with Positions, Market price, and Total value (PHP, plus USD for USD assets). |
| FR-D6 | Within each section, assets are grouped by type (groups ordered by subtotal, highest first) and sorted by total value within each group. |
| FR-D7 | Lump-sum assets (no positions) show "—" for positions and price. |
| FR-D8 | Layout adapts to phone width with no horizontal scrolling. |

### 4.2 Asset management
| ID | Requirement |
|---|---|
| FR-A1 | **Add asset** opens a form: Category (Philippines/Global), Asset type (free text with suggestions), Code/name, Description, Positions, Price currency, and Market price (or Total amount when Positions is blank). |
| FR-A2 | Choosing Global defaults the currency to USD; Philippines defaults to PHP. |
| FR-A3 | Validation: a name is required, the price must be a number ≥ 0, and positions (if given) must be a number ≥ 0. Commas are allowed. |
| FR-A4 | Tapping an asset row opens it for editing. **Delete** needs two taps to confirm. |
| FR-A5 | **FX & date** sets the USD/PHP rate (> 0) and the as-of date. |
| FR-A6 | Any add, edit, or delete sets the as-of date to today. |
| FR-A7 | When editing an asset, an **Add or Remove** box offers **Added / Deposited** and **Sold / Withdrawn** (numbers ≥ 0). These are one-time entries, not stored fields. |
| FR-A8 | For assets with positions, the values are shares/units: new positions = current + added − sold. For lump-sum assets (no positions), they are amounts: new amount = current + deposited − withdrawn. The total value recomputes automatically. |
| FR-A9 | A live preview shows current → new positions (or amount) and the new total value in PHP. Saving is blocked if the result would be below zero. |
| FR-A10 | The History entry records the transaction (`txn: {add, sub, unit, cur}`) and shows it as *Added / Sold N units* or *Deposited / Withdrew ₱N*, plus the positions or amount change. |

### 4.3 Navigation
| ID | Requirement |
|---|---|
| FR-N1 | A ☰ button on every page opens a side drawer with **Dashboard**, **History**, **Accounts & Investment Details** (with a red "N due" badge when a card is overdue or due within 5 days), and **App Lock** (On/Off). |
| FR-N2 | Tapping the backdrop or pressing Esc closes the drawer. |
| FR-N3 | The app reopens on the last page used. |

### 4.4 History
| ID | Requirement |
|---|---|
| FR-H1 | Every add, edit, delete, FX/date change, backup restore, version restore, and sync choice is logged with a timestamp and the portfolio total before and after. |
| FR-H2 | Edits log only the fields that changed, as before → after. Saving with no change logs nothing. |
| FR-H3 | The log is append-only. The app offers no way to edit or delete entries. |
| FR-H4 | One **daily snapshot** of the full dashboard is kept per day with changes, overwritten through the day so it holds the closing state. |
| FR-H5 | The first launch with existing data records a **"History started"** entry and snapshot. |
| FR-H6 | **Net worth over time** chart: line plus area of daily closing totals, gold endpoint, Y axis in ₱K/₱M, X axis dates as `YY-MM-DD`. |
| FR-H7 | The header shows the current total and the change since the first day on record (₱ and %), or *Tracking since Mon D, YYYY* when there is only one day. |
| FR-H8 | The timeline is grouped by day, newest first. Each day header shows the closing total and a **View** button. 14 days show by default, with **Show older days** for more. |
| FR-H9 | **View** opens a read-only snapshot: total, PH/Global split, FX, and every asset. |
| FR-H10 | **Restore this version** needs two taps. It replaces the current dashboard, saves the replaced version on the log entry (**View version before this**), and syncs. |

### 4.5 Accounts & Investment Details — Credit Cards
| ID | Requirement |
|---|---|
| FR-C1 | The page **Accounts & Investment Details** has a **Credit Cards** section. Card amounts are **not** counted in Gross Portfolio. |
| FR-C2 | **Add Card** form: card name (nickname, required, **letters and spaces only**), cut-off day of the month (1–31), credit limit, statement due date, statement due amount, paid/settled on (date), and payment (Not yet paid / Paid the whole amount / Paid the minimum). |
| FR-C3 | Validation: name required, letters (any language, e.g. Ñ) and spaces only. Other characters are removed as typed or pasted, with a notice, and are rejected on save. Cut-off must be a whole number from 1 to 31. Amounts must be numbers ≥ 0 (commas and ₱ allowed). A paid date requires a payment type. Choosing a payment type with no date fills in today. |
| FR-C4 | Each card shows its cut-off ("Every 15th", plus the next cut-off date, using the month's last day when shorter), due date, due amount, paid on, payment, and credit limit. |
| FR-C5 | Each card shows the statement as a % of the limit: green under 30%, gold 30–49%, red 50% and up. |
| FR-C6 | Status pill: **Overdue N days** / **Due today** (red), **Due in N days** for ≤ 5 days (gold), **Due Mon D** (neutral), **Paid in full** (green), **Paid minimum** (gold), and **No statement** / **Nothing due**. |
| FR-C7 | Sort order: overdue, then due today, due soon, upcoming, paid minimum, paid in full, and no statement. Ties sort by due date. |
| FR-C8 | Summary tiles: **Unpaid Due** (sum and count of unpaid cards), **Next Due** (soonest unpaid due date and card), and **Total Limit**. |
| FR-C9 | Unpaid cards with a due date show one-tap **Paid Whole Amount** and **Paid Minimum** buttons, which set the paid date to today. |
| FR-C10 | The edit form has **New Statement**, which clears the due date, due amount, paid date, and payment for the next billing cycle. **Delete** needs two taps. |
| FR-C11 | Every card add, edit (field by field, before → after), and delete is logged in History with no ₱ delta. Cards are included in Drive sync and backup files. Restoring a portfolio version does not change cards. |
| FR-C12 | Card numbers (full or partial) are never requested or stored. The letters-only name rule makes them impossible to enter. |

### 4.6 Backup & restore
| ID | Requirement |
|---|---|
| FR-B1 | **Download backup** saves a JSON file containing holdings, settings, and history. |
| FR-B2 | **Restore** loads a backup file, replaces holdings and settings, merges history, and logs a restore entry. |

### 4.7 Google Drive sync
| ID | Requirement |
|---|---|
| FR-S1 | **Connect Drive** signs in with Google (OAuth implicit flow, `drive.file` scope). On return the app removes the token from the URL and checks that the `state` value matches. |
| FR-S2 | If the Drive file doesn't exist, it is created from this device's data. |
| FR-S3 | **Push:** every local change is sent to Drive about 1.2 seconds later. |
| FR-S4 | **Pull:** while the app is visible it checks Drive every 30 seconds, and also on resume, on reconnecting to the internet, on returning from a cached page, and when the Drive button is tapped. Each check reads only the file's `version`. The full file downloads only when the version changed. |
| FR-S5 | **Conflicts:** if both this device and Drive changed since the last sync, the app shows a choice between "Keep the Google Drive copy" and "Keep this device's copy", with asset counts and timestamps. |
| FR-S6 | History from both sides is always merged: log entries are combined by ID, and for each day the snapshot with the later time wins. If the merged history differs from Drive's, it is pushed back. |
| FR-S7 | **Silent renewal:** when the 1-hour token expires, the app re-authorizes with `prompt=none`, at most once every 10 minutes and never while a dialog is open. If Google needs interaction, the button shows **Tap to sync**. |
| FR-S8 | Button states: Connect Drive (grey), Syncing… (gold), Synced (green), Tap to sync, Sync failed, or Choose copy (red). |
| FR-S9 | **Disconnect** revokes the token and clears the connection. Local data stays on the device. |
| FR-S10 | **Shared use:** all devices must sign in with the **same Google account** to share one file. Different accounts create separate files. |

### 4.8 App lock
| ID | Requirement |
|---|---|
| FR-L1 | **App lock** in the menu turns the lock on or off for this device. Turning it on creates a platform passkey (`userVerification: required`). |
| FR-L2 | When the lock is on, a lock screen covers the app on launch and asks for Face ID, fingerprint, or passcode. Unlock requires the user-verified (UV) flag in the authenticator data. |
| FR-L3 | The app content is hidden as soon as the app goes to the background. Returning after more than **60 seconds** locks it again; returning sooner just uncovers it. |
| FR-L4 | Content is hidden before first paint (a CSS class is set by an early inline script), so numbers never flash. |
| FR-L5 | Turning the lock off requires a successful biometric check. |
| FR-L6 | **Can't unlock?** requires two taps. It erases this device's holdings, history, Drive connection, and lock, then reloads. Data can be recovered by reconnecting Drive. |
| FR-L7 | If the phone has no platform authenticator, the lock setting explains why it isn't available. |

---

## 5. Non-Functional Requirements

| ID | Area | Requirement |
|---|---|---|
| NFR-1 | Privacy | No portfolio data in the repo, in URLs, or on any server other than the user's own Google Drive. |
| NFR-2 | Least privilege | The Drive scope is `drive.file`, so the app can only access the file it created. |
| NFR-3 | Security | The OAuth `state` is checked on every redirect. Tokens are never logged. The client ID is public by design. |
| NFR-4 | Security | The app lock is a privacy screen, not encryption. Data in `localStorage` is not encrypted at rest. |
| NFR-5 | Offline | The app opens and edits work offline. Changes sync when back online with the app open. |
| NFR-6 | Performance | First render is under 1 s on a mid-range phone (no framework). An idle sync check is one small metadata request every 30 s. |
| NFR-7 | Storage | Roughly 1 MB of history per year of daily edits, well within the ~5 MB `localStorage` limit. The app warns if a write fails. |
| NFR-8 | Compatibility | iOS Safari 16.4+ (home-screen app), Android Chrome 110+, desktop Chrome, Edge, and Safari. |
| NFR-9 | Accessibility | Visible focus states, keyboard support for rows and the drawer, aria labels on charts, tabular numbers, and reduced-motion support. |
| NFR-10 | Theming | Light (cream) and dark (forest green) themes follow the phone's setting. All colors come from shared tokens. |

---

## 6. Visual Design

| Token | Light | Dark | Use |
|---|---|---|---|
| `--bg` | `#faf8f2` | `#0b1a17` | Page background |
| `--surface` | `#ffffff` | `#11241f` | Cards, dialogs |
| `--green` | `#0d5248` | `#4fae8c` | Primary, headings, Global |
| `--gold` | `#b8862f` | `#d9ad55` | Accents, section rules, Philippines |
| `--leaf` | `#3f8f4a` | `#7cc06a` | Gains, synced state |
| `--neg` | `#b4432f` | `#e58472` | Losses, errors, delete |

Colors are taken from the family logo (forest green, gold, cream). Asset type colors come from a 10-color green/gold palette, assigned by value rank.

---

## 7. Setup & Release Procedure

### 7.1 One-time Google Cloud setup
1. Create a Cloud project, then enable the **Google Drive API**.
2. On the OAuth consent screen, set Audience to External and leave the app in **Testing**. Add every Gmail that will sign in under **Test users**.
3. Under Data access, add the scope `.../auth/drive.file`.
4. Create a Web application client:
   - JS origin: `https://edmertsison-cpu.github.io`
   - Redirect URI: `https://edmertsison-cpu.github.io/sison-portfolio/` (keep the trailing slash)
5. Put the client ID in `config.js`.

### 7.2 Releasing a change
1. Edit `index.html` (and other files as needed).
2. **Bump `VERSION` in `sw.js`** (`portfolio-vN` → `vN+1`).
3. Add an entry at the top of `what-changed.md`, and update this SPEC if behavior changed.
4. Upload to the repo and commit to `main`. GitHub Pages deploys in about a minute.
5. On each phone, fully close and reopen the app (twice if needed).
6. If an icon changes, remove the home-screen app and add it again. **Back up first**, or confirm Drive sync, because on iOS removing the app deletes its data.

---

## 8. Open Items / Backlog

Effort: **S** = under an hour, **M** = a few hours, **L** = a day or more. Source = where the item came from.

| # | Item | Notes | Effort | Source |
|---|---|---|---|---|
| 1 | **Other account & investment details** | Credit cards done in v11. Remaining sections await requirements from Tep. | — | v7 / v11 |
| 2 | "Connected as name@gmail.com" in the Drive dialog | Would have prevented the separate-accounts mix-up. | S | Drive sync issue, Oct 11 |
| 3 | Liabilities / net worth after loans | Example: the ₱374,540 in loans from the original screenshot. Credit card balances could feed into it. | M | Original screenshot |
| 4 | Cost basis and gain/loss per asset | The source screenshot tracked "Bought" vs "Market". | M | Original screenshot |
| 5 | Passive income / yield | The source screenshot showed annual and monthly passive income and net yield. | M | Original screenshot |
| 6 | Encryption at rest | A key from the WebAuthn PRF extension could encrypt local data. Today the app lock is a privacy screen only. | L | App lock, v10 |
| 7 | Snapshot trimming | Only needed if storage warnings appear. | S | History, v7 |
| 8 | Shared Drive file across different Google accounts | Needs the Google Picker API. Currently solved by using one account. | M | Drive sync issue, Oct 11 |
| 9 | Sale price and realized gain on Sold / Withdrawn | Today, selling only reduces positions. Capture the sale price and gain/loss, with an option to move the proceeds into a cash asset automatically. Pairs with #4. | M | Added/Sold fields, v11 |
| 10 | Credit card due-date reminders | Home-screen apps can't reliably send notifications on iPhone. Options: add due dates to Google Calendar, or export a calendar (.ics) file per statement. | M | Credit cards, v11 |
| 11 | Per-asset merge on sync conflicts | Today a conflict makes you keep one whole copy. Merge asset by asset so both phones' edits survive. | L | Two-way sync, v9 |
| 12 | Switch to turn off silent sign-in renewal | Only if the iPhone shows a flash or gets stuck on a Google page when renewing. Fallback is the manual "Tap to Sync". | S | Silent renewal, v9 |

### 8.1 Housekeeping (not features)

| # | Task | Notes |
|---|---|---|
| H1 | Rename any card nickname that has digits or symbols | Older names (e.g. with "–") display fine, but can't be saved until they are letters and spaces only. |
| H2 | Delete the old portfolio file in Jeng's own Google Drive | Left over from when she connected with her own Gmail. |
| H3 | Regenerate the Word version of this spec | The .docx is still v10. |

---

## 9. Acceptance Checklist (regression)

- [ ] Adding, editing, and deleting an asset updates totals, sections, type bars, the donut, and the date.
- [ ] A USD asset converts at the current FX, and changing FX updates all totals.
- [ ] History logs each action with correct before → after values and ₱ deltas. Saving with no change logs nothing.
- [ ] **View** on a past day shows that day's assets. Restore plus "View version before this" fully undoes.
- [ ] A fresh device that connects Drive loads the data. An edit on device A appears on device B within about 30 s.
- [ ] Both devices editing before a sync shows the "Choose copy" prompt, and history from both sides survives.
- [ ] An expired sign-in renews silently or shows "Tap to sync", with no redirect loop.
- [ ] With App lock on, launch is locked, a failed biometric stays locked, the app locks again after more than 60 s in the background, and the app switcher shows no numbers.
- [ ] Editing ALI with Added 50,000 / Sold 20,000 gives positions +30,000 and an updated total. Cash Deposited / Withdrawn changes the amount. Overselling is blocked. History shows the transaction.
- [ ] Typing digits or symbols in a card name removes them, and saving "Card 1234" is rejected.
- [ ] A card with a past due date shows Overdue and sorts first. Paid Whole Amount / Paid Minimum update the status. New Statement clears the cycle. Gross Portfolio is unaffected.
- [ ] Light and dark themes, a 390 px width, and no horizontal scroll.
- [ ] `sw.js` VERSION is bumped and `what-changed.md` is updated.
