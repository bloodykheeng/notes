# Who I am and how I work: read this first

**For the AI agent reading this.** This is me, how I talk, how we decide things, and the bar I hold work to. Read it
before anything else, then the playbook for the job ([build](build-laravel-nextjs-app-with-an-ai-agent.md),
[deploy](deploy-with-an-ai-agent.md)), then the project's own `CLAUDE.md`, plan and decisions. The project's files
win where they differ. Everything here comes from real corrections I have had to make, mostly while building
Letshego Field Sales (Oct–Nov 2026: [letshego-field-sales-lessons.md](letshego-field-sales-lessons.md)). If you
follow this, I should not have to repeat myself.

**How to use it (for me).** First message in a new project:

> Read `coding-notes/ai-agents/who-i-am-and-how-i-work.md`, then `build-laravel-nextjs-app-with-an-ai-agent.md`, and
> follow them. Then <the task>.

---

## 1. Who I am

- **bloodykheeng** (my git name and handle), a developer in Kampala, Uganda, working at present under **New Wave
  Technologies**. I build business systems for clients: Letshego Uganda (micro-lending field sales), NICE House of Plastics (sales, distribution,
  merchandising), the Alpha Fund (CPA accounts), SDS / MOPS (an inspection tool), GiftNest. They share a foundation:
  each new one is built on the newest one before it.
- **Stack:** Laravel (latest, PHP 8.4) API, Next.js (latest, App Router) web dashboard, Flutter or React Native for
  the field app, MySQL, Reverb for realtime, Ubuntu + nginx servers (two environments on one box). Windows dev PC
  with Laragon, VS Code and Claude Code; the PC is not powerful, so heavy checks go one at a time.
- **People I work with:** clients (who decide), teammates who build parts with their own agents (the mobile app is
  often another developer's, built from a guide I keep in the workspace), and colleagues who send documents that are
  not the client's word (see §4).

## 2. How I talk, and how to read me

- **I type fast and misspell, in Ugandan English.** "men" or "mn" at the end is just emphasis. Read for intent, never
  ask me to fix a typo, never quote my typos back.
- **I send screenshots.** Look at them properly: the URL, the numbers, the state of every control. Most of my
  messages are a screenshot and one line ("is this correct?", "things are squeezed", "why can't I see them").
- **"Don't you think…?", "what's your view", "is this correct behaviour?"** means I want your opinion: say plainly
  whether it is right, recommend one option with the reason, short. Do not list five options and leave me to pick.
- **"I agree", "go ahead", "implement fully", "we have discussed enough"** means build all of it, end to end (API,
  web, mobile guide, docs, tests) without stopping to ask about details you can settle with good defaults.
- **"Hope it's not in other places"** is always in the air: when I report one instance of a problem, find and fix
  every instance of it before answering. A fix to one screen of a pattern that is wrong on six is not a fix.
- I can lose dev data ("we are still in dev and test"), but ask before wiping it, and tell me what to restart after.

## 3. How we decide things

1. **Brainstorm first, in plain words.** For a new module, propose the model as a few sentences ("a target is a
   person, a period, and lines of route + expected prospects") and agree it before any table exists. I reject
   over-built drafts ("so many dates, let's not do unnecessary work").
2. **Recommend.** Give your view with the reason, and the trade-off when there is a real one. Check the reference
   projects AND search online for current practice; cite what you found. References are older; they are not gospel.
3. **I settle it.** Then it is closed: do not reopen it unless I do.
4. **Record it** the same turn in the workspace's `docs/decisions.md`: date, the decision, why (with my words in
   quotes), and where the detail lives. Big rounds of change get their own plan file (`docs/plan-<topic>.md`: the
   decisions table, the steps with status, what the mobile app gets, what was left out and why), kept current.
5. **When a client says "offline", ask what they mean and classify every feature** (offline OK / offline confirmed
   by the server / online only) before designing anything: almost no business app is "fully offline"
   ([offline-and-online-explaining-to-clients.md](offline-and-online-explaining-to-clients.md); MRDRS was built on
   "offline" read as "no server").
6. **Build in dependency order, MVP core first.** No side modules, no frameworks for one value: a setting lives
   where it belongs (a column on the row it governs, a constant), never a "settings module" ("storage... are you
   serious").

## 4. What decides, and what does not

- **Decides:** me, the client's own word, the agreed clickable prototype (built first, deployed so the client can
  click through it), and what we settled (`decisions.md`).
- **Does not decide:** documents a colleague writes from other firms' material. Read them as a list of ideas, never
  as rules; never cite their codes. Keep a `docs/requirements-sources.md` saying what decides and what does not.
- **The prototype follows the system once the system exists.** When we build something differently from the
  mockup (a real sign-in instead of a PIN screen, a field moved to another step), update the prototype too: the
  mobile developer builds from it.

## 5. How a project is set up

- **A workspace repo** (instructions, plan, decisions, handoffs) with the app repos inside it, git-ignored, each its
  own repo: a change across API, web and docs is three commits in three repos. **If I have the root folder above the
  workspace open in VS Code**, add a `.vscode/settings.json` there with `git.scanRepositories` listing each repo, or
  VS Code shows no git colours ([../vscode/git-colours-for-nested-repos.md](../vscode/git-colours-for-nested-repos.md)).
- **`CLAUDE.md`** at the workspace root: what the system is, where patterns come from (the reference projects and
  what to take from each), the domain as settled, binding conventions, how to verify. Short, linking to docs.
- **`WORK-PLAN.md`**: phases, the current one, its checklist and stop point. **`docs/decisions.md`** (above).
  **`.claude/handoffs/`**: a note written when I ask, with goal, status, next steps, decisions, still-open questions
  and gotchas; the next session starts from the latest.
- **Lessons for another project go to coding-notes, never into this project:**
  `coding-notes/project-inbox/<that project>/YYYY-MM-DD-from-<this project>-<topic>.md`, with a header saying who it
  is for, where it came from and its status. When I say "check coding-notes for lessons" inside a project, read its
  inbox folder, apply what holds (read-only checks first), mark the note Resolved, move it into the project for history
  and delete it from the inbox. Rules: [../project-inbox/README.md](../project-inbox/README.md).
- **Each app keeps `docs/*.md`** per topic (the data model, each module, validation, scoping, dashboards, maps), a
  **user manual** in the app (a chapter per module, updated with the module), and a **mobile build guide** in the
  workspace when another developer builds the app.
- **My branch, never `main`:** in client projects I usually work on the branch **`farouk`** (the project's
  `CLAUDE.md` says which). Commit and push **only when I ask**, in the repo the change belongs to. After I
  approve a module I want commit, push, and straight on to the next; I ask for handoffs myself.
- **Latest versions, official installers** (`composer create-project`, `create-next-app@latest`, `npm install
  pkg@latest`); hold a major back only with a stated reason. A package's published files (migrations, config) come
  from the installed version (check Packagist / GitHub first), then re-apply only our deliberate changes with a
  comment each.

## 6. The bar: professional, not "vibe-coded"

- **One place per action.** An action lives in its table (toolbar to start, the row once it exists), not also in a
  header, a bar and a note. Once a one-to-one record exists the action is Edit, never another "Fill in".
- **No clutter, no repeated notices.** Don't print what the screen already shows. Help (i) only on technical fields
  (a rate, a repayment period, an upload limit) and on figures (how they are counted, the formula, an example).
- **Short, plain copy** that says what the reader sees. Labels, not essays. No em dashes. No internal names.
- **Calm attention.** A row of always-on icons or arrows distracts: secondary affordances appear on hover/focus.
- **Consistent everywhere.** The same thing looks and behaves the same on every screen; a new screen starts from an
  existing standard one, never from scratch.
- **Check it with your eyes:** screenshots at 1536×745 (a 1920 laptop at 125%) and 1920×1080, light and dark, with
  realistic volumes of data (a demo seeder, a heavy day). Empty reports prove nothing.

## 7. Rules I have had to repeat (do them unasked)

Working:
- **Write every request down the moment it arrives**, mid-task included, as a row in the project's tracked list, then
  carry on; never only answer in chat. Questions are rows too. How: [docs-and-requests-on-a-shared-project.md](docs-and-requests-on-a-shared-project.md).
- **Add, create and new buttons sit on the right** of a list's toolbar or a header, on phones too (most people are
  right-handed): use the shared button helper, never a header slot that lands on the left.
- **A bug in one place: search for the same pattern everywhere** before calling it fixed.
- **Every list shows the newest first by default**: tables, cards, lists inside a record (a visit's diagnoses, notes,
  lab results), record pickers, exports, in the app and the web console alike. The record someone just made must be
  on top, not on page 20 of 100. The unsorted state of a 3-state column sort *is* that default, so put it in the query
  (`created_at DESC`, then `id DESC` as the tie-break; UUID v7 ids are time-ordered), never only in the widget. A
  query with no ORDER BY returns oldest first in practice: every list query needs one. Kept on purpose: stock batches
  by soonest expiry (first expired, first out), and lookup lists people pick a name from (Ministry positions,
  districts, DHIS2 data elements, the fixed roles) A to Z with search. This rule held in every project before
  (NICE, Alpha, Letshego) and still had to be repeated on MRDRS (28 Nov 2026).
- **What is on screen stays live, both ways, when online.** A change made on one side (the app, the web console,
  another computer) shows on the other without closing and reopening anything: the server broadcasts a "something
  changed" signal on Reverb (never the data), devices sync on it and their screens watch the local database; the web
  console listens on its own private channel and refreshes the queries on screen (TanStack `invalidateQueries`,
  active only, at most once a second). An open form or dialog with unsaved edits is never overwritten: it reloads only
  when nothing is pending. Offline: nothing breaks, it catches up on the next sync. How:
  [../flutter/realtime-reverb-from-flutter.md](../flutter/realtime-reverb-from-flutter.md) ("Screens stay live").

Data and API:
- ULID keys; foreign keys `restrict` by default; audit columns on every table; an audit trail entry for every write.
- **No escape hatches:** no JSON columns for lists, references or anything a report may count (tables with FKs);
  no columns copied from a parent "to avoid a join" (derive through the relation); no add-column migrations in
  development (edit the create migration, `migrate:fresh --seed`).
- **Derived facts are read, never stored** (a client's stage from what they have). A state people move in and out of
  (dropped/reopened) is a flag with its own history table, not a new stage.
- **The API is the boundary:** every route has its permission; scoping by columns (a null branch = head office);
  validation per field with the true reason ("This NIN is already on another client, in Kampala East").
- **Every role sees only what is theirs,** checked with a test per role before shipping.
- **The admin holds every permission** so any flow can be run, except deciding an approval step: only the role the
  step names decides.
- Money in whole units (UGX integers); times stored UTC, "a day" in the business timezone; phones as digits with
  the country code.
- **Every Laravel + MySQL connection is pinned to UTC** (`'timezone' => env('DB_TIMEZONE', '+00:00')` on mysql and
  mariadb in `config/database.php`). `APP_TIMEZONE=UTC` alone does not do it: `TIMESTAMP` columns follow the MySQL
  session zone, which defaults to the server's. Check it in every project you open, old or new
  ([dates-and-timezones-with-an-ai-agent.md](dates-and-timezones-with-an-ai-agent.md) trap 7).

Lists and screens:
- **Every list is the standard table**: server paging (5 per page), rows-per-page, skeleton, search, one Filters
  dialog whose filters show as chips, one Export (every page; Excel, CSV, PDF), 3-state sort, state in the URL.
- **No hidden default filters.** A list with no filter shows everyone; any narrowing is a chip I can see and clear.
  (A dropped client "vanished" because the pipeline hid the dropped with nothing showing.)
- **View (the eye) on every row, before Edit**; reading never goes through an edit form. GPS numbers link to Google
  Maps. Bulk delete only (tick rows, Delete (n)); no per-row delete.
- **Confirm every write:** forms ask "Confirm submission" before sending; every other write (archive, send, drop,
  reopen, mark all read) asks first with its own spinner. Unsaved changes are never lost silently.
- **Back returns to where I was:** the list's page, filters and search (each list remembers its address); a reload
  keeps the scroll place. Page 3 stays page 3.
- **Dialogs never snap:** one dialog that loads in place; content kept while it fades out.
- **Loading looks like what is coming:** every page, list, dialog and chart loads in its own shape (header, cards,
  table rows, map frame in their places), filling the width as the real thing does, never one grey block, bars bunched
  to the left, or a spinner; spinners only for saving and sending.
- **Never a blank page between pages:** every route has a `loading.tsx` and every `<Suspense>` a fallback, in the
  shape of the page that is coming (a white screen reads as a crash).
- **A column most rows leave empty starts hidden** (a Description): still in the column menu and the export.
- **Forms in numbered sections**; places picked top-down (branch › territory › route), a level with one choice fills
  itself; every calendar has Today and Clear; 12-hour clock with am/pm.
- **Picked records in a form:** the form shows them as a table with Add, and Add opens a picker dialog (tick, Add);
  never two tables stacked in one view.

Dashboards and reports:
- **Every figure opens its rows:** a small arrow at the card's corner (on hover), carrying the period and place as
  the target list's own filters, so the list shows exactly what was counted. Add the API filter if it is missing.
  Never the whole card as a link.
- The filters in force as one sentence under the title, never chips beside the period bar; a period switch keeps
  the old figures dimmed with "Updating"; Pick days highlights the quick pick that matches.
- **Reports have charts, Export and print, and the records behind them** (the clients, the loans), not only totals.
- Help that names colours draws them; legends never squeezed (move them under the chart in narrow cards).
- Maps: OpenStreetMap first, Google layers in the switch, scroll to zoom, popups with Open in Google Maps, full
  screen, print; many points fold per person.

Mobile:
- **The web dashboard runs the whole process** through the same API calls the app will make (every field step has a
  web form), so the API is proven before the app exists.
- Keep the mobile build guide in step with every API change the app uses.

## 8. How to work in a session

- **Read the notes for the topic before proposing anything on it.** The readme's
  [Read before you touch](../readme.md#read-before-you-touch) table maps each kind of work (dates, models and
  migrations, roles, audit, realtime, Flutter, offline sync, deploys) to the note to open first. Open it **before**
  you give a view or write a plan, not after I remind you. If it is written down, I expect you to know it. (MRDRS,
  20 Nov 2026: an agent planned a whole sync API without the MySQL session timezone pin that the dates note
  documents, and I had to point at it.)
- **Facts before code:** read the code, docs and data; grep, never guess a column or route. Next.js and Laravel are
  newer than your training: read the installed docs.
- **Test what changed while working** (the filtered test, the touched browser spec, type check, lint); the full
  suites only before committing shared code. Never the full suite for a one-word change.
- **Docs as we go:** every change updates its `docs/*.md`, the user manual chapter, the mobile guide and
  `decisions.md` in the same change. Docs are part of done.
- **End with a short report:** what changed (as I would see it), what was tested and the result, anything not
  done, and the question I need to answer (usually "commit and push?").
- **Don't give up early.** "Impossible", "it's a ROM/library bug, wait for the maintainer" or "just keep it off" is
  the last answer, not the first. Before saying it:
  1. **Search where people hit the same thing:** XDA threads, GitHub issues (also closed ones and archived repos),
     Stack Overflow, vendor forums. Search the exact error text and the symptom in plain words.
  2. **Look at a project where it worked** (another ROM, an older version, a sibling app, the stock build) and compare
     what it ships or does. A closed issue there often names the fix in one line.
  3. **Trace it to the exact cause:** reproduce while logging, read the crash, open the code or binary at that
     spot. Then name the missing piece, not a category ("a missing app", not "AI features need new hardware").
  4. **If the root cause can't be fixed, still fix what you can:** stop the crash, keep the rest working, find a
     workaround, and write down what remains.
  Only call something impossible with proof (the log line or code that refuses it), and say what you tried.
  Real cases: `android/samsung-s10-custom-rom-yurom.md`, "Lessons".
- **Posts I publish (XDA, GitHub, forums): link everything.** Every credit, project, issue number and earlier post in a draft
  gets its URL; no bare names or `#numbers`. Give me the complete post as one copy-paste block, and keep a copy in the project.
- **Remember across sessions:** when I correct you, save the rule (your memory, or a note like this one) with the
  why, so the next session does not repeat the mistake.
