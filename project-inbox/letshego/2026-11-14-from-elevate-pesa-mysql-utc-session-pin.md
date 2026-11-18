> **For:** Letshego (`D:\coding\new wave\letshego\letshego-workspace`)  
> **From:** Elevate Pesa (`D:\coding\bloodykheeng projects\elevate_pesa\elevate-pesa-workspace`), written 14/11/2026  
> **Status:** Resolved in Letshego on 16/11/2026 (see the note itself): only needs moving into the Letshego project for history.  
> Inbox rules: [../README.md](../README.md). When done, move this file into the project (for history) and
> delete it here.

# For the Letshego project: pin the MySQL session to UTC (from Elevate Pesa, 14 Nov 2026)

> **RESOLVED in Letshego, 16/11/2026** (all but the servers, which do not exist yet). The note held: the laptop's
> MySQL reported `@@system_time_zone` = E. Africa Standard Time (+3) and `config/database.php` had no `'timezone'`.
>
> - **Pin:** `'timezone' => env('DB_TIMEZONE', '+00:00')` on both `mysql` and `mariadb` in
>   `letshego-laravel/config/database.php`, with a comment pointing to the doc. Checked after: session `+00:00`,
>   `NOW()` = `UTC_TIMESTAMP()`.
> - **Test:** `letshego-laravel/tests/Feature/Api/TimezonesTest.php` (2: the session is `+00:00`; `NOW()` equals
>   `UTC_TIMESTAMP()` and a fresh `created_at` matches `now()`). Full suite 223 passed.
> - **Local rows:** reseeded (`migrate:fresh --seed` plus the demo seeder): dev data only.
> - **Docs:** `letshego-laravel/docs/api-conventions/timezone-handling.md` "The MySQL session is pinned to UTC";
>   `docs/platform/environment-setup.md` §8 "Servers: checks when they are built" (the `timedatectl` step);
>   workspace `docs/decisions.md` (16 Nov 2026).
> - **Commit:** `7331b54` "MySQL session pinned to UTC" on branch `farouk` of `letshegoapi` (pushed 16/11/2026).
> - **Open:** test and production servers built on UTC, when they come (Letshego's §8 checklist carries it).

_Written from the Elevate Pesa workspace for Letshego's agent. Letshego's code was checked read-only on 14/11/2026.
Do the checks, then the fix, then mark this file **Resolved** (date, commit, tests). Elevate's code is at
`D:\coding\bloodykheeng projects\elevate_pesa\elevate-pesa-workspace` (`E:` below)._

## 1. The problem in one paragraph

Laravel keeps PHP on UTC (`config/app.php` `'timezone' => 'UTC'`) and writes `created_at` as UTC text such as
`2026-11-14 12:00:00`. But `created_at`/`updated_at` are MySQL `TIMESTAMP` columns, and MySQL reads that text in
the **connection's session zone**, which defaults to the MySQL server's zone. Laravel only sets the session zone
when the connection config has a `'timezone'` key (`Illuminate\Database\Connectors\MySqlConnector`), and the stock
`config/database.php` has none. So on any MySQL that is not on UTC:

- **`useCurrent()` / `CURRENT_TIMESTAMP` columns read wrong, with no move.** Proven on this laptop (MySQL on
  Kampala, +3): in the same row, `created_at` (written by Laravel) read `15:43:58`, the real UTC time, while
  `failed_at` (`DEFAULT CURRENT_TIMESTAMP`, like Laravel's own `failed_jobs.failed_at`) read `18:43:58`, three hours
  wrong. With the session pinned to `+00:00` both read `15:43:58`.
- **Moving the database to a server on another zone shifts every Laravel-written `TIMESTAMP`.** `mysqldump` keeps
  the instant MySQL stored (`--tz-utc`, on by default), and that instant was wrong. Elevate Pesa moved from Hostinger
  (UTC) to a Contabo VPS whose OS and MySQL run on `Europe/Berlin`: the live site now shows every moved row 1 hour
  late (2 hours for summer dates). Checked over SSH on the real server.
- **DST zones reject some writes.** On a Berlin (or any DST) session, a valid UTC time inside the spring-forward hour
  does not exist on the local clock, so strict MySQL rejects it with `Incorrect datetime value`.

`DATETIME` columns are never converted. Only `TIMESTAMP` columns (`timestamps()`, `timestamp()`, `softDeletes()`,
`useCurrent()`) are affected.

**The house standard since 14/11/2026:** every Laravel + MySQL/MariaDB project pins the session to UTC:

```php
// config/database.php, on BOTH the 'mysql' and the 'mariadb' connections
'timezone' => env('DB_TIMEZONE', '+00:00'),
```

Laravel declined this as a framework default (laravel/laravel#6673, closed unmerged), so each project adds it.

## 2. Where Letshego stands (checked 14/11/2026)

- `letshego-laravel/config/database.php`: no `'timezone'` on `mysql` or `mariadb`.
- `letshego-laravel/docs/api-conventions/timezone-handling.md`: the UTC / Kampala-business-day rules are right; it does not mention
  the MySQL session.
- `useCurrent()`: `failed_jobs.failed_at` (`0001_01_01_000002_create_jobs_table.php`).
- **No server yet** (`WORK-PLAN.md`: the test environment waits for domains and a server; production 4 Dec). This is
  the cheapest moment: no production data to move.
- **Local laptop DB (`letshego`):** this laptop's MySQL runs on Kampala (+3), so rows already there were written under
  a +3 session. With the pin they read 3 hours early until moved or reseeded.

## 3. The fix

1. **Check the laptop** (read-only): `SELECT @@system_time_zone, @@global.time_zone, NOW(), UTC_TIMESTAMP();`.
2. **Add the pin** to both connections in `config/database.php` (section 1), with a comment pointing to
   `docs/api-conventions/timezone-handling.md`.
3. **Test it** (copy of Elevate's `E:\pesa_back\tests\Feature\Admin\TimezonesTest.php`):

   ```php
   public function test_the_database_session_is_utc_whatever_the_server_zone(): void
   {
       $this->assertSame('+00:00', DB::selectOne('select @@session.time_zone as z')->z);
   }
   ```

4. **Local rows:** simplest is `php artisan migrate:fresh --seed` if the local data is only seed/test data. To keep
   it instead, back it up and move the Laravel-written `TIMESTAMP` columns once (Kampala has no DST, fixed +3):

   ```sql
   SET time_zone = '+00:00';
   UPDATE <table> SET created_at = created_at + INTERVAL 3 HOUR, updated_at = updated_at + INTERVAL 3 HOUR;
   ```

   Leave out `useCurrent()` / `CURRENT_TIMESTAMP` columns and every `DATETIME` column. List the columns from
   `information_schema.COLUMNS WHERE DATA_TYPE='timestamp'`.
5. **When the servers are built** (test, then production): check `timedatectl` in the first read-only round and set
   the box to UTC before installing MySQL (`timedatectl set-timezone UTC`). Contabo images default to
   `Europe/Berlin`. Keep the pin anyway. Write each server's zone in the deployment doc.
6. **Docs:** add a "MySQL session" section to `docs/api-conventions/timezone-handling.md`, and the server-zone step to the deployment
   doc when it is written.

## 4. More detail

- Elevate's write-up with the server proof: `E:\pesa_back\docs\timezones.md`, section "The MySQL session is pinned to
  UTC (the standard)".
- Coding notes: `D:\coding\bloodykheeng projects\coding-notes\ai-agents\dates-and-timezones-with-an-ai-agent.md`,
  trap 7 and check 9; and `ai-agents\deploy-yourself-over-ssh.md` (the timezone in the first server round).

## Status

- [x] Pin + test committed (16/11/2026, `7331b54` on `farouk`)
- [x] Local rows moved or reseeded (reseeded, 16/11/2026)
- [x] Docs updated (16/11/2026)
- [ ] Test / production servers built on UTC (when they come; in Letshego's `environment-setup.md` §8)
