# Increase PHP Upload Limit on Nginx

Useful when uploading large files (e.g. SQL dumps via FileBrowser) and hitting upload size or timeout errors.

---

## Step 1 — Find Your Active php.ini

```bash
php --ini | grep "Loaded Configuration File"
```

Output example:

```
Loaded Configuration File: /etc/php/8.3/fpm/php.ini
```

### Alternative: use `plocate`

```bash
sudo apt install plocate -y
locate php.ini
```

Output example:

```
/etc/php/8.3/cli/php.ini
/etc/php/8.3/cli/php.ini.save
/etc/php/8.3/fpm/php.ini
/usr/lib/php/8.3/php.ini-development
/usr/lib/php/8.3/php.ini-production
```

Use the **fpm** one — that is what Nginx uses:

```
/etc/php/8.3/fpm/php.ini
```

---

## Step 2 — Edit php.ini

```bash
sudo nano /etc/php/8.3/fpm/php.ini
```

### Search in nano

Press `CTRL + W`, type the directive name, press `Enter` to jump to it.
Press `ALT + W` to cycle through further matches.

### Update these values

```ini
max_input_time = 24000
max_execution_time = 24000
upload_max_filesize = 150M
post_max_size = 150M
memory_limit = 256M
```

> For very large databases you can go higher, e.g. `upload_max_filesize = 12000M`

Save and exit: `CTRL + X` → `Y` → `Enter`

---

## Step 3 — Edit nginx.conf

Nginx also has its own body size limit that must match or exceed PHP's limit.

```bash
sudo nano /etc/nginx/nginx.conf
```

Search for `types_hash_max_size` and add `client_max_body_size` directly below it:

```nginx
types_hash_max_size 2048;
client_max_body_size 150M;
```

Save and exit: `CTRL + X` → `Y` → `Enter`

---

## Step 4 — Restart PHP-FPM and Nginx

```bash
sudo systemctl restart php8.3-fpm
sudo systemctl restart nginx
```

---

## Verify the Change

```bash
php -r "echo ini_get('upload_max_filesize');"
php -r "echo ini_get('post_max_size');"
```

Test Nginx config before restarting:

```bash
sudo nginx -t
```

---

## Notes

- `upload_max_filesize` must be ≤ `post_max_size`
- `client_max_body_size` in Nginx must be ≥ `upload_max_filesize` in PHP
- Adjust the PHP version number (`8.3`) to match what your server runs: `php --version`

---

## Reference

- [YouTube walkthrough](https://youtu.be/NTD6-DeYC9I?si=O4yo0E8rHa3KQffe)
