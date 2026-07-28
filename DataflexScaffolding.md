# CLAUDE.md — DataFlex Workspace Scaffolding

This file captures what was learned building the `Claudetest` DataFlex
Web/Mobile workspace, so future workspace-creation requests don't need to
re-derive file formats from scratch or re-ask the same setup questions.

## Defaults — ask only what's not covered here

When asked to scaffold a new DataFlex workspace, assume these unless told
otherwise, and only ask for what's genuinely missing (workspace name, and
the filesystem path if it matters):

- **DataFlex version: 26.0**
- **Database: MySQL**, connected via a Managed Connection / Connection ID
  (never hardcode credentials or a raw connection string — that's always
  set up in Studio's Database → Manage Server/Database Connections)
- **App type: Web/Mobile**, using the `cWebApp` framework with the
  drill-down navigation pattern (`cWebViewStack` + `cWebMenuButton`), themed
  `Df_Flat_Touch`, not a classic desktop menu bar
- **Copy real framework assets in** (CssThemes, Images, WebApi, WebUI,
  favicons/manifest, the Web UI + Web UI Server + theme packages in
  `DfPkg/`) rather than leaving them for Studio's package manager to fetch,
  whenever a working reference workspace is available to copy from

## Golden rule: don't guess file formats — verify against a real workspace

The first scaffolding attempt failed because several core file formats were
guessed instead of verified, and Studio couldn't open the result. If a
reference workspace (a real `.zip` export from Studio) is available, always
extract and inspect it before writing new files — copy real files where
possible, and only hand-write files whose format has been directly
confirmed. If no reference is available, say so explicitly and flag which
parts are best-effort.

## Verified file formats and structure

### Root: `<WorkspaceName>.sws` (JSON — not binary/XML)

```json
{
    "dependencies": [
        "DataFlex-dev/Web UI Server#1.0.51",
        "DataFlex-dev/Web UI Flat Desktop Theme#1.0.4",
        "DataFlex-dev/Web UI Flat Touch Theme#1.0.4",
        "DataFlex-dev/Web UI Material Theme#1.0.3"
    ],
    "df": 26.0,
    "projects": [
        "WebApp.src"
    ]
}
```

- `dependencies` only needs **top-level** packages — don't list a package's
  own transitive dependencies (e.g. "Web UI Server" already depends on
  "Web UI" per its own `df.sws`, so "Web UI" isn't listed again here).
- If exact version numbers for a given DataFlex release aren't known with
  confidence, leave `dependencies: []` and say so — Studio's package
  manager will prompt to resolve `Use` statements it can't find, rather
  than silently failing on a wrong guess.
- `Claudetest.sws.lock` (the resolved-versions lockfile) is Studio-generated
  on first open — don't hand-write it.

### `Programs/Config.ws` (INI) — the *runtime* workspace descriptor

This is NOT at the workspace root — it lives in `Programs/`, next to where
the compiled `.exe` ends up (matches official deployment docs: the `.ws`
file must sit alongside the compiled program).

```ini
[Workspace]
Home=..\
AppSrcPath=AppSrc
AppHTMLPath=AppHTML
BitmapPath=Bitmaps
IdeSrcPath=IdeSrc
DataPath=Data
DDSrcPath=DDSrc
HelpPath=.
ProgramPath=Programs
FileList=Data\Filelist.cfg
Description=<WorkspaceName>
```

### Project naming convention

The main project is conventionally named **`WebApp.src`** /
**`WebApp.cfg`** (matching pair by filename stem) regardless of what the
workspace itself is called. Don't name it after the workspace.

### `AppSrc/WebApp.cfg` (INI) — build/deploy settings

Sections: `[WebApp]` (SessionTimeOut, VirtualDir, BaseURL, TestURL),
`[Application]`, `[Version]` (company/product/file version metadata),
`[Compiler]` (Platform=x64), `[Icons]`.

### `AppSrc/WebApp.src` — main entry point pattern (Web/Mobile)

`WebApp.src` also `Use`s two small **workspace-level** files that live
directly in `AppSrc/`, not inside any `DfPkg` package — easy to miss:

```dataflex
// SessionManager.wo
Use cWebSessionManagerStandard.pkg
Object oSessionManager is a cWebSessionManagerStandard
End_Object
```
```dataflex
// WebResourceManager.wo
Use cWebResourceManager.pkg
Object oWebResourceManager is a cWebResourceManager
End_Object
```

`cWebSessionManagerStandard.pkg` comes from the `Web UI Server` package;
`cWebResourceManager.pkg` ships with core DataFlex. Always create both
files (or copy from a reference) alongside `WebApp.src` — leaving them out
produces `Error 4313: Include file not found` at compile time.

```dataflex
Use AllWebUIServerClasses.pkg
Use cApplication.pkg
Use cConnection.pkg
Use cWebApp.pkg
Use cWebMenuItem.pkg
Use cWebPanel.pkg
Use cWebMenuButton.pkg

Register_Object oDashboard

Object oApplication is a cApplication
    Object oConnection is a cConnection
        Use LoginEncryption.pkg
    End_Object
End_Object

Object oWebApp is a cWebApp
    Set psTheme               to "Df_Flat_Touch"
    Set peAlignView             to alignCenter
    Set psApplicationTitle       to "<AppName>"
    Set peApplicationStyle        to wvsDrillDown   // mobile drill-down nav
    Set peApplicationStateMode     to asmHistoryAndUrls
    Set peThemePreference           to tpSystemPreference

    Object oViewStack is a cWebViewStack
    End_Object

    Object oHeaderPanel is a cWebPanel
        Set peRegion to prTop
        Object oMenuPanel is a cWebPanel
            Set peRegion to prLeft
            Object oMenuButton is a cWebMenuButton
                Object oDashboard_itm is a cWebMenuItem
                    Set psCaption to "Dashboard"
                    WebRegisterPath ntNavigateBegin oDashboard
                    Procedure OnClick
                        Send NavigatePath
                    End_Procedure
                End_Object
                // one oXxx_itm per view, same pattern
            End_Object
        End_Object
    End_Object

    Use SessionManager.wo
    Use WebResourceManager.wo
    // Use <ViewName>.wo — one per view, as built

    // REQUIRED in wvsDrillDown mode: tells oWebApp which view is the root
    // of the stack. Without this, MSG_RESTORESTATE_DRILLDOWN has nothing
    // to restore to on load and fails with "Invalid message
    // GET_NAVIGATEINITIALIZE". Place it after the view Use lines, before
    // OnLoad. Update it if the initial view is renamed/replaced.
    Set phoDefaultView to oDashboard

End_Object

Send StartWebApp of oWebApp
```

### Login: every view is gated by default — build the real flow, don't just disable it

`cWebWindow.AllowAccess` (every view's access check) delegates to
`ValidLogin`, which checks `peLoginMode` on `oWebApp` —
**defaults to `lmLoginRequired`** (`cWebSessionHost_mixin.pkg`, part of
Web UI Server). With no session logged in, this fails and produces a bare
"Access Denied" dialog with no further detail — including on the very
first/initial view.

The real fix is a `Login.wo`, not disabling the check (that's a stopgap
at best). Pattern, copied from a working reference:

- `Login.wo` is self-contained: `cWebView`, `cWebForm`, `cWebButton`,
  `cWebPanel`, `cWebLabel`, `cWebSpacer`, `cWebImage` — all in the core
  `Web UI` package, nothing extra needed. It registers itself as the
  login view via `Delegate Set phoLoginView to Self` inside its own body.
- `DoLogin` calls `UserLogin` on the session manager, which checks
  against the `WebAppUser` table — needs `WebAppUser`'s `.fd`/`.dd`/data
  files present (see the "base system tables" section above).
- In `WebApp.src`: `Set peLoginMode to lmLoginRequired` (the default —
  set it explicitly for clarity) and `Use Login.wo` alongside the other
  views. **Nothing else to wire up** — `ForcedInitialView`
  (`cWebSessionHost_mixin.pkg`) automatically routes to `phoLoginView`
  instead of `phoDefaultView` when nobody's logged in yet.
- If a reference's `WebAppUser.dat` came along with it, check for a demo
  credential hint in the login form's caption/placeholder text before
  assuming you need to create a user manually.

Only reach for `Set peLoginMode to lmLoginNone` as a deliberate, called-out
temporary stopgap when there's no `Login.wo` to copy from anywhere and
building one from scratch is out of scope for the current ask.


- `cApplication.pkg` / `cConnection.pkg` ship with core DataFlex (not a
  package-manager dependency) — always resolvable, nothing to copy.
- `cWebApp.pkg`, `cWebViewStack.pkg`, `cWebMenuButton.pkg`, etc. DO come
  from `DfPkg/` packages (Web UI + Web UI Server) — must be present (copied
  or resolved) for the project to compile.

### Adding a table/feature from a reference workspace — checklist

Copying one feature across is never just its `.wo` files. For each table:

1. `Data/<Table>.dat|.hdr|.tag` + every `<table>.k#` key file (key files
   are lowercase in the reference even when the table files aren't).
2. `DDSrc/<Table>.fd` and `DDSrc/c<Table>DataDictionary.dd`.
3. **Read the DD's `Open` statements and `Add_Client_File` /
   `Add_System_File` lines** — each names another table that must also be
   physically present, even if that table has no UI. This is the usual
   hidden cascade.
4. Check the DD for `Field_Value_Table` (needs the matching code records
   in `CodeMast`), `Field_Auto_Increment` (needs `DFLastID`), and
   `Field_WebPrompt_Object` (needs that lookup `.wo` copied too — often
   `Use`d at the bottom of the `.dd` under `#IFDEF Is$WebApp`).
5. Add the table → DD entry to `DDSrc/DDClassList.xml`.
6. In `WebApp.src`: a `Use <View>.wo` line per view, plus a
   `cWebMenuItem` with `WebRegisterPath ntNavigateBegin <oSelectView>`.
   The menu item can reference the view before its `Use` line without a
   `Register_Object` — `WebRegisterPath` resolves lazily.
7. Grep the copied views for references to views you did NOT copy
   (`Register_Object oSomethingElse`, `WebRegisterPath ... oSomethingElse`)
   and strip those navigation paths, or the cascade never ends.

### File encoding: exactly one BOM

DataFlex source files are UTF-8 with a single leading BOM (CRLF line
endings). **Two BOMs produce `Error 4298: Command not found` on line 1**
with an apparently empty command name — the compiler eats the first as
the encoding marker and parses the second as source.

This is easy to cause when editing files programmatically: reading with
`utf-8-sig` strips a BOM, writing with a `b"\xef\xbb\xbf"` prefix adds
one, and mixing the two conventions across successive edits makes them
accumulate. After any byte-level write, verify:

```python
raw = open(path, "rb").read()
n = 0
while raw[n*3:(n+1)*3] == b"\xef\xbb\xbf": n += 1
assert n == 1
```

Also worth checking for a BOM stranded mid-file and for mixed line
endings inside a single file.

### Playwright tests against a DataFlex web app

Lives in `Tests/` (TypeScript, page objects in `pages/`, specs in
`specs/`). `npm install && npx playwright install chromium`, then
`npm test` or `npm run ui`.

**Selectors:** DataFlex generates its own DOM ids and they are not
stable. Key off the `psCSSClass` values the application sets on itself —
those change only deliberately. Two framework internals are worth
knowing and both should be commented at the point of use:
`.WebList_Selected` (the active row) and `[data-dfrowid]` (serialized
RowID). Otherwise use what the user sees: caption, label, input type.
Put every selector in a page object so a DOM change is one edit.
`npx playwright codegen <url>` finds working selectors; replace its
guesses with application classes before keeping them.

**Config:** `workers: 1` and `fullyParallel: false` — one database and
server-side session state, so parallel runs interfere and produce
failures that are not defects.

**Auth:** log in per test rather than reusing `storageState`. The
session lives on the server (including the drill-down view stack) and
expires on `SessionTimeOut`; a restored cookie can reference a session
that no longer exists, and the failure looks like a broken test.

**Waiting:** the page is an empty shell until the framework builds the
UI into `#viewport` — wait for a real element, not for `load`.

### Naming convention (user's standard)

Procedures and functions the user writes are prefixed **`p`** and **`f`**
respectively — `pCallbackDelete`, `pSendLog`, `fCheckReadonly`. Apply
this to every new class, procedure and function. Events keep the
framework's `On<Name>` form.

This is not only style: `cObject`, `cJsonObject` and other intrinsic
classes are defined in fmac with **no `.pkg` source to read**, so their
member lists cannot be checked. A helper named `HasMember` silently
collided with `cJsonObject.HasMember` and produced 16 "Wrong Number of
Arguments" errors. The prefix keeps user code out of that namespace.

The prefix covers *messages* but **not local variables**, which share
the constant namespace — `iWindow` collides with a framework constant,
for instance. Prefer specific local names (`iPageWindow`) over short
generic ones.


Views are named **`<Entity>Select.wo` / `<Entity>Zoom.wo`** with matching
objects `o<Entity>Select` / `o<Entity>Zoom` — entity first, not
`SelectCustomer.wo` / `oSelectCustomer`. Lookup dialogs follow as
`<Entity>Lookup.wo` / `o<Entity>Lookup`. Reference workspaces use the
opposite order, so rename on copy — and update every reference:
`WebApp.src` (`Use` lines, menu items, `WebRegisterPath`), inter-view
navigation, and the `.dd` (`Register_Object`, `Field_WebPrompt_Object`,
and the `Use <Lookup>.wo` under `#IFDEF Is$WebApp`).

### `AppHTML/CssStyle/application.css` is NOT covered by the themes

Theme packages supply the framework classes (`MobileList`, `RowCaption`,
`RowDetail`, `HeaderPanel`, `LoginView`, `LoginBackground`). App-specific
classes live in `application.css`, which is workspace source, not a
package. Copying a view without it silently produces an unstyled screen
with no error — e.g. `Dashboard.wo`'s `WidgetsDashboard` / `TabButton` /
`TabLine` / `*DashboardBtn` classes all live there. When copying views
from a reference, copy its `application.css` too, and check any
`Images/*.svg` its rules reference.

### `cWebList` card-style rows

Multi-line "card" rows are built with `pbNewLine`, `piListColSpan` and
`piListRowSpan` on the columns; the visual work is CSS. Useful hooks:

- `pbAllowHtml` — column value is injected as markup, not encoded. Fine
  for markup built in `OnSetCalculatedValue`; never point it at raw
  record data without encoding.
- `OnSetCalculatedValue String ByRef sValue` — unbound (no `Entry_Item`)
  computed columns.
- `OnDefineCssClass String ByRef sCSSClass` — per-cell conditional
  styling (e.g. highlight an overdue balance) without adding a column.
- The column's `psCSSClass` lands on the `td` itself, alongside
  `dfData_Text` — so target `td.MyClass`, not `.MyClass .dfData_Text`.
- **Sort menu items address columns by position.** Reordering or
  inserting columns silently breaks every `WebSet piSortColumn` and the
  list's initial `piSortColumn`. Remap them in the same edit.

### Implementing a supplied HTML/Tailwind mockup

- **Don't** add the Tailwind CDN script. It's a browser-side JIT
  compiler for prototyping and its preflight reset fights the DataFlex
  theme across the whole app. Hand-translate the token set into CSS
  variables in `application.css`.
- Scope everything under one class set via `psBodyCSSClass` on the view,
  so other views keep the stock theme.
- Prefer inline SVG over icon webfonts (Material Symbols et al.) — a
  failed icon-font load renders raw ligature text like `more_vert`.
  Text webfonts are fine with a system fallback stack.
- Mockups carry invented data. Map each column to a real field and say
  which ones had no equivalent rather than inventing one.
- Static figures in a mockup ("142") should be computed. A table scan in
  `OnLoad` is acceptable for embedded demo tables; flag it for
  replacement with an aggregate query on SQL-backed tables.

### Applying a mockup to an editable form (zoom) view

Detail mockups are usually read-only label/value pages; a zoom view is a
DD-bound form. Keep the form real and restyle it — don't replace fields
with HTML.

- Set `peLabelPosition to lpTop` on the fields, then style
  `.WebControl > div.WebCon_TopLabel > label` as the uppercase
  micro-label. Group captions are
  `.WebGroup.WebGrp_HasCaption > .WebCon_Inner > div > .WebGrp_Caption`;
  the card surface is `.WebGroup.<YourClass> > .WebCon_Inner`.
- Use `cWebGroup` as the mockup's "cards" and `cWebHtmlBox` only for
  non-editable chrome (header, stat blocks, progress bars).
- **Prefer flowing cards directly in the view's column grid** over
  nesting `cWebPanel` wrappers to build columns: give each card its own
  `piColumnSpan` (8/4, 8/4, 8) and let them wrap. Fewer moving parts,
  and the responsive collapse is one `WebSetResponsive` per card.
- Fill dynamic `cWebHtmlBox` content from `OnNavigateForward` — the
  record is already in the buffer there (that's where the stock
  `SetBreadcrumbCaption (Trim(Customer.Name))` reads from). Handle the
  no-record case explicitly: a new record should get placeholder copy,
  not zeros.
- Fields the DD marks `DD_DISPLAYONLY` are good candidates to move out
  of the form into a read-only stat card — no capability is lost.
- Don't recreate mockup buttons that duplicate the existing action bar,
  and don't add a FAB with nothing to do.

### `cWebHtmlList` — custom row markup, including column buttons

`cWebHtmlList` is a `cWebList` subclass that renders each row from
`psHtmlTemplate`, with `{{oColumnName}}` replaced by that column's
rendered cell content. That includes `cWebColumnButton` columns —
`cellHtml` emits the real `<button data-dfbtnid="...">` markup, with
`pbDynamic` / `OnDefineButtons` support intact. Columns are never laid
out as cells, so extra ones cost nothing; don't set `pbRender False` on
them.

**The one catch is click dispatch, not the class hierarchy.** The two JS
view classes differ:

| | resolves column? | calls |
|---|---|---|
| `cWebList` view | `determineCell(target)` | `cellClick(e, sRowId, iCol)` |
| `cWebHtmlList` view | no | `cellClick(e, sRowId, -1)` |

and `cellClick` gates `cellClickAfter` — which raises a column button's
`OnClick` — behind `index >= 0`. So a button's own `OnClick` never fires
in an HtmlList.

**Bridge it** rather than abandoning the class, and keep each button's
`OnClick` where it belongs. Wrap each button in the template with a span
carrying `data-ServerOnElemClick="<name>"`, which the HtmlList view
*does* look for, then add one `OnElemClick` on the list that positions
the DD on the clicked row and hands off:

```dataflex
Procedure OnElemClick String sRowId String sParam
    Handle hoServer
    Boolean bOk

    Get Server to hoServer
    If (hoServer = 0) Procedure_Return

    // same pair cWebList uses internally
    Get FindByRowIdEx of hoServer (Main_File(hoServer)) (DeserializeRowID(sRowId)) to bOk
    If (bOk = False) Procedure_Return

    WebSet psCurrentRowID to sRowId

    Case Begin
        Case (sParam = "detail")
            Send OnClick of oDetail_wbtn "DEFAULT" sRowId
            Case Break
        Case (sParam = "delete")
            Send OnClick of oTrashCan_wbtn "E" sRowId
            Case Break
    Case End
End_Procedure
```

Positioning the DD first is what makes this equivalent to a plain list:
`OnElemClick` fires *before* the row becomes current, so without the
`FindByRowIdEx` the button would act on whatever record was previously
selected. With it, each `OnClick` — and any `psCurrentRowID` guard
inside it — behaves exactly as it would in a `cWebList`.

The second button id is whatever `OnDefineButtons` passed to `AddButton`
(`"E"` in the house pattern); a non-dynamic button is `"DEFAULT"`.

**Selection highlighting needs a `table.WebList_Row` root.** The
framework marks the selected row with
`containerElem('table.WebList_Row[data-dfrowid="..."]')`, and
`cWebHtmlList` does not override that lookup — its view class only
overrides `tableHtml` and `onTableClick`. A `<div>` template root
therefore renders fine but can never receive `WebList_Selected`, so no
CSS will highlight it. Make the template root
`<table class="WebList_Row ..."><tr><td>…</td></tr></table>` and keep
the card layout on an inner div. `initTemplate` injects
`data-dfisrow`/`data-dfrowid` into the first tag either way, so clicks
are unaffected.

Also note: `cWebListSwipeButton` does not work in an HtmlList (template
rows have no swipe layer) — move anything it carried into the row
template or the action menu.

### Refreshing one view after another changes data

Pattern: the changing view raises a `{ WebProperty=Server }` flag on the
view that needs refreshing; that view acts on it in `OnShow` (with
`pbServerOnShow to True`), which fires when it returns to the top of the
drill-down stack.

Do **not** do the refresh from a navigate-back hook:

- `OnNavigateBack` is sent to the *invoking object*, so a list with row
  buttons needs it in several places (the button for an icon click, the
  list for a row click).
- Anything that moves the record buffer during navigate-back can break
  the return. `cWebList.GetNavigateBackData` re-finds the row and then
  asserts `IsSameRowID(CurrentRowId(DD), psCurrentRowId)`, erroring with
  *"Assert: Different ids in weblist"*.

Any procedure that scans a table should save and restore the DD
position, so it stays safe to call from anywhere:

```dataflex
Get HasRecord of hoDD to bHadRecord
If (bHadRecord) Get CurrentRowId of hoDD to riSave
// ... Clear / Find loop ...
If (bHadRecord) Get FindByRowIdEx of hoDD (Main_File(hoDD)) riSave to bOk
Else Clear <Table>
```

Cross-view references need no `Register_Object` as long as the target
view's `.wo` is `Use`d first in `WebApp.src`.

### Paging a cWebList / cWebHtmlList

The stock list streams a window around the current record and extends it
on scroll — there are no stable page boundaries, and `piLimitRows` only
caps the first N of the current ordering. For numbered pages:

- `Set peDbGridType to gtManual` and supply rows from
  `Procedure OnManualLoadData tWebRow[] ByRef aTheRows String ByRef sCurrentRowID tWebGroupConfig[] ByRef aTheGroups tWebGroupHeader[] ByRef aTheGroupHeaders`.
- **Keep `pbDataAware` True.** Then `Get LoadGridRow to tRow` builds each
  row from the column objects — bound and calculated columns, per-cell
  CSS — and serialises a real `RowID`, so row clicks, selection and any
  `DeserializeRowID` / `FindByRowIdEx` logic keep working. Losing that is
  the main risk of manual loading.
- **Iterate with the read path, not `Send Find`.** `Send Find` on a DD
  routes to `Request_Find`, the interactive DEO find — it checks
  `OPERATION_MODE`, moves focus and fires `OnPreFind`/`OnPostFind`, and
  is not valid during a data load (it fails silently, leaving an empty
  list). Copy what `cWebList.LoadData` does:

  ```dataflex
  Get IndexOrder to iOrdering               // follows the sort column
  WebGet pbReverseOrdering to bReverse
  Get ReadDDFirstRecord hoDD iOrdering bReverse to riFirst   // Request_Read
  Send EstablishDDFindMode GT hoDD iOrdering bReverse
  While (Found)
      If (OnCheckCustomConstrains(Self)) Begin
          Get LoadGridRow to tRow
      End
      Send Locate_Next to hoDD
  Loop
  ```

  Honour `pbReverseOrdering` and `OnCheckCustomConstrains` or reverse
  sorting and row filtering quietly stop working.
- `Send LoadDataPage "first" ""` re-runs the load and ships the new page
  to the client.
- **`gtManual` switches off the automatic load.** Every `GridRefresh` in
  `InitialRefresh` and `Refresh` is wrapped in
  `If (eDBGridType<>gtManual)`, and `LoadData` is only reachable from
  `GridRefresh` or the client-published `LoadDataPage`. So a manual grid
  never loads by itself, and the client won't ask either — manual mode
  reports `bFirst`/`bLast` both True, so it thinks it has everything.
  Symptom: header renders, no rows, no footer.
- **Do not put the load back by augmenting `Refresh` or
  `InitialRefresh`.** Those fire during object construction —
  `End_Construct_Object` → `Attach_DEO_To_Server` →
  `Add_User_Interface` → `Refresh` — long before a web session exists,
  so any `WebGet`/`WebSet` in the load raises
  *"phoWebServerPropStore is not initialized"* (error 4402) at startup.
  Call the reload only from request-time entry points: the view's
  `OnLoad` and `OnShow`, a client click, and `RefreshListFromDD`.
- `RefreshListFromDD` must trigger the reload itself in manual mode —
  the stock version only repositions the DD, and its `Refresh`
  notifications are guarded out.

**General rule:** anything touching a `WebProperty` must be reachable
only from inside a request. Before hanging work off a framework event,
check whether that event also fires during construction — the object
structure is built once at process start, with no session.
- Row totals need a full walk of the index — isolate that in its own
  overridable procedure so a SQL-backed table can swap in a real
  `COUNT(*)`.

Pager markup goes in `psHtmlAfter`. Elements there are outside any row,
so `data-ServerOnElemClick` on them fires `OnElemClick` with an **empty
row id** — handle your own params and `Forward Send OnElemClick` the
rest, so views can still use their own elements in the row template.

### Row action buttons (house pattern)

Subclass `cWebColumnButton`, set the defaults in `Construct_Object`
(`piWidth`, empty `psCaption`, `pbFixedWidth`, `pbResizable`,
`psBtnCssClass`, `peAlign`) and override `OnDefineTooltip`. Confirmed
available in Web UI 1.0.51: `pbDynamic` + `OnDefineButtons`/`AddButton`,
`ShowYesNo` / `ShowInfoBox` (in `cWebHostAPI_mixin.pkg`),
`RefreshListFromDD`, `psCurrentRowId`, and `cmYes` (`cWebBaseDEOServer.pkg`).

Delete pattern: `ShowYesNo` → a `WebPublishProcedure`-published callback
→ `Request_Delete` on the DD → `RefreshListFromDD`.

Row context matters: the DD deletes its *current* record. In a plain
`cWebList`, the button's `OnClick` can fire on a row that is not the
selected one, so guard with `sRowId` vs the list's `psCurrentRowId` and
refuse the mismatch. Via the `cWebHtmlList` bridge above the work
happens in `OnRowClick`, after the row is already current, so no guard
is needed.

Icons: the house `icon-menu-*` class names are wired up in
`application.css` the same way `theme.css` does its own — a `content:`
character in the **`dataflex-mobile`** icon font that ships in
`AppHTML/CssThemes/<theme>/Fonts/`. Compare
`.WebIcon_Delete:before { content: "x"; }`. The theme already applies
that font to `.WebButtonIcon:before`, so any button whose
`psBtnCssClass` includes `WebButtonIcon` picks it up — no extra font
needed. Decode `Fonts/dataflex-mobile.svg` (or open
`Fonts/icons-reference.html`) to find the character for a glyph;
`M` = `df-info-outlined`, `?` = `df-delete-filled`, `x` =
`df-delete-outlined`.

Colour the glyph on `:before`, not on the button — the theme sets
`.WebButtonIcon:before { color: ... }` and a rule on the button element
will not win. Hover rules also need to out-specify
`.WebButtonIcon:hover:before`, which otherwise forces the stock warning
colour.

### Two DataFlex rules worth remembering

- **`Construct_Object` can only be defined in a `Class`**, never in an
  `Object … End_Object`. Error 4391 "Illegal code placement". In an
  object instance just put the `Set` statements in the body — it already
  runs at construction time.
- **A name that resolves to an intrinsic message wins.** Error 4379
  "Wrong Number of Arguments" on a call that looks correct usually means
  the name collided with something built in, not that the arity is
  wrong. Compare against a neighbouring call of the same shape: if that
  one compiles, it is the name.

`cJsonObject` has its own `HasMember`, taking the member name only:
`Get HasMember of hjObject sName to bHas`. Prefer it over inferring
absence from `Get Member` returning 0.

**`HasMember` returns True for a member of type null** (documented). That
answers "does this key exist", not "did the caller supply a value" — so
for optional inputs that drive a write, test the type as well, or an
explicit `null` will overwrite good data:

```dataflex
Get HasMember of hjParent sName to bHas
If (bHas) Begin
    Get Member of hjParent sName to hjMember
    Move (not(IsOfJsonType(hjMember, jsonTypeNull))) to bHas
    Send Destroy of hjMember
End
```

Absence is still the right test where the protocol defines meaning by
absence — e.g. a JSON-RPC notification is a request with no `id`, and
`"id": null` is technically an id.

### Source order in a standalone (non-web) program

A `.dd` carries its `Open <Table>` statements at **file scope**, so they
execute where the `Use` appears — not when a DD object is constructed.
The filelist they need is loaded when `cApplication` opens the workspace
in its `End_Construct_Object` (`psAutoOpenWorkspace`, default
`'Config.ws'`, resolved next to the exe).

So in a console/utility `.src`, every `Use` that reaches a `.dd` must sit
**after** the `oApplication` object:

```dataflex
Use cApplication.pkg
Object oApplication is a cApplication ... End_Object
Use cMyDataDictionary.dd        // and anything that Uses it
Object oMyDD is a cMyDataDictionary End_Object
```

Getting it wrong gives
`Entry does not exist in filelist Table = (nn), Source = Flexdrvr.Open`
at run time. Web apps do not hit this because their view `.wo` files are
`Use`d inside `oWebApp`, well after `oApplication` closes — so the
correct order comes for free there and is easy to forget elsewhere.

Useful for console harnesses: `Seq_Append_Output_Channel` /
`Seq_Close_Channel` (`seq_chnl.pkg`) to tee output to a log file, and the
global `Info_Box` to stop the window closing on exit.

### Building an MCP server in DataFlex

MCP is JSON-RPC 2.0 over a transport. A tools-only server implements
`initialize`, `notifications/initialized`, `tools/list`, `tools/call`
(and optionally `ping`). Notifications have no `id` and get no response
at all — return an empty body, not an error.

Keep the protocol engine transport-agnostic: a
`Function HandleRequest String sBody Returns String` can be driven by
HTTP, by stdio, or by a console harness, and the last of those means the
tools can be tested before any transport question is settled. See
`AppSrc/cMcpJsonRpc.pkg` + `AppSrc/McpTest.src`.

Verified `cWebHttpHandler` API (core DataFlex) — the base for any HTTP
endpoint:

```dataflex
Procedure Construct_Object
    Forward Send Construct_Object
    Set psPath  to "mcp"          // must match <location path> in web.config
    Set psVerbs to "POST"
    Set peErrorType to httpErrorJson   // httpErrorHtml/Json/Soap/Empty
End_Procedure

// One event per verb; the last argument is the body size.
Procedure OnHttpPost String sPath String sContentType String sAcceptType DWord iSize
    UChar[] ucData
    Get RequestDataUChar -1 to ucData      // -1 = whole body
    // ... or Get RequestDataString to sBody, which wraps the above
    Send SetResponseStatus 200 "OK" 0      // BEFORE any output
    Send AddHttpResponseHeader "Content-Type" "application/json; charset=utf-8"
    Send OutputUChar ucData                // or Send OutputString sText
End_Procedure
```

Also available: `OnHttpGet/Put/Patch/Delete/Request`, `HttpRequestHeader`,
`UrlParameter`, `ParseContentType`, and the `psRequest*` properties.

**Status and headers must precede any output** — the class errors with
*"Data has already been sent"* once `pbDataSent` is True.

For JSON, stay on the UChar path (`RequestDataUChar` → `ParseUtf8`,
`StringifyUtf8` → `OutputUChar`) as `cWebServiceDispatcher` does, rather
than round-tripping through a DataFlex string; it preserves non-ASCII
data in both directions.

Error handling splits by context: `ErrorQueueStart` / `ErrorQueueEnd` /
`ErrorCount` are session-based and only exist on `cBaseWebComponent`
descendants, so they belong in the endpoint. Inside plain classes use
the DD's own idiom — `Move False to Err`, run the operation, test `Err`
— which works with no web session and is what `Datadict.pkg` itself does.

Verified `cJsonObject` API (core DataFlex):

```dataflex
Get Create (RefClass(cJsonObject)) to hj
Send InitializeJsonType of hj jsonTypeObject   // or jsonTypeArray
Send SetMemberValue of hj "name" jsonTypeString sValue
Send SetMemberValue of hjArray iIndex jsonTypeString sValue  // arrays: numeric index
Send SetMember of hj "name" hjChild            // object-valued member
Send AddMember of hjArray hjChild              // array element
Get Member of hj "name" to hjChild             // 0 when absent - use as an "exists" test
Get MemberValue of hj "name" to sValue
Get MemberCount of hj to iCount
Get ParseString of hj sText to bOk
Get Stringify of hj to sText
Send DataTypeToJson of hj <struct>             // struct <-> json
Get JsonToDataType of hj to <struct>
Get ParseUtf8 of hj ucData to bOk              // UTF-8 byte input
Get StringifyUtf8 of hj to ucData              // UTF-8 byte output
IsOfJsonType(hj, jsonTypeObject)
Set pbRequireAllMembers of hj to False         // tolerant parsing
```

The type enum is exactly `jsonTypeNull`, `jsonTypeBoolean`,
`jsonTypeDouble`, `jsonTypeInteger`, `jsonTypeObject`, `jsonTypeArray`,
`jsonTypeString` — there is **no** `jsonTypeNumber`. There is no
`SetValue`; scalars always go through `SetMemberValue`. `cJsonObject`
and `cObject` are both intrinsic (defined in fmac), so `cJsonObject.pkg`
holds only the constants and there is no `cObject.pkg` to `Use`.

Every handle you create or `Get Member` must be `Send Destroy`-ed.

Route the endpoint by adding a `<location>` block to
`AppHTML/web.config` pointing `dataflexHttpModule` at the handler
object, exactly as `WebServiceDispatcher.wso` is routed.

**Let the DD do the validating.** Drive every write through the data
dictionary rather than the table, so `DD_REQUIRED`, `DD_NOPUT`,
`DD_DISPLAYONLY` and value tables apply to a model exactly as they do to
the UI — no second, weaker path into the data. Auto-increment keys and
derived figures should not appear in a tool's input schema at all.

**Guardrails worth building in by default:** destructive tools off
unless switched on (and unadvertised in `tools/list` while off, so the
model cannot attempt what it cannot see); an explicit `confirm` argument
on anything irreversible; a capped result set so a broad query cannot
serialise a whole table into a context window; and an audit hook on
every mutating call.

### `DDSrc/*.fd` — field definition macro format

```
#REPLACE FILE25 <TableName>
#REPLACE <TableName>.<Field> |FN25,<index>   // numeric
#REPLACE <TableName>.<Field> |FS25,<index>   // string
```

Normally Studio-generated via Table Editor / SQL Conversion Wizard — hand-
write only as a placeholder template, and say so.

### `AppHTML/Index.html` — structure and section order

1. `<meta charset>`, viewport
2. `<title>`
3. Favicon/manifest/tile block (see below) — copy the referenced icon
   files into `AppHTML/` whenever adding this block, don't leave dangling
   links
4. App-level CSS link (`CssStyle/application.css`)
5. **Managed Includes** block — bounded by
   `<!-- Managed Includes ... -->` / `<!-- End of Managed Includes ... -->`
   comments; Studio owns this block and auto-inserts framework/theme
   CSS+JS lines here as packages are added via Tools → Manage Packages.
   Only hand-write lines here that match files actually copied into
   `AppHTML/` (e.g. `WebUI/system.css?v=1.0.51`,
   `CssThemes/Df_Flat_Touch/theme.css?v=1.0.4`) — never reference a
   package's files without also placing those files.
6. `<!-- DataFlex Custom Controls ... -->` marker (also Studio-owned)
7. WebApp init script: `new df.WebApp("WebServiceDispatcher.wso")` +
   `displayApp("#viewport")`

Favicon/manifest/tile block (goes right after `<title>`):

```html
<link rel="manifest" href="manifest.json">

<link rel="icon" type="image/png" href="favicon-32x32.png" sizes="32x32">
<link rel="icon" type="image/png" href="favicon-16x16.png" sizes="16x16">
<link rel="icon" type="image/png" href="favicon-96x96.png" sizes="96x96">
<link rel="icon" type="image/png" href="favicon-194x194.png" sizes="194x194">

<link rel="apple-touch-icon" sizes="180x180" href="apple-touch-icon-180x180.png">
<link rel="apple-touch-icon" sizes="152x152" href="apple-touch-icon-152x152.png">
<link rel="apple-touch-icon" sizes="120x120" href="apple-touch-icon-120x120.png">
<link rel="apple-touch-icon" sizes="72x72" href="apple-touch-icon-72x72.png">

<meta name="msapplication-TileColor" content="#e64812">
<meta name="msapplication-TileImage" content="mstile-144x144.png">
<meta name="theme-color" content="#ffffff">

<meta name="mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-status-bar-style" content="black">
```

Required binary files if this block is added: `favicon.ico`,
`favicon-{16,32,96,194}x16.png`, `apple-touch-icon-{72,120,152,180}x*.png`,
`mstile-{70,144,150,310x150,310x310}.png`, `android-chrome-{36,48,72,96,
144,192}x*.png`, `browserconfig.xml` (auto-discovered by IE/Edge, no
explicit `<link>` needed). `manifest.json` should list the `android-chrome-*`
icons once those files exist — don't reference icons that aren't there.

### Other `AppHTML/` support files (INI/XML, verified)

- `web.config` — IIS routing via `<dataflexHttpModule>`, two `<location>`
  blocks: `DfResource` (object=`oWebResourceManagerProxy`) and
  `WebServiceDispatcher.wso` (object=`oWebServiceDispatcher`)
- `Global.asa` — VBScript session init,
  `PROGID="WebAppServer.Session.<major>.<minor>"`,
  `WebAppServerSession.Initialize("<AppName>")`
- `WebServiceDispatcher.wso` — `[WebService]` INI block,
  `Application=<AppName>`, `Object=oWebServiceDispatcher`

### Folder structure (top level)

```
<Workspace>/
├── <Workspace>.sws
├── AppHTML/        (Index.html, web.config, Global.asa, WebServiceDispatcher.wso,
│                     manifest.json, CssStyle/, + copied CssThemes/ Images/ WebApi/ WebUI/
│                     + favicon/icon files if that block is added)
├── AppSrc/
│   ├── WebApp.src
│   ├── WebApp.cfg
│   └── config/      (Studio populates classlist.xml here — don't hand-write)
├── Bitmaps/
├── Data/
│   └── Filelist.cfg  (binary — leave as an empty 0-byte file; Studio/tables populate it)
├── DDSrc/
├── DfPkg/            (copy needed packages here if a reference is available —
│                       at minimum Web UI + Web UI Server + one theme, for a Web/Mobile app)
├── Help/
├── IdeSrc/           (Studio's canvas layout state — populated on first save)
└── Programs/
    └── Config.ws
```

## Always Studio-managed — never hand-write these

- `DDSrc/DDClassList.xml` — a **separate** file from
  `AppSrc/config/classlist.xml`, easily confused with it. Maps each table
  number to its preferred DD class, and drives Studio's DD selector. Add
  an entry every time a new table + DD pair is added to the workspace, or
  Studio won't offer that DD.
- `AppSrc/config/classlist.xml` — **correction: Studio does NOT reliably
  regenerate this on open.** Leaving it out produces a "classes not
  defined" error when the workspace loads. If a reference workspace has
  one, don't copy it verbatim — it lists every package the reference has
  (`<libraries>`) plus every class (`<classes>`, each tagged either
  `libraryID=0` for core-DataFlex classes or the literal package ID
  string, e.g. `DataFlex-dev/Web UI`). Filter it: keep only `<library>`
  entries for packages actually present in this workspace's `DfPkg/`, and
  keep `<class>` entries tagged `0` or matching a kept library's ID —
  drop entries tagged with a library you didn't carry over.
- `<Workspace>.sws.lock` — generated when dependencies resolve
- `IdeSrc/Navigation.json`, `IdeSrc/Workspace.loc` — canvas/layout state
- MySQL connection string/credentials (`DFConnId.ini`) — set up via
  Database → Manage Server/Database Connections, never invented
- Registering the workspace / touching the Windows registry — requires
  Studio running as Administrator

## Questions still worth asking per new workspace

1. Workspace name
2. Filesystem path (if it matters to them)
3. Whether there's a reference/example workspace to base it on (always
   prefer this over guessing)

Everything else (version, database engine, app type/theme, framework
package set) defaults per "Defaults" above unless told otherwise.
