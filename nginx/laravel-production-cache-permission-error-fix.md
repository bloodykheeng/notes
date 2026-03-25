# Fixing Laravel `file_put_contents(...storage/framework/cache...)` Error (Production Server)

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
