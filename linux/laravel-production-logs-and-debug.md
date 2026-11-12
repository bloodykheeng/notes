# Laravel on a live server: debug off, daily logs, and only real errors in them

Three settings that decide whether a production log is useful, and how to fix them on a server that is already live
without breaking anything. Learnt on NICE (12/11/2026): `APP_DEBUG=true` on production, one `laravel.log` of 89 MB
that never rotated, and every refused check-in written into it as an ERROR with a full stack trace, so the one real
error of the day was buried. All three were fixed in one small deploy.

## 1. What to aim for

| Setting | Production | Why |
| --- | --- | --- |
| `APP_DEBUG` | `false` | With `true`, any server error shows its internals (stack trace, request, and on some setups config values) to whoever triggered it. With `false` the user gets "Server Error" and the details still go to the log. |
| `LOG_CHANNEL` | `stack` | The default; the stack sends to the channels listed in `LOG_STACK`. |
| `LOG_STACK` | `daily` | One file per day (`storage/logs/laravel-YYYY-MM-DD.log`) instead of one `laravel.log` that grows forever. |
| `LOG_DAILY_DAYS` | `30` | Files older than this are deleted by Laravel itself. No `logrotate` needed for the app log. |
| `LOG_LEVEL` | `warning` (new servers) | Keeps info/debug noise out. On an existing server, check nothing relies on `info` lines (NICE's order renumber migration wrote `info` lines we wanted to read) before raising it. |

These are read in `config/logging.php` (`env('LOG_STACK', 'single')`, `env('LOG_DAILY_DAYS', 14)`) and
`config/app.php` (`env('APP_DEBUG', false)`): check the project's own config files say the same before relying on
the names.

A **test** server may keep `APP_DEBUG=true` if the team wants the details on screen while testing, but it is still on
the internet: decide it, do not inherit it.

## 2. Business refusals are not errors

An exception that means "the rule worked" (a check-in outside the radius, a closed visit, a limit reached) should
answer the client with its 422 and message, and **not** be written to the log:

```php
use Illuminate\Contracts\Debug\ShouldntReport;

final class OutsideGeofenceException extends Exception implements ShouldntReport
{
    public function render(Request $request): JsonResponse { /* 422 with message, errors, details */ }
}
```

- **`report()` returning `false` does NOT do this.** In Laravel, `false` from `report()` means "use the default
  reporting", so the exception is still logged. Use `ShouldntReport` (or `$dontReport` / `dontReport()` in
  `bootstrap/app.php`).
- Prove it with a test that fails without the interface:

```php
it('refuses with a 422 and does not report it', function () {
    Exceptions::fake();
    Route::post('/_test/refuse', fn () => throw new OutsideGeofenceException(/* ... */));
    $this->postJson('/_test/refuse')->assertStatus(422)->assertJsonPath('message', '...');
    Exceptions::assertNotReported(OutsideGeofenceException::class);
});
```

- The audit trail (if the app has one) still records the refused action; the log is for faults.

## 3. Fixing a live server (one box, run as root)

Change one line each, back the file up first, print before and after, then confirm what Laravel actually reads.
Never open the `.env` in an editor over a slow link for this, and never print the whole file (it holds passwords).

```bash
cd /var/www/<api-folder>
echo "--- before"; grep -nE '^(APP_DEBUG|LOG_CHANNEL|LOG_STACK|LOG_DAILY_DAYS)=' .env
cp .env /root/db-backups/<db>/env-before-debug-and-logs-$(date +%F-%H%M)
grep -q '^APP_DEBUG=' .env && sed -i -E 's/^APP_DEBUG=.*/APP_DEBUG=false/' .env || printf '\nAPP_DEBUG=false\n' >> .env
grep -q '^LOG_STACK=' .env && sed -i -E 's/^LOG_STACK=.*/LOG_STACK=daily/' .env || printf '\nLOG_STACK=daily\n' >> .env
grep -q '^LOG_DAILY_DAYS=' .env && sed -i -E 's/^LOG_DAILY_DAYS=.*/LOG_DAILY_DAYS=30/' .env || printf 'LOG_DAILY_DAYS=30\n' >> .env
echo "--- after"; grep -nE '^(APP_DEBUG|LOG_CHANNEL|LOG_STACK|LOG_DAILY_DAYS)=' .env
sudo -u www-data php artisan optimize:clear
sudo -u www-data php artisan config:show app.debug
sudo -u www-data php artisan config:show logging.default
sudo -u www-data php artisan config:show logging.channels.stack
chown -R www-data:www-data storage bootstrap/cache
systemctl reload php8.4-fpm
supervisorctl restart '<api-worker-group>:*'
```

Read the output:
- "after" shows each line **once** with the new value (the `grep -q ... ||` only appends when the key is missing; the
  `printf '\n...'` starts on a new line in case the file has no final newline).
- `app.debug` reads `false`; `logging.default` reads `stack` and its `channels` read `daily`. If `logging.default` is
  something else (`single`, `daily` directly), `LOG_STACK` has no effect: set `LOG_CHANNEL=stack` too.
- Restart **that** API's workers (they hold the old config in memory); never `supervisorctl restart all` on a box with
  two environments.
- The old `laravel.log` stays where it is. Archive it when convenient (`gzip -c laravel.log > /root/...` then
  `: > laravel.log`), never delete it while someone is reading it.

## 4. Reading daily logs

```bash
cd /var/www/<api-folder>
ls -lt storage/logs | head -5                                    # every file owned by www-data
tail -n 30 storage/logs/laravel-$(date -u +%F).log                # today's file (the server runs in UTC)
grep -E '^\[' storage/logs/laravel-$(date -u +%F).log | cut -c1-300 | tail -15   # entry headers only, no stack frames
grep -cE '^\[2026-11-12 13:(1[7-9]|[2-5][0-9])' storage/logs/laravel-2026-11-12.log  # how many since a moment
```

- A `tail` of a long stack trace shows only its bottom: read the **header** lines (`^\[`) to see time and message.
- An entry's `request_id` (if the app adds one) finds the same request in the audit trail.
- After a deploy, count the entries since the deploy and read each kind: expected lines (a migration's `info`), rules
  working (refusals, if still reported), and real faults. Only the last ones need work.

## 5. In the deploy docs

Put the three values in the project's env checklist, the reading commands in its redeploy checks, and the fact that
the log file name changed (`laravel.log` to `laravel-YYYY-MM-DD.log`) wherever a doc says `tail laravel.log`.
