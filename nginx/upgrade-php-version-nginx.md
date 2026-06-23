# Upgrade PHP Version on Nginx (Ubuntu)

Useful when your Laravel/PHP app requires a newer PHP version than the one
shipped by Ubuntu. For example, Ubuntu 24.04 ships **PHP 8.3**, but a project
may require **8.4**, giving a `composer install` error like:

```
spatie/laravel-activitylog 5.0.0 requires php ^8.4 -> your php version (8.3.6) does not satisfy that requirement.
spatie/laravel-model-states 2.14 requires php ^8.4 -> your php version (8.3.6) does not satisfy that requirement.
```

The fix is to install the required PHP version, point Nginx at its FPM socket,
and switch the default CLI version.

> ⚠️ **Before troubleshooting versions, do NOT delete `composer.lock`.**
> The lock file pins exact, known-good versions. Deleting it turns
> `composer install` into `composer update` and pulls the **latest** of every
> package (often requiring an even newer PHP). If you already deleted it,
> restore it with:
>
> ```bash
> git checkout composer.lock
> ```

---

## Step 1 — Check Your Current PHP Version

```bash
php -v
```

Output example:

```
PHP 8.3.6 (cli) ...
```

---

## Step 2 — Add the ondrej/php PPA

Ubuntu's default repos only carry the one PHP version that shipped with the
release. The `ondrej/php` PPA carries all current PHP versions (8.1 → 8.4+).

```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

---

## Step 3 — Install the New PHP Version + Extensions

Install the FPM package (Nginx talks to PHP over FPM) plus the same extensions
your app uses. Replace `8.4` with whatever version you need:

```bash
sudo apt install -y php8.4-fpm php8.4-cli php8.4-mbstring php8.4-xml \
  php8.4-bcmath php8.4-curl php8.4-gd php8.4-zip php8.4-mysql
```

> Match this list to the extensions you installed for the old version. Common
> Laravel set: `mbstring xml bcmath curl gd zip mysql`. Add `php8.4-redis`,
> `php8.4-intl`, etc. if your app needs them.

---

## Step 4 — Make the New Version the Default CLI

This is what `php` (and therefore `composer`) uses on the command line:

```bash
sudo update-alternatives --set php /usr/bin/php8.4
php -v   # confirm it now reports 8.4
```

To pick interactively from all installed versions instead:

```bash
sudo update-alternatives --config php
```

---

## Step 5 — Point Nginx at the New FPM Socket

Each PHP version runs its own FPM service with its own socket:

```
/run/php/php8.3-fpm.sock
/run/php/php8.4-fpm.sock
```

Edit your site's server block (e.g. `/etc/nginx/sites-available/your-site`)
and update the `fastcgi_pass` line:

```nginx
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/run/php/php8.4-fpm.sock;   # was php8.3-fpm.sock
}
```

Test the config and reload:

```bash
sudo nginx -t
sudo systemctl restart php8.4-fpm nginx
```

---

## Step 6 — Disable the Old FPM Service

Stop the old PHP-FPM so it isn't running for nothing:

```bash
sudo systemctl disable --now php8.3-fpm
```

> Leave the old PHP packages installed if other sites still use 8.3.
> Only purge them once nothing depends on that version.

---

## Step 7 — Re-run Composer

```bash
git checkout composer.lock   # if you deleted it earlier
composer install
```

This should now resolve cleanly against the new PHP version.

---

## Verify the Web Server Uses the New Version

Create a temporary info file in your web root:

```bash
echo '<?php phpinfo();' | sudo tee /var/www/html/info.php
```

Visit `http://your-domain.com/info.php` and confirm the **PHP Version** at the
top reads `8.4.x`. Then delete it (don't leave `phpinfo()` exposed publicly):

```bash
sudo rm /var/www/html/info.php
```

---

## Quick Reference

| Task | Command |
|---|---|
| Check CLI version | `php -v` |
| Add PHP PPA | `sudo add-apt-repository ppa:ondrej/php -y && sudo apt update` |
| Install PHP 8.4 FPM | `sudo apt install -y php8.4-fpm php8.4-cli ...` |
| Set default CLI version | `sudo update-alternatives --set php /usr/bin/php8.4` |
| Choose version interactively | `sudo update-alternatives --config php` |
| Restart new FPM + Nginx | `sudo systemctl restart php8.4-fpm nginx` |
| Disable old FPM | `sudo systemctl disable --now php8.3-fpm` |
| List FPM sockets | `ls /run/php/` |
