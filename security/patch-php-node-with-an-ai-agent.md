# Patching PHP and Node.js with an AI agent: the playbook

**For the AI agent reading this.** A PHP or Node.js security release is out and we are taking my machines onto
the patched versions: my Windows dev PC first, then the server(s). The commands themselves are in
[php-node-security-patch.md](php-node-security-patch.md): read it fully before writing anything. This file is
**how** we do it together. The general working model is the deploy playbook,
[../ai-agents/deploy-with-an-ai-agent.md](../ai-agents/deploy-with-an-ai-agent.md) (rounds, paste-safe blocks,
secrets): it applies here too.

**How to use it (for me).** In the project's workspace:

> Read `coding-notes/security/patch-php-node-with-an-ai-agent.md` and follow it. The advisory is `<link>`.
> Server(s): `<host>`, apps `<domains>`.

---

## 1. Rules

- **You do not have the server. I do.** You write the commands, I run them and paste back the output. Nothing is
  "done" on the server until its output shows it.
- **No guessing.** Paths, app names, ports, pm2 names and Supervisor programs come from the project's
  `docs/deployment.md` and from the server's own output (step 3), never from memory or a typical layout.
- **One small block at a time.** Say what it does and exactly what the output must look like. Wait for the paste,
  check it line by line, then the next block. If the paste is cut short, ask for the missing line before a later
  step depends on it.
- **Read-only first, changes after, a check after every change.** Downtime steps go after their checks. Chain a
  risky step to its check with `&&` or an `if`.
- **Never tell me Ctrl+C is safe.** If the screen looks stuck: "press Enter once and wait, send a screenshot".
  And tell me **not to press Ctrl+C before pasting** a block (a habit to "clear the line"): after `pm2 update` it
  killed pm2 and both websites (3 Nov 2026).
- **A long command that looks stuck: read it from a second SSH session**, never interrupt it. Give a read-only block
  for the new tab (`supervisorctl status`, `ps -eo pid,etime,cmd | grep ...`) and decide from that.
- **Commands that might wait a long time get a `timeout`** (`timeout 30 supervisorctl update`), and output sent to a
  log gets a line saying it will be quiet for a while.
- **Patch inside the line already running** (8.4.x to the fixed 8.4.x). Moving to a new line (8.4 to 8.5, Node 22
  to 24) is a separate upgrade with tests, done on dev and server together, and only if I ask.
- **Ask before anything destructive** (deleting old PHP/Node versions, removing files the break-in check found).

---

## 2. Facts first (on my machine, no server needed)

1. Read the advisory and the vendor posts. Write down: what is affected (which SAPIs), the fixed version per line,
   which lines get no fix, and whether it is exploitable without login.
2. Confirm the fixed versions from the source, not from blog posts:
   - PHP: `https://www.php.net/releases/index.php?json&max=8`
   - PHP Windows builds and SHA-256: `https://downloads.php.net/~windows/releases/releases.json`
   - Node: `https://nodejs.org/dist/index.json` (`"security": true`)
3. Read the project: `composer.json` (`php` constraint), `package.json` (`engines`), `docs/deployment.md` (server
   facts: PHP line, FPM, nvm, pm2 names and ports, Supervisor programs, app folders). Check whether a sibling
   project already went through this (a `SECURITY-*.md` in its workspace): reuse what it learned.
4. Tell me in a few lines: affected or not, from which version to which, on the PC and on each server.

---

## 3. The Windows dev PC (you can run these yourself)

You have a shell on my PC, so do section 1 of the guide directly, explaining as you go:

- Inventory (1.1). Point out every PHP folder, the active one, and **any PATH entry that names a PHP version**.
- Download, **verify the SHA-256 against `releases.json`**, install beside the old folder, carry over `php.ini`
  and hand-added extensions, prove `php -m` is identical (1.2, 1.3).
- I switch in Laragon's menu. You check Apache's `X-Powered-By` and the PATH. If the Machine PATH still names the
  old folder, give me the admin PowerShell block (1.4); your own shell keeps the old PATH until VS Code restarts,
  so check with a fresh read of the registry PATH, not `php -v` in your session.
- Node (1.5): list global packages first; `nvm use` may need my admin approval.
- Run the project's tests on the new PHP. Delete old folders only when I say.

---

## 4. The server, in rounds

| Round | Guide section | What I paste back | You check |
| --- | --- | --- | --- |
| 1. Read | 2.1 | the whole output | every PHP in `/etc/php`, which one nginx/Apache uses, CLI alternative, Node version, **each pm2 app's name, folder, script path, args**, boot unit, Supervisor programs. Compare with `docs/deployment.md`; say what differs |
| 2. Break-in check | 2.2 | the whole output | each item against "normal". **Anything unexpected: stop**, explain, and decide with me before patching. Never `git pull` before this round |
| 3. PHP candidates | 2.3 first block | `apt-cache policy` lines | every line has a fixed `Candidate`; if not, wait |
| 4. PHP upgrade | 2.3 second block | the tail | no errors, `--force-confold` kept configs |
| 5. Restart PHP | 2.3 third block | the whole output | FPM `active`, workers `RUNNING`, `php -v` fixed, the APIs answer as before |
| 6. Node install | 2.4 first block | the whole output | new `node -v`, `pm2` under the new version, `PM2 updated` printed |
| 7. Apps on new Node | 2.4 second block | the whole output | each app `script path` under the new version, `node.js version` new, args as in round 1, sites answer |
| 8. Boot unit | 2.4 third block | the whole output | new version in `ExecStart` / `PATH`, `enabled`; do **not** start it |
| 9. Final read | 2.1 again | the whole output | everything on the fixed versions; nothing else changed |
| 10. Reboot (if asked) and OS security updates | 2.4b | the whole output | new kernel, `pm2-root` active, apps back on their own, 0 security updates left |

Before round 5, read every Supervisor program's `stopwaitsecs` (2.3): a websocket server on the workers' 3600 can hang
a restart for an hour.

Build every round's commands from round 1's facts (real folders, real pm2 names and args, real Supervisor
programs). On a box shared with other projects, restart only this project's programs
(`supervisorctl restart '<program>:*'`), and say which other apps share the PHP/Node you are changing.

---

## 5. Leave it written down

- In the project workspace, `SECURITY-<date>-<short-name>.md`: the advisory and CVEs, affected and fixed versions,
  the break-in check result, every command block as run, and a **record table** (date, step, result with the
  versions before and after). Link it from `docs/deployment.md`.
- Update `docs/deployment.md`'s server facts (PHP and Node versions, pm2 script paths).
- Anything new you learned that applies beyond this project: tell me, and we add it to
  [php-node-security-patch.md](php-node-security-patch.md).
- Then a short message for the client, if I ask: what was patched, versions, downtime (seconds). No internal names,
  no "we were exposed" speculation, only facts from the record.
