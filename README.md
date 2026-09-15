# HMIS Payment Notices

Static, self-contained HTML pages embedded via `<iframe>` on HMIS login /
department-selection screens (`select_department.xhtml`, config keys
`Display the payment notice` + `Payment Notice URL`) for hospitals with an
overdue CareCode HMIS subscription payment.

Each `<hospital>.html` here is a standalone page — no external dependencies,
no third-party branding/chrome — served via GitHub Pages so the iframe shows
only the intended message.

This is an **interim measure**. The long-term fix is an inline-HTML config
option (`Payment Notice HTML`) so notices can be authored directly in the
HMIS admin UI without needing a hosted page at all — see
[hmislk/hmis#23813](https://github.com/hmislk/hmis/issues/23813). Once that
ships, hospitals should be migrated off this repo and this repo can be
retired.

## Pages

| File | Hospital | Added |
| --- | --- | --- |
| `southernlanka.html` | Southern Lanka | 2026-09-15 |
| `coop.html` | Co-op (Galle Co-Operative Hospital) | 2026-09-15 |

## Editing a notice

Edit the hospital's `.html` file and push to `main` — GitHub Pages
republishes automatically within a minute or two. No build step.
