# Dates and timezones: the rules, the traps, and how to check a whole app

**For:** an AI agent (or a developer) working on a Laravel API + Next.js web + mobile app where money and
activity are reported by day. **Worked example:** NICE (Kampala, UTC+3), fixed 05/11/2026. Read this before
touching any date, filter, report or chart.

---

## 1. The rules (one flow, no exceptions)

1. **Store UTC. Query UTC.** Laravel `app.timezone = UTC`; every datetime column holds a UTC instant.
2. **"A day" is the business timezone's day**, from one config key (`BUSINESS_TIMEZONE`), never the server's,
   never the viewer's. It decides where today, a month, a report range starts and ends.
3. **The viewer's timezone is display only.** The browser and the phone convert UTC to local by themselves.
4. **Clients send unambiguous values:** an instant with `Z` or an offset (`2026-11-04T10:00:00.000Z`), or a
   bare date for a whole day (`2026-11-04`). Never a naive `2026-11-04 13:00:00`.
5. **A bare date means the whole business day; anything with a time is used exactly.** End date
   `2026-11-05` = to 23:59:59 on the 5th. A picked 1 pm to 3 pm stays 1 pm to 3 pm.
6. **One helper turns request dates into query bounds** (`App\Support\DateBounds`). No controller hand-rolls
   date math.

## 2. Check the column kind before you change anything

| Kind | Example | Compare with | Group by day with |
| --- | --- | --- | --- |
| UTC instant (datetime/timestamp) | `order_date`, `checkin_date`, `created_at` | UTC instants | the business day: `DATE(CONVERT_TZ(col,'+00:00','+03:00'))` |
| Calendar date (a day, no time) | an ERP posting date `DATE` column; days off stored `00:00:00` | the business calendar date (`Y-m-d`) | `DATE(col)` (already a business day) |

**Measure, do not assume:** compare a client-sent column with `created_at` on the same rows, and look at
the time-of-day distribution of a "date" column. Durations and before/after between two real instants are
timezone-free: leave them alone.

## 3. The traps (each one was live in production)

1. **Laravel does not convert a Carbon's timezone when binding.** `Connection::prepareBindings()` sends
   `$carbon->format('Y-m-d H:i:s')`. `Carbon::now('Africa/Kampala')->startOfMonth()` reaches MySQL as
   `2026-11-01 00:00:00` and is compared with UTC: 3 hours late. Wrap it: `->copy()->utc()`.
2. **An end date parsed as an instant is the start of that day.** `Carbon::parse('2026-11-05')` = 00:00 on
   the 5th: the whole last day is dropped. Every "month to date" link ends on today, so today vanishes and
   lists look empty early in the month.
3. **`DATE(col)`, `whereDate`, `whereYear`, `whereMonth` give the UTC day/month/year.** 01:30 in Kampala on
   1 March is February in UTC. Use business-day SQL, or query macros (`whereBusinessDate` etc.).
4. **The same in PHP:** `Carbon::parse($row->order_date)->startOfDay()` / `->format('Y-m-d')` = the UTC day.
   `->setTimezone(business tz)` first.
5. **`new Date("2026-11-01")` in the browser is UTC midnight** (03:00 in Kampala): build
   `${day}T00:00:00+03:00` (and `T23:59:59.999` for an end).
6. **Date pickers that snap on every change throw the chosen time away.** Snap only when the date changes.
7. **MySQL `TIMESTAMP` columns are converted with the session timezone.** A machine whose MySQL runs on
   Kampala time reads every `created_at` 3 hours late. Pin the session in config, not per server: Laravel's
   mysql connection takes `'timezone' => '+00:00'`. Prove production already matches before you ship it: in
   a production dump, compare a TIMESTAMP with a DATETIME written at the same moment (`created_at` against
   `checkin_date`) under both session zones; equal under `+00:00` means production's session was UTC.
8. **Not every business-time value is a bug.** Calendar-date columns must be compared with the business
   date. Converting them to UTC is the bug.
9. **Caches outlive the fix.** Clear them on deploy.
10. **An uncast datetime column goes out without `Z`** and the browser reads it as local time. Compare
    `information_schema.columns` (datetime, timestamp) with each model's casts, all models in one script.
11. **`subMonths()` overflows** (Carbon's default): 31 Dec minus a month is 1 Dec, so a "last three months"
    chart read Oct, Dec, Dec on the 31st. Use `subMonthsNoOverflow()` unless the date is a `startOfMonth()`.
12. **One display format for timestamps**, from one helper (`DD/MM/YYYY hh:mm:ss A`), never
    `toLocaleString()`, whose format follows the browser's language.
13. **Why nobody noticed:** a 3-hour shift only touches the edges of a day or month (3 orders in 2,093 in
    one month), and when every report shares it they agree with each other. Cross-checking two screens
    shows nothing; checking against the raw database does.

## 4. How to check a whole app

1. **Snapshot the reports first** (golden master: save each report's JSON, compare after a change).
2. **A guard test with `DB::pretend()` (seconds, runs in CI forever).** Call every GET route without route
   parameters, with each date style the screens send, inside `DB::pretend(fn () => ...)`, and hook
   `DB::connection()->beforeExecuting(fn ($sql, $bindings) => ...)`. Pretend builds every query and hands the
   SQL and bindings to the hook but executes nothing, so it needs no schema, no data, and takes ~2 seconds
   for ~150 routes. Fail on:
   - a `DateTimeInterface` binding whose offset is not 0, or a string ending `00:00:00`/`23:59:59`, against a
     UTC column (read the column from the SQL in front of the placeholder);
   - a raw ISO string (`/^\d{4}-\d{2}-\d{2}T/`) in any binding: a request value handed to SQL unparsed;
   - `whereDate`/`whereYear`/`whereMonth` on a UTC column (in SQLite they compile to
     `strftime('%Y-%m-%d', "col") >= cast(? as text)`);
   - a UTC Carbon at exactly `00:00:00`/`23:59:59` against a UTC column: the UTC day
     (`Carbon::parse('2026-07-15')`, `now()->startOfDay()`, `Carbon::create(2026, 7, 1)`). A business day in
     UTC+3 starts at `21:00:00` UTC, so a UTC midnight bound is almost never right. This one rule found a
     financial-year card and a map's one-day window that every other check had passed.
   Allow only named calendar-date columns, table-qualified when the name is generic (`date` only on the
   days-off table: another table had a datetime called `date`). Use `withoutMiddleware()`,
   `Sanctum::actingAs(new User)`, `Http::preventStrayRequests()`, `Cache::flush()` per call, and assert that
   most routes built at least one query, so the guard cannot pass by reaching nothing.
   **Prove the guard:** put the old code back (`git show HEAD:file > file`), run it, see it name each bug,
   restore.
3. **The same hook on real data (~1 hour on a production copy), before a release:** it reaches the
   queries that only run per row, which pretend cannot (pretend returns no rows).
4. **Send every required parameter.** A route that answers 422 builds no query and is silently skipped:
   NICE's maps and activity reports require `date_type`, and the first detector runs never reached them.
5. **Read your own diff tool's output critically.** `$actual[$path] ?? '(missing)'` reports every null as
   missing: thousands of false "changes" on NICE's maps until it used `array_key_exists`. And a report
   compared hours after its baseline moves by itself when its windows are relative to now.
6. **Static sweeps** (no runtime check sees these). Start with `Carbon::(now|today|create|parse)\(` with no
   timezone argument (the default "today" and rolling windows hide there), then:
   `(DATE|YEAR|MONTH|DATE_FORMAT)\(col` in raw SQL; `where(Date|Year|Month)\(`;
   `Carbon::parse($x)->startOfDay|endOfDay|format('Y-m-d')` with no timezone argument (a UTC Carbon snapped
   to the UTC day looks right when bound); `substr((string) $x->col, 0, 10)`.
7. **Writes:** find every `Carbon::now($businessTz)` in a file that also creates/updates; prove it never
   reaches a datetime column.
8. **Uncast columns:** one script over every model: the `datetime`/`timestamp` columns from
   `information_schema.columns` that `$model->hasCast()` and `getDates()` do not cover. Each one goes to the
   browser without `Z` and shows 3 hours early (NICE: `failed_jobs.failed_at`, the queues screen).
9. **The database session:** `select @@session.time_zone, @@system_time_zone`. Anything but UTC shifts every
   TIMESTAMP column; pin it in the connection config (`'timezone' => '+00:00'`).
10. **Tally, do not trust:** pin the clock to a day that has data (`Carbon::setTestNow`), call each dashboard
    figure and the list its link opens with the exact parameters the screen sends (a bare day in the link
    becomes `00:00:00.000` / `23:59:59.999` business time as ISO), and compare. Do it on the 1st, a middle day
    and the last day with data. Then tally cards against each other (the Yearly card's current year = the
    Monthly card's months; pin the clock AFTER the last data, or a "whole month" card counts days a "to
    today" card does not). NICE: 33 of 33 matched, and the Monthly/Yearly tally found the Yearly SPLY bug.
11. **Every date edge in a test:** the 31st (month overflow), 1 July (a financial year), 29 February.
12. Fix, then rerun until only calendar-date columns remain, the snapshots move only at the edges, and the
    tests pass on both MySQL and the SQLite test database (`CONVERT_TZ` is MySQL-only: give the helper a
    SQLite branch).

## 5. Mistakes the agent made on NICE (do not repeat them)

Each of these cost a round of the user's time. They are about method, not dates.

1. **Claiming "none left" from a partial search.** "0 `whereDate` remain" was true only for the files the
   sweep had touched. Search the WHOLE `app/` before saying none, and show the command.
2. **A check that passes because it reached nothing.** The first detector never reached 8 endpoints: they
   answered 422 (a required `date_type`) or 500 (a `data_limit` read with no default). Count the routes that
   built queries and fail when most did not; send every required parameter.
3. **An allow-list wider than the truth.** `date` was allowed as a calendar column everywhere; one table's
   `date` was a datetime. Table-qualify generic names, and check the migration and the data.
4. **Excluding routes by name** ("skip anything with sync/export") hid real lists. Exclude nothing you have
   not read.
5. **Trusting a recorded decision was implemented.** The 16/10 docs said every card's SPLY was "same dates
   last year"; the Yearly card never changed. Read the code behind a claim; a tally (section 4, step 10)
   catches it.
6. **Misreading tool output.** `| tail` makes `$?` the exit code of tail, not of `tsc`: capture
   `cmd > out.txt; echo $?`. `??` in a diff script turned nulls into "missing". A stale `.actual.json` from an
   earlier run looked like a fresh result. Read raw output before believing a summary.
7. **Shell quoting on Windows.** Bash heredocs through the agent's tool stripped `\\` from PHP namespaces and
   regexes, and an apostrophe in a comment broke `node -e '...'`. Write scripts to a file with the editor
   tool and run the file.
8. **Bulk edits that change line endings.** Splitting on `\n` and inserting a line left CRLF files mixed.
   Detect the file's ending and keep it; check with `file <path>`.
9. **A negative test assertion that was wrong.** "SPLY must not be last year's whole window" failed on
   correct code, because that window is also the previous year's Actual. Assert what distinguishes old
   from new (here: how many times the window is queried), and prove the test by putting the old code back.
10. **New UI that did not match the existing UI.** A custom hover came out as a dark box in light mode, unlike
    the Highcharts hover beside it. Copy the look of the closest existing component (colours, border, light
    and dark), and say plainly when a screen change has not been seen in a browser.
11. **"Lint clean" without a linter.** `next lint` is gone in Next 16 and the repo had no ESLint config, so
    there was nothing to run; say so, and check unused imports another way, instead of reporting it clean.

## 6. Ship it with docs

- The project's timezone guide: the rules, the column table, the helpers, the traps.
- A contract for the mobile app: send `Z` instants or bare dates, parse UTC, group by the business day.
- A deploy note: `cache:clear` after any date-logic change.

- A display convention for the web: one timestamp format from one helper (`DD/MM/YYYY hh:mm:ss A`).

NICE's versions: `nice-laravel-api-pase-two/docs/timezone-handling-guide.md` (sections 11 to 14),
`docs/dates-and-times-for-the-mobile-app.md`, and the web's `nice-next-js-phase-two/docs/display-conventions.md`.
