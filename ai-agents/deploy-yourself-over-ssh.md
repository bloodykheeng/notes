# Deploying it yourself over SSH: the agent runs the server

**For the AI agent reading this.** The normal playbook ([deploy-with-an-ai-agent.md](deploy-with-an-ai-agent.md))
has me run every command and paste the output back. This note is the other mode: I put the server's SSH login in a
text file on my machine and you run the whole deploy yourself, from your own shell, and report back. It was
written from the Elevate Pesa move (Nov 2026: a Laravel API + two pre-built React sites restored from a Hostinger
backup onto a fresh Contabo VPS, DNS, HTTPS and free mail, in one sitting). Everything in the normal playbook still
applies (study first, read-only first, house rules, secrets, docs at the end); this note adds what changes when
the keyboard is yours.

**How to use it (for me).** Put the login in the project's deployment folder (outside every repo) and say:

> Read `ai-agents/deploy-with-an-ai-agent.md` and `ai-agents/deploy-yourself-over-ssh.md` in my coding-notes and
> deploy `<project>`. The SSH login is in `<path to the txt>`. The code / backup is in `<folder>`.

---

## 1. Where things live on my machine

```
<project>/
├── <project>-workspace/      code repos + docs (a git repo of its own: README, CLAUDE.md, HANDOFF.md, docs/)
├── deployment/
│   ├── <provider>/           "<project> <provider> hosting.txt" (logins), server-env/ (production .env), mail txt
│   └── <old-provider>/       old notes, kept for reference
├── backups/                  hosting backups (files + SQL dumps)
└── flutter/ ...              anything else (signing keys)
```

The logins file is free text (mine are WhatsApp-style notes). Find the line you need; never quote it back.

---

## 2. Secrets: reading them without leaking them

- **Read the logins file with the Read tool or `sed -n 'Np'`** into a variable, never `cat` it into the output.
  When you need to look at a file that holds secrets, mask it first:

  ```bash
  grep -E '^[A-Z_]+=' .env | sed -E 's/^([A-Z_]*(PASS|SECRET|KEY|TOKEN|PRIVATE|CREDENTIAL|URL)[A-Z_]*)=.*/\1=***/'
  ```

  Mask `*_URL` too: `CLOUDINARY_URL=cloudinary://key:secret@cloud` slipped through a mask that only had
  PASS/SECRET/KEY.
- A password never goes into a command line, a heredoc that echoes, a commit or a reply. Move it **file to file**
  (a small Python script that reads the source file and writes the `.env`), and print only "updated N keys,
  password length 10".
- When I paste a password in chat, use it, do not repeat it, and remind me once at the end to change it.
- `.env` values starting with `#` or holding spaces must be quoted (`MAIL_PASSWORD="#..."`), or `#...` is read as
  a comment and the value is empty.

---

## 3. Getting in: password login without a terminal prompt

Your shell cannot type into SSH's password prompt, and `sshpass` / `plink` are usually missing on Windows. OpenSSH
8.4+ (Git for Windows ships it) can take the password from a helper program instead:

```bash
SP="<your scratchpad directory>"                       # session-only, never a repo
sed -n '14p' "<path to logins txt>" | tr -d '\r\n ' > "$SP/pw"; chmod 600 "$SP/pw"   # line 14 = the root password
printf '#!/bin/sh\ncat "%s/pw"\n' "$SP" > "$SP/ap.sh"; chmod 700 "$SP/ap.sh"

cat > "$SP/r.sh" <<EOF
#!/bin/sh
SSH_ASKPASS="$SP/ap.sh" SSH_ASKPASS_REQUIRE=force DISPLAY=:0 exec ssh \
  -o StrictHostKeyChecking=accept-new -o PubkeyAuthentication=no -o PreferredAuthentications=password \
  -o ServerAliveInterval=30 root@SERVER_IP "\$@"
EOF
chmod 700 "$SP/r.sh"

"$SP/r.sh" 'hostname; uptime' < /dev/null          # every call ends with < /dev/null
```

- `scp` takes the same three environment variables and `-o` options.
- `< /dev/null` on every call: without it ssh can swallow your shell's stdin.
- Reading the password from a file (`cat`) instead of baking it into the script survives `$`, quotes and spaces.
- **Do not install your SSH key on my server without asking.** It is persistent access; the harness may block it
  (it did, as "unauthorized persistence"). Ask, or stay on the password helper.
- **Delete `pw`, `ap.sh` and `r.sh` when the job is done** (`rm -f`), and again after any later session that
  recreated them. Say in the final message that you did.

---

## 4. The order of work (what worked)

1. **Study locally** (normal playbook, Phase 0). For a restore from a hosting backup also:
   - the backup's `public_html` per domain, its `.env` (masked), `composer.lock` (PHP version), `.git` (which
     commit is live, and compare it with my local repo: `git log --oneline <live>..HEAD`);
   - which front-ends are **pre-built**: grep the build for the API URL baked in
     (`grep -rhoE 'https?://[a-z0-9.-]*example\.com[^"]*' build/static | sort | uniq -c`). If it is already the
     production URL, upload the build as is, no rebuild;
   - `grep -rn "env(" app routes`: anything there makes `config:cache` / `optimize` unsafe (it returns `null` once
     config is cached). Elevate Pesa reads Firebase credentials that way: cache routes and views only.
   - the SQL dump's first line: a MariaDB 11 dump starts with `/*M!999999\- enable the sandbox mode */`, which the
     MySQL 8 client rejects. `tail -n +2` it before import.
2. **Read-only server round** (OS, RAM, disk, ports, services, DNS of every hostname). A fresh box answers
   "nothing installed" in one call.
3. **Check DNS before you need it.** If the hostnames do not resolve, find out why now (section 5); the server
   work can go on while I fix DNS.
4. **Install in the background** (`run_in_background`), and package uploads meanwhile:
   - `tar` on Windows reads `C:/...` as a remote host: use **`tar --force-local`** (or relative paths).
   - Leave out `vendor/`, `node_modules/`, old `*.zip`, logs, cached sessions/views and the old `.env`; keep `.git`
     so the server can `git pull` later.
   - `DEBIAN_FRONTEND=noninteractive NEEDRESTART_MODE=a` and `-o Dpkg::Options::=--force-confold` so apt never
     stops to ask.
5. **One block per step, each ending with its check** (firewall + fail2ban, DB create + import with row counts,
   extract + ownership, composer, migrate status, caches, nginx `-t`, local `curl -H "Host: ..."` per site).
   Tarballs extracted as root keep Windows uids: `chown -R root:root` the tree first, then `www-data` on
   `storage`, `bootstrap/cache` and upload folders.
6. **HTTPS** once DNS answers: one `certbot --nginx --redirect --non-interactive --agree-tos -m <email> -d ...`
   for every name, then `certbot renew --dry-run`.
7. **Verify from outside** (your machine, not the server): every URL's status, `/.env` 403/404, a JSON endpoint
   with real data, a CORS preflight from the web origin, `http` → `https` redirect.
8. **Clean up**: delete the env file and DB-password SQL from the server's upload folder, the helper files on your
   side, and say what is left (`/root/<project>-upload/` with the tarballs and dump) for me to delete.
9. **Docs and commit**: `docs/deployment.md` (+ redeployment), `HANDOFF.md`, README/CLAUDE.md links, in the
   workspace repo; scan the staged diff for secrets; commit and push when I asked.

Env files: build them locally in `deployment/<provider>/server-env/` from the old `.env` (production values, fresh
DB password via `openssl rand -hex 20`, daily logs), then `scp` + `install -o root -g www-data -m 640`. Back up the
live `.env` to `/root/` before replacing it.

---

## 5. DNS: the traps

- **Nameservers moved but nothing resolves.** Ask the registry and then the new nameserver directly:

  ```bash
  dig +norec NS example.com @a.gtld-servers.net        # who the .com registry delegates to
  dig example.com @ns1.provider.net | grep status      # REFUSED = the provider has no zone for it yet
  ```

  On Contabo, pointing the domain at `ns1/2/3.contabo.net` is not enough: someone must **create the DNS zone** in
  the panel (Network → DNS Management → Create DNS zone, pick the server's IP). Contabo's template then adds `@`,
  `www`, `mail`, `*` A records and an MX to `mail.<domain>`; add the other subdomains (`api`, `admin`...) as A
  records.
- That auto MX points at **your** VPS, which runs no mail server. Replace it when mail is set up
  ([../integrations/free-email-with-maildog.md](../integrations/free-email-with-maildog.md)).
- Contabo's form: an **empty Name means the domain itself**; a number typed in Name creates `10.example.com`.
- Check resolution from more than one place: the provider's nameserver, `8.8.8.8` and `1.1.1.1`
  (`getent ahostsv4` on the server, or `https://dns.google/resolve?name=...&type=A`).
- Find the registrar with RDAP when asked "do they already have email there?":
  `curl -s https://rdap.verisign.com/com/v1/domain/example.com`. Once the nameservers moved, the registrar's own
  records (and any free email they gave) no longer count.

---

## 6. Windows-side traps

| Trap | Fix |
| --- | --- |
| `tar: Cannot connect to C: resolve failed` | `tar --force-local` |
| `mv: Permission denied` moving a folder you once `cd`'d into | a shell still holds it; retry with PowerShell `Move-Item`, or move children |
| `git` "dubious ownership" on old repos (copied from another PC) | `git -c safe.directory=* <cmd>` for read-only checks; do not change global config unasked |
| Backslashes vanish inside a quoted heredoc that writes Python | write the line with the Edit tool, or use `chr(92)` |
| `gh` not installed, GitHub SSH key not accepted | push over HTTPS (the credential manager has the token) |
| Long `du`/`grep` over a backup times out at 2 min | run it in the background, read the output file later |

---

## 7. What to report to me

After each phase, a few lines in plain words: what was done, what the check printed (numbers: tables, rows, latest
order date), what needs me (a DNS zone, an email decision), what is risky. At the end: the URLs that work, the
admin login (verify it against the restored hash with `password_verify` locally, do not guess), open items in
order, where the docs are, and the reminder to change passwords that passed through chat or plain-text files.
