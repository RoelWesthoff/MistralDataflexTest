# Claudetest — DataFlex 26.0 Web/Mobile Workspace Scaffold (v36)

## Playwright end-to-end tests

New `Tests/` folder — 19 tests across 4 specs, using page objects.

```
Tests/
├── playwright.config.ts
├── fixtures.ts
├── pages/      LoginPage · CustomerListPage · CustomerZoomPage
└── specs/      login · customer-list · pagination · customer-zoom
```

```bash
cd C:\DataFlexProjects\Claudetest\Tests
npm install
npx playwright install chromium
npm test          # or: npm run ui
```

### Verified before shipping

Node was available in my sandbox, so unlike the DataFlex code these were
actually executed rather than only reasoned about:

- `tsc --noEmit --strict` — clean.
- `npx playwright test --list` — the config, fixtures and all three page
  objects load, and all 19 tests are collected.

What I could **not** do is run them against your IIS instance, so the
selectors are derived from the application source, not from a live DOM.
See "Honest status" at the end of `Tests/README.md` for the three places
most likely to need a nudge on the first run — each is isolated to one
method so the fix is small.

### Selector strategy

This is what decides whether the tests survive a year of edits.
DataFlex generates its own DOM ids and they aren't stable. What *is*
stable is the `psCSSClass` values the app sets on itself:

`.LoginView` · `.HeaderPanel` · `.StoicList` · `.StoicRow` ·
`.StoicName` · `.StoicChip` · `.StoicAct` · `.IBWPager*` · `.StoicZoom`

Only two framework internals are used, both commented where they appear:
`.WebList_Selected` and `[data-dfrowid]`. Everything else goes through
what a user can see — a caption, a label, an input type.

### Two decisions worth knowing about

**`workers: 1`, no parallelism.** One database, one server-side session.
Parallel tests would page the customer list while another selects a row
in it; the resulting failures aren't defects, and chasing them costs
more than the parallelism saves.

**Each test logs in again**, rather than reusing a saved `storageState`.
The usual advice suits token-based sessions; this app keeps state on the
server — including the drill-down view stack — and expires it on
`SessionTimeOut`. Restoring a cookie can hand you a session the server
already discarded, and that failure reads like a broken test. A login is
one round trip; the isolation is worth it.

### Tests that encode real bugs

Two exist specifically because this code shipped with those defects:

- **pagination** — `gtManual` suppressed the framework's automatic
  `GridRefresh` and the list rendered a header with no rows. Asserting
  rows are present catches it immediately.
- **row highlight** — selection only works because the row template's
  root is a `table.WebList_Row`; the framework looks for exactly that
  when applying `WebList_Selected`. A template edit losing that root
  would otherwise fail silently.

### Nothing writes data

Every test reads. The delete test deliberately **cancels** the
confirmation, so the guard is exercised without losing a record. If you
add create/update coverage, have it create its own uniquely-named
customer and keep it in a separate spec — `Tools/Test-McpServer.ps1`
follows that pattern with `-IncludeWrites`.

`node_modules/` is not in the zip; `npm install` restores it from the
committed `package-lock.json`.

## Still Studio-managed (unchanged)

- `Claudetest.sws.lock`
- MySQL Connection ID (`ClaudetestDB`)
- `DDSrc/Example.fd` / `cExampleDataDictionary.dd` remain unused templates
