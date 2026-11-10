# Building a Laravel + Next.js business app with an AI agent: the playbook

**For the AI agent reading this.** We are building (or extending) a back-office web app: a Laravel REST API and a
Next.js dashboard, staff plus (often) a client portal, with approvals, an audit trail, reports and realtime. This file
is **how we build**: the conventions every screen and endpoint follows, the traps already paid for, and how we work
together. Read it fully before writing code, then read the project's own `CLAUDE.md`, `AGENTS.md` and `docs/`: the
project's word beats this file where they differ.

**Read [who-i-am-and-how-i-work.md](who-i-am-and-how-i-work.md) first** (me, how I talk and decide, the bar, the
rules I keep repeating), and [letshego-field-sales-lessons.md](letshego-field-sales-lessons.md) for the newest worked
example and the corrections it took.

Worked example of everything below: the Alpha Fund system (`alpha-fund-laravel/docs/*.md`, `alpha-fund-nextjs/docs/`);
Letshego Field Sales (`letshego-workspace`) for the field-sales domain, dashboards and reports.
Sibling playbooks: [deploy-with-an-ai-agent.md](deploy-with-an-ai-agent.md), and patching servers in
[../security/patch-php-node-with-an-ai-agent.md](../security/patch-php-node-with-an-ai-agent.md).

**How to use it (for me).** In the project's workspace:

> Read `coding-notes/ai-agents/build-laravel-nextjs-app-with-an-ai-agent.md` and follow it. Then <the task>.

---

## 1. How we work

- **Facts before code.** Read the code, the docs and the data before proposing. Never guess a column, route, enum
  value or package API: grep for it. This Next.js and Laravel are newer than your training data: read
  `node_modules/next/dist/docs/` and Laravel Boost / the package's docs before writing their APIs.
- **Search online for current best practice** when unsure (a library's recommended pattern, a security default, a
  version's breaking change), and say what you found with links. Prefer the vendor's docs and release notes, then
  well-known maintainers; never a random blog alone.
- **The narrowest idiomatic fix, at the source.** No broad disables (`eslint-disable` for a file, `@ts-ignore`,
  `@phpstan-ignore`, baseline entries, `any` casts) to hide a symptom. Before launch, fix a bad design properly: no
  shims or compatibility layers.
- **Root cause before fix.** Reproduce (a test, a Playwright probe, a tinker query), find why, then fix. Prove the fix
  by making the test fail on the old code.
- **Every change is tested** (Pest on the API, Playwright on screens) and **documented** (`docs/`, the user manual,
  the e2e spec list). A doc nobody updates is worse than none.
- **Commit only when asked**, in the repo the change belongs to, with a message that says what and why for a person.
  No AI attribution trailers unless the user wants them.
- **Low-RAM PCs:** heavy checks one at a time (full test suites, `tsc`, a full browser run); lint changed files only.
  A full browser run against the local API makes it slow for the person using it: run it when they are away.
- **Ask before anything destructive** (dropping data, deleting files, force pushes, migrations on live data).
- **Brainstorm, then decide, then record.** A new module starts as a model in plain words, agreed before any table;
  a question gets your view and one recommendation; what I settle goes into the workspace's `docs/decisions.md`
  (date, decision, why in my words, where the detail lives) the same turn. A round of change gets a plan file.
- **One report means every instance.** When I point at a problem on one screen, find and fix the same problem on
  every screen before answering ("hope it's not in other places").

## 2. Repos and folders

```
workspace/                  # its own repo: CLAUDE.md, handoffs, plans, security records
  app-laravel/              # API repo
    app/Enums/              # every fixed value (status, type, currency, method)
    app/Http/Controllers/Api/V1/   # thin: validate, authorize, call a service, return a resource
    app/Http/Resources/     # plain JsonResource; relations only on ?include=
    app/Models/             # casts, relations, snapshotSchema(), deleting events
    app/Services/           # all domain logic
    app/Support/            # small shared helpers (Money, SerialCounter, BusinessDate, DocumentLayout)
    app/Traits/             # Approvable, LoggableTrait, HasReferenceNumber
    database/migrations/    # one round, spaced timestamp per migration
    docs/                   # one file per topic (see 15)
    routes/api.php, routes/console.php (schedule)
  app-nextjs/
    app/(main)/  app/(auth)/  app/dashboard/@admin/  app/dashboard/@investor/   # route groups + parallel portals
    components/common/      # Skeletons, InfoHint, HelpPanel, dialogs
    components/dashboard/   # tables, filters, export, print, layout
    services/<domain>/      # axios calls per domain
    hooks/  lib/  lib/enums/ contexts/
    e2e/staff|investor|guest/   # Playwright, one rule per test
    docs/
```

Each repo has its own `CLAUDE.md` / `AGENTS.md` and `.claude/skills`; the workspace `CLAUDE.md` is the map.

## 3. Backend conventions (Laravel)

- **Allowed values go through `Rule::in([...])` or `Rule::enum(X::class)`, never the string `'in:'.implode(',', $list)`.**
  The string form splits on every comma, so a value that holds one ("Unplanned, with a plan") is refused as invalid
  while every other value works: NICE's dashboard links to two statuses failed with "The selected status is invalid"
  for days (10/11/2026). The same for any code that joins values with commas and splits them back (a survey's
  multi-select answer "Yes, sometimes"): keep them as an array, or as one row per value.
- **Keys:** ULID primary keys everywhere. A ULID is ordered by creation time, **not** a business number.
- **Human numbers** (account number, reference, serial) come from a **counter table** (`serial_counters`: stem,
  table_name, column_name, last_value) raised under a row lock (`SELECT ... FOR UPDATE`), plus a unique index on
  the column. The counter never goes back, so a deleted record's number is never reissued. Never `orderBy('id')`,
  never `count() + 1`, and not "highest used + 1" either: once the NEWEST record is deleted, highest + 1 gives its
  number to the next one (proven on NICE 05/11/2026; Alpha still did it). Month and year in a number are the
  business-timezone month, not `now()` in UTC. A backfill works the numbers out and sets the counters before it
  adds the column, so a record created during the deploy cannot take a backfilled number. Reference: NICE API
  `docs/reference-numbers.md`.
  *Why reuse matters when the column is unique anyway:* the database stays clean, but the number has already
  left it: notifications, prints, exports, emailed documents, WhatsApp, the audit trail. Reused, it points at a
  different record, and a delete-then-recreate looks like an edit. **A number, once anyone has seen it, means
  one record forever**; a unique index only guarantees one record at a time. A never-reused number leaves a
  visible gap that the audit trail explains, which is what auditors expect.
- **Fixed values are backed enums** cast on the model, stored as strings, mirrored in the frontend's `lib/enums`.
  Compare through the enum, never a hand-typed string; the migration comment lists the values and names the enum.
  Details and the traps: [enums-and-migrations.md](enums-and-migrations.md).
- **Money:** one base currency per ledger, nothing converts silently. Display at the currency's decimals (UGX 0,
  USD 2) through one helper (`Money::format` / `lib/money`); prices 4; units 2; exports keep the full number.
- **Time:** store and send UTC (`...Z`, the frontend sends `toISOString()`). The business timezone only for day
  boundaries, schedules and reports (`BusinessDate`); the viewer's clock only for display. A local midnight stored as
  21:00 the day before is correct: prove a timezone "bug" before fixing it. The full rules, the traps that reached
  production (Laravel binding a Carbon without converting its timezone, end dates dropping the last day,
  `whereDate`/`DATE()` on UTC columns) and how to check a whole app: [dates-and-timezones-with-an-ai-agent.md](dates-and-timezones-with-an-ai-agent.md).
- **Migrations:** each gets its own spaced timestamp (`090000`, `091000`), never consecutive seconds. Every column
  written out (no loops), related columns grouped under a one-line comment with blank lines between groups,
  explanations above the column ([enums-and-migrations.md](enums-and-migrations.md)). Foreign keys at
  the bottom with `foreign()`. Before go-live, edit the create migration (and roll it back on servers); after,
  forward migrations only. Delete policy: an owned child cascades, reference and audit rows restrict, set null only
  for disposable links; never orphan financial or KYC data.
- **Columns** carry no entity prefix (`issuers.name`, not `issuer_name`). Repeatable lists are child tables. A JSON
  column is suspect: ask whether it is a count you could GROUP BY or rows that deserve a table.
- **Editable data that other records depend on** (schedules, price lists, registers): one master copy edited directly,
  the facts that define what happened frozen on the records that use them, no versioned rows when a consumer already
  freezes: [master-copy-and-snapshots.md](master-copy-and-snapshots.md).
- **Files:** store disk-relative paths; delete through the model so `deleting` events remove the stored file; never
  bulk or cascade-delete file-bearing rows.
- **Derived facts are read, never stored** (Letshego: a client's stage from what they have, in one SQL `case` used by
  lists, filters and sorts). A state people enter and leave (dropped, reopened) is a **flag with its own history
  table** (who, when, why, the stage at that moment), not one more stage. A record's **History is read from the
  stored facts** (one `UNION ALL` of the event tables, paged in SQL, words built on the API), never a log kept twice.
- **Responses:** plain `JsonResource`; nest a relation only on `?include=` and make every list `include` what its
  dialogs read (no N+1, no lazy loading: strict mode is on in development).
- **Config:** `env()` only inside `config/` (servers cache config).
- **Static analysis:** Larastan level 5 against a baseline; new code adds no findings; when you remove code, remove
  its baseline entries.
- **Queues:** emails, notifications, approval fan-outs and PDFs run on the queue; OTP and onboarding mail send
  synchronously. A worker keeps the code it started with: restart it after changing a job, a mail or a template.
- **Spreadsheet uploads: measure, then send in pieces and process on the queue.** File size misleads: on NICE
  (05/11/2026) an 8,000-row sheet was 0.3 MB but took 12 s and 190 MB to read and 23 s and 240 MB to apply inside
  one request. So: the browser sends 1 MB pieces with a progress bar (no server body limit sees more); the API
  assembles them on a private disk, saves the upload as Reading and queues the read (progress counted on the
  upload row, every 500 rows); the apply is queued too (one transaction, so a running bar, not a fake count); a
  job's `failed()` puts the upload in Failed or back to Preview with the reason, never stuck. The page polls while
  Reading/Applying. **Remember the user's action, don't infer it from an intermediate status**: a fast queue went
  Preview -> Applied between two polls and the page never saw "Applying". Read with `setReadDataOnly(true)`.
  A sheet bounded small by its nature (one row per subcategory) can stay in the request; say why.

## 4. Approvals (maker / checker)

- One generic engine for every approvable module: **Draft → Pending Approval → Active**, workflows with ordered steps
  by role, value thresholds for senior steps, escalation on a business-hours clock.
- An edit to a live record is **deferred**: the record keeps its values, the proposal waits in the approval request;
  approval applies it, rejection restores the status it had. Child-only edits defer their payload too.
- **Every edit is blocked while a request is Pending** (API and UI).
- The maker never approves their own request. Deciding needs `approve`; seeing requests follows module permissions.
- The approval diff and the audit trail read the model's `snapshotSchema()` (labels, formats, which relations) and
  log through the trait's builders, never hand-built arrays. A frozen presentation is stored with the request.
- A new approvable module is added to the frontend module registry (labels, the record's deep link) in the same change.
- Seeders that create live data use `Model::withoutApprovals(fn () => ...)`, never a bare `$bypassApprovals = true`
  (a static stays on for the whole process: in tests every later edit skipped approval).

## 5. Roles, permissions, access

- Spatie roles and permissions. **Staff means "not the client role"** (`! hasRole('Investor')`), never a list of
  staff roles: a role added later must just work.
- Frontend: `proxy.ts` only classifies (public, client portal, staff). What a staff page needs comes from the sidebar
  menu entry's `permission`: the same entry hides the link and guards the page (`PageAccessGuard`). **The API is the
  security boundary**; the UI only mirrors it.
- Never run a "reset all roles" seeder on a live server: grant the one new permission in tinker.
- Data scoping: a client sees only their own accounts; book-wide reports check staff on the API as well.

## 6. Sessions, sign-in and the portals (a bug we paid for)

- When the server picks the portal (or any page) from a cookie, **sign-in and sign-out must load the page afresh**
  (`window.location.assign`), never `router.push`: Next's client cache served a Super Admin the investor portal the
  previous user had just left. Wipe anything kept per tab for the user (scroll, table pages) at both moments.
- A parallel slot's layout never exports `metadata`: Next resolves every slot's, so the other portal's title leaks.
  Choose the title in the parent layout from the same cookie.
- Signing out cancels in-flight queries and clears the session; sign-in clears the query cache before the new session.
- Login: email OTP second factor, reCAPTCHA v3, rate limits. Social sign-in through a bridge page.
- No platform user is created for a client until their account is approved (track the applicant by email).
- Full write-up: Alpha's `alpha-fund-nextjs/docs/switching-users-on-one-tab.md`.

## 7. Lists (every one)

- **Paginate everything**: default 5 rows, max 100; dropdowns search and page; exports walk all pages. No `all=true`.
- **State lives in the URL**: page, page size, search, filters, sort (`?sort=-created_at`), so a refresh or a shared
  link reopens the same view. Push with `{ scroll: false }` so paging keeps the reader's place.
- **3-state column sort**: ascending → descending → off (back to the default), on the API field name.
- **Status chips** above the list with counts from the API (one `StatusCounts` query), and **one Filters button**
  opening one row form for every other filter (dates as `start_date` / `end_date` end to end); applied filters show
  as removable chips.
- **Loading = a shaped skeleton** (`isPending`), on record pages too (composed from the page's own blocks: header,
  figure cards, tabs, details, table, map, in its grid; Letshego `RecordSkeleton.tsx`), never one grey block, never bars bunched to the left of a table (one cell per column across the width, the toolbar,
  the View button, the pager), and never a spinner in an empty area, and **no `placeholderData`**:
  never show the previous page's rows under new filters. Spinners are for saving and sending only.
- **One Export** per list or report: Excel, CSV and Print from the same rows, with **typed cells** (real dates and
  numbers, never formatted text).
- **A new list is born with the house kit, not added on review**: the project's filter drawer (on NICE the shared
  Sell-out Filters: channel, region, territory, route, in step with the top bar's Global Filters), an **Export all**
  that walks every page with the filters and search applied and shows them before the download, and the API
  honouring the same filters in the list, its counts, the rows a count opens, and the export. On NICE (05/11/2026)
  the user had to ask for all of these on the new schedule page; also: actions belong to the tab they act on
  (upload/template under Uploads, not above every tab), and a help (?) sits beside the heading it explains.
- **A tab the URL can open, the URL must follow.** If `?tab=fulfilment` opens a tab, clicking another tab writes it
  back (and drops one-shot params such as `&status=` that only the opening link meant). Otherwise a refresh or a
  copied link lands on the wrong tab: on NICE (05/11/2026) the user arrived from a home card and the address said
  `?tab=fulfilment` on every tab. When the page is keyed by its query (to reset on a new link), write with
  `window.history.replaceState`, not `router.replace`, or every click starts the page again. Spec: click, assert
  `toHaveURL`, reload, assert the tab.
- Tables inside a page keep their page and size for the tab (`useTablePaging`, sessionStorage).
- **No hidden default filters.** A list with no filter in the URL shows everyone; any narrowing is a visible,
  removable chip. Letshego's pipeline hid dropped clients by default with nothing showing and the user thought a
  client was lost (10/11/2026). Rows in a special state stay in the list with a tag (a red "Dropped").
- **Back returns to the list where it was left** (page, filters, search, sort): each list saves its address as it
  changes (sessionStorage, per tab) and record pages' back buttons and the breadcrumb read it (Letshego
  `lib/listReturn.ts`, `useListReturn(path)`); the first paint uses the bare path so the markup matches the server.
  The sidebar opens a list fresh. Spec: open a record from page 3, press back, assert `page=3` in the URL.
- **No blank page between pages** (Next App Router): a `loading.tsx` per route segment (the dashboard's picks the shape
  by path; record routes their own) and a fallback on every `<Suspense>` (a bare one shows nothing while it waits), in
  the shape of the coming page. Letshego: `RecordSkeleton.tsx`, `app/dashboard/loading.tsx`. Prove it with `next build`.
- **A column most rows leave empty starts hidden** (`visible: false`; a plan's Description): it stays in the column
  menu and the export, and the row keeps its width for what is filled.
- **Filter words say what they do** ("Not dropped", not "Working"), and never reuse a word the same dialog already
  uses for something else ("Active" was the record status).
- Deep links: `?open_<thing>_id=` opens a record's dialog (approvals link to records this way).

## 8. Dialogs and forms

- **Record dialogs** use one hook (`useRecordDialog`): closing hides the dialog and keeps the record, so the content
  does not vanish mid-fade. Never null the record in `onHide`.
- **Details dialogs are calm**: a label/value grid, no hero stat boxes. Maximizable; sized for their content.
- **A view is its own read-only screen, never the create form with its fields disabled** (web and mobile). A
  disabled form keeps the input boxes and the empty space of every field (an empty description box), and reads as
  "something I can type in". NICE's Flutter app did this for expenses and journey plans; the brief given to the app's
  agent (10/11/2026): one `DetailRow(label, value)` and one `DetailSection(title, children)`, reused by every view;
  forms only for create and edit.
- **An infinite list says where it is:** skeletons on the first load, "Showing 20 of 134" with a small spinner while
  more pages exist, "That's all: 134" at the end (from the API's `total`), pull to refresh back to page 1.
- **A details dialog reads the record first and the approval last**: what was asked for (the order, its items, the
  amounts, the metadata), then the approval timeline and the decision at the bottom, where the approver acts after
  reading. NICE put a pre-sale order's approval above its items and the user rejected it (09/11/2026, "someone needs
  to first see the order details and items"). Check every dialog that carries an approval, not only the one reported.
- **The sidebar is grouped**: a small heading over each group, related pages folded under one parent, children on a
  guide line with a dot, the menu with everything folded fitting a laptop screen. See
  [nextjs/dashboard-sidebar-groups.md](../nextjs/dashboard-sidebar-groups.md).
- **Form defaults are built when the form mounts**, never a module-level constant holding `new Date()` (it freezes at
  the first import; [dates note](dates-and-timezones-with-an-ai-agent.md), trap 15).
- **Forms**: react-hook-form + zod, in numbered sections; the API answers per field with the true reason and the form
  shows it on the field. A computed figure is read-only and computed on the API too (never trust a typed total).
- **Confirm** before anything final (post, decline, delete), inline in the same box or a confirm dialog.
- Destructive or approvable edits are blocked in the UI while pending, with the reason.
- Errors are handled globally (toasts from one handler); opt out per query with `meta.skipGlobalError`.
- Forward the AbortSignal to axios through the whole `QueryFunctionContext`; invalidate full query keys, the current
  detail key first.
- **A fixed-order form saved step by step** (NICE's merchandising modules, 05/11/2026): one row per run with a
  `<step>_saved_at` per step, so "what is next" is read from the data, never stored. The API enforces the order under
  `lockForUpdate` (two taps must not both pass "is it next"), refuses a step out of order or twice with a sentence
  that names the next one, and starting twice returns the same row (a phone retrying after a dropped connection). A
  reference value the person is judged against (a published price) is snapshotted when they first SEE it, not when
  they finish, or a change in between moves the goalposts. Name the answer columns as the questions they answer;
  null = not reached, never "No".

## 9. Help, text and look

- **Help** (`InfoHint`): `?` for how to do something, `i` for what a figure means, with its formula. Text as
  paragraphs, `- Term:` points and `> formula` lines; long help scrolls in its panel. In JSX, a multi-line help string
  is a JS string (`text={"a\n\nb"}`): a quoted attribute shows `\n` literally (lint rule blocks it).
- **Documents** (statements, reports, notes) read the same on screen, in print and as the PDF: one layout, fixed
  label slots (a missing value reads N/A in its slot, so nothing shifts), the PDF rendered from the same frozen data
  as the dialog. PDFs password-protected with something the holder knows, with a Copy button for it.
- **Chips** stay on one line (`whitespace-nowrap`); dates and figures do not wrap in table cells.
- Dark mode on every screen; brand colours from tokens; no plain blue links in a branded app.
- Scroll memory: a reload or Back returns to the same place on the page (`useScrollMemory` on the scrolling `main`).
- Copy for people: plain words, no internal names, no em dashes.
- **One timestamp format** from one helper (`formatDateTime()` → `23/08/2026 08:39:43 PM`) in every table,
  details view and print; never `toLocaleString()` (its format follows the browser's language: NICE had six
  formats for "Created At" across 78 files). Empty is `N/A`.
- **New UI copies the nearest existing component's look**, light and dark: a hand-made hover next to
  Highcharts ones must look like them (white, thin grey border, dark text in light mode). Check light mode
  as well as dark before calling it done.

## 10. Audit trail

- Every create, update, delete, approval decision, sign-in and send is logged (Spatie activitylog) with the causer,
  the subject, the outcome and a diff built from `snapshotSchema()`: what was **written**, not what was asked for.
  Every logged model declares a schema. The engine and how to upgrade an app that logs sentences:
  [audit-trail-with-snapshot-schemas.md](audit-trail-with-snapshot-schemas.md).
- Secrets never enter the log (passwords become `password_changed: true`).
- The audit screen filters by module, event, user and date, paginates, and exports.

## 11. Realtime (Reverb + Echo)

- Laravel Reverb with Sanctum-authorised private and presence channels; `laravel-echo` in `lib/echo.ts`.
- Broadcast **ids only**, after the database commit; the browser refetches through its query keys. One "ledger
  changed" style event per aggregate, and every screen showing derived data listens.
- Status changes that take time (an email queued → sent → failed) broadcast each step, the first one at the request.
- Disconnect Echo on sign-out (or the user lingers in presence lists).
- Servers: Reverb under Supervisor with `stopwaitsecs=10` (workers 3600), proxied by nginx; restart it with the
  workers on deploy.

## 12. Reports and dashboards

- Report filters, page and sort in the URL (`useReportParams`, page wrapped in `<Suspense>`); one Export.
- Dashboard: one chart wrapper (Highcharts), each chart a component fed by its card's query; every figure has its
  formula in help; currencies never added together.
- Book-wide reports are staff-only on the API.
- **Charts read at a glance and in full:** a short amount on each bar or column ("3K", "412.5M"), the full
  amount in the hover. A hover shows the period (and what a part-way period covers: "1st to today"), every
  series together (Actual, SPLY, Target) with full figures, and the change, red down / green up. Never the
  browser's `title` tooltip (late, unstyled, one line) and never a bare number without its currency.
- **Comparisons are like for like:** a period so far against last year to the same date, everywhere; when a
  rule like that is decided, check EVERY card actually follows it.
- **Tally before saying "it matches":** each dashboard figure against the list its link opens (same filters),
  and cards against each other, with the clock pinned to days that have data.
- Every list a dashboard links to must read the link's URL (filters, sort, dates) and accept every value the
  screen can send; give optional API parameters the screen's default (a missing `data_limit` was a 500).
- **Every figure opens its rows, through a small arrow at the card's corner, never the whole card** (NICE's
  `OpenLink`; Letshego `KpiCard`/`ChartCard` `href`). The link carries the page's period and place as the target
  list's own filters so the list counts exactly the same rows; add the API filter when it is missing (paid out,
  approved, rejected, withdrawn, dropped in these days). The arrows are **quiet**: shown on hover or keyboard focus,
  always on touch screens ("too many arrows, people lose concentration", 10/11/2026).
- **A report shows the records behind its figures** (the loans with their clients, switchable by what was counted),
  as the standard table with its own Export, beside the charts and the page's Export and print.
- A chosen period shows as chosen when the dialog reopens (Pick days highlights the matching quick pick); legends
  sit under charts in narrow cards; never show a figure the system does not compute.

## 13. Packages

- Prefer what the project already uses; add a package only for a real need, after checking it is maintained, its
  licence, its size, and that it works with this Next/React/Laravel version. Pin through the lock file.
- Usual stack: Laravel (Sanctum, Reverb, Spatie permission + activitylog, Pest, Larastan, Pint, mPDF, Scramble),
  Next.js App Router (TanStack Query, react-hook-form + zod, PrimeReact + Tailwind 4, Highcharts, laravel-echo,
  next-auth for Google, Playwright).
- npm 11.19+ holds back packages' install scripts until approved: approve or deny them on the dev PC and commit,
  never on a server.

## 14. Testing

- **API:** Pest feature tests per rule (`php artisan test --compact --filter=...`), Pint, Larastan. The full suite
  before committing a cross-cutting change. A test that only fails in the full run means shared state (a static, a
  cache): find it.
- **A table check that runs while the table is loading checks nothing.** Skeleton rows are rows: a spec that waited
  for "Showing 1 to" (the paginator still showed the old figures) read the skeleton cells, filtered out the blanks and
  passed with no row checked (NICE, 10/11/2026). Wait for the RESPONSE of the filtered request
  (`page.waitForResponse`, not `waitForRequest`), then for the skeletons to go (`tbody .p-skeleton` count 0), and
  assert there were rows (`length > 0`) before asserting what is in them.
- **Screens:** Playwright against the running local API and app with seeded data; specs in `e2e/staff|investor|guest`,
  one rule per test, sessions set by cookies from a helper; a screen change updates or adds its spec. Prove a spec by
  making it fail on the old code.
- **The agent drives the browser itself, every time.** A new screen is not reported done with "not seen in a
  browser": write its spec, run it against the local servers, save screenshots (light and dark) and LOOK at them.
  On NICE (05/11/2026) the screenshots caught a bug every other check had passed: an applied upload's history read
  "2 unchanged" because it was compared with the schedule it had just changed. Dark mode = the app's theme cookie
  and a reload (toggling the `dark` class from the spec skips PrimeReact's theme and shows false colours); wait for
  the table's loading text to go before the picture; a spec that writes names what it makes and deletes it in
  `afterAll`. Then run the WHOLE suite: a change to a shared list can break another screen's spec.
- `tsc --noEmit` clean, ESLint clean on changed files. Capture the real exit code (`cmd > out.txt; echo $?`,
  never `cmd | tail; echo $?`), and if the repo has no working linter, say so rather than "lint clean".
- **Whole-API guards with `DB::pretend()`:** call every GET route inside `DB::pretend()` with a
  `beforeExecuting` hook; queries are built and their bindings seen, nothing runs, no schema needed, seconds
  for 150 routes. Good for rules about what reaches SQL (timezones, raw request values) and for pinning a
  card's periods (`Carbon::setTestNow` + the bindings it sends). Assert most routes built a query, or the
  guard can pass by reaching nothing.
- **Prove every new test by putting the old code back** (copy the file aside, revert the hunk, run, restore).
- Test the calendar edges: the 31st, 1 January, the first day of the financial year, 29 February.

## 15. Documentation the repo keeps

`docs/` one topic per file, updated in the same change: access control, approvals, audit/snapshot schema, enums,
migrations ordering, reference numbers, pagination and exports, timezone handling, file storage, deployment,
environment setup, running locally, scroll memory, e2e testing, every serious bug's write-up (what, why, fix, rules,
test). The in-app user manual is updated with every user-visible change. Session handoffs in
`workspace/.claude/handoffs/`; plans in `.claude/plans/`; a `SECURITY-<date>.md` per server patch.

## 16. Running it

Local (never on a server): `php artisan serve`, `php artisan reverb:start --debug`, `php artisan queue:work`,
`php artisan schedule:work`, `npm run dev`. Servers: Supervisor (workers, Reverb), cron's `schedule:run` every minute,
pm2 (Next.js), nginx; deploy and patch through the sibling playbooks.
