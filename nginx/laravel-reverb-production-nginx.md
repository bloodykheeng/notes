# Laravel Reverb in Production (Nginx + Supervisor)

Reverb is Laravel's WebSocket server. It is a long-running process: Supervisor keeps it
alive, Nginx terminates SSL on 443 and proxies the `/app` and `/apps` paths to it.

```
Browser  wss://api.example.com/app/<key>  ->  Nginx :443  ->  Reverb 127.0.0.1:8080
```

Reverb never faces the internet directly. It listens on an internal port; the public side
is your existing API domain on 443, so there is no extra DNS record and no extra
certificate.

For a full server build (PHP, MySQL, pm2, cron, SSL) see
[laravel-nextjs-two-environments-deploy.md](laravel-nextjs-two-environments-deploy.md).
This note is only the Reverb part.

---

## 1. Install

```bash
php artisan install:broadcasting
```

Installs Reverb, `laravel-echo`, `pusher-js` and creates `config/reverb.php`.

---

## 2. Generate the app id / key / secret

One set **per environment**. Never reuse dev values on a server: the secret signs private
channel auth, and the key is public in the browser bundle.

```bash
echo "REVERB_APP_ID=$(shuf -i 100000-999999 -n 1)"
echo "REVERB_APP_KEY=$(openssl rand -hex 10)"
echo "REVERB_APP_SECRET=$(openssl rand -hex 10)"
```

---

## 3. `.env` (Laravel)

```env
BROADCAST_CONNECTION=reverb

REVERB_APP_ID=<generated>
REVERB_APP_KEY=<generated>
REVERB_APP_SECRET=<generated>

# public: what browsers and Laravel connect to (Nginx, SSL on 443)
REVERB_HOST="api.example.com"
REVERB_PORT=443
REVERB_SCHEME=https

# internal: where the Reverb process listens, behind Nginx. One port per app on the box.
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080
```

Frontend values, same key and public host:

- Blade/Vite app, in the same `.env`:

  ```env
  VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
  VITE_REVERB_HOST="${REVERB_HOST}"
  VITE_REVERB_PORT="${REVERB_PORT}"
  VITE_REVERB_SCHEME="${REVERB_SCHEME}"
  ```

- Next.js app, in its own `.env.production`:

  ```env
  NEXT_PUBLIC_REVERB_APP_ID=<same>
  NEXT_PUBLIC_REVERB_APP_KEY=<same>
  NEXT_PUBLIC_REVERB_APP_SECRET=<same>
  NEXT_PUBLIC_REVERB_HOST="api.example.com"
  NEXT_PUBLIC_REVERB_PORT=443
  NEXT_PUBLIC_REVERB_SCHEME=https
  ```

Frontend values are read at **build time**: rebuild (`npm run build`) after changing them.

---

## 4. Nginx: proxy `/app` and `/apps` inside the API server block

Add to the existing `server { }` of the API domain. `proxy_pass` port = `REVERB_SERVER_PORT`.

```nginx
location ~ ^/(app|apps) {
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
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Firewall: only `Nginx Full` (80, 443). Never open 8080.

---

## 5. Supervisor

`sudo nano /etc/supervisor/conf.d/api.example.com-reverb.conf`

```ini
[program:api.example.com-reverb]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/api.example.com/artisan reverb:start --host=0.0.0.0 --port=8080
autostart=true
autorestart=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/api.example.com/storage/logs/reverb.log
stopwaitsecs=3600
```

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl status
```

Name the file and the program after the domain, so a second app on the box does not
collide in `supervisorctl status`.

The program runs as `www-data`, so its log file must be writable by `www-data`. If you ever run
artisan as root in this folder, re-fix it, otherwise Reverb logs nothing:

```bash
chown -R www-data storage
```

---

## 6. Two or more apps on one box

Each Reverb process needs its own internal port. Two processes cannot bind 8080; the
second one shows `FATAL Exited too quickly` / `ERROR (spawn error)`.

| App | `REVERB_SERVER_PORT` | Supervisor `--port` | Nginx `proxy_pass` | Public |
| --- | --- | --- | --- | --- |
| testapi.example.com | 8080 | 8080 | `http://127.0.0.1:8080` | `wss://testapi.example.com` 443 |
| api.example.com | 8081 | 8081 | `http://127.0.0.1:8081` | `wss://api.example.com` 443 |

Three places must agree per app: the `.env`, the Supervisor command, the Nginx
`proxy_pass`. The public side (`REVERB_HOST`, 443, https) is the same for all.

Check what is bound:

```bash
grep -H command= /etc/supervisor/conf.d/*reverb*.conf
sudo ss -ltnp | grep -E ':808[0-9]'
```

---

## 7. After every deploy

```bash
sudo supervisorctl restart api.example.com-reverb:*
```

Reverb keeps old code in memory until restarted. Restart the queue worker at the same
time: `ShouldBroadcast` events go through the queue.

---

## 8. Load limits (only if you expect >1,000 concurrent connections)

Each connection is one open file. Raise the limit for `www-data` in
`/etc/security/limits.conf`:

```
www-data   soft   nofile   10000
www-data   hard   nofile   10000
```

and install the `uv` event loop (the default `stream_select` caps at 1,024):

```bash
sudo apt install -y php-pear php-dev libuv1-dev
sudo pecl install uv
```

Reverb switches to `uv` automatically when the extension is present.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `FATAL Exited too quickly` / `ERROR (spawn error)` | Port already taken by another app's Reverb. Section 6. |
| Both apps `RUNNING` but one refuses the app key | Its Nginx `proxy_pass` points at the other app's port. |
| Browser `WebSocket connection failed` | `/app` location missing the `Upgrade` / `Connection "Upgrade"` headers. |
| Mixed content / `ws://` blocked | `REVERB_SCHEME=http` on an https site. Set `https`, port `443`, rebuild frontend. |
| Connects then drops | `allowed_origins` in `config/reverb.php` does not include the frontend domain. |
| `502 Bad Gateway` on `/app` | Reverb not running: `sudo supervisorctl status`. |
| Events broadcast, client gets nothing | `BROADCAST_CONNECTION` not `reverb`, or the queue worker is down or stale. |
| Killed the process, it came back | `autorestart=true`. Use `supervisorctl stop <program>:*`. |
| `Failed to create broadcaster for connection "reverb": Pusher::__construct(): Argument #1 ($auth_key) must be of type string, null given` | The app is reading a **cached config** built before `REVERB_APP_KEY` existed. `php artisan config:show broadcasting.connections.reverb` shows the truth; fix with `config:clear && config:cache`, then restart the workers (they hold the old config in memory). |
| Reverb writes nothing to `storage/logs/reverb.log` | The log file is root-owned while Reverb runs as `www-data`. `chown -R www-data storage`. See [laravel-production-cache-permission-error-fix.md](laravel-production-cache-permission-error-fix.md). |

---

## Quick reference

| Task | Command |
| --- | --- |
| Generate id/key/secret | the three `echo` lines in section 2 |
| Start manually (debug) | `php artisan reverb:start --host=0.0.0.0 --port=8080 --debug` |
| Restart under Supervisor | `sudo supervisorctl restart <domain>-reverb:*` |
| Status | `sudo supervisorctl status` |
| Logs | `tail -f storage/logs/reverb.log` |
| Ports per app | `grep -H command= /etc/supervisor/conf.d/*reverb*.conf` |
| Listeners | `sudo ss -ltnp \| grep -E ':808[0-9]'` |

Reference: [Laravel Reverb docs](https://laravel.com/docs/reverb)
