# Deploy PostgreSQL + pgvector for a Laravel AI Project (Ubuntu VPS)

For Laravel AI / RAG projects (`laravel/ai`, embeddings, semantic search) the
database must be **PostgreSQL with the `pgvector` extension**. This stores
embeddings as a native `vector` column and powers similarity queries like
`whereVectorSimilarTo()`.

---

## First: Why PostgreSQL & pgvector Are NOT in `composer.json`

This confuses everyone. `composer install` ran fine and you saw no mention of
Postgres or pgvector — that is **expected**. They live at three different layers:

| Thing | What it is | Installed with |
|---|---|---|
| **PostgreSQL** | The database **server** | `apt` (system package) |
| **pgvector** | A **PostgreSQL extension** (adds the `vector` type) | `apt` + `CREATE EXTENSION vector` |
| **php8.4-pgsql** | The PHP **driver** that lets PHP talk to Postgres | `apt` (PHP extension) |
| **`laravel/ai`** | The PHP library that *uses* the above | ✅ `composer` — this one IS in your `composer.json` |

Composer only manages **PHP libraries**. It has no idea which database you use,
so `composer install` succeeds regardless. The failure shows up later at:

```bash
php artisan migrate
# SQLSTATE[08006] could not connect  -- or --  type "vector" does not exist
```

`laravel/ai` (`0.9.1` in your `composer.json`) is the package doing embeddings
and vector search — it expects the `vector` column type to already exist in the
database. That's what this guide sets up.

> **Nginx is not involved here.** Nginx serves HTTP; Postgres is a separate
> service your app connects to over a local socket/port. Your existing Nginx +
> PHP-FPM config needs no changes for this.

---

## Which PostgreSQL Version Should I Install?

PostgreSQL has **no "LTS" release** — that's a Ubuntu/Node concept. Instead,
**every major version gets 5 years of support** from its release date, and a new
major version ships each year (usually September). So "the LTS" is really just
"the current stable release", which gets the longest remaining support window.

**How to check what's current — official sources:**

| What | Link |
|---|---|
| Ubuntu install instructions (always current) | <https://www.postgresql.org/download/linux/ubuntu/> |
| Versioning + end-of-life dates per version | <https://www.postgresql.org/support/versioning/> |

**How to check what your server can actually install** — this is the
authoritative answer for your machine:

```bash
# Which PostgreSQL versions are offered?
apt-cache search '^postgresql-[0-9]+$'

# Which versions have a matching pgvector build?
apt-cache search 'postgresql-.*-pgvector'
```

> ⚠️ **The single rule that matters:** the PostgreSQL version number and the
> pgvector package number **must match**. `postgresql-18` + `postgresql-17-pgvector`
> gives you an extension PG 18 cannot load. Always install them as a matched pair.

This guide uses **18** (current stable as of writing). Substitute your version
number everywhere `18` appears if you install a different one.

---

## Step 1 — Install PostgreSQL

Ubuntu's default repo carries an older PostgreSQL. Use the official **PGDG**
repo so you get the current version with matching pgvector builds:

```bash
sudo apt update
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt update
sudo apt install -y postgresql-18
```

> Press `Enter` when the PGDG script asks to confirm.
> The `apt update` after adding the repo can be **slow** (a few minutes on a
> modest VPS) — `Waiting for headers` at low kB/s is normal, let it finish.

Verify it's running:

```bash
sudo systemctl status postgresql
psql --version
```

---

## Step 2 — Install the pgvector Extension

The package name follows the pattern `postgresql-<version>-pgvector`:

```bash
sudo apt install -y postgresql-18-pgvector
```

> No compiling needed — PGDG ships a prebuilt package. The version number
> **must match** the PostgreSQL you installed in Step 1.
>
> If `postgresql-18-pgvector` isn't found, PGDG hasn't published a build for
> that PG version yet — drop back to the newest version that *does* have one
> (check with `apt-cache search 'postgresql-.*-pgvector'`) and install that
> matched pair instead.

---

## Step 3 — Install the PHP PostgreSQL Driver

Without this, Laravel cannot connect at all (`could not find driver`):

```bash
sudo apt install -y php8.4-pgsql
sudo systemctl restart php8.4-fpm
```

Confirm PHP sees it:

```bash
php -m | grep pgsql
# pdo_pgsql
# pgsql
```

---

## psql Basics — Read This Before Step 4

Open the PostgreSQL shell as the admin user:

```bash
sudo -u postgres psql
```

```
psql (18.4 (Ubuntu 18.4-1.pgdg24.04+1))
Type "help" for help.

postgres=#
```

### ⚠️ Pressing Enter does NOT run your command

SQL statements can span multiple lines, so psql only executes when it sees a
**semicolon `;`**. Enter alone just moves to the next line and keeps waiting.

**Watch the prompt — it tells you the state:**

| Prompt | Meaning |
|---|---|
| `postgres=#` | **Ready** — waiting for a new command |
| `postgres-#` | **Still waiting** — you're mid-statement, it wants more input or a `;` |
| `jlosapi=#` | Ready, and you're connected to the `jlosapi` database |

If you get stuck on `-#`, type `;` + Enter to run it, or `Ctrl+C` to discard it.

### Two kinds of input

**1. SQL** — needs a trailing `;`

```sql
SELECT version();
CREATE DATABASE jlosapi;
```

**2. Backslash meta-commands** — run **immediately on Enter, no semicolon**

| Command | Meaning |
|---|---|
| `\l` | **L**ist all databases |
| `\c dbname` | **C**onnect / switch to a database |
| `\dt` | **D**escribe **t**ables in the current database |
| `\dx` | **D**escribe e**x**tensions installed (use this to verify pgvector) |
| `\du` | **D**escribe **u**sers (roles) |
| `\d tablename` | **D**escribe one table's columns |
| `\conninfo` | Show which database/user/port you're currently on |
| `\?` | Help — list all backslash commands |
| `\h CREATE TABLE` | **H**elp on a specific SQL statement's syntax |
| `\q` | **Q**uit psql |

> 📝 Shell commands like `ls`, `cd`, `ll` do **nothing** here — psql only accepts
> SQL and backslash commands. `ll` is not `\l`; the backslash is what matters.

---

## Step 4 — Create the Database and User

With psql open, run these one at a time. The **response after each line** is how
you know it worked:

```sql
CREATE DATABASE jlosapi;
-- CREATE DATABASE

CREATE USER jlosadmin WITH ENCRYPTED PASSWORD 'a-strong-password';
-- CREATE ROLE

GRANT ALL PRIVILEGES ON DATABASE jlosapi TO jlosadmin;
-- GRANT
```

Now **switch into the new database** — the next grants and the extension are
per-database, so this step is mandatory:

```sql
\c jlosapi
```

```
You are now connected to database "jlosapi" as user "postgres".
jlosapi=#
```

> The prompt changing from `postgres=#` to `jlosapi=#` is your confirmation.
> Everything after this point applies to `jlosapi`, not the default database.

```sql
GRANT ALL ON SCHEMA public TO jlosadmin;
-- GRANT
```

> ⚠️ On PostgreSQL 15+ the `GRANT ALL ON SCHEMA public` line is **required** —
> without it migrations fail with `permission denied for schema public`.

**Optional — not needed for Laravel:**

```sql
ALTER DATABASE jlosapi OWNER TO jlosadmin;
-- ALTER DATABASE
```

> The two `GRANT`s above are already enough: the user can create tables, and it
> automatically **owns every table it creates**, so migrations, rollbacks, and
> `migrate:fresh` all work. Database *ownership* only matters for
> `DROP DATABASE` / `ALTER DATABASE`, which Laravel never runs. Skip it unless
> you specifically want the app user to administer the database itself.

> ⚠️ **Avoid `#` and `$` in the password.** They're fine in psql, but `#` starts
> a comment in Laravel's `.env` parser — see Step 6.
> Also avoid naming the role `root`: PostgreSQL's `peer` auth maps OS users to
> same-named roles, so an OS root shell could then reach your app database
> without a password.

---

## Step 5 — Enable the `vector` Extension in Your Database

Extensions are installed **per database**, not server-wide. Make sure your
prompt still reads `jlosapi=#` before running this:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
-- CREATE EXTENSION
```

Confirm it's active:

```sql
\dx
```

Expected output — `vector` must appear in the list:

```
                                      List of installed extensions
  Name   | Version | Default version |   Schema   |                     Description
---------+---------+-----------------+------------+------------------------------------------------------
 plpgsql | 1.0     | 1.0             | pg_catalog | PL/pgSQL procedural language
 vector  | 0.8.5   | 0.8.5           | public     | vector data type and ivfflat and hnsw access methods
(2 rows)
```

> `ivfflat and hnsw access methods` in the description confirms the similarity
> **indexes** are available — that's what makes vector search fast at scale.

Exit psql:

```
\q
```

> 📝 The project is called **pgvector**, but the extension name is **`vector`**.
> `CREATE EXTENSION pgvector;` will fail.

---

## Step 5b — Verify the App User Can Log In

So far you've only connected as the `postgres` admin. Test the **app user** now,
so any password problem surfaces here rather than inside a failing migration:

```bash
psql -h 127.0.0.1 -U jlosadmin -d jlosapi -c "\dx"
```

Enter the password when prompted. Getting the extension table back means the
user, password, database, and pgvector are all confirmed working.

---

## Step 6 — Point Laravel at PostgreSQL

Edit your project's `.env`:

```dotenv
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=jlosapi
DB_USERNAME=jlosadmin
DB_PASSWORD="a-strong-password"
```

> ⚠️ **Quote the password if it contains `#`.** Laravel's dotenv parser treats
> `#` as the start of a comment, so `DB_PASSWORD=#@1jlosdb` is read as an
> **empty** password and you get `password authentication failed`. Wrapping it
> in double quotes — `DB_PASSWORD="#@1jlosdb"` — fixes it.

Clear cached config, then migrate:

```bash
php artisan config:clear
php artisan migrate --force
```

> `--force` is required in production (`APP_ENV=production`) — Laravel refuses
> destructive commands without it.

---

## Step 7 — Verify the Vector Setup Works

Check the connection end-to-end from Laravel:

```bash
php artisan tinker
```

```php
DB::select('select version()');
DB::select("select extname from pg_extension where extname = 'vector'");
```

The second query returning a row means pgvector is live and usable by your app.

---

## Step 8 — AI Provider Credentials

Vector search needs embeddings, which come from an AI provider. Make sure your
`.env` has the key your `laravel/ai` config expects, e.g.:

```dotenv
OPENAI_API_KEY=sk-...
# or ANTHROPIC_API_KEY=... depending on the configured provider
```

Then re-cache config for production:

```bash
php artisan config:cache
php artisan route:cache
```

---

## Already Installed 17? Moving to 18

Two situations — pick the one that matches you.

### Case A — Nothing important in the database yet (clean swap)

If you just installed 17 and haven't created real data, remove it entirely:

```bash
# Stop the service
sudo systemctl stop postgresql

# Purge PG 17 and its pgvector build (⚠️ deletes its databases)
sudo apt purge -y postgresql-17 postgresql-17-pgvector
sudo apt autoremove -y

# Remove leftover data directory
sudo rm -rf /var/lib/postgresql/17
sudo rm -rf /etc/postgresql/17

# Install the matched 18 pair
sudo apt install -y postgresql-18 postgresql-18-pgvector
sudo systemctl status postgresql
```

> ⚠️ `apt purge` on PostgreSQL **deletes the databases in that cluster.** Only
> do this if you have nothing to lose, or you've dumped it (Case B).

### Case B — You have data to keep (dump & restore)

Safest approach: back up, install 18 alongside, restore, then remove 17.

```bash
# 1. Dump everything from 17 (roles + all databases)
sudo -u postgres pg_dumpall > ~/pg17-backup.sql

# 2. Install 18 (it will auto-assign port 5433 since 17 holds 5432)
sudo apt install -y postgresql-18 postgresql-18-pgvector

# 3. Check which cluster is on which port
pg_lsclusters
```

Then either restore into 18 and drop 17:

```bash
# Restore into the 18 cluster (adjust port if needed)
sudo -u postgres psql -p 5433 -f ~/pg17-backup.sql

# Re-enable pgvector in the restored database
sudo -u postgres psql -p 5433 -d myapp -c "CREATE EXTENSION IF NOT EXISTS vector;"

# Verify data looks right, THEN remove 17
sudo pg_dropcluster --stop 17 main
sudo apt purge -y postgresql-17 postgresql-17-pgvector
```

Or use PostgreSQL's built-in upgrade helper, which moves the cluster for you:

```bash
sudo pg_dropcluster --stop 18 main        # remove 18's empty default cluster
sudo pg_upgradecluster 17 main            # migrate 17 → 18
sudo pg_dropcluster --stop 17 main        # remove old cluster once verified
```

### After Either Case — Point Things at 18

```bash
# Confirm 18 is on port 5432
pg_lsclusters
psql --version

# If the port changed, update .env accordingly, then:
php artisan config:clear
```

> `pg_lsclusters` is the command to remember — it shows every installed cluster,
> its version, port, and status in one table.

---

## Note on SSE Streaming (`ChatController@stream`)

This project streams AI responses via Server-Sent Events. Nginx buffers proxied
responses by default, which makes streamed tokens arrive all at once at the end.
The controller already sends `X-Accel-Buffering: no`, which Nginx respects — but
if streaming still feels buffered, add this to your site's PHP location block:

```nginx
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/run/php/php8.4-fpm.sock;

    fastcgi_buffering off;
    fastcgi_read_timeout 300;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## Troubleshooting

| Error | Cause / Fix |
|---|---|
| `could not find driver` | `php8.4-pgsql` not installed — Step 3, then restart PHP-FPM. |
| `type "vector" does not exist` | Extension not enabled **in that database** — rerun Step 5 while connected to it. |
| `CREATE EXTENSION pgvector` fails | Wrong name — use `CREATE EXTENSION vector;`. |
| `permission denied for schema public` | Missing the `GRANT ALL ON SCHEMA public` from Step 4 (PG 15+). |
| `SQLSTATE[08006] could not connect` | Postgres not running, or wrong `DB_HOST`/`DB_PORT`. Check `sudo systemctl status postgresql`. |
| `password authentication failed` | Wrong `DB_PASSWORD`, or `pg_hba.conf` needs `md5`/`scram-sha-256` for local TCP. |
| Migrations still use SQLite | `DB_CONNECTION` still `sqlite`, or stale cache — run `php artisan config:clear`. |
| Dimension mismatch on insert | Vector column size must match the embedding model (e.g. `1536` for `text-embedding-3-small`). |

---

## Quick Reference

| Task | Command |
|---|---|
| Add PGDG repo | `sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh` |
| See available PG versions | `apt-cache search '^postgresql-[0-9]+$'` |
| See available pgvector builds | `apt-cache search 'postgresql-.*-pgvector'` |
| List installed clusters + ports | `pg_lsclusters` |
| Install PostgreSQL | `sudo apt install -y postgresql-18` |
| Install pgvector | `sudo apt install -y postgresql-18-pgvector` |
| Upgrade a cluster in place | `sudo pg_upgradecluster 17 main` |
| Remove an old cluster | `sudo pg_dropcluster --stop 17 main` |
| Install PHP driver | `sudo apt install -y php8.4-pgsql` |
| Open psql as admin | `sudo -u postgres psql` |
| Open psql as app user | `psql -h 127.0.0.1 -U jlosadmin -d jlosapi` |
| Enable extension | `CREATE EXTENSION IF NOT EXISTS vector;` |
| Run migrations | `php artisan migrate --force` |
| Postgres service status | `sudo systemctl status postgresql` |

### Inside psql (no semicolon needed on these)

| Command | Meaning |
|---|---|
| `\l` | List all databases |
| `\c dbname` | Connect / switch to a database |
| `\dt` | List tables in the current database |
| `\dx` | List installed extensions (verify pgvector) |
| `\du` | List users / roles |
| `\d tablename` | Show a table's columns |
| `\conninfo` | Show current database, user, and port |
| `\?` | Help — all backslash commands |
| `\h SQL_STATEMENT` | Help on SQL syntax, e.g. `\h CREATE TABLE` |
| `\q` | Quit psql |

> Remember: **SQL needs a `;` to execute**; backslash commands run on Enter.
> Prompt `=#` means ready, `-#` means it's still waiting for a `;`.
