# Deploy Laravel API + Next.js on CentOS / RHEL 8 with Apache (httpd) and SELinux

Copy-paste order for a **CentOS 8 / RHEL 8 / Rocky / Alma** box running **Apache httpd** (not nginx) with
**SELinux Enforcing**: a Laravel API (PHP-FPM, queue worker, scheduler) and a Next.js frontend (pm2) behind
Apache, with Let's Encrypt. First used on a shared CentOS 8 Linode, Oct 2026.

It is the same stack as [../nginx/laravel-nextjs-two-environments-deploy.md](../nginx/laravel-nextjs-two-environments-deploy.md);
this note is what changes on a Red Hat family box. Redeploying: [laravel-nextjs-apache-centos-redeploy.md](laravel-nextjs-apache-centos-redeploy.md).

---

## 0. What is different from the Ubuntu / nginx recipe

| Ubuntu + nginx | CentOS / RHEL + Apache | Consequence |
| --- | --- | --- |
| `apt` | `dnf` | CentOS 8 is end of life: its repos moved to `vault.centos.org` (section 2) |
| web user `www-data` | **`apache`** | every `sudo -u www-data` becomes `sudo -u apache`; files owned `apache:apache` |
| `/etc/nginx/sites-available` + symlink | **`/etc/httpd/conf.d/*.conf`**, loaded directly | no enabling step; a backup `.conf` left in `conf.d` is **loaded too**, keep backups in `/root` |
| certbot edits the server block | `certbot --apache` **creates a second file** `<site>-le-ssl.conf` for 443 | the 443 file is the live site; a `ProxyPass` added only to the port 80 file does nothing (section 7) |
| no SELinux | **SELinux Enforcing** | Apache cannot proxy to Node, and PHP cannot write `storage/`, until you allow it (section 6) |
| Supervisor | not in the base repos | run the queue worker as a **systemd** service (section 9) |
| `ufw` | `firewalld` | `firewall-cmd` (section 2) |
| glibc 2.35+ | **glibc 2.28** on EL8 | Next.js 16's native compiler needs 2.29: **build with `--webpack`** (section 8) |

`composer` as root stops at "Continue as root/super user?" and swallows the following pasted lines as answers.
Always run it as `COMPOSER_ALLOW_SUPERUSER=1 composer ...`.

---

## 1. Names used below

Set these once per SSH session (they are re-used by every block; re-run after a reconnect):

```bash
WEB=app.example.com            # Next.js site
API=api.example.com            # Laravel API
WEBDIR=/var/www/html/app       # Next.js folder
APIDIR=/var/www/html/api       # Laravel folder
WEBPORT=3000                   # pm2 port; one per Next.js app on the box
```

Before picking a port, check what is taken: `ss -ltnp | grep -E ':(3[0-9]{3}|80[0-9]{2})\b'`.

---

## 2. First look and packages

```bash
cat /etc/os-release | head -4
getenforce                                  # Enforcing
free -h && df -h /
systemctl is-active httpd php-fpm mysqld
ss -ltnp
ls /etc/httpd/conf.d/                       # what else is hosted here
```

**CentOS 8 only**: the mirrors are gone. If `dnf` fails with "Failed to download metadata", point it at the vault:

```bash
sed -i 's/^mirrorlist/#mirrorlist/' /etc/yum.repos.d/CentOS-*.repo
sed -i 's|^#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|' /etc/yum.repos.d/CentOS-*.repo
dnf clean all && dnf makecache
```

Packages (skip whatever the box already has; `php -v`, `httpd -v`, `mysql --version` tell you):

```bash
dnf install -y httpd mod_ssl git unzip curl policycoreutils-python-utils
dnf install -y epel-release
dnf install -y certbot python3-certbot-apache
dnf install -y mysql-server && systemctl enable --now mysqld
```

PHP 8.4 from Remi (EL8 ships 7.2/7.4/8.0 only). Install the version the project was developed on:

```bash
dnf install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm
dnf module reset -y php
dnf module enable -y php:remi-8.4
dnf install -y php php-cli php-fpm php-mysqlnd php-mbstring php-xml php-bcmath php-curl php-gd php-zip php-intl php-opcache
systemctl enable --now php-fpm httpd
php -v && ls /run/php-fpm/                  # www.sock
```

`/etc/httpd/conf.d/php.conf` (from the php package) hands every `.php` file to `proxy:unix:/run/php-fpm/www.sock`
for **all** sites. Nothing to add per site.

Composer:

```bash
curl -sS https://getcomposer.org/installer | php && mv composer.phar /usr/local/bin/composer
COMPOSER_ALLOW_SUPERUSER=1 composer -V
```

Node via nvm, then pm2 (Node 18+ official builds need glibc 2.28, which EL8 has):

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.nvm/nvm.sh
nvm install --lts && nvm alias default node
npm install -g pm2
node -v && pm2 -v
```

Firewall: only 80, 443 and SSH. Never open the Node, Reverb or MySQL ports.

```bash
firewall-cmd --permanent --add-service=http --add-service=https
firewall-cmd --reload
firewall-cmd --list-all
```

---

## 3. Code

```bash
cd /var/www/html
git clone -b main https://gitlab.com/group/api-repo.git "$APIDIR"
git clone -b main https://gitlab.com/group/web-repo.git "$WEBDIR"
git -C "$APIDIR" config core.fileMode false
git -C "$WEBDIR" config core.fileMode false
```

Change the repo URLs and branches before pasting. HTTPS clones ask for credentials on every pull; a read-only
deploy key (`ssh-keygen -t ed25519`, then GitLab > Settings > Repository > Deploy keys) avoids that:
[../linux/github-ssh-authentication.md](../linux/github-ssh-authentication.md).

`core.fileMode false` keeps a later `chmod` from making `git pull` refuse:
[../git/git-pull-refuses-after-chmod-file-mode.md](../git/git-pull-refuses-after-chmod-file-mode.md).

---

## 4. Database

```bash
mysql -u root -p          # or plain `mysql` if root uses socket auth on this box
```

```sql
CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'myapp_user'@'localhost' IDENTIFIED BY 'A_LONG_RANDOM_PASSWORD';
GRANT ALL PRIVILEGES ON myapp.* TO 'myapp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Generate the password (`openssl rand -base64 24`), never a short or tutorial-style default. Use `DB_HOST=localhost`: PHP
then talks over the MySQL socket and SELinux needs no database boolean. With `DB_HOST=127.0.0.1` (TCP) you must
also run `setsebool -P httpd_can_network_connect_db 1`.

---

## 5. Laravel

```bash
cd "$APIDIR"
cp .env.example .env
nano .env
```

Values that must not stay at their development defaults:

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://api.example.com
FRONTEND_URL=https://app.example.com
CORS_ALLOWED_ORIGINS=https://app.example.com,https://www.app.example.com
LOG_LEVEL=warning
DB_HOST=localhost
DB_DATABASE=myapp
DB_USERNAME=myapp_user
DB_PASSWORD="A_LONG_RANDOM_PASSWORD"
QUEUE_CONNECTION=database
```

`APP_DEBUG=true` on a public server prints the `.env` (DB and mail passwords) on any error page. A `.env` copied
from a laptop also keeps `localhost` CORS origins, and the browser then blocks every call from the real site.
`CORS_ALLOWED_ORIGINS` is our projects' own key, read by `config/cors.php`; check that file in a new project.

To change one value later without nano: `sed -i 's|^LOG_LEVEL=.*|LOG_LEVEL=warning|' .env`.

Install and set ownership:

```bash
cd "$APIDIR"
COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader
chown -R apache:apache storage bootstrap/cache
find storage bootstrap/cache -type d -exec chmod 775 {} \;
find storage bootstrap/cache -type f -exec chmod 664 {} \;
chown root:apache .env && chmod 640 .env
```

Folders 775 and files 664, **never `chmod -R 775`** (it marks every file executable and git sees it as a change).

Then the SELinux labels (section 6), and only then the artisan commands, **as `apache`**:

```bash
cd "$APIDIR"
sudo -u apache php artisan key:generate          # only on a new .env without APP_KEY
sudo -u apache php artisan migrate --force       # add --seed on the very first deploy if the project seeds
sudo -u apache php artisan storage:link
sudo -u apache php artisan optimize
sudo -u apache php artisan config:show app.url   # proves what the app loaded
```

Run artisan as root and it creates root-owned files in `storage/` that `apache` cannot write later: a bare 500 with
an empty body and nothing in `laravel.log`. Tinker needs a writable home: `sudo -u apache HOME=/tmp php artisan tinker`.

`optimize` caches config, so `env()` returns `null` anywhere outside `config/`. Check first:
`grep -rn "env(" app routes` must print nothing. Re-run `optimize` after **every** `.env` edit.

---

## 6. SELinux

Three things, once per app:

```bash
# Apache may open network connections: proxy to Node / Reverb, PHP to SMTP and external APIs
setsebool -P httpd_can_network_connect 1        # takes ~30 s

# PHP may write Laravel's two writable folders (default /var/www label is read-only)
semanage fcontext -a -t httpd_sys_rw_content_t "$APIDIR/storage(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "$APIDIR/bootstrap/cache(/.*)?"
restorecon -R "$APIDIR/storage" "$APIDIR/bootstrap/cache"
ls -Zd "$APIDIR/storage" "$APIDIR/bootstrap/cache"     # httpd_sys_rw_content_t
```

`semanage ... already defined` is harmless. `semanage: command not found`: `dnf install -y policycoreutils-python-utils`.

Is SELinux the reason something fails? It logs every refusal:

```bash
ausearch -m avc -ts recent | tail
getsebool -a | grep httpd_can
```

Never `setenforce 0` as the fix: it switches protection off for every site on the box.

---

## 7. Apache sites

`certbot --apache` works from port 80 sites and writes the 443 copy itself. So: write the two port 80 sites,
run certbot, then put the real content into the two `-le-ssl.conf` files it created.

### 7.1 Port 80 (before certbot)

```bash
cat > /etc/httpd/conf.d/$API.conf <<EOF
<VirtualHost *:80>
    ServerName $API
    DocumentRoot $APIDIR/public
</VirtualHost>
EOF

cat > /etc/httpd/conf.d/$WEB.conf <<EOF
<VirtualHost *:80>
    ServerName $WEB
    ServerAlias www.$WEB
    DocumentRoot /var/www/html
</VirtualHost>
EOF

apachectl configtest && systemctl reload httpd
```

(The heredocs are unquoted on purpose so the shell fills in `$API`, `$WEB` and the folders; nothing in them is an
Apache variable.)

### 7.2 Certificate

DNS `A` records for every name must already point at the server (`getent hosts $WEB $API`). Then, on its own line
(it asks for an email and the terms):

```bash
certbot --apache -d $WEB -d www.$WEB -d $API
```

Certbot creates `$WEB-le-ssl.conf` and `$API-le-ssl.conf` and adds an http → https redirect to the port 80 files.
Renewal: `systemctl status certbot-renew.timer` (or `certbot renew --dry-run`).

### 7.3 Port 443: the live sites

Back up, then overwrite the two files certbot made. The certificate paths are the ones `certbot certificates`
prints; one certificate can cover several names, so both sites may point at the same `live/<first-name>/` folder.

```bash
cp /etc/httpd/conf.d/$WEB-le-ssl.conf /etc/httpd/conf.d/$API-le-ssl.conf /root/
certbot certificates | grep -E 'Certificate Name|Domains|Certificate Path'
CERT=/etc/letsencrypt/live/$WEB          # the Certificate Path folder printed above
```

Next.js: everything is proxied, and there is **no `DocumentRoot`**, so Apache can never list the source files:

```bash
cat > /etc/httpd/conf.d/$WEB-le-ssl.conf <<EOF
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName $WEB
    ServerAlias www.$WEB
    ErrorLog /var/log/httpd/$WEB-error.log
    CustomLog /var/log/httpd/$WEB-requests.log combined

    ProxyPreserveHost On
    ProxyRequests Off
    ProxyPass / http://127.0.0.1:$WEBPORT/
    ProxyPassReverse / http://127.0.0.1:$WEBPORT/

    Include /etc/letsencrypt/options-ssl-apache.conf
    SSLCertificateFile $CERT/fullchain.pem
    SSLCertificateKeyFile $CERT/privkey.pem
</VirtualHost>
</IfModule>
EOF
```

Laravel: `DocumentRoot` is `public/`, and Laravel's own `public/.htaccess` does the routing (hence `AllowOverride All`):

```bash
cat > /etc/httpd/conf.d/$API-le-ssl.conf <<EOF
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName $API
    DocumentRoot $APIDIR/public

    <Directory $APIDIR/public>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
        CGIPassAuth On
    </Directory>

    ErrorLog /var/log/httpd/$API-error.log
    CustomLog /var/log/httpd/$API-requests.log combined

    Include /etc/letsencrypt/options-ssl-apache.conf
    SSLCertificateFile $CERT/fullchain.pem
    SSLCertificateKeyFile $CERT/privkey.pem
</VirtualHost>
</IfModule>
EOF

apachectl configtest && systemctl reload httpd
```

`CGIPassAuth On` passes the `Authorization: Bearer ...` header through to PHP (Sanctum tokens).

Never point `DocumentRoot` at the Laravel project root and patch it with a root `.htaccess` that rewrites to
`public/`: one mistake in that file and `.env` is downloadable.

### 7.4 Reverb (only if the API broadcasts with Reverb)

Inside the API's 443 `<VirtualHost>`, with Reverb on internal port 8080 (`proxy_wstunnel_module` ships with httpd):

```apache
    ProxyPass /app ws://127.0.0.1:8080/app
    ProxyPassReverse /app ws://127.0.0.1:8080/app
    ProxyPass /apps http://127.0.0.1:8080/apps
    ProxyPassReverse /apps http://127.0.0.1:8080/apps
```

Laravel has no route under `/app`, so nothing clashes. Run Reverb as a second systemd unit
like section 9 with `ExecStart=/usr/bin/php artisan reverb:start --host=127.0.0.1 --port=8080`. Env values and keys:
[../nginx/laravel-reverb-production-nginx.md](../nginx/laravel-reverb-production-nginx.md). (Not yet used on an
Apache box; the nginx note is the tested one.)

---

## 8. Next.js

```bash
cd "$WEBDIR"
nano .env.production                  # NEXT_PUBLIC_API_BASE_URL=https://api.example.com/api ...
npm ci
npx next build --webpack
pm2 start npm --name app-web -- start -- --port=$WEBPORT
pm2 save
pm2 startup                           # once per box; run the command it prints
curl -sI http://127.0.0.1:$WEBPORT | head -3
```

**Why `--webpack`.** Next.js 16 builds with Turbopack and a native compiler (`@next/swc-linux-x64-gnu`) that needs
glibc 2.29. EL8 has 2.28, so you get

```
Attempted to load @next/swc-linux-x64-gnu, but an error occurred: /lib64/libm.so.6: version `GLIBC_2.29' not found
```

With `--webpack`, Next falls back to its WASM compiler (downloaded once to `~/.cache/next-swc/`) and the build works,
only slower. Plain `npm run build` fails, **and a failed build empties `.next`**, so the running site breaks on the
next restart. Best fix: make it the project's script, `"build": "next build --webpack"` in `package.json`.

`NEXT_PUBLIC_*` values are baked in at build time: change them, rebuild.

Only one process may own the port. Two pm2 apps (or a pm2 app and a systemd unit) on the same port: one
crash-loops with `EADDRINUSE`. `pm2 status` and `ss -ltnp | grep :$WEBPORT` before starting anything.

---

## 9. Queue worker (systemd) and scheduler

Without Supervisor, systemd keeps the worker alive just as well:

```bash
cat > /etc/systemd/system/$API-worker.service <<EOF
[Unit]
Description=$API queue worker
After=network.target mysqld.service

[Service]
User=apache
Group=apache
WorkingDirectory=$APIDIR
ExecStart=/usr/bin/php artisan queue:work database --sleep=3 --tries=3 --max-time=3600
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now $API-worker
systemctl status $API-worker --no-pager | head -5
```

Logs: `journalctl -u $API-worker -n 50 --no-pager`. Restart after every deploy (it keeps old code in memory):
`systemctl restart $API-worker`. Need more workers? Copy the unit as `$API-worker2.service`.

Scheduler, in `apache`'s crontab (never root's):

```bash
(crontab -u apache -l 2>/dev/null; echo "* * * * * cd $APIDIR && php artisan schedule:run >> /dev/null 2>&1") | crontab -u apache -
crontab -u apache -l
sudo -u apache php "$APIDIR/artisan" schedule:list
```

---

## 10. Verify

```bash
systemctl is-active httpd php-fpm mysqld $API-worker
pm2 status
curl -s -o /dev/null -w "%{http_code} web\n"  https://$WEB
curl -s -o /dev/null -w "%{http_code} api\n"  https://$API
curl -s -o /dev/null -w "%{http_code} .env (must be 404)\n" https://$API/.env
curl -si -X OPTIONS -H "Origin: https://$WEB" -H "Access-Control-Request-Method: POST" https://$API/api/login | grep -i '^access-control-allow-origin'
ls -la "$APIDIR/storage/logs/"
```

`curl -sI https://$WEB` showing `Content-Type: text/html;charset=ISO-8859-1` is Apache's own **directory listing**,
not your app: the 443 site is not proxying (section 7.3).

Then log in through the browser.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Browser shows **"Index of /"** and the project's files | The 443 vhost (`-le-ssl.conf`) has a `DocumentRoot` and no `ProxyPass`. The proxy was added only to the port 80 file, which just redirects. Section 7.3. |
| `503 Service Unavailable` through Apache, but `curl http://127.0.0.1:3000` works | SELinux: `setsebool -P httpd_can_network_connect 1`. |
| `503` and the curl also fails | Next.js is down: `pm2 status`, `pm2 logs`. |
| `GLIBC_2.29 not found`, build fails | Build with `npx next build --webpack` (section 8). |
| pm2 app `errored`, `Could not find a production build in the '.next' directory` | Not built yet, or the last build failed. Build, then `pm2 restart`. |
| Bare 500, empty body, no `laravel.log` | Root-owned files in `storage`/`bootstrap/cache`: `chown -R apache:apache storage bootstrap/cache`. |
| `Permission denied` on `storage` with correct owner | SELinux label: `ls -Z storage`, section 6, `ausearch -m avc -ts recent`. |
| `SQLSTATE[HY000] [2002] Permission denied` | `DB_HOST=127.0.0.1` without the boolean: use `localhost`, or `setsebool -P httpd_can_network_connect_db 1`. |
| Mail fails with `Connection could not be established` | SELinux blocks outbound SMTP from PHP: `httpd_can_network_connect` (or `httpd_can_sendmail`). |
| Browser CORS error | `CORS_ALLOWED_ORIGINS` still lists localhost; fix, `sudo -u apache php artisan optimize`. |
| A `.env` change has no effect | Cached config: `optimize`, then `systemctl restart $API-worker`. |
| Pasted commands disappear after a `composer` line | Composer's root prompt ate them: `COMPOSER_ALLOW_SUPERUSER=1`. |
| `dnf`: `Failed to download metadata for repo 'appstream'` | CentOS 8 mirrors are gone: section 2, vault. |
| `git pull` refuses, nobody edited anything | Mode bits from a `chmod -R`: `git config core.fileMode false`. |
| An old site config keeps winning | A backup `.conf` inside `conf.d` is loaded too. Move backups to `/root`. `httpd -S` lists every vhost and the file it came from. |

---

## Security, once it works

- Root password logins on a public IP collect thousands of failed attempts. Use SSH keys, then
  `PasswordAuthentication no`, and fail2ban: [../linux/fail2ban-installation-and-ssh-protection.md](../linux/fail2ban-installation-and-ssh-protection.md)
  (on EL8 install it with `dnf install -y fail2ban` from EPEL).
- Change any seeded admin password whose value is in the repo.
- CentOS 8 gets no security updates. Plan a move to Rocky / Alma 8 or 9 (same commands), or to Ubuntu.
