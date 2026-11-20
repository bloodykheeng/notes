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

## 7. Quality gate (before every commit)

```
dart format .
flutter analyze            # zero warnings
flutter test
```

Plus the changed flow by hand on the real target (Windows build, Android device) against the local API. Unit tests
for logic (business day, permissions, sync conflicts, report figures), widget tests for screen states, an integration
test for the main flow. Release builds obfuscated (`--obfuscate --split-debug-info`). CI runs analyze + test + a build.

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
| Fixing by trial and error across many commits | reproduce, root cause, one tested fix |
