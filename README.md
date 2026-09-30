# Ipon

A personal budget and investment tracker that works in Philippine pesos (PHP), US dollars (USD), or both. *Ipon* is Filipino for "savings".

It is a single self-contained HTML file: no build step, no dependencies, no server. Sign up with an email and password, and your data stays in your browser under your account.

## Features

**Accounts**
- Sign up and sign in with an email and password; several accounts can share one browser
- The email is only a sign-in name for this device: it is not verified and nothing is ever sent to it
- Each account has its own separate data
- Optional "keep me signed in on this device"
- Change password, sign out, and delete account from Settings

**Overview**
- Income, spending, money left over, and savings rate for any month
- A bar showing how the month's income split across spending categories and what was left
- Six-month income vs. spending chart
- Budget status, upcoming recurring entries, top savings goals, and a portfolio summary

**Budget**
- Log income and expenses in either currency, with category, note, and date
- Set a monthly limit per category; the meter turns amber at 80% and red when you go over

**Recurring**
- Weekly, every 2 weeks, twice a month (15th and month-end), monthly, or yearly
- Entries log themselves each time you sign in, up to today
- Pause, resume, or delete; deleting keeps entries already logged
- Month-end dates clamp correctly (a monthly entry on the 31st lands on Feb 28)

**Goals**
- Target amount, currency, optional starting balance, optional target date
- Add or withdraw money with a history you can correct
- With a target date, shows the monthly amount needed to finish on time

**Investments**
- Holdings by type: stocks, ETF/index fund, mutual fund/UITF, bonds and treasuries, crypto, Pag-IBIG MP2, time deposit, real estate, other
- Edit the amount invested and the current value inline
- Gain or loss, return %, and allocation by type

**About us**
- A page for your story, the people behind the app, a dedication, and a contact email, all filled in from one place (see [Editing the About us page](#editing-the-about-us-page))

**Everything else**
- Light and dark theme, following your device until you choose
- Backup and restore as JSON, transactions export as CSV
- Optional sample data, tagged so you can remove it in one click

## Using it

1. Open the page and choose **Create account**.
2. In **Settings**, set the exchange rate (pesos per US dollar). The default of 58 is only a placeholder.
3. Pick a display currency at the top. Each entry keeps the currency you entered it in; totals are converted at your rate.
4. Add transactions, budgets, recurring entries, goals, and holdings.

New here? **Settings > Add sample data** fills in three months of made-up entries so you can see how the charts behave.

## Currency handling

- The header toggle sets the *display* currency only. Nothing you entered is rewritten.
- Conversion uses one manual rate per account. There is no live lookup, so update it when the market moves.
- Currency gains and losses on USD holdings appear when you change the rate, not from historical rates.
- Goals stay in their own currency; a converted estimate appears when it differs from the display currency.

## Accounts and security

Read this before relying on the login.

**What it does**
- Passwords are never stored. Each account keeps a random salt and a PBKDF2-SHA256 hash (600,000 iterations) made with the browser's Web Crypto API.
- After 5 wrong passwords an account is locked for 60 seconds.
- Sign-in errors don't reveal whether an email has an account.
- Signing out clears the account's data from memory.

**What it does not do**
- **It is a gate on the app, not encryption.** Accounts and data live in this browser's `localStorage`. Anyone who can open your browser's developer tools on this device can read the stored data and can edit the session. Do not treat it as protection against someone with access to your computer.
- **There is no password reset.** There is no email or server. If you forget a password, you can create a new account and restore from a backup file.
- **Backups are plain text.** Keep them somewhere private.
- The lockout is a speed bump; it can be bypassed by anyone who can edit browser storage.

If you need real protection, the next steps are encrypting each account's data with a key derived from the password, or moving accounts to a server.

## Your data

Stored in this browser's `localStorage`:

| Key | Contents |
| --- | --- |
| `ipon:users:v1` | Account list: id, email, display name, salt, password hash, lockout state |
| `ipon:data:v1:<userId>` | One account's budget, transactions, recurring entries, goals, and holdings |
| `ipon:session:v1` | Who is signed in (in `sessionStorage` unless "keep me signed in" is ticked) |
| `ipon-theme` | Light or dark choice, shared across accounts on this device |

- It does **not** sync across devices or browsers, and clearing site data deletes everything, accounts included.
- Nothing is sent anywhere.
- Back up from **Settings > Backup (JSON)** now and then. A backup covers the signed-in account only and never includes the password. Restore from a file or by pasting the backup text; it replaces that account's data.
- If saving a file isn't available in your environment, the backup text appears in a box you can copy.

### Backup format

```json
{
  "v": 1,
  "cur": "PHP",
  "rate": 58,
  "tx":      [{ "id": "", "date": "2026-09-30", "type": "expense", "cat": "Utilities", "desc": "", "amt": 3100, "cur": "PHP" }],
  "budgets": [{ "id": "", "cat": "Food & groceries", "limit": 9000, "cur": "PHP" }],
  "inv":     [{ "id": "", "name": "", "kind": "ETF / index fund", "cur": "USD", "cost": 1800, "value": 1962, "date": "2026-09-30" }],
  "rec":     [{ "id": "", "type": "income", "cat": "Salary", "desc": "", "amt": 58000, "cur": "PHP", "freq": "monthly", "start": "2026-09-01", "n": 0, "paused": false }],
  "goals":   [{ "id": "", "name": "", "target": 150000, "cur": "PHP", "saved": 0, "deadline": "", "contribs": [{ "id": "", "date": "2026-09-30", "amt": 5000 }] }]
}
```

The backup format is not yet frozen: the app is pre-release, so it may change without a migration path.

## Editing the About us page

The **About us** tab is driven by one constant near the top of the script in `ipon-tracker.html`:

```js
var ABOUT={
  story:'',
  people:[],
  dedicationTitle:'',
  dedication:'',
  dedicationFrom:'',
  contactEmail:''
};
```

| Field | What it does | When blank |
| --- | --- | --- |
| `story` | Your introduction. Separate paragraphs with a blank line. | A short default intro about the name *Ipon* |
| `people` | The people behind the app: `[{name:'', role:'', note:''}]`. Entries without a `name` are skipped. | The "Who made it" section is hidden |
| `dedicationTitle` | Optional heading above the dedication | No heading |
| `dedication` | The dedication message; line breaks are kept | "A dedication to family will go here." |
| `dedicationFrom` | Optional sign-off line under the dedication | No sign-off |
| `contactEmail` | Shown as a mail link | The "Get in touch" section is hidden |

Example:

```js
var ABOUT={
  story:'Why we made this.\n\nWhat we hope it helps with.',
  people:[{name:'Name', role:'Role', note:'A line about them.'}],
  dedicationTitle:'For our family',
  dedication:'Your message here.',
  dedicationFrom:'Your name',
  contactEmail:'you@example.com'
};
```

All text is escaped, so it is shown exactly as typed. The page always ends with the version and a line saying data stays in the browser.

## Limitations

- No cross-device sync; data is per browser.
- The login is not encryption and has no password reset (see above).
- Exchange rate and investment values are manual.
- Recurring entries are logged when an account is opened, not in the background.
- Money added to a goal is tracked separately from transactions; it is not logged as an expense.

## Running and hosting

Open `ipon-tracker.html` in any modern browser, or host it anywhere that serves static files over HTTPS (or open it from disk). Account passwords need the browser's Web Crypto API; if a browser doesn't provide it, sign-up and sign-in are disabled rather than falling back to something weaker. It also runs as a published Claude artifact. Web fonts (Bricolage Grotesque and Instrument Sans) load from Google Fonts; if they can't load, system fonts are used.

## Development

There is nothing to install. Edit `ipon-tracker.html` and reload. See `CLAUDE.md` for the code layout, conventions, a syntax check, and a manual test list.

## License

Personal project. Add a license before sharing it more widely.
