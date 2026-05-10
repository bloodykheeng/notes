# Export & Import MySQL Database Between Servers via FileBrowser

Workflow: export dump inside your project folder → download via FileBrowser on Windows → upload to same path on target server → import into MySQL.

---

## Step 1 — SSH into Source Server & Go to Project

```bash
ssh ubuntu@api.devnicetwo.nice.co.ug
cd /var/www/api.devnicetwo.nice.co.ug
```

---

## Step 2 — Create `db-extracts` Folder

```bash
mkdir -p db-extracts
cd db-extracts
```

---

## Step 3 — Install `pv` (if not installed)

```bash
sudo apt update && sudo apt install pv -y
```

---

## Step 4 — Export the Database

### Basic export with progress

```bash
mysqldump -u dbuser -p dbname | pv > dbname.sql
```

### Compressed export (recommended for large databases)

```bash
mysqldump -u dbuser -p dbname | pv | gzip > dbname.sql.gz
```

### Production-safe export (includes routines, triggers, no table locks)

```bash
mysqldump \
  -u dbuser \
  -p \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  dbname | pv > dbname.sql
```

You should now have the file at:
```
/var/www/api.devnicetwo.nice.co.ug/db-extracts/dbname.sql
```

---

## Step 5 — Download via FileBrowser (on Your Windows Browser)

1. Open FileBrowser for the **source server** in your Windows browser
   - e.g. `https://api.devnicetwo.nice.co.ug/files`
2. Navigate to:
   ```
   /var/www/api.devnicetwo.nice.co.ug/db-extracts/
   ```
3. Click `dbname.sql` → click **Download**
4. File saves to your Windows `Downloads` folder

---

## Step 6 — Upload via FileBrowser to Target Server

1. Open FileBrowser for the **target server** in your Windows browser
   - e.g. `https://api.devthree.nice.co.ug/files`
2. Navigate to the same project path:
   ```
   /var/www/api.devthree.nice.co.ug/db-extracts/
   ```
   Create the folder first if it doesn't exist (FileBrowser has a **New Folder** button)
3. Click **Upload** → select `dbname.sql` from your Windows Downloads
4. Wait for upload to complete

---

## Step 7 — SSH into Target Server & Import

```bash
ssh ubuntu@api.devthree.nice.co.ug
cd /var/www/api.devthree.nice.co.ug/db-extracts
```

### Create the database if it doesn't exist

```bash
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS dbname CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

### Import with progress

```bash
pv dbname.sql | mysql -u dbuser -p dbname
```

### Import compressed file

```bash
pv dbname.sql.gz | gunzip | mysql -u dbuser -p dbname
```

---

## Step 8 — Verify

```bash
mysql -u dbuser -p dbname -e "SHOW TABLES;"
```

Check row count on a key table:

```bash
mysql -u dbuser -p dbname -e "SELECT COUNT(*) FROM users;"
```

---

## Cleanup (Optional)

Remove the dump after confirming the import is good:

```bash
# On source server
rm /var/www/api.devnicetwo.nice.co.ug/db-extracts/dbname.sql

# On target server
rm /var/www/api.devthree.nice.co.ug/db-extracts/dbname.sql
```

---

## Troubleshooting

### Access denied on import
```bash
mysql -u root -p -e "GRANT ALL PRIVILEGES ON dbname.* TO 'dbuser'@'localhost' IDENTIFIED BY 'yourpassword'; FLUSH PRIVILEGES;"
```

### Large file upload times out in FileBrowser / Nginx
Add this to your Nginx server block:
```nginx
client_max_body_size 2G;
```
Then reload:
```bash
sudo nginx -t && sudo systemctl reload nginx
```

### Unknown character set error
```bash
pv dbname.sql | mysql --default-character-set=utf8mb4 -u dbuser -p dbname
```

---

## Quick Reference

| Step | Command |
|---|---|
| Go to project | `cd /var/www/api.devnicetwo.nice.co.ug` |
| Create dump folder | `mkdir -p db-extracts && cd db-extracts` |
| Export (plain) | `mysqldump -u user -p db \| pv > db.sql` |
| Export (compressed) | `mysqldump -u user -p db \| pv \| gzip > db.sql.gz` |
| Import (plain) | `pv db.sql \| mysql -u user -p db` |
| Import (compressed) | `pv db.sql.gz \| gunzip \| mysql -u user -p db` |
| Verify | `mysql -u user -p db -e "SHOW TABLES;"` |
