# The audit trail: snapshot schemas on every model

For an AI agent adding or upgrading an audit trail in a Laravel + Next.js app. The reference implementation is
**SDS** (`sdstool-laravel/docs/audit-trail.md`, `docs/snapshot-schema.md`) and **Alpha** (the same engine). **NICE**
still logs a sentence and an id ("Customer 'X' was updated"); its upgrade is planned for after the scheduled journey
plans module (the user, 06/11/2026: "schemas model by model for all, best practices, no escape hatches").

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
