# Building a Laravel + Next.js business app with an AI agent: the playbook

**For the AI agent reading this.** We are building (or extending) a back-office web app: a Laravel REST API and a
Next.js dashboard, staff plus (often) a client portal, with approvals, an audit trail, reports and realtime. This file
is **how we build**: the conventions every screen and endpoint follows, the traps already paid for, and how we work
together. Read it fully before writing code, then read the project's own `CLAUDE.md`, `AGENTS.md` and `docs/`: the
project's word beats this file where they differ.

Worked example of everything below: the Alpha Fund system (`alpha-fund-laravel/docs/*.md`, `alpha-fund-nextjs/docs/`).
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

- **Keys:** ULID primary keys everywhere. A ULID is ordered by creation time, **not** a business number.
- **Human numbers** (account number, reference, serial) come from a counter: the highest used + 1 within its prefix
  (`SerialCounter`), with a unique index. Never `orderBy('id')`, never `count() + 1` (deletes reuse numbers).
- **Fixed values are backed enums** cast on the model, stored as strings, mirrored in the frontend's `lib/enums`.
  Compare through the enum, never a hand-typed string.
- **Money:** one base currency per ledger, nothing converts silently. Display at the currency's decimals (UGX 0,
  USD 2) through one helper (`Money::format` / `lib/money`); prices 4; units 2; exports keep the full number.
- **Time:** store and send UTC (`...Z`, the frontend sends `toISOString()`). The business timezone only for day
  boundaries, schedules and reports (`BusinessDate`); the viewer's clock only for display. A local midnight stored as
  21:00 the day before is correct: prove a timezone "bug" before fixing it.
- **Migrations:** each gets its own spaced timestamp (`090000`, `091000`), never consecutive seconds. Foreign keys at
  the bottom with `foreign()`. Before go-live, edit the create migration (and roll it back on servers); after,
  forward migrations only. Delete policy: an owned child cascades, reference and audit rows restrict, set null only
  for disposable links; never orphan financial or KYC data.
- **Columns** carry no entity prefix (`issuers.name`, not `issuer_name`). Repeatable lists are child tables.
- **Files:** store disk-relative paths; delete through the model so `deleting` events remove the stored file; never
  bulk or cascade-delete file-bearing rows.
- **Responses:** plain `JsonResource`; nest a relation only on `?include=` and make every list `include` what its
  dialogs read (no N+1, no lazy loading: strict mode is on in development).
- **Config:** `env()` only inside `config/` (servers cache config).
- **Static analysis:** Larastan level 5 against a baseline; new code adds no findings; when you remove code, remove
  its baseline entries.
- **Queues:** emails, notifications, approval fan-outs and PDFs run on the queue; OTP and onboarding mail send
  synchronously. A worker keeps the code it started with: restart it after changing a job, a mail or a template.

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
- **Loading = a shaped skeleton** (`isPending`), never a spinner in an empty area, and **no `placeholderData`**:
  never show the previous page's rows under new filters. Spinners are for saving and sending only.
- **One Export** per list or report: Excel, CSV and Print from the same rows, with **typed cells** (real dates and
  numbers, never formatted text).
- Tables inside a page keep their page and size for the tab (`useTablePaging`, sessionStorage).
- Deep links: `?open_<thing>_id=` opens a record's dialog (approvals link to records this way).

## 8. Dialogs and forms

- **Record dialogs** use one hook (`useRecordDialog`): closing hides the dialog and keeps the record, so the content
  does not vanish mid-fade. Never null the record in `onHide`.
- **Details dialogs are calm**: a label/value grid, no hero stat boxes. Maximizable; sized for their content.
- **Forms**: react-hook-form + zod, in numbered sections; the API answers per field with the true reason and the form
  shows it on the field. A computed figure is read-only and computed on the API too (never trust a typed total).
- **Confirm** before anything final (post, decline, delete), inline in the same box or a confirm dialog.
- Destructive or approvable edits are blocked in the UI while pending, with the reason.
- Errors are handled globally (toasts from one handler); opt out per query with `meta.skipGlobalError`.
- Forward the AbortSignal to axios through the whole `QueryFunctionContext`; invalidate full query keys, the current
  detail key first.

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

## 10. Audit trail

- Every create, update, delete, approval decision, sign-in and send is logged (Spatie activitylog) with the causer,
  the subject, the outcome and a diff built from `snapshotSchema()`: what was **written**, not what was asked for.
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
- **Screens:** Playwright against the running local API and app with seeded data; specs in `e2e/staff|investor|guest`,
  one rule per test, sessions set by cookies from a helper; a screen change updates or adds its spec. Prove a spec by
  making it fail on the old code.
- `tsc --noEmit` clean, ESLint clean on changed files.

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
