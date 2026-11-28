# MRDRS: mistakes and lessons from a system that grew without a plan

**For the AI agent (or developer) starting a system that must run in many facilities, offline, with several
people per site.** MRDRS is a medical records and DHIS2 reporting app for Ugandan health facilities. I took it
over on 20 Nov 2026, a week before a field deployment, and found it could not be deployed. This note records **what
went wrong, why, and the rule now**, so the next system starts right. The project itself lives in
`D:\coding\new wave\mrdrs\mrdrs-workspace` (start with its `CLAUDE.md`, `brainstorm/`, `docs/decisions.md`,
`docs/plan-track-a-multi-facility.md`). Read [who-i-am-and-how-i-work.md](who-i-am-and-how-i-work.md) first.

**Status:** started 20 Nov 2026, last added 27 Nov 2026 (lessons 21 to 26: the first download, encryption, the
UI-thread freeze); add to it as Track A (the fix) and Track B (the rebuild) go on. How the app is built today:
[../architecture/mrdrs-track-a-architecture.md](../architecture/mrdrs-track-a-architecture.md).

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
10. **A form saves once.** Disable the button while saving, make the new record's id once per form (insert or
    update on it), and close the dialog only after every write succeeded. A write that threw after the insert left
    the dialog open and each press added a user, past the position's vacancies.
11. **No `!` on a record's optional field when an edit form opens.** Data synced from other devices and older builds
    has the gaps: `user.loginRole!` made Edit a grey screen in release for every staff member who cannot sign in.
    Read with `?? ''` and validate on save.
12. **Ship your TLS roots in a Flutter Windows app.** Dart trusts only the roots already in the Windows store, and
    Windows 10 adds a root only when a browser meets it: on a fresh PC the app said "Could not reach the server"
    while the browser worked. Add the CA's roots from assets to `SecurityContext.defaultContext` at start-up, and
    say "check the date and time" on a `TlsException` instead of a generic network error.
13. **A visit closes when its bill is paid, not when the bill is opened**: closing on "make the bill" hid the visit
    from the clinician while nothing was paid.
14. **A saved server address outlives every reinstall.** An old build's `http://uhfmrisapi.test/api/` override in
    `shared_preferences.json` made a laptop fail on every release, while we chased certificates and firewalls.
    Release builds drop a non-HTTPS saved address, and the error names the server it tried. Go look at the machine
    early: [remote-debug-a-windows-pc-over-ssh.md](remote-debug-a-windows-pc-over-ssh.md).
15. **A desktop window sized in fixed pixels runs off small screens.** Size it from the screen's work area
    (`screen_retriever`), maximise on small screens, and keep the minimum below the screen, or the title-bar buttons
    are unreachable. And give an old Windows runner the `FLUTTER_VERSION` defines, or the exe says 1.0.0 forever.
16. **A business rule must never trap the person using the app.** We first answered "can a drug be added after the
    bill is paid?" with a block ("start a new visit"), and suggested reopening old visits for 7 days only. Both
    would have stopped real work. The rule now: a rule may ask one question or record a reason, but always leaves a
    way forward (reopen with one click; close with a preselected reason; backdating open). General note:
    [../architecture/rules-that-never-trap-users.md](../architecture/rules-that-never-trap-users.md).
17. **Not every visit is paid for.** Free and exempted services, referrals and follow-ups never have a "paid in full"
    moment, so "the visit closes when the bill is paid" needs a Close button beside it (with a reason when money is
    owed), or open visits pile up for ever.
18. **Check every link column, not just the usual name.** An "is this record empty?" check looked for children by
    `vid` only; one child table used `visit_id`, so a visit with notes passed as empty and could be deleted.
19. **Make test accounts the way the product makes them** (the console or the models), never with raw SQL. A
    hand-made account skipped the "last write" stamp, so devices that were syncing never received it, and it lacked
    its role link, so it saw only the dashboard. Both looked like app bugs for an hour.
20. **Agent habits that cost time:** a git worktree has none of the git-ignored files (signing keys, so the APK was
    debug-signed and every install failed); after `adb install` check the installed version, never hide the output
    with `| tail`; never mask secrets with `sed` on the way to the screen (read them file to file); on a phone, move
    between fields with TAB, since taps miss once the keyboard shifts the layout; and clear a password field with
    End plus backspaces before retyping, as select-all did not take.
21. **Finish the first download before letting anyone in.** We let people in after the staff tables and brought the
    rest down in the background (it "felt faster"). The owner, testing on a laptop, saw the gap at once: someone opens
    Patients mid-download, does not find a returning patient who has not arrived yet, and registers them twice. Now
    one full pass runs on the sign-in screen with real progress, waits for the internet and carries on by itself, and
    reopens on the download if the app is closed. How: [../flutter/first-download-with-progress.md](../flutter/first-download-with-progress.md).
22. **Heavy local work at the sign-in moment broke the network.** Loading 84,500 places into the new file on the UI
    thread (drift's default) froze the app for seconds; the download's first request timed out. Two "bugs" came from
    it: permissions and staff downloaded twice (the step that marks tables done never got the server's time), and the
    chip said "Offline" while online. The server's access log showed no failed request: when the server saw nothing,
    suspect the client freezing. Defer and chunk heavy writes.
23. **A Windows release can miss a native library that every test loads.** Encryption moved SQLite into a build-hook
    DLL; the project's older Windows CMake template never copied it beside the exe, so the release had no SQLite while
    all tests passed (they load it from the build folder). Run the real release on a laptop, not only the tests:
    [../flutter/encrypting-the-device-database.md](../flutter/encrypting-the-device-database.md).
24. **A server command that writes with raw SQL is invisible to syncing devices** (the same trap as lesson 19, in
    production code): a "reopen unpaid visits" command updated rows without stamping the site's last write, so every
    idle device was told "nothing changed" and moved its cursor past them for good. Stamp the site in every raw write,
    and prove it with a test of an idle device.
25. **"Remove" then "add back" reused the old file.** Removing a site deleted its file only at the next start, and
    adding it back took it off the delete list, so the "download" was one quick check. On a flaky line that check
    failed and the app showed "Offline" right after sign-in, while the owner saw a bar that never moved. Fixed: the
    file stays on the delete list and the database deletes it before opening it fresh (Remove is only offered when
    everything is synced). Keep what the dialog promises ("this device forgets the facility") literally true, and
    let the sign-in screen report only a clean pass to the status chip.
26. **Agent habits from the phone and laptop tests:**
    - Before every tap on a phone, check that the app is in front (`dumpsys window` → `mCurrentFocus`). The owner
      took a call mid-test and the taps landed in the call and a chat; a tap may have muted them.
    - When a password from the accounts file is refused, check the server's account (active, last edited) before
      blaming the app. The file had been edited. Then reset the test account the product's way (the console's
      one-time password), save it to its own file and never print it.
    - Time the network with `curl -w` before blaming the app.
    - Don't stop at "it passed once": prove the hard cases (internet cut mid-download, app killed mid-download,
      a truly fresh file) with the access log.
27. **A slow answer acted on "whatever is current now" removed the wrong facility.** Remove signed the device out
    on the server (up to 10 s on a slow line) while the background sync kept running with that token; its "not
    allowed" reply then made the app forget the facility in use **at that moment**, the other one, and Remove went
    on to remove another. The owner saw "removing one facility removes all the others". The rule: **name the target
    before anything slow** (the facility id, its token), pause background work during the change, and let a reply
    act only on what it was about (a pass remembers the facility it started for and ignores its own answer once the
    app has moved on). Any "current X" read after an `await` is a suspect.
28. **A form box sized from the screen ("screen height - 100") was taller than its dialog on a phone**, so Save sat
    half below the screen however far one scrolled. The same box was pasted into 17 forms. The fix lives in one
    shared helper: on a phone the steps take their natural height with Save after them; the fixed box stays for PCs.
29. **When you find a bug, look for the same pattern everywhere before calling it fixed.** The two above were in one
    place and in 17 places; the owner asked for this explicitly. Search the code for the pattern (`grep` the
    construct, not the screen's name), fix it in a shared place, and list what you checked.
30. **"Is there a Laravel Echo for Flutter?" Check pub.dev's dates and platforms before following a tutorial.** The
    tutorials (2020 to 2022) use `laravel_echo` / `flutter_laravel_echo` (Dart 3 incompatible, socket.io) or
    `pusher_client` (no Windows). For Windows plus Android the pure-Dart `dart_pusher_channels` worked against Reverb
    with the device token at `broadcasting/auth`; the newest version needed the app's `http` moved from 0.13 to 1.x.
    Keep polling as the fallback so the socket can only help. How: [../flutter/realtime-reverb-from-flutter.md](../flutter/realtime-reverb-from-flutter.md).
31. **A status gets one colour everywhere: tables, details dialogs and view pages.** The tables had coloured pills
    while the details showed "Is dead: Yes" as text, and five old details pages had their own label-and-text
    helper. The fix: one map from status to colour per codebase (Flutter `StatusColors`, React `StatusTag`), one
    details-value widget that turns known statuses and yes/no flags into pills, and every details page routed
    through it. When adding a status, add it to both maps the same day; check the details view, not just the list.
32. **Opening a screen must never write.** A patient details page stored the age worked out from the date of birth
    every time it opened, and marked the row changed. In an offline-first app that syncs, a view became an edit:
    the row went to every device, "Last changed" showed the viewing time, and with newest-edit-wins a device holding
    an older copy could overwrite someone's real edit just by looking. Found only because a tester's "Last changed"
    showed the minute the page was opened. Rule: a write from a screen's open or build compares first and writes only
    a real change, or better, derive the value when shown. Search for writes in `initState` and page loads.
