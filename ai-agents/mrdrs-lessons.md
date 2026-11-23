# MRDRS: mistakes and lessons from a system that grew without a plan

**For the AI agent (or developer) starting a system that must run in many facilities, offline, with several
people per site.** MRDRS is a medical records and DHIS2 reporting app for Ugandan health facilities. I took it
over on 20 Nov 2026, a week before a field deployment, and found it could not be deployed. This note records **what
went wrong, why, and the rule now**, so the next system starts right. The project itself lives in
`D:\coding\new wave\mrdrs\mrdrs-workspace` (start with its `CLAUDE.md`, `brainstorm/`, `docs/decisions.md`,
`docs/plan-track-a-multi-facility.md`). Read [who-i-am-and-how-i-work.md](who-i-am-and-how-i-work.md) first.

**Status:** started 20 Nov 2026, add to it as Track A (the fix) and Track B (the rebuild) go on.

---

## 1. The story in five steps

1. A **single-person desktop app** (Flutter, SQLite) to fill DHIS2 forms and push them to the MoH HMIS.
2. **More DHIS2 forms** were added, one register screen per paper register.
3. **Facilities came in**, with reception, triage, clinicians, lab, pharmacy and cashier, all wanting it at once,
   offline.
4. The developers **shared the app folder on the network**, everyone opening the same exe on the same SQLite file:
   it locked and crashed.
5. A **Laravel + MySQL server on each facility's LAN** (Laragon, a static IP or Tailscale), every PC syncing its
   own SQLite to it every 2 seconds. It worked on a desk, but **nobody could install it at 40 facilities**: PHP,
   MySQL, migrations, IPs at each site, and no one to support it afterwards.

## 2. The mistakes, and the rule now

| # | Mistake | What it cost | The rule now |
| --- | --- | --- | --- |
| 0 | **"Offline" read as "no server".** The client wanted the work not to stop without internet; the developers built a standalone app, then bolted on sharing (a network folder, then a server per site). | Every later mistake in this table follows from it. | Ask what "offline" means and classify each feature: [offline-and-online-explaining-to-clients.md](offline-and-online-explaining-to-clients.md). Most business apps are offline-capable with sync, with a few online-only features. |
| 1 | **Deployment never designed.** Each site needed a mini server install. | The field deployment was blocked a week before going. | **Decide where it runs and who installs it before writing features.** Field install must be "run an installer, type a code". One server we run beats 40 we can't reach. |
| 2 | **SQLite file shared over the network** for many users. | Locks, corruption, crashes. | SQLite is for **one process on one machine**. Many users means a server. |
| 3 | **Single tenant** (no facility anywhere in the schema), and **fixed ids shared by every install** (one "Admin role" id for the whole fleet). | Could not put two facilities on one server without them merging. | **Multi-tenant from the first table** (`facility_id`, or the tenant column of that domain), and **no hardcoded shared ids**: per-tenant seeds come from the server. |
| 4 | **Built around the report, not the work.** Screens copied the paper HMIS registers (4,000-line forms); staff fill registers instead of serving the patient. | Fails UAT: the facility speaks "the patient in front of me", the app spoke DHIS2. | **Capture the work, derive the register, derive the report.** Each desk does its step on one short screen; codes (DHIS2, ICD) are tags set up once in master data; reports are computed on the server and approved before pushing. |
| 5 | **Reports computed on each client** (a 19,000-line monthly report screen). | Two PCs can give different numbers; untestable. | **Figures are computed in one place, the server**, with tests. |
| 6 | **Credentials and keys handled unsafely on the client** (details in the project's own docs, not here). | Unshippable to a shared server; a data protection risk. | **Hash passwords always** (bcrypt/argon2), even for offline login (verify against the hash on the device). **No secret in a client build**: the server holds third-party credentials per tenant; each device gets its own revocable token (enrolment code → Sanctum token). |
| 8 | **Roles built by hand in the field**: create a role, tick its permissions, then attach it to each user. | Hectic at every site; inconsistent roles between facilities. | **Roles come ready-made with their permissions** (a declarative seeder, as SDS and Letshego), one role per user, a free-text **title**, and per-user switches against the role for the exceptions. |
| 9 | **Phone layouts on a PC.** | Wasted screen, slow work at busy desks. | **Desktop-first where desks work** (tables, list and detail side by side, 2 to 3 column forms), then phone. Check at 1536×745 and 1920×1080. |
| 10 | **Poll every 2 seconds** from every PC. | Fine on one LAN; on a cloud server for 160 PCs it is 80 requests and ~4,000 queries per second with nobody working. | **Push, don't poll**: Reverb tells the tenant's devices "something changed"; a slow safety poll stays. |
| 11 | **Everything a string** (weight, BP, dates as text). | Reports parse text; bad values slip in. | Typed columns, validated at the boundary. |
| 12 | **Shipped as a zipped folder** (exe, DLLs, Laragon and Tailscale installers), updated by copying files. | Unprofessional; versions drift between PCs. | **A real installer and in-app updates** from our server, with a minimum version the server can force. |
| 13 | **No MySQL session timezone pin** while every synced table used `TIMESTAMP` columns, and the sync cursor sent to MySQL as a raw ISO string; Flutter reports built months in the device's zone. | A server not on UTC (or a moved database) skips or re-pulls rows; a PC set to the wrong zone shifts DHIS2 month edges. | Pin the connection to `+00:00` in every Laravel + MySQL project, parse request instants to UTC once, and on devices compute business days with a fixed-offset helper plus a wrong-clock/zone banner ([dates-and-timezones-with-an-ai-agent.md](dates-and-timezones-with-an-ai-agent.md) trap 7). |
| 14 | **Copy-paste instead of one shared piece**: 52 near-identical sync controllers on the API, 52 near-identical sync services in the app, the same permission-loading code pasted into 72 screens, 4,000-line screen files. | A rule change means 52 or 72 edits; bugs differ file to file; nobody can review it. | One generic piece driven by a table list (one sync controller/service configured per table, one permission service, small widgets). When you inherit copy-paste, change behaviour in the shared place it already funnels through (a base controller, a trait, one DAO function) instead of touching every copy. |
| 15 | **The whole local database in one 9,466-line file** (64 tables, 63 DAOs); migrations written `if (from == N)`. | Each schema change regenerated ~108,000 lines: build_runner took 6.1 GB of RAM and 28+ CPU minutes without finishing and froze a 16 GB PC; a device skipping a version misses migration steps. | Drift split by area with modular generation (first build 2.6 GB, then incremental), cumulative `from < N` migrations with migration tests: [../flutter/flutter-engineering-standards-for-agents.md](../flutter/flutter-engineering-standards-for-agents.md) §4a. |
| 16 | **Build junk and a SQLite database committed** to the repo. | Possibly real data in git history. | `.gitignore` from day one; never real data in a repo. |
| 17 | **Settings no user should see, on the main screens**: a button to type the server's address on the sign-in screen and the top bar, a manual sync button, roles to build by hand. | Looks unprofessional; one wrong address and the PC syncs nowhere. | **The device learns its server when it is enrolled; the sync runs by itself and shows a status, not a button; roles come ready-made.** Technical settings live in an admin console, not in front of nurses. |
| 18 | **Human numbers with a random part**: a patient number made of day + year + month + one random digit 0-4. | Five possible numbers a day per facility: patients shared numbers within hours, and 1 Nov / 11 Jan read the same (no zero padding). No test ever registered more than a few patients. | House style (prefix + month + year in the business timezone + serial), a device code from the server for offline devices, a counter that never goes back; a data-entry test that registers a dozen records. |
| 19 | **Reference data as read-only JSON in the app** (84,500 places) re-read on every use, with no way to add a missing village. | Slow forms, and staff typing places that do not exist or picking a wrong one. | A local table filled once, searchable pickers chained by level, facilities add missing places (synced), the national list read-only until a review process exists (two sites editing one national id collide). |
| 20 | **A dropdown package that places its list in window coordinates**, kept after the app moved into a shell (sidebar, custom title bar, nested navigator). | The multi-select for lab and imaging tests did not open where it should (or at all), and a request with nothing picked saved an empty test row. Nobody noticed until the client clicked it. | Option lists open in the app's own overlay or a dialog; many choices = chips + a searchable checklist dialog; nothing picked is a field error; an empty catalogue says where to fill it. The typing runner opens every select. |
| 21 | **A "pipeline" built as a second page** beside the list of people, its View opening a sub-record. | Two places for the same patients, and View landed on the visits list instead of the patient. | One list with status tabs and counts; View opens the person; the open visit's step and its work sit on the person's page (see [navigation-and-client-pipeline.md](navigation-and-client-pipeline.md)). |

## 3. What was done well, and is kept

- **Zed's sync protocol** (Jul to Sep 2026): UUID keys made on the device, a sync cursor on the **server's** clock (not
  the device's), soft deletes with tombstone endpoints, one change probe for all tables, a rule that schema changes
  are additive and new fields nullable so old app builds keep syncing. Reuse these ideas in any offline outbox.
- **The domain knowledge** in the report code: how each HMIS 105/033b/106a figure is counted. It is the spec for the
  rebuild.

## 4. How we got out (the decisions, 20 Nov 2026)

- **Two tracks.** Track A: deploy the current app now, fixed for the field. Track B: the rebuild around the
  patient's journey, after this phase. Don't try to do a rebuild in a week.
- **Track A:** one cloud Laravel server and **one** database for 40+ facilities, `facility_id` stamped by the server
  from the device's token (the client never sends it); devices enrolled with a code; Reverb instead of polling; UUID
  v7 ids (time-ordered like ULID, still UUIDs); hashed passwords; third-party credentials held by the server; ready-made roles;
  desktop-first restyle; installer and in-app updates. **Offline stays first-class**: each PC works with no internet
  as long as needed and syncs when data returns.
- **Firebase waits for the mobile/web apps**: Firebase Cloud Messaging has no supported Windows client.

## 5. For the next system like this

1. Before features: **where does it run, who installs it, who supports it, how does it update.**
2. **Tenant column in the first migration.** No shared fixed ids.
3. **Shadow the users for half a day** before designing screens; build from their flow, not from the report.
4. **Offline = a local store + an outbox + a server**, never a shared file and never a server per site.
5. Secrets never in the client; one token per device.
6. Roles ready-made; title free text; per-user exceptions as switches.
7. **Ticked rows belong to the rows on screen.** Clear the ticks whenever the page, search, sort or filter changes,
   and count only ticked rows still shown; otherwise a search leaves "Delete (4)" over one row.
8. **Open a detail page on top of its list, never pop the list first.** Popping then pushing left Back going home
   and the sidebar losing its place.
9. **Master data that ships with the app can still be kept from the console**: give it the app's own ids and send
   the changed rows in the normal pull, so no app release is needed for a missing village. Hide, never delete.
