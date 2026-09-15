# Upgrade PHP Version on Nginx (Ubuntu)

Useful when your Laravel/PHP app requires a **different** PHP version than the one
shipped by Ubuntu. Both directions happen:

- Ubuntu 24.04 ships **PHP 8.3**, but a project may require **8.4**:

```
spatie/laravel-activitylog 5.0.0 requires php ^8.4 -> your php version (8.3.6) does not satisfy that requirement.
spatie/laravel-model-states 2.14 requires php ^8.4 -> your php version (8.3.6) does not satisfy that requirement.
```

- Ubuntu 26.04 "Resolute" ships **PHP 8.5**, which is too NEW for a lock file built on 8.4
  (seen Sep 2026):

```
phpoffice/phpspreadsheet 1.30.4 requires php >=7.4.0 <8.5.0 -> your php version (8.5.4) does not satisfy that requirement.
```

Same fix either way: install the PHP version the project was developed on. Do not
`composer update` on the server to chase the OS's PHP; keep prod on the same PHP as dev.

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

## Step 2 — Add Ondřej Surý's PHP repository

Ubuntu's default repos only carry the one PHP version that shipped with the
release. Ondřej Surý's repository carries all current PHP versions (8.1 → 8.5).

### Ubuntu 22.04 Jammy / 24.04 Noble: the Launchpad PPA

```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

### Ubuntu 26.04 Resolute and newer: packages.sury.org (the PPA is being retired)

On Resolute `add-apt-repository ppa:ondrej/php` adds the source but `apt update` then fails
with `404 Not Found ... does not have a Release file`, and `apt install php8.4-fpm` says
`Unable to locate package`. The PPA no longer publishes for new releases; the canonical
source is now `packages.sury.org`.

```bash
# remove the dead PPA entry first, or every apt update keeps erroring
sudo rm /etc/apt/sources.list.d/ondrej-ubuntu-php-resolute.sources
sudo apt update

# add the sury repository and its signing key
sudo apt install -y lsb-release ca-certificates curl apt-transport-https
sudo curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
sudo dpkg -i /tmp/debsuryorg-archive-keyring.deb
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'
sudo apt update
```

> If sury does not have the new codename yet (404 on `resolute`), rerun the `sh -c` line
> with `noble` in place of `$(lsb_release -sc)`.

---

## Step 3 — Install the New PHP Version + Extensions

Install the FPM package (Nginx talks to PHP over FPM) plus the same extensions
your app uses. Replace `8.4` with whatever version you need:

```bash
sudo apt install -y php8.4-fpm php8.4-cli php8.4-mbstring php8.4-xml \
  php8.4-bcmath php8.4-curl php8.4-gd php8.4-zip php8.4-mysql php8.4-intl
```

> Match this list to the extensions you installed for the old version. Common
> Laravel set: `mbstring xml bcmath curl gd zip mysql intl`. Add `php8.4-redis`
> etc. if your app needs them.

Verified on Ubuntu 26.04 Resolute via packages.sury.org:

```
$ sudo update-alternatives --set php /usr/bin/php8.4
$ php -v
PHP 8.4.25 (cli) (built: Aug 28 2026 07:33:38) (NTS)
```

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
>
> When the OS shipped a NEWER version you are stepping down from (Resolute's 8.5), the same
> applies: `sudo systemctl disable --now php8.5-fpm`, and make sure no server block still
> points at `php8.5-fpm.sock`, or web requests run on 8.5 while the CLI runs on 8.4.

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
| Add PHP repo (Jammy/Noble) | `sudo add-apt-repository ppa:ondrej/php -y && sudo apt update` |
| Add PHP repo (Resolute+) | packages.sury.org keyring + `/etc/apt/sources.list.d/php.list` (Step 2) |
| Install PHP 8.4 FPM | `sudo apt install -y php8.4-fpm php8.4-cli ...` |
| Set default CLI version | `sudo update-alternatives --set php /usr/bin/php8.4` |
| Choose version interactively | `sudo update-alternatives --config php` |
| Restart new FPM + Nginx | `sudo systemctl restart php8.4-fpm nginx` |
| Disable old FPM | `sudo systemctl disable --now php8.3-fpm` |
| List FPM sockets | `ls /run/php/` |
