# Deploying with an AI agent: the playbook

**For the AI agent reading this.** This is how I deploy and redeploy my projects (usually a Laravel API + a
Next.js frontend) with an AI agent's help. Follow it from top to bottom. It was distilled from deploys that went
fast and clean (two environments on one Ubuntu box with nginx; a CentOS 8 box with Apache), and from the mistakes
made on the way.

**How to use it (for me).** Start a session in the project's workspace and say:

> Read `https://github.com/bloodykheeng/notes/blob/main/ai-agents/deploy-with-an-ai-agent.md` (or the local copy
> in my coding-notes) and follow it. We are deploying `<project>` to `<server / domains>`.

---

## 1. The working model

- **You do not have the server. I do.** You write the commands; I run them over SSH and paste back everything the
  terminal printed. You read the output, decide, and write the next commands. Never claim something is done on the
  server until its output shows it.
- **Unless I hand you the login.** When I say the SSH login is in a txt file, you run the deploy yourself: read
  [deploy-yourself-over-ssh.md](deploy-yourself-over-ssh.md) for how (password helper, secrets file to file,
  background installs, DNS checks, clean-up). Everything else in this playbook still applies.
- **You do have the code.** The repos are on my machine in your workspace: read them yourself instead of asking me.
- **Rounds.** Every round is: a few short blocks of commands → I paste the output → you report what it shows (good
  and bad) → the next blocks. Read-only checks first, changes after, a verification line after every change.
- **Plain explanations.** Say what each block does and what output to expect, so I can spot a problem before
  pasting it back.
- **The recipes are your source.** Before writing a single command, read the recipe that matches the server:

  | Server | Recipe (in this repo) |
  | --- | --- |
  | Ubuntu + nginx (the default) | [../nginx/laravel-nextjs-two-environments-deploy.md](../nginx/laravel-nextjs-two-environments-deploy.md), with [../nginx/laravel-reverb-production-nginx.md](../nginx/laravel-reverb-production-nginx.md) if it broadcasts |
  | CentOS / RHEL + Apache (httpd) | [../centos/laravel-nextjs-apache-centos-deploy.md](../centos/laravel-nextjs-apache-centos-deploy.md) and [../centos/laravel-nextjs-apache-centos-redeploy.md](../centos/laravel-nextjs-apache-centos-redeploy.md) |
  | Anything else / adding an app to a busy box | the closest recipe above, adapted; say what you changed and why |

  Mail, Firebase, Google sign-in and reCAPTCHA values:
  [../integrations/google-workspace-mail-firebase-oauth-recaptcha.md](../integrations/google-workspace-mail-firebase-oauth-recaptcha.md).
  A client with no paid email (app sending + an inbox for free):
  [../integrations/free-email-with-maildog.md](../integrations/free-email-with-maildog.md).

---

## 2. Phase 0: study the project (on my machine, before anything on the server)

Read, do not ask:

- `README`, `CLAUDE.md` / `AGENTS.md`, any existing `docs/deployment*.md` (a redeploy may already be written down).
- **Laravel:** `composer.json` + `composer.lock` (the PHP version it was built on: install exactly that),
  `.env.example`, `config/` (queue, broadcasting, cors, mail, filesystems), `routes/console.php` or the scheduler
  (does it need cron?), seeders (which accounts and roles exist, which seeders are safe to re-run).
  Run `grep -rn "env(" app routes`: anything found makes `config:cache` unsafe until it moves into `config/`.
- **Next.js:** `package.json` (Next and Node versions, the `build` script), `.env*` files, which `NEXT_PUBLIC_*`
  values it needs (API URL, Reverb key/host, Firebase ...).
- Uploads (size limits for nginx `client_max_body_size` and PHP), payment or webhook callbacks (they need the
  public HTTPS API address), outgoing mail/SMS (a real gateway costs money: who turns it on?).

Then give me a short summary: what has to run (PHP-FPM, queue workers, Reverb, scheduler, pm2), what the env needs,
and the questions only I can answer (domains, branch, which environments, real payments/SMS or sandbox).

---

## 3. Phase 1: look at the server (read-only)

First round is **always** read-only. Something like this, adapted to the OS:

```bash
cat /etc/os-release | head -4
free -h && df -h /
nproc
ss -ltnp                                   # every port already taken, by which process
systemctl is-active nginx apache2 httpd php8.4-fpm php-fpm mysql mysqld supervisor
ls /etc/nginx/sites-enabled/ /etc/httpd/conf.d/ 2>/dev/null     # what else lives here
php -v | head -1; node -v; pm2 -v; composer -V 2>/dev/null | head -1
pm2 status 2>/dev/null; supervisorctl status 2>/dev/null
certbot certificates 2>/dev/null | grep -E 'Certificate Name|Domains|Expiry'
getenforce 2>/dev/null                     # SELinux (Red Hat family)
for h in app.example.com api.example.com; do getent hosts $h; done
```

From the answer decide, and show me as a table before changing anything:

- OS and web server → which recipe. A shared box? List its other apps and **do not touch them**.
- **Free ports**: one Next.js port and one Reverb port per environment (my defaults: test 3000 / 8080, prod
  3001 / 8081; on a busy box pick free ones, e.g. 3008 / 8088, and check again with `ss -ltnp`).
- Names: folders `/var/www/<domain>`, pm2 apps, Supervisor programs `<domain>-worker` / `<domain>-reverb`,
  database `<app>` / `<app>_test` with users `<app>_user` / `<app>_test_user`.
- Memory: under ~4 GB with other apps running, every build stops the other apps first (section 6).

---

## 4. Phase 2: env files, made on my machine

Generate the server env files **locally**, in a folder **outside every git repo**, e.g.
`D:\coding\<company>\<project>\deployment\server-env\<server-name>\`:

| File | Goes to |
| --- | --- |
| `<api-domain>.env` (one per environment) | `/var/www/<api-domain>/.env` |
| `<web-domain>.env.production.local` (one per environment) | `/var/www/<web-domain>/.env.production.local` (git-ignored, read by `next build` over `.env.production`) |
| `<project>-mysql.sql` | run once to create the databases and users, then deleted from the server |

- **Fresh secrets per environment**, never copied from development and never shared between environments:
  `APP_KEY` (`php artisan key:generate --show`), DB passwords (`openssl rand -base64 24`), Reverb
  id/key/secret (`shuf -i 100000-999999 -n 1`, `openssl rand -hex 10`). The web file's
  `NEXT_PUBLIC_REVERB_APP_KEY` must equal its API's `REVERB_APP_KEY`.
- Production values: `APP_ENV=production`, `APP_DEBUG=false`, real `APP_URL` / `FRONTEND_URL`,
  `CORS_ALLOWED_ORIGINS` with the real web origins, `LOG_LEVEL=warning`, **`LOG_CHANNEL=stack`, `LOG_STACK=daily`,
  `LOG_DAILY_DAYS=30`** (one log file per day, 30 kept: a single `laravel.log` reached 89 MB on NICE), `REVERB_HOST` =
  public API host, port 443, scheme https; `REVERB_SERVER_PORT` = that environment's internal port. Fixing these on a
  server that is already live, and reading daily logs: [../linux/laravel-production-logs-and-debug.md](../linux/laravel-production-logs-and-debug.md).
- Test environments: payments in sandbox, SMS to the log, unless I say otherwise.
- I paste each file on the server with `rm -f .env && nano .env` (**never a heredoc** for an env file: long pasted
  heredocs get mangled), then `chown root:www-data .env && chmod 640 .env`.
- These files never go into a repo, a doc or a chat reply. Tell me where you wrote them; do not print their contents.

---

## 5. Phase 3: install, round by round

The recipe gives the order (packages → git access → clone → databases → API → workers/scheduler → web server
sites → HTTPS → web build → checks). How to write each round:

### Command blocks

- **Short blocks, one purpose each**, each ending with a check that prints something (`nginx -t`, `supervisorctl
  status`, `curl -s -o /dev/null -w "%{http_code}\n" ...`), and say what that check should print.
- **Loops for repeated environments.** Two environments share every step, so write it once:

  ```bash
  for env in api testapi; do
    cd /var/www/$env.example.com
    sudo -u www-data php artisan migrate --force
    sudo -u www-data php artisan optimize
  done
  ```

  The same for Supervisor files (port picked inside the loop), crontab lines, nginx sites
  (`for s in ...; do ln -sf /etc/nginx/sites-available/$s /etc/nginx/sites-enabled/; done`), DNS checks and
  final `curl` checks. Write config files with heredocs: **quoted `<<'EOF'`** when the file has its own `$`
  variables (nginx), unquoted `<<EOF` only when you want the shell to fill in your loop variables (and the file
  has no `$` of its own).
- **No `<placeholders>` inside a command**: bash reads `<your` as a file redirect. Write the real value, or a
  variable set at the top of the block.
- **Nothing interactive in the middle of a pasted block**: it swallows the lines after it.
  - Composer as root asks "Continue as root?": use `COMPOSER_ALLOW_SUPERUSER=1 composer ...` (or run it as the
    web user).
  - `certbot` asks questions: give it **its own block**, nothing after it.
  - `exec $SHELL` after installing nvm drops the rest of the paste: use `source ~/.nvm/nvm.sh`.
  - `nano` is fine only as the last line of a block.
- **Label where each block runs**: "on the server, bash" (default) or "on Windows, PowerShell". A PowerShell
  `foreach` pasted into bash is a syntax error, and the other way round.
- Long steps (`apt upgrade`, `npm ci`, `next build`) can run inside `tmux new -s deploy`, so an SSH drop does not
  kill them (`tmux attach -t deploy`).

### Changing things safely

- **Back up before you overwrite** any server config: `cp file /root/` (never into `sites-enabled` or
  `conf.d`, where the backup would be loaded too). Then `nginx -t` / `apachectl configtest` before every reload.
- **Ask me first** before anything that destroys or reaches beyond this app: dropping or `migrate:fresh` on a
  database with data, deleting folders, stopping or editing another app's processes, sites or cron lines,
  `setenforce 0`, opening firewall ports. Stopping and restarting all apps to free memory for a build is the one
  exception (section 6), because they all come back.
- **Shared box**: check a port is free before using it; name everything after the domain so it never collides.

### House rules (the reasons are in the recipes)

- Laravel runs as the web user (`www-data`, or `apache` on Red Hat): PHP-FPM, queue workers, Reverb, the
  scheduler. **Every artisan command that writes runs as that user**: `sudo -u www-data php artisan ...`;
  tinker with `HOME=/tmp`. A root-run artisan leaves root-owned files and a silent 500 later.
- `storage` and `bootstrap/cache`: owned by the web user, folders 775, files 664, **never `chmod -R 775`**;
  `git config core.fileMode false` in every clone.
- `composer install --no-dev --optimize-autoloader`. `php artisan optimize` is the last API step, and after every
  `.env` edit. Restart workers (and Reverb) after every deploy: they hold old code in memory.
- One MySQL database and one user per environment, never root.
- The scheduler line goes in the web user's crontab, never root's.
- Next.js: `NEXT_PUBLIC_*` are baked in at build, so env changes need a rebuild. pm2 app per environment on its own
  port, `pm2 save`, `pm2 startup` once.
- One `certbot` command covering every hostname (they all need DNS records first).
- Git from the server with a read-only deploy key per repo.

---

## 6. Memory and shared boxes

`next build` and `composer install` get `Killed` when RAM is short. On a small or shared box every (re)deploy:

```bash
sudo supervisorctl stop all
pm2 stop all
# ... pull, install, build ...
sudo supervisorctl restart all
pm2 restart all
sudo supervisorctl status && pm2 status
```

Restarting the other apps is part of the procedure, not a side effect. Build one Next.js app at a time.

---

## 7. Secrets

- Never ask me to paste an `.env`, a password or a key. When you need to see config, filter it:
  `grep -E '^[A-Z_]+=' .env | grep -vEi 'PASS|SECRET|KEY|TOKEN'`. To check a value the app actually loaded:
  `sudo -u www-data php artisan config:show app.url`.
- If I send you something with a password in it (WhatsApp notes, a screenshot), use it but **never repeat it** in a
  reply, a command or a doc. Remind me once at the end to rotate whatever was shared.
- Docs that go into a repo hold **no secrets**: say where a secret lives (`.env` on the server, the env folder on
  my machine), never its value. Before committing to a **public** repo, grep the staged diff for IPs, domains,
  passwords and emails.

---

## 8. Reading what I paste back

- The terminal echoes pasted lines out of order; judge by the command outputs, not by where the echo landed.
- If a command's output is missing, look for an interactive prompt just before it ("Continue as root?", a
  `[Y/n]`): it ate the lines. Give me those lines again.
- `curl` printing `000` means nothing answered on that scheme/port (no HTTPS yet, firewall, wrong host).
- A bare 500 with an empty body and an empty `laravel.log`: root-owned files in `storage`/`bootstrap/cache`.
- Apache's `Content-Type: text/html;charset=ISO-8859-1` on a Next.js site is a directory listing: the live (443)
  site is not proxying.
- When something fails, find the cause before the fix; do not stack workarounds (no `chmod 777`, no
  `setenforce 0`, no `APP_DEBUG=true` on a public server).

---

## 9. Phase 4: verify, then write the docs into the repo

### Verify

The final round proves everything, per environment: services active, pm2 apps online, ports listening,
`curl` 200 on every site and on one public API endpoint, `/.env` returning 404, a CORS preflight from the real web
origin, the WebSocket (`wss://.../app/<key>`, status 101 in DevTools) if Reverb is used, `storage/logs` owned by the
web user. Then I log in through the browser. For real payments, one small real transaction ending on a receipt.

### Write `docs/deployment.md` and `docs/redeployment.md` in the code repo

**Every deploy ends with these two files in the repo**, so the next agent (or person) does not start from zero.
Put them in the backend repo's `docs/` folder (with a separate workspace repo, link them from its README too), and
add one line to the repo's `CLAUDE.md` / `AGENTS.md` and `README` pointing at them. If they already exist, update
them instead: a new server or environment is a new section, not a new file.

Write them with **the real names and commands** of this deploy, as copy-paste blocks that worked, and **no
secrets**.

**`docs/deployment.md`** (first deploy):

1. **The environment(s)**: a table per environment: web and API URLs, server (provider, OS, IP, `ssh` line),
   folders, repos, branch, pm2 app and port, Reverb port, Supervisor programs (or systemd units), database and
   user, payments/SMS/mail mode, where the env files live on the deploying machine.
2. **The server**: anything particular (shared with which apps, memory, OS quirks, web user, SELinux).
3. **Who owns what**: root vs the web user, and why artisan runs as the web user.
4. **Env files**: the table from section 4 of this playbook, the keys that differ between environments, the
   "must match" pairs (Reverb key), "paste with nano, never a heredoc".
5. **First deploy, step by step**: DNS, packages, git access, clone, databases, API, workers/scheduler, web
   server sites, HTTPS, web build. The exact blocks that worked (with the loops).
6. **Checks**: the commands and what each must print, then the browser checks.
7. **What the first deploy ran into**: a table of what happened / why / what the doc does now.
8. **Troubleshooting**: symptom → cause and fix.
9. **Open items**: what is still to do (password changes, deploy keys, OS end of life ...).

**`docs/redeployment.md`** (every code change after that):

1. One `ssh` line, then **one block per stack, looped over environments**: maintenance mode on, `git pull`,
   `composer install`, `migrate --force`, safe-to-repeat seeders (e.g. roles and permissions), cache resets,
   `optimize`, `chown`, maintenance off; then the web: `git pull`, `npm ci`, build, `pm2 restart`; then restart
   workers/Reverb (stop-all / restart-all on a small box).
2. When steps may be skipped (no lock file changed, no migrations).
3. The checks to run after.
4. `git pull` refusing: `git diff --stat` first; mode changes only → `core.fileMode false`; real edits → stop.
5. Rolling back to the previous commit.
6. A link back to `deployment.md` for troubleshooting.

Commit and push them (with the repo's own commit style) when I ask, or ask me at the end whether to.

### Session handoff (when the deploy took more than one sitting)

Also leave a `HANDOFF.md` next to the deployment env folder (outside the repo when it mentions server details
that should not be public): state at the end, what was wrong and what you changed in order (with backup
locations), documents written, open items by priority, how to pick it up again.

---

## 10. Redeploying later

When I ask for a redeploy, read `docs/redeployment.md` in the repo first and follow it. Check what the pulled
commits bring (`git log`, lock files, migrations, new env keys in `.env.example`) and tell me before running it if
something needs more than the usual block: a new env value, a new queue/cron/service, a new seeder, a new
`NEXT_PUBLIC_*` (rebuild). If the doc turns out wrong or incomplete, fix the doc as part of the job.

After every redeploy, read the log **since the deploy** by its entry headers (`grep -E '^\['` on today's
`laravel-YYYY-MM-DD.log`, never just a `tail` of a stack trace) and sort each kind: expected lines, rules doing their
job, real faults. On the first redeploy of a server, check `APP_DEBUG` is `false` and the log is daily; if not, fix
both with the box in [../linux/laravel-production-logs-and-debug.md](../linux/laravel-production-logs-and-debug.md).
