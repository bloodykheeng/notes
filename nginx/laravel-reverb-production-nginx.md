# Deploy Laravel Reverb in Production (Nginx + Supervisor)

Laravel Reverb is a first-party WebSocket server for real-time broadcasting
(notifications, chat, live dashboards). It is a **long-running process**, so on
a server you run it under **Supervisor** (to keep it alive) and put **Nginx**
in front of it as a reverse proxy that terminates SSL and upgrades the
connection to WebSocket.

```
Browser  ──wss://your-domain.com:443──►  Nginx (SSL)  ──proxy──►  Reverb (127.0.0.1:8080)
```

> Architecture rule: **Reverb never faces the internet directly.** It listens on
> an internal port (default `8080`); Nginx handles `443`/SSL and proxies to it.

> **Already running Reverb for another site on this box?** The second app needs
> its own internal port — 8080 serves one process and Nginx cannot share it.
> Read [Two or More Reverb Apps on One Box](#two-or-more-reverb-apps-on-one-box)
> before you start, or the second app will never come up.

> **Two ways to expose it — pick one:**
>
> - **Option A (preferred): reuse your existing app domain** (e.g.
>   `alphapi.nwtdemos.com`). Reverb only uses the `/app` and `/apps` URL paths,
>   so you just proxy those paths to Reverb and leave the rest of the site alone.
>   No new DNS record, no new SSL cert.
> - **Option B: a dedicated `ws.` subdomain** (e.g. `ws.your-domain.com`). Cleaner
>   separation, but needs its own DNS record + SSL cert. Use this only if your app
>   actually serves routes at `/app` or `/apps`.
>
> The steps below show **Option A first**, with Option B noted where it differs.

Reference: [Laravel Reverb docs](https://laravel.com/docs/12.x/reverb)

---

## Step 1 — Install Reverb

From your project root:

```bash
php artisan install:broadcasting
```

This runs `reverb:install`, pulls in the `laravel/reverb` package, sets sensible
defaults in `config/reverb.php`, and adds the `REVERB_*` variables to `.env`.

> Requires PHP **8.2+**. If broadcasting prompts about Laravel Echo / front-end
> scaffolding, accept it so the client side (`laravel-echo` + `pusher-js`) is
> wired up too.

---

## Step 2 — Configure Environment Variables

The key thing to understand: there are **two pairs** of host/port variables and
they mean different things.

| Variable | Meaning |
|---|---|
| `REVERB_SERVER_HOST` / `REVERB_SERVER_PORT` | Where the **Reverb process itself binds** (internal). |
| `REVERB_HOST` / `REVERB_PORT` | Where **clients + Laravel connect to** (your public domain, port 443). |

Edit `.env`:

```dotenv
BROADCAST_CONNECTION=reverb

REVERB_APP_ID=my-app-id
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret

# Where the Reverb process listens internally (behind Nginx)
# One app per port. A second app on this box needs 8081, a third 8082.
# See "Two or More Reverb Apps on One Box" below.
REVERB_SERVER_HOST=127.0.0.1
REVERB_SERVER_PORT=8080

# What clients/Laravel use publicly (Nginx terminates SSL on 443)
# Option A (preferred): your existing app domain
REVERB_HOST=alphapi.nwtdemos.com
# Option B: a dedicated subdomain → REVERB_HOST=ws.your-domain.com
REVERB_PORT=443
REVERB_SCHEME=https

# Vite / Echo front-end values (read at build time)
VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
VITE_REVERB_HOST="${REVERB_HOST}"
VITE_REVERB_PORT="${REVERB_PORT}"
VITE_REVERB_SCHEME="${REVERB_SCHEME}"
```

> **Option A** points Reverb at the domain you already run the app on — nothing
> extra to register. **Option B** uses a dedicated subdomain; if you go that
> route, point its DNS `A` record at the same server IP as your app.

Apply config + rebuild front-end:

```bash
php artisan config:clear
npm run build
```

---

## Step 3 — Nginx Reverse Proxy (WebSocket Upgrade)

Reverb only uses two URL paths — `/app` (WebSocket connections) and `/apps`
(its API). Your Laravel app doesn't use those, so the trick is to proxy **only
those paths** to Reverb and let everything else keep hitting PHP-FPM.

### Option A (preferred) — Reuse your existing app server block

Open the server block you already have, e.g.
`/etc/nginx/sites-available/alphapi.nwtdemos.com`, and add this **above** your
main `location /` (PHP) block:

```nginx
# Reverb WebSocket — only /app and /apps go to Reverb
location ~ ^/(app|apps) {
    proxy_http_version 1.1;
    proxy_set_header Host $http_host;
    proxy_set_header Scheme $scheme;
    proxy_set_header SERVER_PORT $server_port;
    proxy_set_header REMOTE_ADDR $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # These two lines are what turn the request into a WebSocket
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";

    # Must match this app's supervisor --port. Second app on the box: 8081.
    proxy_pass http://127.0.0.1:8080;
}
```

Leave your existing `location /` and `location ~ \.php$` blocks untouched.
Because this server block is **already on HTTPS** (your app's existing Certbot
cert), `wss://` works immediately — **you can skip Step 4.**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

> ⚠️ Only safe if your app has **no route at `/app` or `/apps`**. A typical API
> (routes under `/api/...`) is fine. If you do use those paths, go with Option B.

### Option B — Dedicated `ws.` subdomain

Create a new server block, e.g.
`/etc/nginx/sites-available/ws.your-domain.com`:

```nginx
server {
    listen 80;
    server_name ws.your-domain.com;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header Scheme $scheme;
        proxy_set_header SERVER_PORT $server_port;
        proxy_set_header REMOTE_ADDR $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";

        proxy_pass http://127.0.0.1:8080;
    }
}
```

Here a whole-server `location /` is fine because the subdomain serves nothing
but Reverb. Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/ws.your-domain.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## Step 4 — Add SSL (Option B only)

> **Option A users: skip this** — your app domain already has a certificate that
> covers these paths.

Browsers on an HTTPS page **refuse insecure `ws://`** — you must serve `wss://`.
For the dedicated subdomain, issue a cert with Certbot:

```bash
sudo certbot --nginx -d ws.your-domain.com
```

Certbot rewrites the server block to listen on `443` with your certificate and
adds an HTTP→HTTPS redirect. After this, clients connect to
`wss://ws.your-domain.com` and Nginx proxies to the plain internal `8080`.

---

## Step 5 — Open the Firewall (proxy only, not Reverb)

Only open the public web ports. Do **not** expose `8080`.

```bash
sudo ufw allow "Nginx Full"   # 80 + 443
sudo ufw status
```

---

## Step 6 — Keep Reverb Running with Supervisor

A bare `php artisan reverb:start` dies when you close the shell. Supervisor
restarts it on crash and on boot.

Install Supervisor:

```bash
sudo apt install -y supervisor
```

Create `/etc/supervisor/conf.d/reverb.conf`. Name the file and the program after
the site rather than `reverb`, so a second app on the same box does not collide
with this one in `supervisorctl status`:

```ini
[program:reverb]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/your-app/artisan reverb:start --host=127.0.0.1 --port=8080
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/www/your-app/storage/logs/reverb.log
stopwaitsecs=3600
numprocs=1
```

> Change `/var/www/your-app` to your real project path, and `user` to whoever
> owns the app (often `www-data`).
>
> The `--port` flag **overrides `REVERB_SERVER_PORT`**, so this line is the one
> that decides. Keep `.env` in step with it anyway, or the next person to read
> the file is misled.

**Raise Supervisor's open-file limit** — each WebSocket connection is an open
file. Edit `/etc/supervisor/supervisord.conf` and under `[supervisord]` add/set:

```ini
[supervisord]
minfds=10000
```

Load it:

```bash
sudo systemctl restart supervisor
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start reverb:*
sudo supervisorctl status        # should show reverb_00  RUNNING
```

---

## Two or More Reverb Apps on One Box

Everything above is written for one site. If the server already runs another
Laravel app with Reverb, **the second one gets its own internal port.** Two
processes cannot bind `8080`, and Nginx cannot rescue that: it picks the
destination by `server_name`, but a port serves one process and no more.

What it looks like when you get it wrong — the second app never starts:

```
sdsbetapi.nwtdemos.com-reverb:..._00   FATAL   Exited too quickly (process log may have details)
sudo supervisorctl start sdsbetapi.nwtdemos.com-reverb:*
sdsbetapi.nwtdemos.com-reverb:..._00: ERROR (spawn error)
```

Confirm it in one line. If both print `--port=8080`, that is the fault:

```bash
grep -H command= /etc/supervisor/conf.d/*reverb*.conf
```

### One port per app

| App | Internal port | Nginx `proxy_pass` | Public |
|---|---|---|---|
| alphapi.nwtdemos.com | 8080 | `http://127.0.0.1:8080` | 443, `wss://alphapi.nwtdemos.com` |
| sdsbetapi.nwtdemos.com | 8081 | `http://127.0.0.1:8081` | 443, `wss://sdsbetapi.nwtdemos.com` |

**Only the internal port differs.** Every app still publishes on 443 through its
own server block, so `REVERB_HOST`, `REVERB_PORT=443` and `REVERB_SCHEME=https`
are unchanged and no front-end build changes.

### Three places must agree

For the second app, and they are the whole job:

```bash
# 1. Supervisor — the flag that actually decides
sudo sed -i 's/--port=8080/--port=8081/' \
  /etc/supervisor/conf.d/sdsbetapi.nwtdemos.com-reverb.conf

# 2. Nginx — inside the /app and /apps location blocks
sudo sed -i 's|127.0.0.1:8080|127.0.0.1:8081|g' \
  /etc/nginx/sites-available/sdsbetapi.nwtdemos.com

# 3. .env — kept in step with the flag
sed -i 's/^REVERB_SERVER_PORT=.*/REVERB_SERVER_PORT=8081/' \
  /var/www/sdsbetapi.nwtdemos.com/.env
```

Apply and restart:

```bash
cd /var/www/sdsbetapi.nwtdemos.com && php artisan config:cache
sudo supervisorctl reread && sudo supervisorctl update
sudo supervisorctl start sdsbetapi.nwtdemos.com-reverb:*
sudo nginx -t && sudo systemctl reload nginx
```

### Verify

Both ports listening, one PHP process each:

```bash
sudo ss -ltnp | grep -E '808[01]'
```

```
LISTEN 0 511 127.0.0.1:8080 users:(("php",pid=1297634,fd=5))
LISTEN 0 511 127.0.0.1:8081 users:(("php",pid=1297659,fd=5))
```

Then a real handshake, per app. `curl -I` sends HEAD and Reverb answers `500`,
which proves nothing — do the upgrade instead:

```bash
curl -i -N -o- \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  https://sdsbetapi.nwtdemos.com/app/YOUR_REVERB_APP_KEY
```

```
HTTP/1.1 101 Switching Protocols
X-Powered-By: Laravel Reverb

{"event":"pusher:connection_established", ...}
```

`101` plus that frame is the only proof. Ctrl+C to exit.

### The failure this catches

Point app B's Nginx at app A's port and **both apps keep running**. Nothing is
`FATAL`, nothing is in a log. App B's browsers connect to app A's Reverb, get
refused on the app key, and live updates are simply dead. So after any port
change, check `proxy_pass` and the supervisor `--port` name the same number.

### Ports other things want

Reverb's default 8080 is the most contested port on a box. File Browser, Jenkins,
Tomcat and half the admin tools reach for it. Give them something else:

```bash
filebrowser -r /var/www -p 8090
```

If a process is fighting you and keeps coming back after `kill`, it is
Supervisor's `autorestart=true` doing its job. Stop it properly:

```bash
sudo supervisorctl stop alphapi.nwtdemos.com-reverb:*
```

A program left in `STOPPED` stays down until you `start` it by name. It does not
come back on `systemctl restart supervisor`.

---

## Step 7 — Raise OS Open-File Limits

The OS also caps open files per user. Each connection = one file.

Check current limit:

```bash
ulimit -n
```

Raise it in `/etc/security/limits.conf` (use the user from your Supervisor
config, e.g. `www-data`):

```
www-data   soft   nofile   10000
www-data   hard   nofile   10000
```

> For **more than ~1,000 concurrent connections**, also install the `ext-uv`
> event loop (the default `stream_select` is capped at 1,024 files):
>
> ```bash
> sudo apt install -y php-pear php-dev libuv1-dev
> sudo pecl install uv
> ```
>
> Reverb auto-switches to the `uv` loop when the extension is present.

---

## Step 8 — Front-End (Laravel Echo) Sanity Check

`resources/js/bootstrap.js` (or `echo.js`) should read the Vite vars:

```js
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';
window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT ?? 80,
    wssPort: import.meta.env.VITE_REVERB_PORT ?? 443,
    forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```

Rebuild after any change: `npm run build`.

---

## Restarting After Code Changes

Reverb is long-running, so deploying new code does **not** reload it. After a
deploy, gracefully restart it:

```bash
php artisan reverb:restart
```

If running under Supervisor, this terminates connections cleanly and Supervisor
brings the process back up automatically.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `FATAL Exited too quickly` or `ERROR (spawn error)` | Another process holds the port. `grep -H command= /etc/supervisor/conf.d/*reverb*.conf` — two apps on 8080 cannot both run. See "Two or More Reverb Apps on One Box". |
| Second app connects but is refused on the app key | Its Nginx `proxy_pass` points at the other app's port. Both stay `RUNNING`, so nothing looks wrong. |
| Killed the process, it came straight back | Supervisor's `autorestart=true`. Use `supervisorctl stop <program>:*`. |
| Browser console: `WebSocket connection failed` | Nginx not upgrading — check the `Upgrade` / `Connection "Upgrade"` headers. |
| Mixed-content / `ws://` blocked | You're on HTTPS but `REVERB_SCHEME=http`. Set it to `https` and finish Step 4 (SSL). |
| Connects then drops instantly | `allowed_origins` in `config/reverb.php` doesn't include your site. Add your domain (or `'*'` temporarily). |
| `502 Bad Gateway` on `ws.` subdomain | Reverb process isn't running — `sudo supervisorctl status reverb:*`. |
| Stops accepting connections under load | Raise `ulimit -n`, `minfds`, and install `ext-uv` (Step 7). |
| Events broadcast but client gets nothing | `BROADCAST_CONNECTION=reverb` not set, or queue worker not running for `ShouldBroadcast` events. |

---

## Quick Reference

| Task | Command |
|---|---|
| Install Reverb | `php artisan install:broadcasting` |
| Start manually (testing) | `php artisan reverb:start --host=127.0.0.1 --port=8080` |
| Start with debug output | `php artisan reverb:start --debug` |
| Graceful restart after deploy | `php artisan reverb:restart` |
| Supervisor status | `sudo supervisorctl status reverb:*` |
| Tail Reverb logs | `tail -f storage/logs/reverb.log` |
| Which port each app uses | `grep -H command= /etc/supervisor/conf.d/*reverb*.conf` |
| What is listening | `sudo ss -ltnp \| grep -E '808[0-9]'` |
| Prove a websocket works | the `curl` upgrade in "Two or More Reverb Apps on One Box" |
| Check OS open-file limit | `ulimit -n` |
| Issue SSL for WS subdomain | `sudo certbot --nginx -d ws.your-domain.com` |
