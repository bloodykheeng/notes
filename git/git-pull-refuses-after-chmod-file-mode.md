# `git pull` refuses on a server after `chmod -R` (file mode changes)

A deployed Laravel app that pulled fine for months suddenly refuses:

```
error: Your local changes to the following files would be overwritten by merge:
        storage/app/private/compendium_data/questions_2024.json
Please commit your changes or stash them before you merge.
Aborting
```

Nobody edited that file on the server. What changed is its **permission bits**, and git on
Linux tracks the executable bit.

---

## Why it happens

Every Laravel deploy guide says to make `storage` and `bootstrap/cache` writable, and the
command people reach for is:

```bash
sudo chmod -R 775 storage bootstrap/cache
```

`-R 775` sets the executable bit on every **file** as well as every folder. Any file git
tracks under those folders (`.gitignore` placeholders, seed fixtures, spreadsheets) goes from
mode `100644` to `100755`, and with git's default `core.fileMode true` each one now counts as
a local modification.

It stays invisible for as long as no commit touches those files. `git pull` only refuses when
an incoming commit changes a file you have "modified". The day somebody updates a seed
fixture in `storage/app`, the pull stops, and it looks like somebody edited the server.

## Why one server has it and another does not

Look at who you log in as.

- **A non-root user** (a VPS where you log in as `user` or `ubuntu`, and the app runs as
  `www-data`): both have to write to `storage`, you when you run commands and `www-data` when
  the app runs. Sooner or later something fails with a permission error, and the quick fix is
  group-writable permissions via `chmod -R 775`. That is the command that sets the bits.
- **Root** (Linode and other boxes that hand you root by default): root can write anywhere,
  nothing ever fails, nobody runs `chmod`, and the bits stay `100644`.

So the same app, deployed the same way, pulls fine as root and refuses on the non-root box.
The user is not what git checks, though: git compares permission bits whoever runs the pull,
and root would stop on the same files if the `chmod` had been run there. A problem that really
is about the user shows a different error, `Permission denied` or
`detected dubious ownership in repository`.

## Diagnose first: mode or content?

Never discard before looking. A real edit on a server is somebody's work.

```bash
cd /var/www/api.example.com
git status --short
git diff --stat
git diff <one of the files> | head
```

A mode-only change looks like this, with **no content lines at all**:

```
 storage/app/private/.../questions_2024.json | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
diff --git a/storage/... b/storage/...
old mode 100644
new mode 100755
```

If every listed file shows `old mode / new mode` and `0 insertions, 0 deletions`, it is
the `chmod`, and the fix below is safe. If any file shows `+` or `-` lines, stop: that is a
real edit, so find out whose it is before doing anything.

## Fix, once per folder

```bash
cd /var/www/api.example.com
git config core.fileMode false
git status --short          # must print nothing now
git pull
```

`core.fileMode false` tells this clone to ignore permission bits. The files keep the
permissions the app needs; only git's view of them changes. It is stored in the folder's
`.git/config`, so set it in each clone on the box (every API, test and prod).

The same check as one guarded block, which skips a folder with real edits instead of
overwriting them:

```bash
for d in testapi.example.com api.example.com; do
  cd /var/www/$d
  git config core.fileMode false
  if [ -n "$(git status --short)" ]; then
    echo "STOP: $d has real local edits:"; git status --short; continue
  fi
  git pull
done
```

## Prevent it: 775 on folders, 664 on files

Files never need the executable bit. Give folders `775` and files `664`: Laravel can write
exactly the same things, and git sees no change.

```bash
sudo chown -R www-data:www-data storage bootstrap/cache
sudo find storage bootstrap/cache -type d -exec chmod 775 {} \;
sudo find storage bootstrap/cache -type f -exec chmod 664 {} \;
```

`chown` never trips git (git does not track ownership), only `chmod` does.

## Related

- [../nginx/laravel-production-cache-permission-error-fix.md](../nginx/laravel-production-cache-permission-error-fix.md):
  ownership, the other half of storage permissions (root vs www-data).
- Seen on the SDS portal, 2026-09-29: both APIs on the Datanet VPS (worked as a non-root `user`)
  had it; the web apps there and the Linode staging box (worked as root) did not, because the
  `chmod` had only been run on the Datanet API folders.
