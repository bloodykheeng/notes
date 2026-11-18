> **For:** Alpha (`D:\coding\new wave\CPA accounts system\alpha-fund-system-workspace`)  
> **From:** Elevate Pesa (`D:\coding\bloodykheeng projects\elevate_pesa\elevate-pesa-workspace`), written 14/11/2026  
> **Status:** Open: not applied yet.  
> Inbox rules: [../README.md](../README.md). When done, move this file into the project (for history) and
> delete it here.

# For the Alpha project: pin the MySQL session to UTC (from Elevate Pesa, 14 Nov 2026)

_Written from the Elevate Pesa workspace for Alpha's agent. Alpha's code was checked read-only on 14/11/2026. Do the
read-only checks first, then the fix, then mark this file **Resolved** (date, commit, tests). Elevate's code is at
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
- **DST zones reject some writes.** On a Berlin (or any DST) session, a valid UTC time inside the spring-forward hour,
  e.g. `2027-03-28 02:30:00`, does not exist on the local clock, so strict MySQL rejects it with `Incorrect datetime
  value`.

`DATETIME` columns are never converted, so `dateTime()` business columns are safe. Only `TIMESTAMP` columns
(`timestamps()`, `timestamp()`, `softDeletes()`, `useCurrent()`) are affected.

**The house standard since 14/11/2026:** every Laravel + MySQL/MariaDB project pins the session to UTC:

```php
// config/database.php, on BOTH the 'mysql' and the 'mariadb' connections
'timezone' => env('DB_TIMEZONE', '+00:00'),
```

Laravel declined this as a framework default (laravel/laravel#6673, closed unmerged), so each project adds it.

## 2. Where Alpha stands (checked 14/11/2026)

- `alpha-fund-laravel/config/database.php`: no `'timezone'` on `mysql` (line 47) or `mariadb`.
- `alpha-fund-laravel/docs/timezone-handling.md`: the UTC / Kampala-business-day rules are right; it does not mention
  the MySQL session.
- `useCurrent()`: `failed_jobs.failed_at` (`0001_01_01_000002_create_jobs_table.php`).
- **Local laptop DB (`alpha_fund_laravel`): rows were written under a Kampala (+3) session.** Proven: 136 audit rows
  whose recorded moment equals `created_at` match when read under the default session and none match under
  `+00:00`. So the moment the pin goes in, the laptop shows every existing Laravel-written `TIMESTAMP` **3 hours
  early** until they are moved (section 4).
- Servers (`docs/deployment.md`): production + test on the Hostinger VPS `187.7.23.63`, staging on New Wave's Linode
  `80.85.84.128`. **Their zones are not known yet.** If a server's MySQL is on UTC, its data is already right and the
  pin changes nothing there. If it is on anything else, its rows need the same move.

## 3. Read-only checks first (do not change anything yet)

On each server (production, test, staging) and on the laptop:

```bash
timedatectl | grep 'Time zone'
mysql -e "SELECT @@system_time_zone, @@global.time_zone, NOW(), UTC_TIMESTAMP();"
```

`@@global.time_zone` = `SYSTEM` means MySQL follows the OS zone. If `NOW()` equals `UTC_TIMESTAMP()`, that database
is on UTC and already right.

For a database that is not on UTC, confirm with one real row: create something in the app now, then read its
`created_at` with the session left alone and with `SET time_zone='+00:00'`. The read that equals the real UTC time of
the click tells you how the existing rows were written. Also ask whether the database was ever moved from another
server (Alpha's incident file `INCIDENT-2026-11-10-hostinger-outage.md`): rows written on a server in another zone
carry that server's offset.

## 4. The fix

1. **Back up** each database you will touch (`docs/deployment.md` has the `mysqldump` line).
2. **Add the pin** to both connections in `config/database.php` (section 1), with a comment pointing to
   `docs/timezone-handling.md`.
3. **Test it** (copy of Elevate's `E:\pesa_back\tests\Feature\Admin\TimezonesTest.php`):

   ```php
   public function test_the_database_session_is_utc_whatever_the_server_zone(): void
   {
       $this->assertSame('+00:00', DB::selectOne('select @@session.time_zone as z')->z);
   }
   ```

4. **Move the rows written under a non-UTC session, once per database**, in the same release as the pin:
   - Only `TIMESTAMP` columns that **Laravel wrote** (`created_at`, `updated_at`, `deleted_at`, other `timestamp()`
     columns the app fills). **Leave out** `useCurrent()` / `CURRENT_TIMESTAMP` columns (MySQL wrote those with the
     right instant; the pin makes them read right) and every `DATETIME` column.
   - List the columns from `information_schema.COLUMNS WHERE DATA_TYPE='timestamp' AND TABLE_SCHEMA=...`.
   - Kampala has no DST, so the laptop is a fixed +3:

     ```sql
     SET time_zone = '+00:00';
     UPDATE <table> SET created_at = created_at + INTERVAL 3 HOUR, updated_at = updated_at + INTERVAL 3 HOUR;
     ```

     A column with `ON UPDATE CURRENT_TIMESTAMP` would change on this UPDATE; name it in the SET too.
   - For a DST zone (Berlin etc.) the offset differs per row: convert per row (a small PHP script with Carbon, or
     `CONVERT_TZ` if the server has the zone tables).
   - The laptop alternative: re-import a fresh copy of production instead, if production is on UTC.
   - Check: a row you know (a recent transaction's real time) reads correctly afterwards.
5. **A server whose OS is not on UTC:** pin anyway. Setting the OS to UTC (`timedatectl set-timezone UTC`) is a
   separate production change for the owner to approve.
6. **Docs:** add a "MySQL session" section to `docs/timezone-handling.md` (what, why, the Contabo story, the move
   done), and a line in `docs/deployment.md` (each server's zone, the pin).

## 5. More detail

- Elevate's write-up with the server proof and deploy steps: `E:\pesa_back\docs\timezones.md`, section "The MySQL
  session is pinned to UTC (the standard)".
- Coding notes: `D:\coding\bloodykheeng projects\coding-notes\ai-agents\dates-and-timezones-with-an-ai-agent.md`,
  trap 7 and check 9; and `ai-agents\deploy-yourself-over-ssh.md` (check `timedatectl` in the first server round).

## Status

- [ ] Read-only checks done (laptop, production, test, staging zones written here)
- [ ] Pin + test committed
- [ ] Rows moved where needed
- [ ] Docs updated
