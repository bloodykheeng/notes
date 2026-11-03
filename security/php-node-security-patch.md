# Patching PHP and Node.js after a security release (server and Windows dev PC)

How to take a Laravel + Next.js stack onto patched PHP and Node.js when a security release comes out: first
on your own Windows PC (Laragon + nvm-windows), then on an Ubuntu server (nginx, PHP-FPM from Ondřej Surý's
repository, Node through nvm, pm2, Supervisor). Follow it top to bottom and copy-paste.

Giving the job to an AI agent instead? Hand it
[patch-php-node-with-an-ai-agent.md](patch-php-node-with-an-ai-agent.md): same steps, in rounds.

---

## 0. The release this was written for (October 2026)

On **21 Oct 2026** PHP fixed two critical holes, both **CVSS v4.0 9.2**:

| CVE | Advisory | In short |
| --- | --- | --- |
| CVE-2026-109618 | [GHSA-w8hw-w9fx-j79w](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w) | Running out of memory mid-request leaves half-built request data that cleanup then trusts |
| CVE-2026-109619 | [GHSA-6wmf-4rv5-p3rp](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp) | State from one request survives being freed and the next request on that worker uses it |

- **Every PHP that serves web requests**: PHP-FPM, Apache's PHP, `php-cgi`, and `php artisan serve` (the CLI's
  built-in server). Exploitable **without logging in, before any app code runs**, so Laravel cannot protect you.
- **Fixed in:** 8.5.12, **8.4.27**, 8.3.36, 8.2.35. **8.1 and older get no fix**: move off them.
- **Node.js** security releases the day before (20 Oct): **24.22.1** and **22.23.4**.

Sources: [Laravel response](https://laravel.com/blog/php-critical-vulnerability-response-updates),
[Laravel News](https://laravel-news.com/php-critical-rce-security-releases),
[Forge steps](https://laravel.com/forge/docs/knowledge-base/php-security-release-october-2026),
[php.net archive](https://www.php.net/archive/2026.php).

### The next time: find the fixed versions yourself

```bash
# PHP: newest release per branch (look for the security tag)
curl -s 'https://www.php.net/releases/index.php?json&max=8' | head -c 2000
# PHP for Windows: version, zip names and SHA-256 per build
curl -s https://downloads.php.net/~windows/releases/releases.json | head -c 3000
# Node.js: releases flagged "security": true, newest first
curl -s https://nodejs.org/dist/index.json | grep -o '"version":"v2[0-9.]*"[^}]*"security":true' | head
```

**Patch inside the line you already run** (8.4.5 to 8.4.27 is a patch, not an upgrade). Changing line (8.4 to
8.5, Node 22 to 24) is a separate, tested upgrade, and dev must match production.

---

## 1. Windows dev PC (Laragon)

Your PC serves web requests too: Laragon's Apache listens on the LAN, and `php artisan serve` (inside
`composer dev`) is a web server. Patch it first, it costs nothing.

### 1.1 See what you have

```powershell
Get-ChildItem C:\laragon\bin\php -Directory | ForEach-Object { "{0} -> {1}" -f $_.Name, (& "$($_.FullName)\php.exe" -r "echo PHP_VERSION;") }
(Get-Command php).Source; php -v | Select-Object -First 1
node -v; nvm list
[Environment]::GetEnvironmentVariable('Path','Machine') -split ';' | Where-Object { $_ -match 'php|node|nvm' }
[Environment]::GetEnvironmentVariable('Path','User')    -split ';' | Where-Object { $_ -match 'php|node|nvm' }
```

Note which folder is active, and whether the **Machine** or **User** PATH names a PHP folder by version (it
usually does, and Laragon does not update it when you switch, see 1.4).

### 1.2 Download the patched PHP and check it

Same line, same build type as the active folder: `nts` vs thread safe, and `vs17` vs `vs16`, as in its name.
Example for `php-8.4.5-nts-Win32-vs17-x64`:

```powershell
$v = '8.4.27-1'; $name = "php-$v-nts-Win32-vs17-x64.zip"
$dl = "$env:TEMP\php-dl"; New-Item -ItemType Directory -Force $dl | Out-Null
Invoke-WebRequest -UseBasicParsing "https://downloads.php.net/~windows/releases/$name" -OutFile "$dl\$name"
$want = ((Invoke-WebRequest -UseBasicParsing https://downloads.php.net/~windows/releases/releases.json).Content |
  ConvertFrom-Json).'8.4'.'nts-vs17-x64'.zip.sha256
(Get-FileHash "$dl\$name" -Algorithm SHA256).Hash.ToLower() -eq $want   # must print True
```

### 1.3 Install it beside the old one (old stays until the new one is proven)

```powershell
$old = 'C:\laragon\bin\php\php-8.4.5-nts-Win32-vs17-x64'
$new = 'C:\laragon\bin\php\php-8.4.27-nts-Win32-vs17-x64'
Expand-Archive "$dl\$name" -DestinationPath $new

# Extensions added by hand to the old one (not in the official zip): here imagick and its ImageMagick DLLs
Add-Type -AssemblyName System.IO.Compression.FileSystem
$inZip = [IO.Compression.ZipFile]::OpenRead("$dl\$name").Entries | ForEach-Object FullName
Get-ChildItem "$old\ext" -File | Where-Object { "ext/$($_.Name)" -notin $inZip } | Select-Object Name
```

Copy each extra the list shows. For imagick:

```powershell
Copy-Item "$old\ext\php_imagick.dll" "$new\ext\"
Get-ChildItem $old -File | Where-Object { $_.Name -match '^(CORE_RL_|IM_MOD_RL_|FILTER_).*\.dll$|^(ImageMagickObject|X11|Xext)\.dll$' } |
  Copy-Item -Destination $new

# Same php.ini, with paths that name the old folder (extension_dir) pointed at the new one
(Get-Content "$old\php.ini" -Raw).Replace((Split-Path $old -Leaf), (Split-Path $new -Leaf)) |
  Set-Content "$new\php.ini" -NoNewline

& "$new\php.exe" -v | Select-Object -First 1
Compare-Object (& "$old\php.exe" -m) (& "$new\php.exe" -m)   # must print nothing: same extensions
```

A copied extension must be built for the same line (an 8.4 DLL works across 8.4.x, not on 8.5).

### 1.4 Switch

1. Laragon: **Menu › PHP › Version › php-8.4.27-...** Apache restarts on it. Check:
   `(Invoke-WebRequest -UseBasicParsing http://localhost -Method Head).Headers['X-Powered-By']` says `PHP/8.4.27`.
2. **The PATH.** If 1.1 showed the old folder in the **Machine** or **User** PATH, every terminal outside Laragon
   keeps running the old PHP: switching the version changes Apache and Laragon's own terminal (Cmder), not that
   entry. Let Laragon rewrite it: **Menu › Tools › Path › Add Laragon to Path**. That updates the **User** PATH
   only. A PHP folder in the **Machine** PATH was put there by hand or by another installer, Laragon never
   touches it, and Windows reads the Machine PATH first, so it shadows Laragon's entry. Remove it (not replace:
   then Laragon's User entry covers every later switch). In an **admin** PowerShell:

   ```powershell
   $k = 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment'
   $raw = (Get-Item $k).GetValue('Path', $null, 'DoNotExpandEnvironmentNames')   # raw, keeps %SystemRoot% entries
   $raw | Out-File "$env:USERPROFILE\machine-path-backup.txt"                      # backup, to undo by hand
   $new = ($raw -split ';' | Where-Object { $_ -and $_ -notlike 'C:\laragon\bin\php\*' }) -join ';'
   Set-ItemProperty $k -Name Path -Value $new -Type ExpandString
   ($new -split ';') | Where-Object { $_ -match 'php' }                            # prints nothing
   ```

3. **Sign out of Windows and back in** (Laragon says so too: running programs, VS Code and its terminals keep
   the PATH they started with). Then `where.exe php` lists the new folder first and `php -v` says 8.4.27. Start
   `composer dev` / `queue:work` again.
4. Run the project's tests once. When all is well, delete the old folder.

Other PHP folders you no longer use (an old 7.4, a superseded 8.3.x): delete them, or patch them the same way if a
project still needs them. 7.4 has no fix.

### 1.5 Node.js (nvm-windows)

nvm-windows does not carry global packages over. Either `npm install -g` each one again on the new Node (newer
versions), or, within the same major version, **copy** them to keep the exact versions:

```powershell
npm ls -g --depth=0                                   # note the list
nvm install 22.23.4
$o = "$env:LOCALAPPDATA\nvm\v22.19.0"; $n = "$env:LOCALAPPDATA\nvm\v22.23.4"
foreach ($d in Get-ChildItem "$o\node_modules" -Directory | Where-Object Name -ne 'corepack') {
  robocopy $d.FullName "$n\node_modules\$($d.Name)" /MIR /NFL /NDL /NJH /NJS /NP | Out-Null }   # npm too: keeps your npm version
Get-ChildItem $o -File | Where-Object { -not (Test-Path "$n\$($_.Name)") } | Copy-Item -Destination $n   # claude.cmd, codex.cmd, ...
nvm use 22.23.4                                       # may ask for admin (it re-points C:\nvm4w\nodejs)
node -v; npm ls -g --depth=0                          # same list as before
```

**Except a tool something else has patched in place.** clodex's patch of Claude Code is tied to the install's
path: on the copied `claude.exe`, `clodex patch` refuses ("The patch manifest records a different Claude Code
install"). With clodex, install Claude Code and clodex **fresh** on the new Node and patch again; the full order
is [../clodex/update-procedure.md](../clodex/update-procedure.md#after-a-node-version-switch-nvm-use).

Restart the dev servers (`npm run dev`). Then `nvm uninstall 22.19.0`. "Error removing" / `node.exe` access
denied: a process started before the switch still runs the old `node.exe`. It shows as `C:\nvm4w\nodejs\node.exe`
with a start date before the switch (on 3 Nov 2026 a forgotten `relay.mjs --watch` from 15 Oct). Read its command
line, stop it, start it again if wanted, then delete the folder.

---

## 2. Ubuntu server (nginx, PHP-FPM, nvm, pm2, Supervisor)

Run as root, in bash, one block at a time; read each output before the next.

### 2.1 Read the server (changes nothing)

```bash
lsb_release -ds; uptime
echo "== PHP installed"; ls /etc/php
for v in $(ls /etc/php); do echo "-- $v"; dpkg -l "php$v-*" | awk '/^ii/ {print $2, $3}' | head -3; done
echo "== CLI"; php -v | head -1; update-alternatives --query php | grep -E 'Value|Status'
echo "== FPM services"; systemctl list-units --type=service --no-legend 'php*-fpm*'
echo "== Which PHP nginx sends to"; grep -rhR 'fastcgi_pass' /etc/nginx/sites-enabled/ /etc/nginx/conf.d/ | grep -v '#' | sed 's/^[ \t]*//' | sort | uniq -c
echo "== PHP source"; ls /etc/apt/sources.list.d/
echo "== Node"; source ~/.nvm/nvm.sh 2>/dev/null; nvm ls 2>/dev/null | head; which node; node -v
echo "== pm2"; pm2 list
for a in $(pm2 jlist | grep -o '"name":"[^"]*"' | cut -d'"' -f4 | sort -u); do echo "-- $a"; pm2 describe $a | grep -E 'script path|script args|exec cwd|node.js version'; done
echo "== pm2 boot unit"; systemctl cat pm2-root 2>&1 | grep -E 'Environment=PATH|ExecStart'
echo "== Supervisor"; supervisorctl status
echo "== Apache?"; systemctl is-active apache2 2>/dev/null
```

Write down: every PHP in `/etc/php`, which one nginx uses, the Node version, **each pm2 app's name, folder,
script and args**, and the Supervisor programs. You need them in 2.4.

### 2.2 Look for signs of a break-in FIRST (changes nothing)

Before patching and before any `git pull` (a pull changes file dates and hides what this looks for). Set the
folders and the date the holes became public:

```bash
APPS="/var/www/api.example.com /var/www/testapi.example.com"; WEBS="/var/www/example.com /var/www/test.example.com"; SINCE=2026-10-19
echo "== PHP changed since $SINCE (outside vendor and caches)"
for d in $APPS; do find $d -name '*.php' -newermt $SINCE -not -path '*/vendor/*' -not -path '*/storage/framework/*'; done | head -50
echo "== PHP files where uploads and public files live (should be none)"
for d in $APPS; do find $d/storage/app $d/public -name '*.ph*' -not -path '*/public/index.php'; done | head -20
echo "== Files edited or added on the server, outside git (should be none)"
for d in $APPS $WEBS; do echo "-- $d"; git -c safe.directory=$d -C $d status --short --untracked-files=all | head -15; done
echo "== Crontabs"; crontab -l; echo "-- www-data"; crontab -u www-data -l; ls -la /etc/cron.d
echo "== www-data processes that are not php-fpm or artisan"; ps -u www-data -o pid,etime,cmd | grep -vE 'php-fpm|artisan|nginx|grep'
echo "== Listening ports"; ss -ltnp
echo "== SSH logins since $SINCE"; journalctl -u ssh -u sshd --since $SINCE --no-pager | grep Accepted | awk '{print $1,$2,$3,$9,$11}' | tail -20
echo "== Failed passwords since $SINCE"; journalctl -u ssh -u sshd --since $SINCE --no-pager | grep -c 'Failed password'
echo "== Firewall"; ufw status | head -15
```

Notes from a real run (Ubuntu 26.04, Nov 2026):

- `grep -r` does **not** follow the links in `sites-enabled`: 2.1's nginx line printed nothing until it used `-R`.
- `last` is not installed on Ubuntu 26.04 (wtmp moved to `wtmpdb`): read logins from `journalctl` as above.
- **A deploy since `$SINCE` lists your own files** under "PHP changed since": that is fine *only if* `git status` is
  clean in that folder (every changed file is one git knows). An untracked or edited file is what a break-in leaves.
- `git` as root on a `www-data` folder refuses ("dubious ownership") without `-c safe.directory=...`.

Normal: the first two print nothing (or only files from your own recent deploy, see above); `git status` prints nothing per folder; crontabs hold only your
`schedule:run` lines and system/certbot entries; ports are nginx, node, MySQL (local), sshd and things you can
name; logins are people you know. **Anything else: stop.** Patching does not remove something already inside.

### 2.3 Patch PHP (every version in `/etc/php`)

```bash
apt-get update
for v in $(ls /etc/php); do echo "== $v"; apt-cache policy php$v-fpm php$v-cli | grep -E 'php|Installed|Candidate'; done
```

Each `Candidate` must be the fixed version or newer (`8.4.27-1+ubuntu24.04.1+deb.sury.org+1` and similar). If it
is older, the packages have not reached the repository yet: wait and look again. No `ondrej-php` / `php.list` in
`sources.list.d`: PHP came from elsewhere, stop and find out.

```bash
for v in $(ls /etc/php); do
  apt-get install --only-upgrade -y -o Dpkg::Options::="--force-confold" $(dpkg -l "php$v*" | awk '/^ii/ {print $2}')
done
```

`--only-upgrade` touches only what is installed; `--force-confold` keeps your `php.ini` and pool files.
**FPM restarts: the APIs drop for a few seconds.**

Then the long-running PHP, which keeps the old binary until restarted:

```bash
for s in $(systemctl list-units --type=service --no-legend 'php*-fpm*' | awk '{print $1}'); do systemctl restart $s; echo "$s $(systemctl is-active $s)"; done
supervisorctl restart '<api>-worker:*' '<api>-reverb:*' '<testapi>-worker:*' '<testapi>-reverb:*'   # your programs, by name
supervisorctl status
php -v | head -1
for u in https://api.example.com https://testapi.example.com; do curl -s -o /dev/null -w "%{http_code} $u\n" $u; done
```

Every `-v` says the fixed version, FPM `active`, workers `RUNNING`, the APIs answer as before (Laravel's
`https://api.example.com/up` health route proves PHP answers, not only nginx). The cron scheduler starts a fresh
PHP every minute, so it needs nothing. Apache with `libapache2-mod-php`: `systemctl restart apache2`.

**Before any Supervisor restart, check each program's `stopwaitsecs`** (`grep -h stopwaitsecs
/etc/supervisor/conf.d/*.conf`). `supervisorctl restart` stops *every* named program before it starts any, and waits
up to `stopwaitsecs` for each. Queue workers need a long one (3600) to finish a job; **Reverb (or any websocket
server) does not**, and one copied from the worker config hangs the whole restart: on 3 Nov 2026 a Reverb that ignored
the stop signal held the restart for over ten minutes with production's workers already stopped. Give Reverb
`stopwaitsecs=10`:

```bash
f=/etc/supervisor/conf.d/<api>-reverb.conf
cp "$f" "$f.bak-$(date +%F)" && sed -i 's/^stopwaitsecs=3600$/stopwaitsecs=10/' "$f" && grep stopwaitsecs "$f"
supervisorctl reread
timeout 30 supervisorctl update; echo "exit $?"     # timeout: this terminal waits 30 s at most
supervisorctl status | grep reverb
```

**If a restart already hangs:** never Ctrl+C it. Open a second SSH session and read `supervisorctl status`: the
program shown `STOPPING` is the one it waits on. `supervisorctl signal KILL '<program>:*'` makes it stop, and the
waiting restart carries on by itself.

### 2.4 Patch Node.js (nvm + pm2)

pm2 apps started with `pm2 start npm ...` keep the **full path of the old Node** (`script path` in 2.1).
Installing a new Node does not move them: each app is deleted and started again. Each site is down a few seconds.
The existing `.next` builds keep working within the same major version.

```bash
source ~/.nvm/nvm.sh
nvm install 24.22.1 --reinstall-packages-from=24.15.0      # your old version after "from"
nvm alias default 24.22.1
node -v; which pm2
pm2 update
```

`pm2 update` restarts pm2's daemon on the new Node and brings the apps back (still on the old Node).
**Never press Ctrl+C after `pm2 update`, even if the screen looks stuck.** It ends with `>>>>>>>>>> PM2 updated`
and can swallow the next pasted line. Press Enter once and wait. A Ctrl+C there once killed the new daemon and
every web app with it (502 until restarted). **Nor a Ctrl+C at the prompt "to clear it" before pasting the next
block:** on 3 Nov 2026 exactly that left both sites on 502 for minutes (the next command printed `Spawning PM2
daemon`: none was running). Just paste.

`pm2 update` may also end with an **empty table** even though it printed `restored` for each app. Do not `pm2 save`
then (it would save the empty list, and the next reboot would start nothing); go straight to the next block.

Start each app again from the new Node, exactly as 2.1 recorded it (name, folder, args):

```bash
source ~/.nvm/nvm.sh
pm2 delete web-test && cd /var/www/test.example.com && pm2 start npm --name web-test -- start -- --port=3001
pm2 delete web-prod && cd /var/www/example.com      && pm2 start npm --name web-prod -- start
for a in web-test web-prod; do pm2 describe $a | grep -E 'script path|script args|node.js version'; done
# Save the list only when every app is online: it is what pm2 restores at boot.
if [ "$(pm2 jlist | grep -o '"status":"online"' | wc -l)" -eq 2 ]; then pm2 save; else echo "NOT SAVED: not all online"; fi
for u in https://test.example.com https://example.com; do curl -s -o /dev/null -w "%{http_code} $u\n" $u; done
```

(An app started from an `ecosystem.config.js`: `pm2 delete <name> && pm2 start ecosystem.config.js --only <name>`.
`pm2 delete <name> 2>/dev/null; ...` instead of `&&` when the list may be empty after `pm2 update`.)

The boot unit also holds the old Node's path. Rewrite it (this does not touch the running apps):

```bash
source ~/.nvm/nvm.sh
pm2 startup systemd
systemctl daemon-reload
systemctl is-enabled pm2-root
systemctl cat pm2-root | grep -E 'Environment=PATH|ExecStart'     # shows the new version
```

`systemctl is-active pm2-root` saying `inactive` is expected while pm2 runs from your shell. **Do not
`systemctl start pm2-root` now**: it would start every app a second time. It takes over at the next reboot.

Keep the old Node until everything is checked, then `nvm uninstall 24.15.0`.

### 2.4b Reboot if the kernel asks, and the OS's own security updates

`System restart required` at login (`cat /var/run/reboot-required.pkgs` names the kernel) is a good moment once 2.4 is
done: the reboot also proves the boot unit brings every app back by itself. Check everything starts at boot first:

```bash
systemctl is-enabled nginx mysql php8.4-fpm supervisor pm2-root cron     # all "enabled"
reboot
```

Log in again after a minute or two: `uname -r` the new kernel, `systemctl is-active pm2-root` **active** (systemd
started pm2 this time), every app online on the new Node, Supervisor `RUNNING`, sites 200.

Then the OS security updates (`apt list --upgradable | grep -i security`). Install only those packages
(`apt-get install --only-upgrade -y <packages>`); a library the web stack loads at start (`libpcre2-8-0`, `libssl`)
needs nginx, PHP-FPM and the Supervisor programs restarted (the stopwaitsecs warning in 2.3 applies).

### 2.5 Write it down

In the project's docs (e.g. `SECURITY-<date>.md`): versions before and after, the break-in check result, anything
odd, and every outage with its cause. The next person starts from there. Alpha's of 3 Nov 2026 is a full example.

---

## 3. Gotchas

| Symptom | Cause | Fix |
| --- | --- | --- |
| `php -v` still old on Windows after switching in Laragon | A PHP folder in the Machine PATH shadows Laragon's User PATH entry; open programs keep their PATH | 1.4 step 2 (remove the Machine entry), then sign out and in |
| `Unable to load dynamic library 'imagick'` on the new PHP | Hand-added extension not copied, or built for another line | 1.3: copy the DLL and its libraries; same line only |
| Queue jobs behave as before the patch | Workers still on the old PHP | `supervisorctl restart ...` |
| Web app 502 after the Node upgrade | pm2 apps killed (Ctrl+C after `pm2 update`, even at an empty prompt) or still pointing at a removed Node | 2.4: start each app again, `pm2 save` once all are online |
| `pm2 update` ends with an empty table | The new daemon did not take the apps over | Do not `pm2 save`; start each app again (2.4) |
| `supervisorctl restart` prints nothing for minutes, some programs `STOPPED`, one `STOPPING` | A program ignores the stop signal and its `stopwaitsecs` is long (Reverb copied from the workers' 3600) | Second session: `supervisorctl signal KILL '<program>:*'`; then give it `stopwaitsecs=10` (2.3) |
| 2.1 says nginx uses no PHP | `grep -r` skips the symlinks in `sites-enabled` | `grep -rhR` |
| `last: command not found` (Ubuntu 26.04) | wtmp moved to `wtmpdb` | `journalctl -u ssh -u sshd \| grep Accepted` |
| Apps come up twice | `systemctl start pm2-root` while pm2 already runs | `pm2 kill`, then `systemctl start pm2-root` |
| `apt` offers nothing newer | Packages not in the repository yet | Wait, `apt-get update`, look again |
