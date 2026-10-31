# Redeploy Laravel API + Next.js on CentOS / RHEL with Apache

Every code change, after the first deploy in
[laravel-nextjs-apache-centos-deploy.md](laravel-nextjs-apache-centos-deploy.md). Run as root, in order.

```bash
WEB=app.example.com            # same names as the deploy note
API=api.example.com
WEBDIR=/var/www/html/app
APIDIR=/var/www/html/api
APIBRANCH=main
WEBBRANCH=main
WEBAPP=app-web                 # pm2 name
```

---

## 1. API

```bash
cd "$APIDIR"
sudo -u apache php artisan down --retry=60
git pull origin $APIBRANCH
COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader
sudo -u apache php artisan migrate --force
sudo -u apache php artisan optimize
chown -R apache:apache storage bootstrap/cache
sudo -u apache php artisan up
systemctl restart $API-worker
```

- `composer install` and `migrate`: skip them when the pull changed no `composer.lock` and brought no migration.
  `git diff --stat HEAD@{1} -- composer.lock database/migrations` tells you.
- The `chown` after composer is not optional: composer (root) rewrites `bootstrap/cache/packages.php` as root.
- The worker restart is not optional: `queue:work` keeps the old code in memory.
- New permissions or roles in a seeder: `sudo -u apache php artisan db:seed --class=RolesAndPermissionsSeeder --force`
  (only if that seeder is safe to repeat), then `sudo -u apache php artisan permission:cache-reset` for Spatie.

## 2. Web

```bash
cd "$WEBDIR"
git pull origin $WEBBRANCH
npm ci
npx next build --webpack
pm2 restart $WEBAPP
```

- **`--webpack` every time** on EL8 (glibc 2.28). A plain `npm run build` fails and leaves `.next` empty, which
  takes the site down at the next restart. If the project's `build` script already says `next build --webpack`,
  `npm run build` is fine.
- Skip `npm ci` when `package-lock.json` did not change.
- Low memory (`Killed` during the build): stop the heavy apps first, build, start them again. Check with `free -h`.

## 3. Check

```bash
systemctl is-active httpd php-fpm $API-worker
pm2 status
curl -s -o /dev/null -w "%{http_code} web\n" https://$WEB
curl -s -o /dev/null -w "%{http_code} api\n" https://$API
tail -n 20 "$APIDIR/storage/logs/laravel.log" 2>/dev/null
journalctl -u $API-worker -n 20 --no-pager
```

Then log in through the browser and open a page that loads data.

---

## When `git pull` refuses

"Your local changes to the following files would be overwritten by merge":

```bash
git diff --stat
git diff | head -40
```

- Only `old mode 100644 / new mode 100755` lines: a `chmod -R` changed permission bits.
  `git config core.fileMode false`, pull again.
- Real content changes: somebody edited on the server. Find out whose before `git stash` / `git checkout`.

## When something broke after the deploy

| Symptom | Fix |
| --- | --- |
| Site shows "Index of /" | The 443 vhost lost its `ProxyPass` (certbot re-ran?). Restore it from the deploy note, section 7.3. |
| 503 on the web site | `pm2 logs $WEBAPP --lines 30 --nostream`; a failed build shows `Could not find a production build`. Rebuild with `--webpack`. |
| Bare 500 from the API | `ls -la storage/logs bootstrap/cache`: anything root-owned, `chown -R apache:apache storage bootstrap/cache`. |
| Change not visible in the API | `sudo -u apache php artisan optimize` and `systemctl restart $API-worker`. |
| Put it back as it was | `git log --oneline -5` to see the commits, `git checkout HEAD~1` (one commit back), then that folder's steps above; for the web, rebuild. Return with `git checkout $APIBRANCH` (or `$WEBBRANCH`) once fixed. |
