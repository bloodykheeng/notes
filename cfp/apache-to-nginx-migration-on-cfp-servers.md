# Apache → Nginx Migration & Production Incident Log

**Server:** `hqcfptestsvr` (Ubuntu 24.04 Noble)
**Date:** 25 August 2026
**Apps affected:** `oagapi` (Laravel API), `cfpwebapp` (Next.js), `oagdash` (React SPA)

---

## Table of contents

1. [Starting symptom](#1-starting-symptom)
2. [Root causes found](#2-root-causes-found-in-order-of-discovery)
3. [Fix walkthrough](#3-fix-walkthrough-step-by-step)
4. [Final nginx configs](#4-final-nginx-configs)
5. [SSL certificate setup](#5-ssl-certificate-setup)
6. [Frontend apps](#6-frontend-apps)
7. [Queue workers](#7-queue-workers)
8. [Application bugs found](#8-application-bugs-found)
9. [Diagnostic cheat sheet](#9-diagnostic-cheat-sheet)
10. [Outstanding items](#10-outstanding-items)

---

## 1. Starting symptom

`POST https://cfpapidemo.oag.go.ug/api/login` returned **500 Internal Server Error**.
Worked fine on localhost. The React frontend showed only a generic "Server Error" toast because
`auth-api.js` discards the response body.

**Key early insight:** the Laravel controller catches its own exceptions and returns the real
message inside the JSON body. Always read the **Network → Response** tab, not just the console.

`AuthController::login()` has two deliberate 500 paths:

| Location | Response |
| --- | --- |
| `AuthController.php:742-745` | LDAP bind failure → `"Whoops! There was an error connecting to the LDAP server."` |
| `AuthController.php:722` | catch-all → `"An error occurred while signig in"` + `error` field |

### The critical distinction used throughout

> **404** = the request never reached PHP.
> **500 with an empty body** = PHP died before Laravel booted.
> **500 with Laravel headers** (`Cache-Control: no-cache, private`, `Vary: Origin`) = Laravel booted and threw.

---

## 2. Root causes found (in order of discovery)

The server was mid-migration from Apache to Nginx. Apache had been purged
(`apt purge apache2 apache2-utils apache2-bin`), which silently removed **mod_php** — so nothing
was executing PHP at all. Every subsequent error was a layer underneath that.

| # | Problem | Real cause |
| --- | --- | --- |
| 1 | Nginx would not start | `CAP_DAC_OVERRIDE` stripped by an OAG systemd hardening drop-in |
| 2 | No PHP execution | `mod_php` removed with Apache; `php8.2-fpm` never installed |
| 3 | Blank 500, empty `laravel.log` | `www-data` could not read `vendor/` |
| 4 | `could not find driver` | `php8.2-mysql` (pdo_mysql) not installed |
| 5 | `Access denied for user 'user_oag'` | wrong `DB_PASSWORD` in `.env` |
| 6 | HTTPS dead | Apache's certbot SSL config removed with the purge |
| 7 | Next.js `EADDRINUSE :::3000` | an existing `cfpwebapp.service` systemd unit already owned the port |
| 8 | Next.js chunks 403 | `.next/` built as root, unreadable by `www-data` |
| 9 | Firebase 500 (earlier) | `env()` returning `null` under a cached config |

---

## 3. Fix walkthrough (step by step)

### Step 1 — Nginx would not start

`nginx -t` passed when run by hand, but failed under systemd:

```
[emerg] open() "/var/log/nginx/error.log" failed (13: Permission denied)
```

**Cause.** `/etc/systemd/system/nginx.service.d/oag-hardening.conf` contains:

```ini
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_SETUID CAP_SETGID
```

`CAP_DAC_OVERRIDE` is absent. That capability is what lets root ignore file ownership and
permission bits. Ubuntu ships `/var/log/nginx/*.log` as `www-data:adm 640`, and the nginx master
starts as root — not the owner, not in `adm`. Without the override it gets `EACCES`.
Running `nginx -t` manually kept the full root capability set, which is why the two behaved
differently.

> **Note:** `ProtectSystem=full` does *not* make `/var` read-only (that is `strict`), and
> `NoNewPrivileges=true` does *not* strip capabilities. Neither of those lines caused this.

**Fix** — make root the owner so no capability override is needed:

```bash
sudo chown root:adm /var/log/nginx/*.log /var/log/nginx
sudo systemctl reset-failed nginx
sudo systemctl start nginx
```

**Keep it fixed after rotation** (logrotate recreates as `www-data:adm`):

```bash
sudo sed -i 's/create 0640 www-data adm/create 0640 root adm/' /etc/logrotate.d/nginx
```

**Why the drop-in survived a fresh nginx install:** `/etc/systemd/system/nginx.service.d/` is the
systemd *administrator override* directory. It is not owned by the nginx package, so
`apt install/remove/purge nginx` never creates or deletes it.

**Two latent landmines still in that drop-in:**

- `MemoryDenyWriteExecute=true` — breaks PCRE JIT (`pcre_jit on;`) and any JIT module (njs, Lua).
  Expect segfaults if regex-heavy `location` blocks are added later.
- `ProtectHome=true` — `/home` is invisible to nginx. Fine while apps live under `/var/www`.

### Step 2 — PHP was gone

Purging Apache removed `mod_php`. Nginx has no built-in PHP handler.

```bash
sudo apt install php8.2-fpm -y
sudo systemctl enable --now php8.2-fpm
ls -l /run/php/            # confirm the socket path
```

Version 8.2 chosen to match `composer.json` (`"php": "^8.2"`). The unversioned `php-fpm`
meta-package would have pulled 8.3 on Noble and mismatched the existing install.

### Step 3 — `www-data` could not read the project

Symptom: 500 with an **empty body** and **nothing in `laravel.log`**.

```
PHP Warning:  require(/var/www/oagapi/vendor/composer/../laravel/prompts/src/helpers.php):
              Failed to open stream: Permission denied
PHP Fatal error:  Uncaught Error: Failed opening required ...
```

PHP died during Composer autoload, before Laravel could initialise its log handler — hence the
blank response and silent log.

```bash
cd /var/www/oagapi
sudo chown -R www-data:www-data .
sudo find . -type d -exec chmod 755 {} \;
sudo find . -type f -exec chmod 644 {} \;
sudo chmod -R 775 storage bootstrap/cache
sudo chmod 640 .env
```

What each part does:

- `chown -R www-data` — php-fpm runs as `www-data`; it must own or be able to read everything.
- `755` on dirs — without `x` on a directory, PHP cannot descend into it.
- `775` on `storage` + `bootstrap/cache` — these need **write**, not just read.
- `640` on `.env` — keeps DB password and Firebase private key off other users.

Do **not** `chmod -R 777` the project. That makes `.env` world-readable.

After this the 500 changed to carry Laravel headers (`Cache-Control: no-cache, private`,
`Vary: Origin`) — proof the app was finally booting.

### Step 4 — Missing PHP extensions

```
SQLSTATE: could not find driver (Connection: mysql, SQL: select * from `sessions` ...)
```

`session.driver` is `database`, and `StartSession` middleware runs on **every** request before
routing — so a DB failure took down the entire site, not just `/api/login`.

```bash
sudo apt install php8.2-mysql php8.2-mbstring php8.2-xml php8.2-curl \
                 php8.2-zip php8.2-gd php8.2-bcmath php8.2-intl php8.2-ldap -y
sudo systemctl restart php8.2-fpm
php -m | grep -E "pdo_mysql|ldap|curl|zip"
```

Why each is needed by this project:

| Extension | Required by |
| --- | --- |
| `pdo_mysql` | database sessions, all Eloquent |
| `ldap` | `directorytree/ldaprecord-laravel` (login falls through to LDAP) |
| `curl` | `kreait/firebase-php`, SMS gateway |
| `zip`, `gd` | `maatwebsite/excel` |
| `mbstring`, `xml`, `bcmath`, `intl` | Laravel framework baseline |

### Step 5 — Wrong DB password

```
SQLSTATE[HY000] [1045] Access denied for user 'user_oag'@'localhost' (using password: YES)
```

`DB_PASSWORD` in `.env` did not match the MySQL user.

```bash
grep -E "^DB_" .env
mysql -u user_oag -p -e "SELECT 1;"     # verify by hand
```

Corrected `.env`, then **cleared config — did not cache it**:

```bash
php artisan config:clear
sudo systemctl restart php8.2-fpm
```

Result: **`HTTP/1.1 200 OK`**.

> **Watch for special characters.** A `DB_PASSWORD` containing `#` or spaces must be quoted in
> `.env` or it gets truncated: `DB_PASSWORD="kX6fYxgXc?Mdapcc"`

### Step 6 — The `env()` / `config:cache` trap

Earlier in the incident, a Firebase exception fired *before* the controller ran:

```
#15 Application->make()                    ← container builds the controller
#2  App\Services\FirebaseService->__construct()
#1  Factory->createMessaging()
#0  Factory->getProjectId()                ← throws
```

`AuthController::__construct()` type-hints `FirebaseService`, so the container must build it
before `login()` can run — which is why the `try/catch` inside `login()` never caught it.

`FirebaseService.php:18-29` reads all eleven credentials via **`env()`**, not `config()`.

> **Rule:** once `php artisan config:cache` has run, Laravel stops loading `.env` at runtime and
> **every `env()` call outside a config file returns `null`.**

**Immediate fix:**

```bash
php artisan config:clear
php artisan cache:clear
```

**Never run `php artisan config:cache` on this project until the code fix below is applied.**

---

## 4. Final nginx configs

### `/etc/nginx/sites-available/cfpapidemo.oag.go.ug` — Laravel API

```nginx
server {
    listen 80;
    server_name cfpapidemo.oag.go.ug;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name cfpapidemo.oag.go.ug;
    root /var/www/oagapi/public;
    index index.php;

    ssl_certificate     /etc/ssl/certs/fullchain.crt;
    ssl_certificate_key /etc/ssl/private/privkey.key;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

`try_files ... /index.php?$query_string` replaces Apache's `.htaccess` mod_rewrite rules —
nginx does not read `.htaccess` at all.

### `/etc/nginx/sites-available/cfpdemo.oag.go.ug` — React SPA (static)

```nginx
server {
    listen 80;
    server_name cfpdemo.oag.go.ug;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name cfpdemo.oag.go.ug;
    root /var/www/oagdash;          # use /var/www/oagdash/build if index.html lives there
    index index.html;

    ssl_certificate     /etc/ssl/certs/fullchain.crt;
    ssl_certificate_key /etc/ssl/private/privkey.key;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

`try_files $uri $uri/ /index.html` is the nginx equivalent of the Apache SPA fallback:

```apache
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^ /index.html [L]
```

### `/etc/nginx/sites-available/cfp.oag.go.ug` — Next.js (reverse proxy)

```nginx
server {
    listen 80;
    server_name cfp.oag.go.ug;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name cfp.oag.go.ug;

    ssl_certificate     /etc/ssl/certs/fullchain.crt;
    ssl_certificate_key /etc/ssl/private/privkey.key;

    gzip on;
    gzip_types application/javascript text/css application/json;
    gzip_min_length 256;

    location /_next/static/ {
        alias /var/www/cfpwebapp/.next/static/;
        expires 365d;
        access_log off;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Enable and reload

```bash
sudo ln -s /etc/nginx/sites-available/cfpapidemo.oag.go.ug /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/cfpdemo.oag.go.ug    /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/cfp.oag.go.ug        /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Always `nginx -t` before reloading — a bad cert path takes the whole site down otherwise.

---

## 5. SSL certificate setup

### Certbot did not work

```
Detail: 154.72.196.174: Fetching http://cfpapidemo.oag.go.ug/.well-known/acme-challenge/...
        Timeout during connect (likely firewall problem)
```

`ufw` allowed 80/443, but **inbound port 80 is blocked upstream** (corporate firewall / NAT).
Earlier `curl` tests passed only because they ran *from* the server itself.

Let's Encrypt HTTP-01 validation is therefore impossible on this box. Options were DNS-01
(manual TXT record every 90 days) or the existing commercial certificate.

### Used the existing DigiCert wildcard

```
subject   = CN = *.oag.go.ug
notBefore = Apr 17 00:00:00 2026 GMT
notAfter  = Nov  1 23:59:59 2026 GMT
SAN       = DNS:*.oag.go.ug, DNS:oag.go.ug
```

Covers all three subdomains. Conversion from the supplied `.pfx`:

```bash
sudo openssl pkcs12 -in oag.go.ug.pfx -clcerts -nokeys -out oag.go.ug.crt
sudo openssl pkcs12 -in oag.go.ug.pfx -nocerts -out oag.go.ug.key
sudo openssl rsa -in oag.go.ug.key -out oag.go.ug.nopass.key
sudo mv oag.go.ug.nopass.key privkey.key
sudo mv oag.go.ug.crt cert.crt
sudo bash -c "cat cert.crt DigiCertCA.crt TrustedRoot.crt > fullchain.crt"
sudo chmod 600 privkey.key
sudo chmod 644 fullchain.crt
sudo cp fullchain.crt /etc/ssl/certs/
sudo cp privkey.key   /etc/ssl/private/
sudo update-ca-certificates
```

### Verification (do this before reloading nginx)

```bash
# Does the cert cover the domain, and is it in date?
sudo openssl x509 -in /etc/ssl/certs/fullchain.crt -noout -subject -dates -ext subjectAltName

# Do the cert and key actually match? These two hashes MUST be identical.
sudo openssl x509 -noout -modulus -in /etc/ssl/certs/fullchain.crt  | openssl md5
sudo openssl rsa  -noout -modulus -in /etc/ssl/private/privkey.key | openssl md5
```

> **The cert file path does not need to contain the domain name.** What matters is that the
> certificate *inside* covers the `server_name`.

**⚠ Expires 1 November 2026** — set a calendar reminder. There is no auto-renewal.

---

## 6. Frontend apps

### Next.js — `EADDRINUSE :::3000`

pm2 kept crashing with 16+ restarts. Killing the process on 3000 respawned it instantly with a
new PID.

```bash
ps -o pid,ppid,user,cmd -p $(pgrep -f next-server)
#    PID    PPID USER      CMD
# 164349       1 cfpweba+  next-server (v16.0.10)     ← PPID 1 = systemd

systemctl list-units --type=service | grep -iE "next|cfp|pm2"
# cfpwebapp.service   loaded active running   CFP Next.js frontend
```

The app had **always** been run by a systemd unit, not pm2. pm2 was never part of the original
deployment — installing it created a second supervisor competing for port 3000.

To switch to pm2 deliberately:

```bash
systemctl cat cfpwebapp             # read WorkingDirectory, User, EnvironmentFile first
sudo systemctl stop cfpwebapp
sudo systemctl disable cfpwebapp
sudo ss -tulpn | grep ':3000'       # confirm free

cd /var/www/cfpwebapp
pm2 start npm --name cfpwebapp -- start
pm2 save
pm2 startup                          # run the command it prints, or it dies on reboot
```

> That systemd unit was unrelated to Apache — Node services were untouched by the purge.

### Next.js — 403 on `/_next/static/chunks/*`

```
Failed to load resource: the server responded with a status of 403 (Forbidden)
Uncaught Error: Failed to load chunk /_next/static/chunks/...
```

`.next/` was built as **root**; nginx serves those chunks directly via the `alias` block and its
workers run as `www-data`.

```bash
sudo chown -R cfpwebapp:cfpwebapp /var/www/cfpwebapp/.next
sudo chmod -R a+rX /var/www/cfpwebapp/.next
sudo chmod a+x /var/www /var/www/cfpwebapp
sudo systemctl reload nginx
pm2 restart cfpwebapp
```

`a+rX` (capital X) sets execute on **directories only**, which is what traversal needs.

**Prevention — always build as the app user:**

```bash
sudo -u cfpwebapp npm run build
```

### CORS — a false alarm

Chrome reported:

```
Access to XMLHttpRequest at 'https://cfpapidemo.oag.go.ug/api/get-logged-in-user'
from origin 'https://cfpdemo.oag.go.ug' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present
```

Server-side testing showed CORS headers **were** being sent correctly:

```bash
curl -i -H "Origin: https://cfpdemo.oag.go.ug" \
  https://cfpapidemo.oag.go.ug/api/get-logged-in-user
# → Access-Control-Allow-Origin: https://cfpdemo.oag.go.ug   ✓
```

The same flow worked perfectly in **DuckDuckGo browser**. The tell was in Chrome's own console:

```
Unchecked runtime.lastError: A listener indicated an asynchronous response by
returning true, but the message channel closed before a response was received
```

That is a **Chrome extension** interfering with requests. Ad blockers, privacy tools and password
managers commonly strip or rewrite `Origin` / `Authorization` headers.

**Test with:** incognito window, or `chrome://extensions` → disable all → retry.
Also tick DevTools → Network → **Disable cache** (Chrome caches CORS preflights separately).

Two genuine findings surfaced along the way:

1. Unauthenticated API requests return **302 → `/api/login`** instead of 401 JSON. CORS headers do
   not survive a redirect, so the browser reports it as a CORS error. Fix in `bootstrap/app.php`:

   ```php
   ->withExceptions(function (Exceptions $exceptions) {
       $exceptions->shouldRenderJsonWhen(
           fn ($request) => $request->is('api/*') || $request->expectsJson()
       );
   })
   ```

2. `config/cors.php:33` has `'supports_credentials' => false`. If the frontend sends
   `withCredentials: true`, the browser rejects the response regardless of origin.

---

## 7. Queue workers

`QUEUE_CONNECTION=database`, with seven queued jobs — **a worker must be running** or feedback
notifications silently pile up and never send:

`SendEmailNotificationToUsersJob`, `SendSmsNotificationToUsersJob`,
`SendFirebaseNotificationToUsersJob`, `SendFeedbackRemindersToDirectorsJob`,
`SendNotificationJob`, `StoreLocationsJob`, `ProcessSublocationChunk`

Dispatched from `FeedbackController`, `FeebackAtFeedbackCoordinatorController`,
`FeedbackNotificationsController`, and a scheduled command.

### Duplicate supervisor group

```
cfpapi-worker:cfpapi-worker_00   RUNNING
oagapi-queue:oagapi-queue_00     FATAL   Exited too quickly
```

Two groups were configured for the same queue; the stale one kept failing.

```bash
# find and read the failing config
grep -rn "oagapi-queue" /etc/supervisor/
sudo tail -n 30 /var/log/supervisor/oagapi-queue*.log

# remove it
sudo supervisorctl stop oagapi-queue:*
sudo mv /etc/supervisor/conf.d/oagapi-queue.conf /root/oagapi-queue.conf.bak
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl status
```

If `reread` says "No config updates", the file lives outside `conf.d/` — check the `[include]`
section of `/etc/supervisor/supervisord.conf`. Last resort: `sudo systemctl restart supervisor`.

### Health checks

```bash
sudo supervisorctl status
mysql -u user_oag -p oag_laravel -e "SELECT COUNT(*) FROM jobs; SELECT COUNT(*) FROM failed_jobs;"
```

A growing `jobs` table means no worker is consuming.

> **Restart workers after every deploy** — `queue:work` holds the old code in memory:
> `sudo supervisorctl restart cfpapi-worker:*`

---

## 8. Application bugs found

### a. Bulk delete — `Undefined array key "feedback_no"`

`app/Http/Controllers/API/FeedbackController.php:1115-1116`

```php
$feedbackNo = $feedbackObj['feedback_no'];   // ✗ throws when key absent
$message    = $feedbackObj['message'];       // ✗ same
```

Validation declares both `nullable` (lines 1101-1102). When the frontend omits the keys — which it
does for rows where `feedback_no` is `null` and the UI shows **N/A** — they never appear in
`$validatedData`, and direct array access throws.

```php
$feedbackNo = $feedbackObj['feedback_no'] ?? null;   // ✓
$message    = $feedbackObj['message'] ?? null;       // ✓
```

`SummarisedFeedbackController.php:1785` emits the same error message and likely shares the pattern.

### b. `FirebaseService` uses `env()` instead of `config()`

`app/Services/FirebaseService.php:18-29`. Makes `php artisan config:cache` fatal.
Move the eleven `FIREBASE_*` keys into `config/firebase.php` and read them via `config()`.

### c. `AuthController` injects `FirebaseService` but never uses it

`app/Http/Controllers/AuthController.php:37,41,47` — assigned and never referenced again.
A Firebase misconfiguration currently takes down **login and every other route on that
controller** for no benefit. Remove the constructor dependency or resolve it lazily at point of use.

### d. Login blocks on SMTP for ~13 seconds

`AuthController.php:707-709` calls `Mail::send()` **synchronously** — the one mail path that
bypasses the queue. It is also not individually wrapped, so any SMTP failure becomes a 500 via the
catch-all. Should be a queued job.

### e. LDAP failure returns 500

`AuthController.php:742-745` returns 500 when the LDAP server is unreachable, making an
infrastructure problem look like an application crash. 503 would be more accurate.

Note the login flow at `AuthController.php:608-619` tries local DB auth first and **falls through
to LDAP whenever that fails** — so every wrong password attempts an LDAP bind against
`172.16.10.8`, an internal address.

---

## 9. Diagnostic cheat sheet

### Where to look first

```bash
tail -f /var/www/oagapi/storage/logs/laravel.log      # app exceptions
sudo tail -f /var/log/nginx/error.log                 # nginx / permissions
sudo tail -f /var/log/php8.2-fpm.log                  # PHP fatals
sudo journalctl -xeu nginx.service -n 30 --no-pager   # unit start failures

# both at once
sudo tail -f /var/www/oagapi/storage/logs/laravel.log /var/log/nginx/error.log
```

Only the **most recent** log file is being written — Laravel may be on daily rotation:

```bash
ls -lt /var/www/oagapi/storage/logs/ | head
tail -n 100 storage/logs/$(ls -t storage/logs/ | head -1)
```

### Reading the signal

| Observation | Meaning |
| --- | --- |
| 404, no PHP involved | rewrite/`try_files` wrong, or wrong vhost |
| 500, **empty** body, empty `laravel.log` | PHP died before Laravel booted → permissions or extensions |
| 500 with `Cache-Control: no-cache, private` + `Vary: Origin` | Laravel booted and threw → read `laravel.log` |
| 502 / 503 | upstream (php-fpm socket or Node) not reachable |
| `Content-Type: text/html` with no `Content-Length` | php-fpm responded, not an nginx error page |

### Service checks

```bash
systemctl status nginx php8.2-fpm mysql --no-pager
apachectl -M | grep -E "rewrite|php"     # legacy Apache only
sudo ss -tulpn | grep -E ':80|:443|:3000'
sudo nginx -t
apachectl -S                              # which vhost owns which port
```

### Permission checks

```bash
sudo -u www-data test -w storage/logs && echo WRITABLE || echo NOT WRITABLE
namei -l $(pwd)/storage/logs             # walks the whole path for a broken link
getfacl /var/www/oagapi/vendor | head    # ACLs can deny even when mode bits look fine
```

`namei -l` catches the classic case where `storage/` is perfect but a parent directory lacks `x`.

### Laravel cache

```bash
php artisan config:clear
php artisan cache:clear
php artisan route:clear      # a stale routes-v7.php can serve responses from old code
php artisan view:clear
```

A stale **route cache** was what briefly made `/` return JSON from an older deployment instead of
the `welcome` view in `routes/web.php:5`.

### Verify PHP the web SAPI actually sees

```bash
php -m                                                # CLI extensions
php -i | grep "Loaded Configuration File"             # CLI ini
ls /etc/php/8.2/fpm/conf.d/                           # what FPM loads
```

CLI and FPM read **different** `php.ini` files. A mismatch produces "works in artisan, fails in
browser".

---

## 10. Outstanding items

- [ ] **SSL expires 1 Nov 2026** — no auto-renewal. Calendar reminder required.
- [ ] **Inbound port 80 blocked upstream** — blocks all future Let's Encrypt HTTP-01 validation.
      Ask the network team to open it, or commit to DNS-01.
- [ ] Apply the `?? null` fix in `FeedbackController.php:1115-1116` and check
      `SummarisedFeedbackController.php:1785`.
- [ ] Move `FIREBASE_*` from `env()` to `config()` in `FirebaseService.php`, then `config:cache`
      becomes safe in production.
- [ ] Remove the unused `FirebaseService` injection from `AuthController.php:41`.
- [ ] Queue the login OTP mail (`AuthController.php:707`) to kill the 13-second response.
- [ ] Set `supports_credentials => true` in `config/cors.php:33` if the frontend sends
      `withCredentials`.
- [ ] Add `shouldRenderJsonWhen()` so API routes return 401 JSON rather than a 302 to HTML.
- [ ] Decide pm2 **or** systemd for `cfpwebapp` — never both.
- [ ] Review `MemoryDenyWriteExecute=true` in the nginx hardening drop-in before adding
      regex-heavy `location` blocks.
- [ ] Confirm `logrotate` change survived one rotation cycle.
- [ ] `oag-cfp-dev` Firebase service-account key was exposed in terminal output during debugging —
      rotate it in the Firebase console.

---

## Deployment checklist (for next time)

```bash
# 1. Pull
cd /var/www/oagapi && git pull

# 2. Dependencies
composer install --no-dev --optimize-autoloader

# 3. Migrate
php artisan migrate --force

# 4. Clear caches — do NOT config:cache until FirebaseService is fixed
php artisan config:clear && php artisan cache:clear && php artisan route:clear

# 5. Permissions (composer install can create root-owned files)
sudo chown -R www-data:www-data /var/www/oagapi
sudo chmod -R 775 storage bootstrap/cache
sudo chmod 640 .env

# 6. Restart PHP + workers (queue:work holds old code in memory)
sudo systemctl restart php8.2-fpm
sudo supervisorctl restart cfpapi-worker:*

# 7. Verify
curl -I https://cfpapidemo.oag.go.ug
tail -n 20 storage/logs/laravel.log
```

For the Next.js app, **build as the app user** so `.next/` stays readable by nginx:

```bash
cd /var/www/cfpwebapp
sudo -u cfpwebapp npm ci && sudo -u cfpwebapp npm run build
pm2 restart cfpwebapp
```
