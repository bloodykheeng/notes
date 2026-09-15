# Deploy Laravel API + Next.js on one Ubuntu VPS, two environments (prod + test)

Copy-paste order for a fresh Ubuntu box hosting **two environments** of a Laravel API
(queue workers, scheduler, Reverb WebSockets) and a Next.js frontend (pm2), behind Nginx
with Let's Encrypt. Written from the Alpha Fund deploy (Hostinger VPS, Ubuntu 26.04, Sep 2026).

Replace the domains and folders once at the top and keep them consistent everywhere:

| | Production | Test |
| --- | --- | --- |
| Frontend | `invest.alphaeastafrica.com` | `testinvest.alphaeastafrica.com` |
| API | `investapi.alphaeastafrica.com` | `testinvestapi.alphaeastafrica.com` |
| Folder | `/var/www/<domain>` | `/var/www/<domain>` |
| Next.js port (pm2) | `3001` | `3000` |
| Reverb port (internal) | `8081` | `8080` |
| MySQL db / user | `alpha_fund` / `alpha` | `alpha_fund_test` / `alpha_test` |
| PHP-FPM socket | `/run/php/php8.4-fpm.sock` | same |

Rules that avoid every problem met so far:

- Everything Laravel runs as **`www-data`**: PHP-FPM, queue workers, Reverb, the cron scheduler.
  Root-created cache folders break `www-data` later (see
  [laravel-production-cache-permission-error-fix.md](laravel-production-cache-permission-error-fix.md)).
- Each environment gets its **own** MySQL user scoped to its own database, its own Reverb
  app id/key/secret, its own Reverb port, its own Next.js port.
- Install the PHP version the project was **developed on**, not whatever the OS ships.

---

## 1. DNS

`A` records for all four hosts (plus `www.` variants) pointing at the VPS IP. Verify from
your machine before touching the server:

```bash
nslookup investapi.alphaeastafrica.com 8.8.8.8
```

---

## 2. Server packages

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx mysql-server supervisor certbot python3-certbot-nginx git unzip curl ufw
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

### PHP 8.4 (the version the project uses)

Ubuntu 22.04 / 24.04:

```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

Ubuntu 26.04 and newer (the PPA is retired, use packages.sury.org):

```bash
sudo apt install -y lsb-release ca-certificates apt-transport-https
sudo curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
sudo dpkg -i /tmp/debsuryorg-archive-keyring.deb
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'
sudo apt update
```

Then:

```bash
sudo apt install -y php8.4-fpm php8.4-cli php8.4-mbstring php8.4-xml php8.4-bcmath \
  php8.4-curl php8.4-gd php8.4-zip php8.4-mysql php8.4-intl
sudo update-alternatives --set php /usr/bin/php8.4
php -v
```

If the OS shipped a different PHP (26.04 ships 8.5), stop its FPM so nothing picks it up:

```bash
sudo systemctl disable --now php8.5-fpm
```

### Composer, Node, pm2

```bash
curl -sS https://getcomposer.org/installer | php && sudo mv composer.phar /usr/local/bin/composer
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - && sudo apt install -y nodejs
sudo npm install -g pm2
```

---

## 3. Git access (deploy key)

```bash
ssh-keygen -t ed25519 -C "alpha server" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

Paste the public key in GitLab: Project > Settings > Repository > Deploy keys (read-only) for
**each** repository. Then:

```bash
ssh -T git@gitlab.com
```

---

## 4. Clone (four folders)

```bash
cd /var/www
git clone -b farouk git@gitlab.com:dbuwembo/fundapi.git   investapi.alphaeastafrica.com
git clone -b farouk git@gitlab.com:dbuwembo/fundapi.git   testinvestapi.alphaeastafrica.com
git clone -b farouk git@gitlab.com:dbuwembo/funddash.git  invest.alphaeastafrica.com
git clone -b farouk git@gitlab.com:dbuwembo/funddash.git  testinvest.alphaeastafrica.com
```

---

## 5. MySQL: one database + one user per environment

```bash
sudo mysql
```

```sql
CREATE DATABASE alpha_fund      CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE alpha_fund_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'alpha'@'localhost'      IDENTIFIED BY 'PROD_PASSWORD';
CREATE USER 'alpha_test'@'localhost' IDENTIFIED BY 'TEST_PASSWORD';

GRANT ALL PRIVILEGES ON alpha_fund.*      TO 'alpha'@'localhost';
GRANT ALL PRIVILEGES ON alpha_fund_test.* TO 'alpha_test'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Never point the app at `root`: Ubuntu's MySQL root uses `auth_socket`, so Laravel gets
`SQLSTATE[HY000] [1698] Access denied for user 'root'@'localhost'`.

---

## 6. Laravel: env, generated values, install

Do this once per API folder (`investapi...` with prod values, `testinvestapi...` with test values).

```bash
cd /var/www/testinvestapi.alphaeastafrica.com
cp .env.example .env
nano .env
```

Values to set (everything else per the project's `docs/environment-setup.md`):

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://testinvestapi.alphaeastafrica.com
FRONTEND_URL=https://testinvest.alphaeastafrica.com

DB_DATABASE=alpha_fund_test
DB_USERNAME=alpha_test
DB_PASSWORD=TEST_PASSWORD

QUEUE_CONNECTION=database
BROADCAST_CONNECTION=reverb
```

Reverb: generate a fresh id/key/secret **per environment** (do not reuse dev values):

```bash
echo "REVERB_APP_ID=$(shuf -i 100000-999999 -n 1)"
echo "REVERB_APP_KEY=$(openssl rand -hex 10)"
echo "REVERB_APP_SECRET=$(openssl rand -hex 10)"
```

```env
REVERB_APP_ID=<generated>
REVERB_APP_KEY=<generated>
REVERB_APP_SECRET=<generated>
REVERB_HOST="testinvestapi.alphaeastafrica.com"   # public: what browsers connect to
REVERB_PORT=443
REVERB_SCHEME=https
REVERB_SERVER_HOST=0.0.0.0                        # internal: where the process listens
REVERB_SERVER_PORT=8080                           # prod uses 8081
```

Install:

```bash
composer install --no-dev --optimize-autoloader
php artisan key:generate
php artisan storage:link
php artisan migrate --force
php artisan config:cache && php artisan route:cache && php artisan view:cache
```

Ownership (once, and again after any root-run command touched these folders):

```bash
sudo chown -R www-data:www-data /var/www/testinvestapi.alphaeastafrica.com/storage \
  /var/www/testinvestapi.alphaeastafrica.com/bootstrap/cache
sudo chmod -R 775 /var/www/testinvestapi.alphaeastafrica.com/storage \
  /var/www/testinvestapi.alphaeastafrica.com/bootstrap/cache
```

> `config:cache` is safe only if no app code calls `env()` directly (it returns `null`
> once config is cached). Check with `grep -rn "env('" app routes`; move any hits into a
> `config/*.php` file.

---

## 7. Next.js: env, build, pm2

Next.js reads **`.env.production`** at `npm run build` on every server, so on the test
server that file holds the **test** values. Do this per frontend folder.

```bash
cd /var/www/testinvest.alphaeastafrica.com
nano .env.production
```

```env
NEXT_PUBLIC_BASE_URL=https://testinvestapi.alphaeastafrica.com
NEXT_PUBLIC_API_BASE_URL=https://testinvestapi.alphaeastafrica.com/api
NEXTAUTH_URL=https://testinvest.alphaeastafrica.com
NEXTAUTH_SECRET=<npx auth secret>

NEXT_PUBLIC_REVERB_APP_ID=<same as the API .env>
NEXT_PUBLIC_REVERB_APP_KEY=<same as the API .env>
NEXT_PUBLIC_REVERB_APP_SECRET=<same as the API .env>
NEXT_PUBLIC_REVERB_HOST="testinvestapi.alphaeastafrica.com"
NEXT_PUBLIC_REVERB_PORT=443
NEXT_PUBLIC_REVERB_SCHEME=https
```

```bash
npm ci
npm run build
pm2 start npm --name alpha_test -- start                 # port 3000
```

Prod, on a different port:

```bash
cd /var/www/invest.alphaeastafrica.com
npm ci && npm run build
pm2 start npm --name alpha_prod -- start -- --port=3001  # port 3001
```

Survive reboots:

```bash
pm2 save
pm2 startup   # run the command it prints
```

---

## 8. Nginx

### API server block (per environment)

`sudo nano /etc/nginx/sites-available/testinvestapi.alphaeastafrica.com`

```nginx
server {
    server_name testinvestapi.alphaeastafrica.com www.testinvestapi.alphaeastafrica.com;

    root /var/www/testinvestapi.alphaeastafrica.com/public;
    index index.php index.html;
    client_max_body_size 50M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }

    # Reverb WebSocket: only /app and /apps go to the Reverb process.
    # proxy_pass port = this environment's REVERB_SERVER_PORT (prod 8081).
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
}
```

### Frontend server block (per environment)

`sudo nano /etc/nginx/sites-available/testinvest.alphaeastafrica.com`

```nginx
server {
    server_name testinvest.alphaeastafrica.com www.testinvest.alphaeastafrica.com;

    gzip on;
    gzip_proxied any;
    gzip_types application/javascript application/x-javascript text/css text/javascript;
    gzip_comp_level 5;
    gzip_buffers 16 8k;
    gzip_min_length 256;

    location /_next/static/ {
        alias /var/www/testinvest.alphaeastafrica.com/.next/static/;
        expires 365d;
        access_log off;
    }

    # proxy_pass port = this environment's pm2 port (prod 3001)
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable all four and reload:

```bash
sudo ln -s /etc/nginx/sites-available/investapi.alphaeastafrica.com     /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/testinvestapi.alphaeastafrica.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/invest.alphaeastafrica.com        /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/testinvest.alphaeastafrica.com    /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 9. SSL (one command, all hosts)

```bash
sudo certbot --nginx \
  -d invest.alphaeastafrica.com -d www.invest.alphaeastafrica.com \
  -d investapi.alphaeastafrica.com -d www.investapi.alphaeastafrica.com \
  -d testinvest.alphaeastafrica.com -d www.testinvest.alphaeastafrica.com \
  -d testinvestapi.alphaeastafrica.com -d www.testinvestapi.alphaeastafrica.com
```

Certbot edits the server blocks (443 + redirect). Renewal is automatic:
`sudo systemctl status certbot.timer`.

---

## 10. Supervisor: queue workers + Reverb (one file per process)

`sudo nano /etc/supervisor/conf.d/testinvestapi.alphaeastafrica.com-worker.conf`

```ini
[program:testinvestapi.alphaeastafrica.com-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/testinvestapi.alphaeastafrica.com/artisan queue:work database --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/var/www/testinvestapi.alphaeastafrica.com/storage/logs/worker.log
stopwaitsecs=3600
```

`sudo nano /etc/supervisor/conf.d/testinvestapi.alphaeastafrica.com-reverb.conf`

```ini
[program:testinvestapi.alphaeastafrica.com-reverb]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/testinvestapi.alphaeastafrica.com/artisan reverb:start --host=0.0.0.0 --port=8080
autostart=true
autorestart=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/testinvestapi.alphaeastafrica.com/storage/logs/reverb.log
stopwaitsecs=3600
```

Same two files for `investapi.alphaeastafrica.com` with `--port=8081`. Then:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl status
```

Manage:

```bash
sudo supervisorctl restart all
sudo supervisorctl restart testinvestapi.alphaeastafrica.com-worker:*
sudo supervisorctl restart investapi.alphaeastafrica.com-reverb:*
```

---

## 11. Scheduler (cron as www-data, never root)

```bash
sudo crontab -u www-data -e
```

```cron
* * * * * cd /var/www/investapi.alphaeastafrica.com && php artisan schedule:run >> /dev/null 2>&1
* * * * * cd /var/www/testinvestapi.alphaeastafrica.com && php artisan schedule:run >> /dev/null 2>&1
```

Make sure the same lines are **not** in `crontab -e` (root) or `sudo crontab -e`.

---

## 12. Verify

```bash
sudo supervisorctl status                       # all RUNNING
pm2 status                                      # alpha_prod + alpha_test online
sudo ss -ltnp | grep -E ':(3000|3001|8080|8081)' # four listeners
curl -I https://testinvestapi.alphaeastafrica.com/api/v1/health 2>/dev/null | head -1
```

Browser: open the frontend, log in, DevTools > Network > WS shows
`wss://testinvestapi.alphaeastafrica.com/app/<key>` with status 101.

---

## 13. Redeploy (every code change)

API:

```bash
cd /var/www/testinvestapi.alphaeastafrica.com
git pull
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan config:cache && php artisan route:cache && php artisan view:cache
sudo chown -R www-data:www-data storage bootstrap/cache
sudo supervisorctl restart testinvestapi.alphaeastafrica.com-worker:* testinvestapi.alphaeastafrica.com-reverb:*
```

The worker restart is not optional: a queue worker keeps the old code in memory until
restarted, which is the most common "my change is not working" on this stack.

Frontend:

```bash
cd /var/www/testinvest.alphaeastafrica.com
git pull
npm ci && npm run build
pm2 restart alpha_test
```

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `composer install`: `requires php ... your php version (8.5.x) does not satisfy` | OS PHP is newer than the lock file. Install the project's PHP version (section 2), `update-alternatives --set php`. Do not `composer update`. |
| `SQLSTATE[HY000] [1698] Access denied for user 'root'` | App is using MySQL root (auth_socket). Use the per-environment user (section 5). |
| `file_put_contents(.../storage/framework/cache...): Failed to open stream` | Something ran as root. Move cron to `www-data`, re-run the chown/chmod (section 6). |
| Supervisor `FATAL Exited too quickly` on the second Reverb | Both environments on the same `--port`. `grep -H command= /etc/supervisor/conf.d/*reverb*.conf`. |
| Realtime events never arrive | Worker not running or stale (`supervisorctl restart ...-worker:*`), or `BROADCAST_CONNECTION` not `reverb`. |
| Browser `WebSocket connection failed` | Nginx `/app` block missing the `Upgrade` headers, or `proxy_pass` port is the other environment's. |
| Push notifications silently stop after deploy | App code reads `env()` directly and `config:cache` is on. Move the values into `config/`. |
| Next.js shows the wrong API | `.env.production` on that server has the other environment's values; fix and `npm run build` again. |
