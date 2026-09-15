# HMIS Payment Notices

Static, self-contained HTML pages embedded via `<iframe>` on HMIS login /
department-selection screens (`select_department.xhtml`, config keys
`Display the payment notice` + `Payment Notice URL`) for hospitals with an
overdue CareCode HMIS subscription payment.

Each `<hospital>.html` here is a standalone page — no external dependencies,
no third-party branding/chrome (unlike a Google Docs "Publish to web" link,
which was tried first and rejected: it always adds "Published using Google
Docs" / "Report abuse" / the doc title / an auto-refresh notice around the
actual content) — served via GitHub Pages so the iframe shows only the
intended message.

This is an **interim measure**. The long-term fix is an inline-HTML config
option (`Payment Notice HTML`) so notices can be authored directly in the
HMIS admin UI without needing a hosted page at all — see
[hmislk/hmis#23813](https://github.com/hmislk/hmis/issues/23813). Once that
ships, hospitals should be migrated off this repo and this repo can be
retired.

**Repo visibility is intentionally public.** GitHub Pages on a Free-plan org
(CareCodes) only serves a *public* repo — private-repo Pages needs a paid
Pro/Team plan. The page content itself is a generic payment reminder, not
sensitive; nothing here should ever be sensitive (see Security note below).

## Pages currently live

| File | Hospital | Added | Notes |
| --- | --- | --- | --- |
| `southernlanka.html` | Southern Lanka | 2026-09-15 | Visually confirmed live |
| `coop.html` | Co-op (Galle Co-Operative Hospital) | 2026-09-15 | API-confirmed |
| `digasiri.html` | Digasiri (New Puttalam Digasiri Hospital) | 2026-09-15 | API write returned 200; admin UI grid is broken on this old deployment so it could not be visually re-checked from the admin side (department-selection screen itself was not re-verified either) |
| `rmh.html` | RMH | 2026-09-15 | API-confirmed via `/api/config/search` |
| `roseth.html` | Roseth Hospital | 2026-09-15 | Visually confirmed live |

**Not yet done:** Engage Wellness Center (EWC) — no working login or Config
API key on file yet. Pick up later; see "Adding a new hospital" below once
credentials are available.

## Credentials

All hospital app logins and `Config`-type API keys used for this rollout are
kept in **`C:\Credentials\Credentials.txt`** on the Windows dev machine —
**never in this repo, never in any HMIS repo** (per the parent HMIS project's
`CLAUDE.md` security rule: no passwords/keys/hostnames in tracked or
untracked project files). That file has a section per hospital, e.g.
`SOUTHERN LANKA PRODUCTION - App Login` and `... - API Keys`.

As of 2026-09-15 this is the **only** credentials store used for this
rollout — no separate copy exists on any Ubuntu/Linux machine. If that
changes, or if you're working from a machine without access to
`C:\Credentials\Credentials.txt`, ask the project owner where the
credentials live for that environment rather than guessing or searching for
them — never commit a found/guessed credential here to "make it work".

## How the notice is wired into HMIS

Two `ConfigOption` keys, `APPLICATION`-scoped, read by
`select_department.xhtml` (the screen shown right after login, before a
department is chosen):

- **`Display the payment notice`** (Boolean) — when `true`, renders an
  `<iframe src="#{configOptionApplicationController.getLongTextValueByKey('Payment Notice URL')}" width="600" height="400">`
  above the department picker.
- **`Payment Notice URL`** (Long Text) — the URL loaded into that iframe.
  Point this at `https://carecodes.github.io/hmis-payment-notices/<hospital>.html`.

A third, unrelated key, **`Execute the payment notice`** (Boolean), fully
blocks login (hides the department-picker form entirely) when `true`. This
repo/rollout only ever sets `Display the payment notice` +
`Payment Notice URL` — **never** `Execute the payment notice`. Flipping that
one is a separate, deliberate escalation the hospital's account owner
decides on later, once a notice period has passed with no payment — treat it
as out of scope unless explicitly asked.

See `hmislk/hmis` source: `src/main/webapp/resources/ezcomp/select_department.xhtml`.

## Adding a new hospital's notice

1. **Get access** for that hospital from `C:\Credentials\Credentials.txt`
   (or ask the project owner if it's not there yet):
   - App login (URL + username + password), for visual verification, and/or
   - A `Config`-type API key, for the fast path below.

2. **Set the two config keys.** Fastest path — via the REST API (works on
   every hospital tried so far except the read-back endpoints on some older
   deployments, see Gotchas below):

   ```bash
   curl -X POST \
     "https://<slug>.carecode.org/<slug>/api/config/setBoolean/Display%20the%20payment%20notice/true" \
     -H "Config: <that hospital's Config API key>"

   curl -X POST \
     "https://<slug>.carecode.org/<slug>/api/config/setLongText/Payment%20Notice%20URL/<url-encoded page URL>" \
     -H "Config: <that hospital's Config API key>"
   ```

   Both should return `200` with body `Configuration updated successfully.`
   Verify with:

   ```bash
   curl "https://<slug>.carecode.org/<slug>/api/config/search?keyword=payment%20notice" \
     -H "Config: <key>"
   ```

   If no Config API key exists for that hospital, log in via the app UI
   instead: **Administration → Manage Institutions → Preferences tab →
   Application Options**, filter for `Display the payment notice` and
   `Payment Notice URL`, and edit each value there. (On some older
   deployments this grid's filter/sort never fires its AJAX update — if so,
   fall back to the API, or ask the project owner to check it live.)

3. **Add the page to this repo.** Copy an existing `<hospital>.html` as a
   template (all five current pages share one design — a bordered red/pink
   card, ⚠ "Payment Notice" header, "Dear `<Hospital>`," body). Swap the
   hospital name and any hospital-specific wording. Keep it plain
   inline-styled HTML with **zero external requests** (no CDN fonts/scripts,
   no analytics) — the whole point is a clean iframe with nothing but the
   message.

4. **Commit and push to `main`.** GitHub Pages rebuilds automatically,
   usually within 1–3 minutes. Poll before telling anyone it's live:

   ```bash
   curl -o /dev/null -s -w "%{http_code}\n" \
     "https://carecodes.github.io/hmis-payment-notices/<hospital>.html"
   ```

   (repeat until it returns `200` instead of `404`)

5. **Update the table above** in this README with the new row.

6. **Verify** — ideally by logging into the hospital's app and confirming
   the department-selection screen shows the notice cleanly. If you can't
   log in yourself, ask the project owner to check, or trust the `200`
   API-write response as a weaker fallback (as was done for Digasiri).

## Removing a hospital's notice (payment received / notice no longer needed)

Two ways, from least to most permanent:

**A. Turn it off in HMIS (recommended — keeps the page here for reuse)**

```bash
curl -X POST \
  "https://<slug>.carecode.org/<slug>/api/config/setBoolean/Display%20the%20payment%20notice/false" \
  -H "Config: <that hospital's Config API key>"
```

This is the normal "payment received, stop showing the notice" action. The
`Payment Notice URL` value and the page in this repo are left alone — if the
notice needs to come back later (e.g. a new missed payment), just flip the
boolean back to `true` with the same call; no need to touch this repo again.

**B. Delete the page from this repo (only if the notice is retired for
good, e.g. that hospital churned, or it moved to the long-term HTML
config-option mechanism from #23813)**

1. Confirm `Display the payment notice` is already `false` for that
   hospital (step A above) — don't delete a page a hospital is still
   pointed at.
2. Remove `<hospital>.html` from this repo and update the README table.
3. Optionally also clear `Payment Notice URL` back to empty via
   `setLongText/Payment%20Notice%20URL/` with an empty value, so the config
   doesn't keep pointing at a dead link.

Never delete a page while its hospital's `Display the payment notice` is
still `true` — that would turn a clean notice into a broken iframe /
404 for real users mid-login.

## Gotchas

- **Some deployments are on an old build** (Digasiri, Roseth confirmed so
  far) where `GET /api/config/{key}` and `GET /api/config/search` both
  404, even though the `POST /api/config/setBoolean` / `setLongText` write
  endpoints work fine (200 + success message). Treat the 200 write response
  as sufficient confirmation on these; don't waste time debugging the GET
  404 as if it were a failed write.
- **Account username isn't always `buddhika`/`Buddhika123@`.** RMH uses
  `bud`; Roseth uses `buddhikaAri`/`1234`. Check
  `C:\Credentials\Credentials.txt` per hospital rather than assuming a
  shared login — a wrong guess risks an account lockout on a live
  production system.
- **The admin "Application Options" grid's filter/sort AJAX doesn't always
  fire** (seen on Digasiri) — don't burn time retrying `fill()`/`Enter`/sort
  clicks past a couple of attempts; fall back to the API path instead.
- **Never navigate to an inner HMIS page by typing its URL** when driving
  the browser for verification — reach `select_department.xhtml` by
  logging in through the app's own login page, the same way a real user
  would; a typed-URL load of an inner page can render against null session
  state and produce a false "it's broken" result.
