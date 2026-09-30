# CLAUDE.md

Guidance for working on **Ipon**, a personal budget and investment tracker (PHP and USD) with multi-user login. See `README.md` for what it does and how people use it.

## Project shape

- One file: `index.html`. HTML, CSS, and JavaScript all live inside it. No build, no bundler, no npm, no framework.
- Keep it a single self-contained file. Do not split into modules or add dependencies.
- Vanilla JS in an IIFE with `'use strict'`. Style is ES5-flavored: `var`, `function` declarations, string concatenation for templates. `async`/`await` is used only where Web Crypto or file saving requires it. Match the existing style rather than mixing in new ones.

## Run and check

There is no test suite. After any edit:

```bash
# extract the inline script and syntax-check it
python3 - <<'EOF2'
import re
s = open('index.html', encoding='utf-8').read()
open('/tmp/ipon.js', 'w').write(re.search(r'<script>(.*)</script>', s, re.S).group(1))
EOF2
node --check /tmp/ipon.js && echo SYNTAX_OK
```

Then open the file in a browser and exercise what you changed. Two things are worth testing in Node because they are pure logic:

- `occ()` (recurrence dates): check month-end clamping, `semimonthly`, and a start on the 31st.
- `Auth` and `Store` (accounts): Node 19+ has Web Crypto globally. Run the extracted script in a `vm` context with stub `window`, `document`, `localStorage`, and `sessionStorage`, and temporarily expose `Auth`, `Store`, and `normalize` by editing the final `})();`. Never commit that exposure.

### Manual test list for accounts

1. First run (no accounts yet) shows **Create account**.
2. Create two accounts; each sees only its own data.
3. Wrong password 5 times locks the account for 60 seconds.
   Sign-up rejects malformed and duplicate emails (case-insensitive).
4. Reload with and without "keep me signed in"; without it, a closed tab signs you out.
5. Change password; old one stops working. Delete account; the other account is unaffected.
6. Sign out, then check that nothing from the previous account (including backup text in Settings) is on screen.
7. Toggle light/dark on the sign-in screen and inside the app.
8. About us: blank `ABOUT` shows the default intro and the dedication placeholder; a filled one shows story, people, dedication and contact, with markup in the text shown literally.

## Hosting constraints

The page is also published as a Claude artifact, which imposes rules:

- No remote scripts except from `cdnjs.cloudflare.com`, `cdn.jsdelivr.net/npm/`, `cdn.tailwindcss.com`, `code.jquery.com`. Styles only from Google Fonts. No network requests, no remote images.
- Do not use `window.storage`, `window.claude.complete`, or calls to `api.anthropic.com`. They don't exist in a published page.
- Browser storage works but is per viewer and per browser. The app keeps everything in `localStorage`, so data and accounts are per browser too.
- File saving goes through the `downloads` capability (`await window.claude.use('downloads')`). It returns `null` outside the artifact host, so `saveFile()` must keep its fallback of putting the text in the Settings textarea.
- `window.claude` may be undefined when the file is opened locally. Always guard it.
- Keep the viewport meta tag with `viewport-fit=cover` and the `env(safe-area-inset-*)` padding on `:root`.

Republishing: publish the same file path, and pass the existing artifact URL so it updates in place instead of creating a second artifact.

## Code layout (inside the script)

In order:

1. Constants: `APP_VERSION`, storage keys, `PBKDF2_ITER`, `MAX_FAILS`/`LOCK_MS`, `ABOUT` (About us content), `EXP`/`INC` categories, `KINDS`, `COLORS`, `TABS`, `FREQ`
2. State: `defaults()`, `normalize()`, `Store` (storage adapter), `S` (signed-in account's data), `user` (signed-in account), `save()`
3. UI state: `ui`, `resetUi()`
4. Theme: `effectiveTheme()`, `applyTheme()`
5. Money and month helpers: `conv()`, `fmt()`, `shiftMonth()`, `monthLabel()`, and friends
6. Calculations: `summary(month)`, `portfolio()`, `budgetRows(month)`
7. Views: `viewOverview`, `viewBudget`, `viewRecurring`, `viewGoals`, `viewInvest`, `viewSettings`, `viewAbout` (with `paras()`), plus `header()`, `tabs()`, `render()`
8. Recurring: `occ()`, `runRecurring()`, `addRec()`
9. Goals: `goalSaved()`, `goalInfo()`, `goalBlock()`, `addGoal()`, `goalMove()`
10. Accounts: crypto helpers, `Auth`, `enterAccount()`, `signOut()`, `renderAuth()`, `authSubmit()`, `changePw()`, `deleteAccount()`, `accountRows()`
11. Data actions: `addTx`, `addBudget`, `addInv`, export/restore, `sampleData`, `removeSample`
12. Events: `act(action, el)` switch, then delegated `click`, `keydown`, and `change` listeners
13. Startup: `applyTheme()`, restore session if valid, `render()`

## Accounts and storage

Storage keys (all `localStorage` unless noted):

- `ipon:users:v1`: array of `{id, email, name, salt, hash, iter, created, fails, lockedUntil}`. `salt` and `hash` are base64. Emails are trimmed and lowercased (`normEmail()`) and unique across accounts.
- `ipon:data:v1:<userId>`: that account's `S`.
- `ipon:session:v1`: `{uid, at}`. In `sessionStorage` by default, or `localStorage` when "keep me signed in" is ticked.
- `ipon-theme`: device-wide theme, deliberately not per account so the sign-in screen can use it.

Rules:

- **Never store or log a plain password.** Passwords are not trimmed, are normalized with NFKC, and are hashed with PBKDF2-SHA256 via `derive()`. Compare with `sameBytes()`, not `===`.
- The email is only a login identifier. It is never verified, and nothing may be sent to it, since there is no server. Don't add copy that implies email verification or recovery.
- If `window.crypto.subtle` is missing, `hasCrypto` is false and accounts are disabled. Do not add a weaker fallback.
- Login errors must stay generic ("The email or password is incorrect.") and unknown emails still run a derive to keep timing similar.
- Per-account state lives in `S` and `user`. Whenever the account changes (sign in, sign out, delete), call `resetUi()` so no form values, month, or backup text from the previous account survive. `resetUi()` leaves `ui.theme` and `ui.auth` alone.
- `save()` writes to the signed-in account's key and does nothing when signed out. Anything that changes `S` must call `save()`.
- `iter` is stored per user so the iteration count can be raised later. Rehash on successful login when `iter < PBKDF2_ITER` is not yet implemented.

### Swapping to a backend later

`Store` (storage) and `Auth` (signup, login, changePassword, remove, logout) are the only places that touch accounts and persistence. To move to a server:

1. Reimplement `Store` and `Auth` against the API; keep the same method names and the `{ok, user}` / `{ok:false, error}` return shape.
2. Make `Store.data()` and `Store.saveData()` async, then `await` them in `enterAccount()`, `save()`, and the startup path. Today they are synchronous.
3. Replace client-side hashing and the lockout with server-side equivalents, and issue real session tokens.
4. Keep `normalize()` as the single place that shapes loaded data.

## Data model

`S` is the signed-in account's whole state and also the backup format:

```
S = { v:1, cur:'PHP'|'USD', rate:<PHP per 1 USD>,
      tx:      [{id, date:'YYYY-MM-DD', type:'income'|'expense', cat, desc, amt, cur, rec?, sample?}],
      budgets: [{id, cat, limit, cur, sample?}],
      inv:     [{id, name, kind, cur, cost, value, date, sample?}],
      rec:     [{id, type, cat, desc, amt, cur, freq, start, n, paused}],
      goals:   [{id, name, target, cur, saved, deadline, contribs:[{id, date, amt}], sample?}] }
```

- Amounts are stored in the currency they were entered in (`cur`). Convert only at display time with `conv(amount, from)`. Never store converted values.
- `rec` rules keep `n`, the count of occurrences already generated. `occ(rule, n)` computes the n-th date from `start`, so generation is idempotent. `runRecurring()` logs every occurrence up to today and advances `n`. Generated transactions carry `rec: ruleId`.
- Goal saved amount = `saved` + sum of `contribs[].amt` (negative entries are withdrawals). Goals do not create transactions.
- `sample: true` marks generated demo data. Editing a sample item removes the flag.
- If you change the shape of `S`, update `defaults()`, `normalize()`, `sampleData()`, `removeSample()`, and the README backup example. The app is pre-release, so there is no obligation to keep old accounts, stored data, or backups working; change the shape freely. Keep `normalize()` as the single loader. Before a public release, decide on data versioning (`S.v`) and migrations, and update this section.

## Conventions

- **Escape all user text.** Anything user-entered that goes into an HTML string must pass through `esc()`. This includes display names and emails.
- **Rendering.** `render()` rebuilds the whole `#app` from state, and shows `renderAuth()` when nobody is signed in. Change state, call `save()`, then `render()`. The exception is inline edits of holding values, which patch `#inv-summary` and `#gain-<id>` directly so keyboard focus isn't lost.
- **Events.** Use delegation, not per-element listeners:
  - `data-action` on clickable elements, handled in `act()`
  - `data-keep="obj.key"` on inputs whose value should survive re-renders (stored in `ui`)
  - `data-enter="action"` to trigger an action on Enter
  - `data-action-change` and `data-inv` for change-event inputs
  - `data-pw` on password inputs so "Show password" can toggle them
- **No `<form>` tags.** Use divs, buttons, and the Enter handler. Read values by element id via `num()` and `val()`.
- **Feedback.** Use `toast(message)` for confirmations and validation errors, in plain sentences that say what to fix. Sign-in and account errors appear inline next to the form.
- **Destructive actions.** Single-row deletes are immediate. "Clear all data" and "Delete account" need a second step (`ui.confirmReset`, `ui.confirmDelete`); account deletion also needs the password first.
- **Persistence.** All storage access goes through `Store`, which wraps every call in `try/catch`. The page must work with empty or blocked storage.

## About us page

The **About us** tab (`viewAbout()`) reads the `ABOUT` constant near the top of the script. Fields: `story`, `people` (`[{name, role, note}]`), `dedicationTitle`, `dedication`, `dedicationFrom`, `contactEmail`.

- `people` and `contactEmail` sections are hidden when blank. A blank `story` falls back to a short default intro. The dedication shows the placeholder "A dedication to family will go here." until `dedication` has text.
- `story` is split into paragraphs on blank lines; `dedication` keeps line breaks (`white-space: pre-line`).
- Everything goes through `esc()`. `contactEmail` is shown as a `mailto:` link only if it passes `EMAIL_RE`.
- The footer (version and the "nothing is sent to a server" line) is always shown. Keep that statement true.
- Keep the copy plain. Don't add tracking, external links, or images there.

## Styling

- All colors come from CSS custom properties on `:root`. Do not hardcode colors in components, except the category palette in `COLORS`, which is chosen to work on both themes.
- Theme is defined three ways and all three must stay in sync: light tokens in `:root`, dark tokens in `@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) }`, and dark tokens again in `:root[data-theme="dark"]`. Adding a token means adding it in all three places.
- Fonts: Bricolage Grotesque for headings (`--display`), Instrument Sans for body (`--body`), each with a system fallback. Numbers use `font-variant-numeric: tabular-nums`.
- Layout is responsive with one breakpoint at 720px. Wide content must scroll inside its own container, never the page body.
- Keep `prefers-reduced-motion` respected and keyboard focus visible.
- Copy: sentence case, plain verbs, buttons say what they do ("Add expense", "Set limit", "Create account"). Empty states tell the user what to do next.

## Known limitations (don't "fix" without asking)

- The login is a gate on the UI, not encryption. Data is readable in `localStorage`, and the session can be edited by anyone with devtools. Don't describe it as secure storage in copy or docs.
- No password reset; there is no email or server.
- No sync between devices; data is per browser.
- Exchange rate and investment values are manual by design. There are no network calls.
- Recurring entries are generated when an account is opened, not in the background.
- Goal contributions are intentionally separate from transactions.

## Ideas not yet built

- Encrypt each account's data with an AES-GCM key derived from the password (trade-off: a forgotten password then means unrecoverable data)
- Rehash passwords on login when the stored iteration count is out of date
- Option to log a matching expense when adding money to a goal
- Editing existing transactions (currently delete and re-add)
- Cost-averaging view for regular investment contributions
- Per-category trends over time
