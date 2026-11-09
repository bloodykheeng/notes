# The audit trail: snapshot schemas on every model

For an AI agent adding or upgrading an audit trail in a Laravel + Next.js app. The reference implementation is
**SDS** (`sdstool-laravel/docs/audit-trail.md`, `docs/snapshot-schema.md`) and **Alpha** (the same engine). **NICE**
was upgraded on 07/11/2026 (`nice-laravel-api-pase-two/docs/audit-trail-upgrade-plan.md`: decisions, how to write an
entry, progress); section 5 below is what that upgrade found.

## 1. What an entry must answer

| Question | Where |
| --- | --- |
| Who | `causer_id`/`causer_type`, plus the name and email frozen in the properties |
| What | `event` (`district_updated`), `description` (a sentence), `outcome` (success / failed) |
| Which record | `subject_type`/`subject_id`; `log_name` = the **module** (an enum), not the event |
| What changed | `properties.changes` (updates: field => label, type, old, new) or `properties.snapshot` (creates, deletes) |
| What else went | `properties.cascade`: what a cascading delete took with it |
| When | `created_at`, the moment it happened (passed to the job, not when the job ran) |
| Where | IP, location, device, browser, the request id, method, route, the sanitised request |

"District was updated" with an id answers none of the useful ones.

## 2. The pieces

- **`HasSnapshotSchema`** (a contract) with `public static function snapshotSchema(): array`, on **every model** whose
  writes are logged. One declaration, read by the audit trail (and an approval engine's before/after view later).
- **Schema entries:** `label`, `type` (a closed list: string, text, enum, relation, date, datetime, boolean, number,
  decimal, money, percentage, file, link, list), `resolve` (`fn ($id) => ?string` for a relation), `enum`
  (value => label), `decimals`, `hidden`, `only`, `kind` (`list` / `object`), `current`, `item`, `fields`, `strategy`,
  `max`. A test holds the type list closed (SDS found `integer` silently rendering numbers as text).
- **`SnapshotPresenter`**: turns raw attributes into `snapshot` and `changes` envelopes **at write time**, resolving
  foreign keys to `{id, label}` then, so a later rename never rewrites history. Unwraps enums before comparing.
- **`LoggableTrait::logActivity($event, $description, $properties, ?Model $subject, ?string $outcome, ?User $causer)`**:
  the only way to write (never `activity()` directly); adds the outcome and the request context, drops empty keys,
  redacts secrets (passwords become `password_changed: true`), derives the module from the subject.
- **`LogActivityJob`**: queued, dispatched `afterCommit`, so a save never waits on IP geolocation and a rolled-back
  save leaves no entry claiming it happened. Sign-in and other pre-auth events pass the causer explicitly.
- **`AuditModule` enum**: every `log_name`; a test fails when a module is written that is not a case.
- **The screen**: a list with a Module column and filters (module, event, user, date), paginated and exported, and a
  **generic view dialog** that renders any entry's `changes` (old → new table) and `snapshot` from the envelopes, with
  no per-module code.

## 3. Rules that make it trustworthy

- **Diff what was written, not what was asked for:** capture before, save, diff after (from the model's attributes).
- **Destructive actions get the most detail:** a delete stores a full snapshot and the cascade (the record will not
  exist to look at later). A bulk delete names every record.
- **No bare ids in the readable sections:** relations are `{id, label}`; the raw request is the one exception.
- **The trail is a copy beside the data, never the only home for a fact.** Retention deletes it (NICE prunes after 6
  months); a fact people need later belongs in a column of the record itself (see
  [master-copy-and-snapshots.md](master-copy-and-snapshots.md)).

## 4. Upgrading an app that logs sentences (the NICE plan)

1. Port the engine (contract, presenter, trait signature, job, module enum, resource) so the existing
   `logActivity($name, $message, $props)` calls keep working and old entries still display.
2. The audit screen: Module column, filters, the generic view dialog.
3. Model by model, **all of them**: a `snapshotSchema()` per model, and each controller's update captures before and
   diffs after. A test lists the models that log and fails when one lacks a schema.
4. No escape hatch: no "we will add schemas later" for some models, no free-text modules, no JSON dumps of whole
   requests in place of a diff.
5. Re-file the old entries once (a data migration: the old action becomes the event, the module becomes the log
   name), so the filters cover them.

## 5. What the NICE upgrade found (07/11/2026)

In the old entries:

- **Secrets.** The SMS trait logged the whole text: one-time password codes and new accounts' passwords. An SMS entry
  now keeps the recipient and the provider's answer; the re-filing migration removed the text from every old one.
- **Noise.** A sync wrote "cron called", "dispatched", "job started", one entry per batch of a hundred, "complete" in
  the trait and "success" in the job: forty per cent of the trail. One entry per action, with its result capped
  (`boundedResult`: lists cut at 20, the number cut kept).
- **Dumps.** Stack traces (26 calls), base64 photos in the request body, whole job payloads. A failure keeps its message
  and file:line; a request value over 1,000 characters is kept as its size.
- **Broken calls** that threw on their own failure path: an array passed as the description, request fields spread as
  named arguments. Nobody noticed because the failure path is the one nobody runs.
- **A past tense built as `"{$action}d"`**: "rejectd" in the events, the sentences and the answers people read.
- **Who could read it:** any signed-in user, through the API. Now the people who see Settings; deletes a System Admin
  only, each delete recording what it removed.

Mistakes an agent made doing it, and the check that caught each:

- **A converter guessed the record from the sentence** (`"... by {$user->name}"`) and logged the user as the subject of
  a sell-in order. Caught by listing every converted call with its variable. Read the subject from the write, never the
  description.
- **The before-state captured in the wrong place:** before a null check, or overwritten by the method's own `$before`.
  A script listed every `captureOriginal()` with the line it follows.
- **`getChanges()` emptied by a `refresh()`** (and by a save made through a service), so an update showed no change.
  `buildChanges()` now diffs the record's attributes; the presenter keeps only what moved.
- **"No to No"**: a flag stored as empty, written as false. A field whose old and new read the same is left out.
- **A module named after a class that was not a case** (`Job`, `DataSync`): it would have thrown in production code
  paths no test ran. Grep every `AuditModule::X` against the enum.
- **A converter run twice inserted a line twice.** Make a converter skip what it already did, and grep for doubles.
- **A trace replaced in an application `Log::error()`**, where a trace belongs. Look at the call before changing its
  arguments.
- Proof that is not a test: drive the real endpoints on a copy (create, update, delete, a failed sign-in) and read every
  entry back; and plant a call that breaks the rule to see the coverage test fail.

## 6. Sign-ins, password resets, and which app (NICE, 09/11/2026)

**Record every authentication step** under one module (`Authentication`), the person as the causer when known:
`logged_in`, `login_failed` (the identifier typed, never the password), `login_refused` (inactive account, role with
no access), `logged_out`, `password_reset_code_sent` (by which channels, never the code),
`password_reset_code_email_failed` / `_sms_failed`, `password_reset_code_refused` (wrong or expired: repeated ones are
someone guessing), `password_reset`, the confirmation's delivery failures, and a password changed on a profile. Alpha
and GiftNest record the same set; check EVERY way in, NICE's third sign-in (the AI connector's browser page) recorded
nothing.

**Name the app on every entry, not only on sign-ins.** Give each client its own token name at sign-in (NICE:
`auth_token` for the dashboard, `mobile_auth_token` for the Flutter app, Passport for the connector) and have the
audit trait read `$request->user()->currentAccessToken()->name` on every entry; before sign-in fall back to the user
agent (`Dart/` is the app, a browser the dashboard); outside a request it is `System`. An enum for the values, a column
and a filter on the screen. Document that the token names must not change.

**What reading the reset controller turned up** (read the whole flow, not just where the log call goes): codes that
never expired (Carbon 3's signed `diffInMinutes`), a person with only a phone number who could not ask for a code (a
non-null `email` column), the SMS signed with ANOTHER SYSTEM's name (copied from an older project: grep the repo for
other projects' names), one channel's failure stopping the other, and `$e->getMessage()` returned to the screen. Prove
each with a test that fails on the old code (put the old file back, run, restore).
