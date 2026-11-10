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
5. **Build in dependency order, MVP core first.** No side modules, no frameworks for one value: a setting lives
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
  own repo: a change across API, web and docs is three commits in three repos.
- **`CLAUDE.md`** at the workspace root: what the system is, where patterns come from (the reference projects and
  what to take from each), the domain as settled, binding conventions, how to verify. Short, linking to docs.
- **`WORK-PLAN.md`**: phases, the current one, its checklist and stop point. **`docs/decisions.md`** (above).
  **`.claude/handoffs/`**: a note written when I ask, with goal, status, next steps, decisions, still-open questions
  and gotchas; the next session starts from the latest.
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

- **Facts before code:** read the code, docs and data; grep, never guess a column or route. Next.js and Laravel are
  newer than your training: read the installed docs.
- **Test what changed while working** (the filtered test, the touched browser spec, type check, lint); the full
  suites only before committing shared code. Never the full suite for a one-word change.
- **Docs as we go:** every change updates its `docs/*.md`, the user manual chapter, the mobile guide and
  `decisions.md` in the same change. Docs are part of done.
- **End with a short report:** what changed (as I would see it), what was tested and the result, anything not
  done, and the question I need to answer (usually "commit and push?").
- **Remember across sessions:** when I correct you, save the rule (your memory, or a note like this one) with the
  why, so the next session does not repeat the mistake.
