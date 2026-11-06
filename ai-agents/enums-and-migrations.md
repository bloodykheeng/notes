# Enums in the models, and migrations people can read

For an AI agent writing Laravel models and migrations. Binding on Alpha (`docs/enums-and-statuses.md`) and, since
06/11/2026, NICE (`docs/enums-and-statuses.md`, `docs/migration-conventions.md`). Both came from the user correcting an
agent's work.

## 1. Every fixed list of values is an enum

A status, type, mode, outcome, frequency, module: a PHP **string-backed enum** in `app/Enums`, **cast on the model**,
validated with `Rule::enum(...)`, mirrored on the web in `lib/enums/*.ts`. Never string constants on a model
(`const STATUS_APPLIED = 'Applied'`), never bare strings in code.

- The column is a plain `string` (indexed when filtered), never `$table->enum()`: a new value is a new case, no
  migration. Its `comment()` **lists the values and names the enum**: `'Staff or Schedule (App\Enums\JourneyPlanSource)'`.
- Behaviour lives on the enum (`ScheduleFrequency::weeks()`, `label()`, `fromSheet()`); lists of cases are constants on
  it (`ScheduleUploadStatus::BUSY`).
- Once cast, the attribute is an object: compare with `=== Enum::Case`; pass the case to queries; use `->value` in
  messages, array keys, `countBy`/`groupBy` callbacks and raw SQL or `DB::table()` rows; convert request input once
  (`Enum::from($validated['mode'])`, `tryFrom` for an optional filter). JSON serialises to the value.
- Before casting an existing column, read its distinct values: one outside the enum throws on read.
- Name values in the business's words, one vocabulary end to end. NICE renamed `source` "User" to **"Staff"** and
  every "associate" to "staff" because anyone (a Regional Manager too) can be on the schedule.
- A heading names the column it shows. An agent headed an upload's `result` column "Does" / "Did" (by tense), then
  "Change to the schedule"; the user: "in the back end it's result, call it result". Short, the stored name, the
  same before and after an action; the values can still read as sentences ("New customer", "No change").
- Keep a "not yet cast" list in the docs for old columns, and cast them when their module is next changed.

## 2. A migration is read more than it is run

- **Every column written out.** No `foreach` making columns (the user: "loops look cool but are confusing"): six
  weekday booleans are six lines. Loops are for data migrations.
- **Group related columns**, each group under a one-line comment, a blank line between groups: keys, then the field
  groups, then audit columns and timestamps, then indexes, then foreign keys.
- **Explanations above the column**, `comment()` short: the values, the enum, the format with an example
  (`'e.g. "Mon", "Mon, Thu"'`), or what moment a UTC `dateTime` holds.
- **No `date` columns**: every date is a UTC `dateTime` (Letshego's rule).
- **Foreign keys** at the bottom, `->onDelete('restrict')` for references and audit columns, `cascade` only for parts,
  `set null` only with a comment saying why (and ask whether the key should exist at all).
- **The file name says what it does now**; rename a migration whose job changed before the deploy.
- **Before any deploy, edit in place** (rebuild local tables with a one-off script, re-apply local data through the
  app's services). **After a deploy, never**: write a new migration.

## 3. Mistakes an agent made doing this (NICE, 06/11/2026)

- Added a JSON `summary` column to hold per-person counts; the user asked whether it was an escape hatch. It was: the
  counts were a GROUP BY over rows already stored, plus one small table for what had no row. Ask that question of
  every JSON column.
- Kept string constants for statuses after Alpha had already made enums binding: read the reference project's
  conventions doc before writing models.
- Ran the formatter on whole folders (`pint app/Models app/Jobs ...`) and reformatted fifty files nobody had touched,
  then had to check and restore each. **Format only the files you changed**:
  `pint $(git status --porcelain | awk '{print $NF}' | grep '\.php$')`.
- Left a queue worker running code from before the enum change; four browser specs failed together. Restart the
  worker after changing anything a job touches, and read its output first when queue-backed tests fail at once.
