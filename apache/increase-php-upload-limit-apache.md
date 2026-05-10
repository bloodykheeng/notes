# Increase PHP Upload Limit on Apache

Useful when uploading large files (e.g. SQL dumps via FileBrowser) and hitting upload size or timeout errors.

---

## Edit php.ini

Find and open the Apache php.ini:

```bash
sudo nano /etc/php/<php_version>/apache2/php.ini
```

Example for PHP 8.3:

```bash
sudo nano /etc/php/8.3/apache2/php.ini
```

### Search in nano

Press `CTRL + W`, type the directive name, press `Enter` to jump to it.
Press `ALT + W` to cycle through further matches.

### Update these values

```ini
post_max_size = 150M
upload_max_filesize = 150M
```

Save and exit: `CTRL + X` → `Y` → `Enter`

---

## Restart Apache

```bash
sudo service apache2 restart
```

---

## Verify the Change

```bash
php -r "echo ini_get('upload_max_filesize');"
php -r "echo ini_get('post_max_size');"
```

---

## Notes

- `upload_max_filesize` must be ≤ `post_max_size`
- Changes only apply after Apache restarts
- If you have multiple PHP versions, make sure you edit the one Apache is actually using:
  ```bash
  php --version
  ```
