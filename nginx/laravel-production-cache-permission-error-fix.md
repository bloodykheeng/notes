# Laravel storage-permission failures on a production server (root vs www-data)

Two symptoms, one root cause: something ran as **root** and left files under `storage/` that
**www-data** can no longer write.

| Symptom | Where it shows |
| --- | --- |
| `file_put_contents(.../storage/framework/cache/...): Failed to open stream` | in the response / log |
| **Bare HTTP 500, empty response body, nothing written to `laravel.log`** | only in the nginx access log |

The second one is the nastier of the two and is covered in
[Silent 500 with an empty log](#silent-500-with-an-empty-log) below.

---

## Problem

While running a Laravel application in production, the following error appeared:

```bash
file_put_contents(/var/www/niceprodapi/storage/framework/cache/data/15/b3/15b392c6b2cb6177e66adcf317fc7472b9a9a0ed):
Failed to open stream: No such file or directory
```

This error happens intermittently and usually returns after clearing cache temporarily.

---

## Root Cause

The issue occurs when Laravel attempts to write cache/session files but **does not have permission to create the required directory structure**.

Typical reasons include:

* Laravel scheduler (`php artisan schedule:run`) running as **root**
* PHP-FPM running as **www-data**
* Cache/session directories being created by **root**
* Later write attempts failing when Laravel runs under **www-data**

Example conflict:

```
cron (root) → creates cache folders
php-fpm (www-data) → cannot write to them
```

This results in:

```
file_put_contents(...): Failed to open stream
```

---

## How the Issue Was Identified

Checking cron jobs revealed the scheduler was running under `root`:

```bash
crontab -l
```

Output:

```bash
* * * * * cd /var/www/niceapi && php artisan schedule:run >> /dev/null 2>&1
* * * * * cd /var/www/niceprodapi && php artisan schedule:run >> /dev/null 2>&1
```

But PHP-FPM runs as:

```bash
ps aux | grep php-fpm
```

Output included:

```
www-data php-fpm: pool www
```

This confirmed ownership mismatch.

---

## Permanent Solution

### Step 1: Remove incorrect cron jobs (running as root)

Edit root cron:

```bash
crontab -e
```

Remove:

```bash
* * * * * cd /var/www/niceapi && php artisan schedule:run >> /dev/null 2>&1
* * * * * cd /var/www/niceprodapi && php artisan schedule:run >> /dev/null 2>&1
```

Then check system cron:

```bash
sudo crontab -e
```

Remove the same entries if present.

---

### Step 2: Add scheduler under correct user (`www-data`)

Run:

```bash
sudo crontab -u www-data -e
```

Add:

```bash
* * * * * cd /var/www/niceapi && php artisan schedule:run >> /dev/null 2>&1
* * * * * cd /var/www/niceprodapi && php artisan schedule:run >> /dev/null 2>&1
```

Save and exit.

---

### Step 3: Restore correct ownership

Run:

```bash
sudo chown -R www-data:www-data /var/www/niceapi/vendor
sudo chown -R www-data:www-data /var/www/niceapi/storage
sudo chown -R www-data:www-data /var/www/niceapi/bootstrap/cache

sudo chown -R www-data:www-data /var/www/niceprodapi/vendor
sudo chown -R www-data:www-data /var/www/niceprodapi/storage
sudo chown -R www-data:www-data /var/www/niceprodapi/bootstrap/cache
```

---

### Step 4: Restore permissions

Run:

```bash
sudo chmod -R 775 /var/www/niceapi/storage
sudo chmod -R 775 /var/www/niceapi/bootstrap/cache

sudo chmod -R 775 /var/www/niceprodapi/storage
sudo chmod -R 775 /var/www/niceprodapi/bootstrap/cache
```

---

## Why This Fix Works Permanently

Laravel writes runtime files into:

```
storage/
bootstrap/cache/
```

These include:

* session files
* cache files
* compiled views
* scheduler locks
* queue metadata

If these folders are created by **root**, then later requests handled by **www-data** fail.

Correct configuration:

```
cron scheduler → runs as www-data
php-fpm → runs as www-data
filesystem owner → www-data
```

Now all Laravel processes share the same permission context.

This prevents directory recreation conflicts permanently.

---

## Verification Steps

Check scheduler ownership:

```bash
sudo crontab -u www-data -l
```

Expected output:

```bash
* * * * * cd /var/www/niceapi && php artisan schedule:run >> /dev/null 2>&1
* * * * * cd /var/www/niceprodapi && php artisan schedule:run >> /dev/null 2>&1
```

Check root scheduler is empty:

```bash
crontab -l
```

and

```bash
sudo crontab -l
```

Check directory ownership:

```bash
ls -l storage/framework/cache
```

Expected:

```
www-data www-data
```

---

## Silent 500 with an empty log

### Symptom

A write endpoint (an approval, a form submit) returns **500** in about a second. The browser
shows only a generic network/CORS failure because the response body is empty, and
`storage/logs/laravel.log` has **no entry for it at all**: its newest lines are days old,
from the last commands you ran by hand.

### Cause

`laravel.log` is owned by root:

```bash
ls -la storage/logs/
-rw-r--r--  1 root     root  23772 Sep 15 13:11 laravel.log
-rw-r--r--  1 www-data root     14 Sep 14 18:15 .gitignore
```

The file was created (or last written) by an artisan command run as **root**, typically
`php artisan migrate` during deployment. PHP-FPM runs as **www-data**, which now has read-only
access. The moment a request writes any log line, Monolog throws
`failed to open stream: Permission denied` **inside the exception handler**, so Laravel cannot
render an error page and cannot record what happened. You get a naked 500.

GET endpoints keep working (they log nothing), which makes it look like a bug in one feature.

### Diagnosis

```bash
cd /var/www/<your-app>

# 1. Is there any entry at all for the failing request's timestamp?
grep -a "production.ERROR" storage/logs/*.log | tail -5 | cut -c1-500

# 2. What status did the request actually get, and did it reach PHP?
sudo grep -a "<the endpoint path>" /var/log/nginx/access.log | tail -10

# 3. Who owns the log, and can the web user write it?
ls -la storage/logs/
sudo -u www-data touch storage/logs/laravel.log && echo "www-data CAN write" || echo "www-data CANNOT write"

# 4. Rule out the queue as the source
php artisan queue:failed | tail -20
```

A `500` in the access log plus **no** matching line in `laravel.log` is the signature. (A `504`
or `499` there would mean a timeout instead, a different problem.)

### Fix

```bash
chown -R www-data storage
chown -R www-data bootstrap/cache
ls -la storage/logs/          # confirm both .log files now show www-data
```

Retry the request; it succeeds, and any genuine error is now recorded properly.

### Prevention

Run artisan as the web user so ownership never flips back:

```bash
sudo -u www-data php artisan migrate --force
sudo -u www-data php artisan config:cache
sudo -u www-data php artisan optimize
```

The Laravel deployment docs only require that "the web server process owner has permission to
write to `bootstrap/cache` and `storage`", and they do not say which user runs artisan, which is
exactly why this keeps happening. Laravel Forge avoids it by running PHP-FPM, the queue workers
and the deploys all as the **same** non-root user.

If you prefer to keep working as root, make the directories self-correcting instead:

```bash
chown -R www-data:www-data storage bootstrap/cache
chmod -R ug+rwX storage bootstrap/cache
find storage bootstrap/cache -type d -exec chmod g+s {} \;   # new files inherit the group
setfacl -R -d -m g:www-data:rwx storage bootstrap/cache       # default ACL for new files
```

> Do not forget the other long-running services: `storage/logs/reverb.log` and the worker log
> are written by www-data too, so a root-owned copy of either silences that service's logging
> in the same way.

---

## Reference Source (Where Solution Idea Came From)

Discussion that helped identify the issue:

Laracasts thread:

**file_put_contents(...): failed to open stream: No such file or directory – Always this problem**

[https://laracasts.com/discuss/channels/general-discussion/file-put-contents-failed-to-open-stream-no-such-file-or-directory-always-this-problem](https://laracasts.com/discuss/channels/general-discussion/file-put-contents-failed-to-open-stream-no-such-file-or-directory-always-this-problem)

Key discovery from the thread:

> Task scheduler was running as root instead of www-data

Which causes Laravel cache/session directory ownership conflicts.

---

## Additional Recommendation (Best Practice for Laravel Production Servers)

Always ensure these run as **www-data**:

* cron scheduler
* queue workers
* Horizon (if used)
* deploy scripts writing cache
* artisan commands triggered automatically

Example correct scheduler:

```bash
sudo crontab -u www-data -e
```

This prevents recurring filesystem permission failures in production environments.
