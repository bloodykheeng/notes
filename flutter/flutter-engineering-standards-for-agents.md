# Flutter engineering standards for AI agents (and the developers driving them)

**For the AI agent working on any of our Flutter apps.** This is the bar. It exists because we have seen what happens
without it: MRDRS (2024 to 2026) grew by trial and error into 52 copy-pasted sync services, permission code pasted
into 72 screens, 4,000-line screen files, analyzer output files committed and credentials handled unsafely on the client ([../ai-agents/mrdrs-lessons.md](../ai-agents/mrdrs-lessons.md)). Letshego's field app was built from a
written guide and did not ( `D:\coding\new wave\letshego\letshego-workspace\docs\flutter-app-guide.md` §0 and §6, the
source of most of these rules). **Don't gamble: look it up, decide once, write it down, test it.**

Read with: [flutter-offline-desktop-and-mobile.md](flutter-offline-desktop-and-mobile.md) (offline, dates, devices,
push, desktop), [../ai-agents/who-i-am-and-how-i-work.md](../ai-agents/who-i-am-and-how-i-work.md), and the app's own
`CLAUDE.md`/`AGENTS.md`.

---

## 1. How to work (the anti-gambling rules)

1. **Facts before code.** Read the code you are changing and what calls it; grep, never guess a table, column, route
   or provider name.
2. **Official docs and current practice before any pattern** (state, navigation, offline queue, pagination, push,
   permissions, a plugin's setup): docs.flutter.dev, dart.dev, the package's README and changelog, firebase.google.com
   /docs; then search for current practice. Write the choice and its sources in the repo's `docs/decisions.md`. Your
   memory of a package is probably a version behind.
3. **Packages from the official tool, at the latest stable version**: `flutter pub add <name>`. Before adding one,
   check pub.dev: publisher, last release, Dart 3 / null safety, pub points, open issues for **your platforms**
   (Windows support is often missing or partial). Hold a version back only with a written reason.
4. **Spike the risky thing first** (a plugin on Windows, a websocket client, background work): a half-day throwaway
   proof on the real platform before building on it.
5. **Root cause, not trial and error.** Reproduce, read the full error and stack, find the line. No "try this, try
   that" commits, no `analyze_output7.txt`. If a fix is a guess, say so and prove it.
6. **One way of doing each thing.** Before writing a new pattern, look for the existing one and use it; if the
   existing one is bad, change it in its one place.
7. **Small steps, each verified**, and the user's rules: own branch, commit and push only when asked, docs with every
   change.

## 2. Structure: feature first, thin screens

```
lib/
  app/        app.dart, router.dart, theme/ (Material 3, ColorScheme.fromSeed, one font, light + dark)
  core/       api/ (one HTTP client, interceptors, ApiError), auth/, config/ (flavour env), db/ (drift),
              sync/ (one engine), time/ (business day helper), permissions/, push/, realtime/, storage/
  features/<feature>/
              data/ (repository: the only place that talks to the API or the database)
              domain/ (models)
              presentation/ (screens, widgets, controllers)
  shared/     widgets used by several features (tables, form sections, empty/error/loading states)
```

- **Screens never call the HTTP client or the database directly**; they call a repository through a controller.
- **No business logic in widgets.** Calculations (a report figure, a BMI, a balance) live in plain Dart classes with
  unit tests.
- **Keep files small**: a screen over ~400 lines is split into widgets; a form is sections, each its own widget. A
  4,000-line file can't be reviewed.
- **The local database too:** tables and DAOs in files per area with drift's modular generation, never one giant
  `database.dart` (build_runner must load the whole file at once to generate it).

## 3. The stack (new apps; check versions on pub.dev at the time)

| Need | Use |
| --- | --- |
| Lints | `very_good_analysis` (or `flutter_lints`), **zero warnings** |
| State | **Riverpod 3** with code generation, `AsyncValue` for every load (Letshego) |
| Navigation | **go_router** (typed routes), redirect on auth state |
| HTTP | **dio** with interceptors: base URL, Bearer token, `Accept`, `X-App-Version`, `X-Socket-ID`, 401 → sign out, errors mapped to one `ApiError` |
| Models | **freezed** + **json_serializable**: no `Map<String, dynamic>` passed around screens |
| Local data | **drift** with DAOs; indexes on what you filter by; migrations tested |
| Secure storage | `flutter_secure_storage` (tokens only) |
| Config per flavour | `--dart-define-from-file=env/<flavour>.json` (local, test, prod); no URLs or keys in source |
| Realtime | a pure-Dart Reverb client (spike on each platform) |
| Push | `firebase_messaging` on Android/iOS/web; Windows: tray + Reverb + `flutter_local_notifications` |
| Desktop | `window_manager`, `tray_manager`, `launch_at_startup` |
| Tests | `flutter_test`, `mocktail`, `integration_test` |

**An app you inherit keeps its state library during a fix-up phase** (do not run Provider and Riverpod side by side
because one screen was easier): new code follows the existing one, in the new folder structure; the switch is planned
as its own step.

## 4. Data, sync and offline

- **One generic sync engine driven by a table registry** (endpoint, table, dependencies, mapper), never one
  copy-pasted service per table.
- The rules in [flutter-offline-desktop-and-mobile.md](flutter-offline-desktop-and-mobile.md) §1 to §4: local store
  + outbox + server, server-clock cursor, tombstones, additive schema changes, ids made on the device (UUID v7), UTC
  on the wire, business days with a fixed offset, device tokens, no secrets in the build, hashed passwords.
- **Typed columns** (numbers as numbers, instants as `DateTime`), not strings for everything.
- **Money in whole units** (UGX `int`), never `double`.
- **Balances are derived from movements** (stock, payments), never a column edited on several devices.

### 4a. The local database (drift) done right

Measured on a real project (MRDRS, Nov 2026, 16 GB Windows PC): the whole drift database (64 tables, 63 DAOs) in one
9,466-line file regenerated ~108,000 lines in one go: **build_runner held 6.1 GB of RAM, ran 28+ CPU minutes and
never finished**, the PC paged to disk (memory 92%, disk 100%). After splitting by area with modular generation,
the same first full build ran at **2.6 GB and finished in 8.5 minutes**. Later builds regenerate only what depends
on the change: a DAO edit is small, but a **table** edit also regenerates the DAOs and the database that use it (7.5
minutes measured), so batch table changes into one build.

- **One file per area, never one giant `database.dart`:** tables in `lib/data/tables/<area>.dart`, one DAO per file in
  `lib/data/daos/<name>_dao.dart`, barrels (`tables.dart`, `daos.dart`) re-exported by the database file so the rest
  of the app keeps one import.
- **Modular generation** (`build.yaml`: `drift_dev` off, `drift_dev:analyzer` + `drift_dev:modular` on): each file
  gets its own `<file>.drift.dart`; the database `extends $AppDatabase`, a DAO `with $NameDaoMixin` and imports its
  own `.drift.dart`. (https://drift.simonbinder.eu/generation_options/modular/)
- **Migrations are cumulative and tested:** `if (from < N)` for every step (an `if (from == N)` chain silently skips
  steps on a device that jumps versions); use drift's migration tooling (schema snapshots and generated migration
  tests, see drift's "Migrations" docs) so each version upgrade is proven before release.
- **Every foreign key says what happens on delete** (SQLite's default refuses deleting a parent with children, so a
  synced tombstone for a parent cannot be applied on a device holding its children): `ON DELETE CASCADE` for parts
  (bill items → bill), `RESTRICT` written out for references (anything → patient), `SET NULL` only for a disposable
  link with a comment why; a Dart non-null foreign key also says `NOT NULL` in SQL. Health records are deleted softly
  and refused in the screens when they have records, never cascaded away. (Found on a real app: 80 foreign keys,
  none with ON DELETE.)
- **Dates:** a `DateTimeColumn` stored as Unix time is an instant (keep `store_date_time_values_as_text: false`); a
  calendar day (date of birth) is written as the business day's start with a fixed-offset helper, never midnight in
  the device's zone; text dates, where they exist, keep one format (`yyyy-MM-dd`, `HH:mm`) until they can be typed.
- **Queries live in DAOs**, never in screens; screens call a repository or DAO method.
- **Run build_runner alone** on a modest PC (close browsers), `--low-resources-mode` if memory is short, and
  `dart run build_runner watch` while editing tables so each change is a small incremental build.
- **Keep drift current** (read the changelog before a major); generated code from an old drift can be slower and miss
  fixes.
- **Weighed and rejected for a relational offline app:** plain SQL with hand mapping (no type safety; a rewrite of every
  DAO), Isar and Hive (not relational, original maintainers left; community forks only), ObjectBox (not SQL), Realm
  (its sync service deprecated). PowerSync (with drift) is worth a look when choosing a sync service. (From knowledge
  as of Nov 2026; verify online before choosing.)

## 5. Permissions, errors, forms

- **One permission service** (`can('record-payment')`), loaded at sign-in and on sync; screens ask it. Never paste
  permission-loading code into screens.
- **Errors:** the server's per-field reason shown on the field (422); network failure says what will happen ("Saved on
  this computer, will send when online"); never swallow an exception, never `print` as error handling (use a logger).
- **Forms:** numbered sections, shape checks on the device, rules on the server, confirm before writing, keep what was
  typed on failure, dates with Today and Clear, 12-hour times.

## 6. Screens that look professional

- One theme, one app shell, the same table/form/dialog widgets everywhere (`shared/`).
- **Desktop-first where desks work**: data tables, list and detail side by side, 2 to 3 column forms from 1280 px,
  keyboard friendly; phone: one column, bottom tabs, big touch targets.
- Every list: paging (or infinite scroll on phones), search (debounced), empty / loading (shaped skeleton) / error /
  "no more" states. Never an endless spinner.
- Check with screenshots at 1536×745 and 1920×1080, light and dark, with realistic data, and on a phone.

### 6a. Theme lessons from a real restyle (Nov 2026)

- **`ColorScheme.fromSeed` makes pastels by default** (`DynamicSchemeVariant.tonalSpot`). The client said the app
  "looks like paper, pale". Use `dynamicSchemeVariant: DynamicSchemeVariant.fidelity` to keep the seed as the
  primary, and set `primary` / `onPrimary` yourself for dark mode if the generated one is too dull.
- **Find the colour helper everyone calls.** An old app had `ThemeColors.accentColor(context)` returning
  `Colors.blue` in 2,000+ places. Making that one function return `Theme.of(context).colorScheme.primary` restyled
  most screens at once. Grep for it before touching screens one by one.
- **Dark mode dies on hard-coded colours.** Count them first (`fillColor: Colors.white`, `TextStyle(color:
  Colors.black)`, `Colors.blue`...). Remove field fills (the theme's `inputDecorationTheme` fills), replace black
  text with an empty `TextStyle()`, and map old blues to one const accent. Do it with a script, then
  `flutter analyze` and a diff of warnings against a saved baseline.
- **After changing the app bar colour, look for white icons.** A sync button drawn `Colors.white` on a white app bar
  was invisible for days.
- **A shell with a nested Navigator: decide where Back belongs.** Pages opened from the sidebar are top level, so the
  app bar shows no Back (`automaticallyImplyLeading: false` when the route's name is a section id). Only detail
  pages get Back. Keep the drawer button on narrow screens.
- **Desktop users expect to select text and open photos.** Wrap the content area in `SelectionArea`; give photos one
  shared full-screen viewer (`InteractiveViewer`, Save as..., Esc to close).
- **`Image.memory` in a list flickers** when a stream re-emits rows (a new byte list each time means a new decode).
  `gaplessPlayback: true` plus a thumbnail `cacheWidth` stop it.
- **Register every new asset in `pubspec.yaml`.** A widget test that pumps the screen catches a missing one
  ("Unable to load asset"); a hand check on an old build does not.
- **Don't hand-make `Scrollbar(thumbVisibility: true, controller: c)` around a page.** It asserts ("no
  ScrollPosition attached" / "attached to more than one ScrollPosition") whenever its scroll view is not built (an
  empty list) or the controller is shared by two scroll views: a red error that flashes on navigation. On desktop
  the default `ScrollBehavior` already gives every scroll view a scrollbar. One controller, one scroll view.
- **No `onDoubleTap` to open a record.** Inside a `SelectionArea` a double-click also means "select the word", and
  the two gestures fight (it felt slow). Open on one click, and give each row a View action.
- **Draw small brand marks in code** (`CustomPainter`) when the only logo file is a small PNG: scaled down it looks
  faded; drawn, it is sharp at every size. Ask the client for the vector file for icons and installers.
- **Widget tests with drift:** a screen that writes to the database in `initState` never finishes under the test's
  fake clock. Pump it inside `tester.runAsync`, use `AppDatabase.forTesting(NativeDatabase.memory())`, set a
  desktop `tester.view.physicalSize`, and focus fields with `tester.showKeyboard` (a `tap` outside the 800×600
  default window silently misses).

## 7. Quality gate (before every commit)

```
dart format .
flutter analyze            # zero warnings
flutter test
```

Plus the changed flow by hand on the real target (Windows build, Android device) against the local API. Unit tests
for logic (business day, permissions, sync conflicts, report figures), widget tests for screen states, an integration
test for the main flow. Release builds obfuscated (`--obfuscate --split-debug-info`). CI runs analyze + test + a build.

## 7a. A new Flutter version can break old packages (and the analyzer won't tell you)

On Flutter 3.50, `IconData` became a `final` class, so `font_awesome_flutter` 10.x (which extends it) stopped
compiling: **the whole app would not build**, yet `flutter analyze` showed nothing, because it does not report errors
inside packages. Only `flutter test` or a build caught it. The fix was `font_awesome_flutter` 11 (`FaIcon` instead of
`Icon` for Font Awesome icons; 18 places). Rules:

- After any Flutter upgrade (or a fresh SDK install on a project last built on an older one), **build for each target
  and run `flutter test`**, not only `flutter analyze`.
- Read a package's breaking changes before a major bump, and search for the old usage across lines (a pattern split
  over two lines escapes a one-line grep).

Every build error seen so far, with its fix: [flutter-build-troubleshooting.md](flutter-build-troubleshooting.md).

## 8. Repo hygiene

- `.gitignore` covers `build/`, `.dart_tool/`, `*.sqlite`, `*.db`, analyzer/test output files, keystores, `env/*.json`
  with secrets. **Never commit a database file** (it may hold real data) or a signing key.
- A `CLAUDE.md` + `AGENTS.md` at the root (what the app is, stack and versions, branch rule, commands, conventions, where
  the docs are), `docs/*.md` per settled pattern, `docs/decisions.md`, handoffs in `.claude/handoffs/`.
- Old packages upgraded on a schedule, one at a time, reading the changelog, tests green after each.

## 9. Anti-patterns seen (do not repeat)

| Seen | Instead |
| --- | --- |
| 52 copy-pasted sync services, 72 screens each loading permissions | one engine from a registry; one permission service |
| 4,000 to 19,000-line screen files mixing UI, queries and report maths | features, repositories, small widgets, logic in tested classes |
| Everything a `String` (weight, BP, dates) | typed columns and models |
| Credentials handled unsafely on the client | bcrypt hashes; third-party credentials held by the server; per-device tokens; flavour config |
| `analyze_output*.txt`, `db.sqlite`, DLLs committed | `.gitignore`; run the tools, don't commit their output |
| Phone layouts stretched on a PC | desktop-first layouts |
| A fixed id shared by every install | per-tenant data from the server |
| Polling every 2 s | push signal + slow safety poll |
| The whole drift database (64 tables, 63 DAOs, ~9,500 lines) in one file: each schema change regenerated ~108,000 lines, using 6 GB of RAM and 20+ minutes on a 16 GB PC | one file per area (`lib/data/tables/`, `lib/data/daos/`) with drift's modular generation, so a change regenerates only its file |
| Fixing by trial and error across many commits | reproduce, root cause, one tested fix |
