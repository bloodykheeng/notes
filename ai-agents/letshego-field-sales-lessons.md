# Letshego Field Sales: how we built it, and what an agent should learn from it

**For the AI agent reading this.** Letshego Field Sales (Oct–Nov 2026) is the system where my way of working was
written down most completely. This note tells the story briefly, shows how decisions were made, lists the patterns
worth reusing, and, most useful of all, lists **the mistakes an agent made and how I corrected them**, so you do not
make them. Read [who-i-am-and-how-i-work.md](who-i-am-and-how-i-work.md) first. The code and its docs are in
`D:\coding\new wave\letshego\letshego-workspace` (start with its `CLAUDE.md`, `docs/decisions.md`,
`docs/plan-after-6-nov.md`).

---

## 1. What it is

Letshego Uganda's field sales system: sales agents on routes capture **prospects** (name, phone, GPS), turn the ones
who want a loan into **leads** (NIN, amount, where); a branch manager plans the leads for a **relationship officer**,
who checks in within 50 m of the lead's saved point (geofence), completes the **client KYC**, and takes **loan
applications**, approved maker-checker; a disbursed loan makes the person a **customer**. Plus prospect targets,
dashboards, reports, notifications, an audit trail. Laravel 13 API, Next.js 16 dashboard, a Flutter app built by a
teammate from a guide in the workspace.

## 2. How it was set up (copy this)

1. **A clickable prototype first** (Python-generated SVG screens, deployed on Vercel, a briefing for the demo). The
   client clicked through it and agreed it before any code. It stays the design reference.
2. **A workspace repo** with three app repos inside (git-ignored), a `CLAUDE.md`, a `WORK-PLAN.md` by phase, a
   `docs/decisions.md`, `docs/requirements-sources.md`, handoff notes, and a Flutter build guide for the teammate.
3. **The foundation copied from the newest system** (GiftNest: auth, users, roles, audit, queues, lists, exports,
   dashboards, deploy), stripped of its domain and rebranded, then reinstalled at the latest versions with the
   official installers. Each reference project is named in `CLAUDE.md` with what to take from it (Alpha: access
   control and the approval engine; SDS: scoping by columns; NICE: the field domain, geofence, maps).
4. **Base, then the MVP core in dependency order:** clients (prospects) → prospect targets → leads → lead journey
   plans → visits and KYC → loans → dashboards. Each module: propose the model in words, agree, build API + web +
   tests + docs + manual chapter, I review in the browser, fix, commit, next.

## 3. How decisions were made (examples)

Each of these was proposed, argued briefly, settled by me, and recorded in `decisions.md` the same turn:

- **One client per person at every stage**, the stage **read, never stored** (Customer if a loan was disbursed, else
  KYC completed, else Lead, else Prospect). No copied stage column to drift.
- **The lead is a loan request made once** (no number of its own); **the client KYC is once per client, edited in
  place** (the audit trail keeps what it said before); **loan applications are many**, each with its own
  assessment, each taken during a geofenced visit.
- **Names fixed when they caused confusion:** `customers` → `clients` (26 Oct); then the last stage "Client" →
  **"Customer"** (10 Nov), because "client" meant both the person and the stage ("client KYC" read as if KYC made
  them a client). Client = the person at every stage; customer = the one with a loan.
- **Dropped is a flag with a reason, not a stage** (`client_drops`, stage frozen at the drop, reopened by/at): the
  stage stays read from the facts, so reopening brings them back where they were; a customer is never dropped;
  dropping withdraws an application in progress; nothing moves while dropped.
- **Who sees what:** a sales agent only the people they captured; a relationship officer **only what is assigned to
  them** (leads in their journey plans, current or past, plus their own captures, visits and applications), kept at
  every stage after; a branch manager the branch; head office all. ("That's the purpose of assigning.")
- **No settings module:** the geofence radius is a column on branches with a default in code.
- **The System Admin holds every permission** (any flow can be run from the dashboard) **except deciding an
  approval step**, which only the step's role does.
- **A lost phone needs no button:** a new password signs out every other device, as Google and Microsoft do.
- **A colleague's requirements document was not a source** ("they will mislead us"). When a second version arrived,
  it was read as a list of ideas: three were taken (drop with a reason, an activity timeline, the calculator in the
  field), some deferred, the rest left out with the reason written down.

## 4. The mistakes an agent made, and the correction

| The agent did | I said | The rule now |
| --- | --- | --- |
| Drew prospect targets as weekly plans with per-day rows and many dates | "So many dates, let's brainstorm" | Propose the model in plain words first; the simplest that says what the business means |
| Proposed a settings table for one value | "Are you serious" | No frameworks for one value: a column where it belongs |
| Copied `branch_id` beside `route_id` "to avoid a join"; wrote add-column migrations | "Not professional"; "we're in dev, edit the migration" | Derive through relations; edit the create migration in development |
| Stored reasons, makers and check results as JSON | "Is it really professional or escape hatches?" | Tables with FKs for anything filtered, counted or referenced; checks read from the facts |
| Shipped writes without a confirm dialog; added per-row delete | "Be serious, I've told you this many times" | Confirm every write; bulk delete only |
| Built lists with a spinner, a raw table, no per-page, no export | "Use our standard pagination" | Every list (dialogs too) is the standard table |
| Gave a list only Edit | "Not always we view from edit" | View (the eye) first on every row |
| Put choices and chosen as two tables in one form | "Like pouring water from a jerrican into a tank" | Picker dialog (tick, Add) into the form's table |
| Showed three "Fill in KYC" buttons and repeated notices | "Ugly clutter, like a vibe-coded system" | One place per action, no repeated notices |
| Opened a loading dialog then swapped it for another | "That snap is ugly, and it's in many places" | One dialog loading in place; fix every instance |
| Ran the full suites after changing one letter | "Is that really professional?" | Test what changed; full suites before shared commits |
| Made whole dashboard cards clickable | "We add a link the user clicks, like NICE" | A small arrow at the card's corner, carrying the period and place |
| Then showed ten arrows at once | "Too many arrows, people lose concentration" | Secondary affordances on hover/focus |
| Hid dropped clients by default with no chip ("Working") | "Why can't I see them? No filter is applied" | No hidden default filters; a list with no filter shows everyone |
| Named that filter "Working" | "Is working the correct wording?" | Plain words for what it does ("Not dropped"); avoid words already used (Active = record status) |
| Linked "← Client pipeline" to the bare list | "Page 3, open someone, back, I'm on page 1" | Back returns to the list's page and filters (each list remembers its address) |
| Showed "2.6 h prospecting" in the prototype though nothing measures it | Noticed in review | Never show a figure the system does not compute |
| Left the prototype on a PIN sign-in after building password + emailed code | "You forgot the login screen" | The prototype follows the built system |
| Officers saw every lead in their territory | "Show them their leads assigned to them" | Reach follows assignment, read from the records |
| Quick pick not highlighted when the dialog reopened | "Is that correct behaviour?" | State the user chose must show as chosen when they come back |
| A legend beside a ring chart in a third-width card | "Things are squeezed" | Legends under the chart in narrow cards; look at every card at 1536×745 |
| Record pages loaded as one flat grey block | "The skeleton doesn't match the data, and it's not only here, be wise" | Every page loads in its own shape (header, cards, table, map); fix every page at once |
| Then a table skeleton with short bars bunched to the left | "The skeleton looks so poor, a lot of empty space on the right" | Skeletons fill the width as the real table does (toolbar, a cell per column, View button, pager) |
| A Description column empty on almost every row | "Make description visible false by default" | A mostly empty column starts hidden; the column menu shows it |
| A white page while a list opened (no `loading.tsx`, bare `<Suspense>`) | "Why is this showing a white screen, is it a crash?" | A `loading.tsx` per route and a fallback on every `<Suspense>`, in the page's shape |
| A report with totals only | "We need to see the clients too" | Reports list the records behind the figures, with their clients |
| A report card with no way to the rows | "We need some links, i.e. to visits" | Every figure, chart or quote opens its rows |
| Browser tests failing on a database seeded two weeks earlier | (data aged) | Seeded "yesterday" data ages: reseed before blaming code; specs search by full name, not a common first name |
| Ran `migrate:fresh` while Reverb was running | Presence stopped working | Reverb polls the cache table: restart it (and the queue worker) after a rebuild |

## 5. Patterns worth reusing (where they live in Letshego)

Web (`letshego-nextjs`):
- **List memory for Back**: `lib/listReturn.ts` (`useListParams` saves each list's URL to sessionStorage;
  `useListReturn(path)` for back buttons; the breadcrumb reads it after mount). The sidebar opens lists fresh.
- **Card link arrow**: `OpenLink` + `KpiCard` `href`/`openLabel`, `ChartCard` `href` (`components/dashboard/insights/parts.tsx`),
  quiet until hover (`group-hover`, always shown on touch screens).
- **Report scope**: one period bar (presets + Pick days with quick picks), one Filters dialog, the filters as a
  sentence (`ScopeLine`), old figures dimmed while new ones load, one Export (Excel sheet per card, PDF via print).
- **History as a timeline** read from stored facts (`ClientTimeline.tsx` over the API's `GET clients/{id}/timeline`:
  one SQL `UNION ALL` of stage events, plan lines, check-ins, refusals, visits, application steps, drops; paged in
  SQL, then only the page's records loaded; the words built on the API so web and app say the same).
- **Confirm**: `confirmSubmit()` for forms, `confirmAction()` for every other write; `guardedClose` for unsaved
  changes; the `Dialog` wrapper (never PrimeReact's directly) and `useLastValue` so dialogs fade with their content.
- **Help** (`InfoHint`): how it works, the formula and one worked example (the Loan calculator's help).
- **Maps**: one Leaflet kit (layers, popups, full screen, print), people's points folded to one flag per person.

API (`letshego-laravel`):
- `Client::scopeVisibleTo($user)`: one function for who sees what; everything hanging off a client follows it.
- `Client::refuseIfDropped()`: one guard called by every write that moves a client.
- `ReportScope`: period, buckets, place and person, shared by every dashboard and report endpoint; a card's list
  link uses list filters that count exactly the same rows (`disbursed_from/to`, `approved_from/to`,
  `rejected_from/to`, `withdrawn_from/to`, `dropped_from/to`).
- Tests: a `planFor($officer, ...$clients)` helper gives an officer reach the way the product does; fixtures named
  after people and places; an `e2e:cleanup` command removes throwaway records named `E2E …`.

## 6. Process lessons

- **Brainstorm rounds pay for themselves:** after a client presentation, collect my notes, read any documents as
  ideas, propose what to take with reasons, agree, then write one plan file and build it in steps, each landing API,
  web, mobile guide, manual and docs together.
- **Write the mobile guide for the other developer's agent:** the calls, the rules, the screens, what changed and
  when; it is how a second team builds the app without me in the middle.
- **Keep a handoff note current when I ask** and point it at the plan file; the next session reads the plan first.
- **Save every correction as a rule** with my words and the reason; the list in §4 exists because of that habit.
