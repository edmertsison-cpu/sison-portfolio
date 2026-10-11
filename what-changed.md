# What Changed

The change history of the **Sison Family Portfolio** app, newest first.

Each version has a matching cache name in `sw.js` (`portfolio-vN`). After uploading a new version, fully close and reopen the app so phones load it.

> This file lists only app changes. Your holdings and history are never stored in this repo. They stay on your device and in your Google Drive file.

---

## v10 — 2026-10-11
**App lock with Face ID / fingerprint**
- New **App lock** item in the side menu. When it's on, the app asks for Face ID, fingerprint, or the phone passcode (whichever the phone has set up) when it opens, and again when you return after more than 1 minute away.
- Uses a device passkey saved by the phone as "Sison Family Portfolio". Your face or fingerprint never leaves the phone, and the app never sees it.
- Numbers are hidden as soon as the app goes to the background, so they don't show in the app switcher.
- The lock is per phone, so each person sets it up on their own phone. Turning it off asks for Face ID or fingerprint first.
- **Can't unlock?** erases this phone's copy and removes the lock. Data comes back by connecting Google Drive again.

## v9 — 2026-10-11
**Always pull the latest data from Google Drive**
- While the app is open, it checks Drive every 30 seconds. It also checks when you switch back to the app, when your phone reconnects to the internet, and when you tap the Drive button.
- Each check is one tiny request that asks whether the Drive file changed. The full data downloads only when it did, so changes made on another device show up within about 30 seconds.
- Sign-in renewal: Google sign-in lasts about an hour. When it expires, the app tries a silent renewal (no screens, about a second) at most once every 10 minutes, and never while a form or dialog is open. If Google needs you to confirm, the button shows "Tap to sync".
- The Drive dialog now shows "Last checked" instead of "Last synced".

## v8 — 2026-10-10
**History date formats**
- "Tracking since" and "since" now include the year, for example *Tracking since Oct 10, 2026*.
- Dates on the net worth chart now read as `YY-MM-DD`, for example `26-10-10`.

## v7 — 2026-10-10
**Side menu, History, and Investment details**
- Added a ☰ side menu with three pages: **Dashboard**, **History**, and **Investment details**. The app reopens on the last page used.
- **History** page:
  - Net worth over time line chart, plus the change since the first day on record.
  - Change log that records each add, edit (with before → after for every changed field), delete, FX/date change, backup restore, version restore, and Drive sync choice, along with the change to the total. The app has no way to edit or delete entries.
  - Daily snapshot of the full dashboard. **View** shows the dashboard as it stood at the end of a day.
  - **Restore this version** asks for a second tap to confirm. Every restore saves the version that came before it, so **View version before this** can undo it.
  - A "History started" entry is recorded the first time the new version opens.
- **Investment details** shows a "coming soon" placeholder until its requirements are ready.
- Google Drive sync and backup files now include history. When two devices sync, their histories merge and no entries are lost.

## v6 — 2026-10-09 / 2026-10-10
**Google Drive sync**
- Added a **Connect Drive** button. After you sign in with Google, every change saves to *Sison Family Portfolio data.json* in your Drive.
- Uses the `drive.file` permission, so the app can only see the one file it creates.
- Opening the app loads the Drive copy if it is newer.
- If this device and Drive both changed, the app asks which copy to keep.
- A sign-in lasts about an hour. After that the button shows "Tap to sync".
- Added `config.js`, which holds the Google OAuth client ID (not a secret). The client ID was set on 2026-10-10.

## v5 — 2026-10-08
**By asset type chart: bar colors**
- Each bar now spans the full width, split by that type's Philippines (gold) vs Global (forest green) share. A type held only in the Philippines is all gold, and one held only globally is all green.

## v4 — 2026-10-08
**By asset type chart**
- Added a **By asset type** section showing each type's total value, position count, % of total, and the Philippines/Global split.

## v3 — 2026-10-08
**Family logo and theme**
- New app icon taken from the Sison Family Portfolio logo. The logo also appears in the header.
- Colors now match the logo (forest green, gold, cream), with light mode as the main look and a deep forest-green dark mode.
- New bold heading font (Montserrat).

## v2 — 2026-10-08
**Title and date**
- Made the "Gross Portfolio" title bigger.
- The "as of" date now updates to today whenever an asset is added, edited, or deleted.

## v1 — 2026-10-08
**First release**
- Installable phone app (PWA) hosted on GitHub Pages, which also works offline.
- Philippines and Global sections with free-form asset types. Each asset shows positions, market price, and total value.
- Assets are grouped by type and sorted by value.
- Summary with gross total, USD equivalent, Philippines/Global split, and an allocation donut chart.
- USD assets are converted to PHP at the rate set under **FX & date**.
- Data is stored on the device, with **Backup** download and restore.
