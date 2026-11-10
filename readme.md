# Notes

Technical notes and solutions for deployments, server configuration and recurring fixes.
Every note is written to be followed top to bottom and copy-pasted.

## Start here

- **[ai-agents/deploy-with-an-ai-agent.md](ai-agents/deploy-with-an-ai-agent.md)**: give this to the AI agent
  before any deploy or redeploy. How we work (agent writes the commands, I run them and paste the output back),
  read-only checks first, loops over environments, paste-safe blocks, secrets, and the
  `docs/deployment.md` + `docs/redeployment.md` it must leave in the code repo.
- **[nginx/laravel-nextjs-two-environments-deploy.md](nginx/laravel-nextjs-two-environments-deploy.md)**:
  full build of a fresh Ubuntu VPS for a Laravel API (queue workers, scheduler, Reverb) plus
  a Next.js frontend (pm2), two environments on one box, Nginx + Let's Encrypt. Includes the
  redeploy procedure and a troubleshooting table.
- **[centos/laravel-nextjs-apache-centos-deploy.md](centos/laravel-nextjs-apache-centos-deploy.md)**:
  the same stack on a CentOS / RHEL 8 box with Apache (httpd) and SELinux Enforcing, plus its
  [redeploy procedure](centos/laravel-nextjs-apache-centos-redeploy.md).

## Contents

### ai-agents
| Note | What it covers |
| --- | --- |
| [build-laravel-nextjs-app-with-an-ai-agent.md](ai-agents/build-laravel-nextjs-app-with-an-ai-agent.md) | The playbook for an AI agent building a Laravel + Next.js business app with me: how we work (facts first, search best practice online, root cause, tests and docs with every change), folders, backend conventions (ULIDs, serial counters, enums, money, UTC, migrations, files), the maker/checker approval engine, roles and permissions, sessions and the portal cache bug, lists (pagination, URL state, 3-state sort, chips, one filter form, skeletons, typed exports), dialogs and forms, help text, documents, audit trail, Reverb realtime, reports, packages, testing and docs |
| [dates-and-timezones-with-an-ai-agent.md](ai-agents/dates-and-timezones-with-an-ai-agent.md) | Dates done right in Laravel + Next.js + a mobile app: store and query UTC, "a day" from one business timezone, bare date = whole day and a time = exact; the column kinds (UTC instant vs calendar date); the traps that were live in production (Laravel binds a Carbon without converting its timezone, an end date that drops the last day, `whereDate`/`DATE()` on UTC columns, browser `new Date("YYYY-MM-DD")`, pickers that discard the time, MySQL TIMESTAMP and the DB session timezone, uncast datetime columns, Carbon's month overflow, caches); how to check a whole app (golden master, a `DB::pretend()` guard test, a real-data detector, tallies of dashboard figures against the lists they link to, static sweeps); and the method mistakes an agent made on NICE, so the next one does not |
| [master-copy-and-snapshots.md](ai-agents/master-copy-and-snapshots.md) | Editable data and its history: one master copy edited directly, facts frozen on the records that use them (SDS standards per inspection period; NICE's journey plan schedule, first built with versioned rows and rebuilt), live identities referenced directly, no keys to mutable rows, changes applied forward, upload modes and ownership so mistakes clean up in one step, and a checklist |
| [audit-trail-with-snapshot-schemas.md](ai-agents/audit-trail-with-snapshot-schemas.md) | The audit trail engine from SDS and Alpha: what an entry must answer, `HasSnapshotSchema` on every model, `SnapshotPresenter` (values resolved and frozen at write time), the queued `LogActivityJob`, the module enum, the generic view dialog, the rules that make it trustworthy, how to upgrade an app that logs sentences, and what NICE's upgrade found (secrets in SMS entries, noise, dumps, broken calls) and the mistakes an agent made |
| [enums-and-migrations.md](ai-agents/enums-and-migrations.md) | Every fixed list as a PHP enum cast on the model (strings in the database, values listed in the migration comment, web mirrors), reading and writing cast values, and migrations people can read (every column written out, grouped and spaced, explanations above, no date columns, edit in place only before a deploy), with the mistakes an agent made |
| [deploy-with-an-ai-agent.md](ai-agents/deploy-with-an-ai-agent.md) | The playbook for an AI agent deploying a Laravel + Next.js project with me: study the repo first, read-only server checks, env files made locally (fresh secrets, never in git), rounds of short verified blocks, `for` loops over environments, nothing interactive mid-paste, backups and asking before destructive steps, stop-all/restart-all on small boxes, secrets never echoed, and writing `docs/deployment.md` + `docs/redeployment.md` (plus a handoff) into the repo |

### security
| Note | What it covers |
| --- | --- |
| [php-node-security-patch.md](security/php-node-security-patch.md) | Taking PHP and Node.js onto a security release (Oct 2026: PHP CVE-2026-109618/109619, fixed in 8.4.27 / 8.3.36 / 8.5.12; Node 24.22.1 / 22.23.4). Windows dev PC (Laragon: verified download, side-by-side install, imagick and `php.ini` carried over, the stale Machine PATH; nvm-windows), then the Ubuntu server: read-only inventory, break-in check, PHP-FPM upgrade for every version in `/etc/php`, workers restarted, Node through nvm with pm2 apps restarted from the new Node and the boot unit rewritten (never Ctrl+C after `pm2 update`) |
| [patch-php-node-with-an-ai-agent.md](security/patch-php-node-with-an-ai-agent.md) | The same job given to an AI agent: facts from the source (php.net, nodejs.org JSON), the dev PC done by the agent, the server in nine checked rounds, and the `SECURITY-<date>.md` record it leaves in the project |

### nginx
| Note | What it covers |
| --- | --- |
| [laravel-nextjs-two-environments-deploy.md](nginx/laravel-nextjs-two-environments-deploy.md) | The complete stack, prod + test on one server |
| [laravel-reverb-production-nginx.md](nginx/laravel-reverb-production-nginx.md) | Reverb only: env, generated keys, Nginx proxy, Supervisor, multiple apps per box |
| [nginx-laravel-deploy.md](nginx/nginx-laravel-deploy.md) | Single Laravel app with MySQL and phpMyAdmin |
| [nginx-nextjs-deployment.md](nginx/nginx-nextjs-deployment.md) | Single Next.js app with pm2 |
| [deployment_larvel_nextjs_on_already_setup_server_guide.md](nginx/deployment_larvel_nextjs_on_already_setup_server_guide.md) | Adding another app to a server that already runs one |
| [upgrade-php-version-nginx.md](nginx/upgrade-php-version-nginx.md) | Installing the PHP version the project needs (ondrej PPA / packages.sury.org) |
| [laravel-production-cache-permission-error-fix.md](nginx/laravel-production-cache-permission-error-fix.md) | `file_put_contents ... storage/framework/cache` (root vs www-data) |
| [install-ssl-on-nginx.md](nginx/install-ssl-on-nginx.md) | Certbot |
| [increase-php-upload-limit-nginx.md](nginx/increase-php-upload-limit-nginx.md) | Upload limits |
| [install-filebrowser-nginx.md](nginx/install-filebrowser-nginx.md) | File Browser behind Nginx |
| [postgres-pgvector-laravel-deploy.md](nginx/postgres-pgvector-laravel-deploy.md) | Postgres + pgvector for Laravel |

### integrations
| Note | What it covers |
| --- | --- |
| [google-workspace-mail-firebase-oauth-recaptcha.md](integrations/google-workspace-mail-firebase-oauth-recaptcha.md) | Where every Google-side env value comes from: Workspace SMTP + App Password, Firebase config and service account, OAuth client, reCAPTCHA, and the walls (2SV, org policy, project quota) |

### nextjs
| Note | What it covers |
| --- | --- |
| [firebase-messaging-sw-and-nextauth-google.md](nextjs/firebase-messaging-sw-and-nextauth-google.md) | One Firebase service worker for every environment (config via query string) and Sign in with Google with NextAuth v4 |
| [firebase-push-token-errors-check-versions-first.md](nextjs/firebase-push-token-errors-check-versions-first.md) | `VersionError (1) < (2)` and `messaging/permission-blocked` when getting a push token: Firebase 13 raised its IndexedDB to v2, `localhost:3000` storage is shared by every project, how to compare versions in five minutes, the guard that drops a newer copy, and why restarting never helps |
| [primereact-tailwind-dark-mode-missing-classes.md](nextjs/primereact-tailwind-dark-mode-missing-classes.md) | PrimeReact's Tailwind preset + Tailwind 4: white-on-white selected options, date-picker days that look disabled, no "today" circle. Why (node_modules is never scanned) and the CSS that fixes it |
| [laravel-validation-error-toasts-without-duplicates.md](nextjs/laravel-validation-error-toasts-without-duplicates.md) | The same Laravel 422 error shown in two toasts: why (Laravel's `message` repeats the first field error) and the hook logic that shows each thing once |
| [dashboard-sidebar-groups.md](nextjs/dashboard-sidebar-groups.md) | A dashboard sidebar people can scan: a few small headings, related pages folded under one parent, children on a guide line, one lit row, gates that drop empty parents and groups, the collapsed rail, and the Playwright check that the folded menu fits a laptop screen (SDS, GiftNest, NICE) |
| [primereact-inputnumber-zod-form-traps.md](nextjs/primereact-inputnumber-zod-form-traps.md) | InputNumber `onChange` vs `onValueChange`, `min`/`max` that only clamp the screen, `z.coerce` and 0 defaults letting empty fields through, AutoComplete text vs a picked item; plus an audit grep |

### linux
| Note | What it covers |
| --- | --- |
| [jobs-and-queues.md](linux/jobs-and-queues.md) | Laravel scheduler (cron) and queue workers (Supervisor) |
| [github-ssh-authentication.md](linux/github-ssh-authentication.md) | SSH keys for Git hosting |
| [fail2ban-installation-and-ssh-protection.md](linux/fail2ban-installation-and-ssh-protection.md) | SSH hardening |
| [install-google-authenticator-on-linux.md](linux/install-google-authenticator-on-linux.md) | 2FA on SSH |
| [server-unreachable-host-or-app.md](linux/server-unreachable-host-or-app.md) | Every site on a VPS times out: telling a host outage from an app fault (curl, ports, `tracert`), what to check in the provider panel, how long it was off and how it ended (`journalctl --list-boots`, no shutdown lines = hard stop), MySQL recovery, failed jobs and missed schedules, and who asks the provider (Hostinger: hPanel chat, ask for a human) |

### apache
| Note | What it covers |
| --- | --- |
| [apache-laravel-deploy-by-nick.md](apache/apache-laravel-deploy-by-nick.md), [apache-laravel-deploy-by-susan.md](apache/apache-laravel-deploy-by-susan.md) | Laravel on Apache |
| [deploy-nextjs-apache.md](apache/deploy-nextjs-apache.md), [apache-deploy-react.md](apache/apache-deploy-react.md) | Next.js / React on Apache |
| [letsencrypt-apache.md](apache/letsencrypt-apache.md) | SSL on Apache |
| [increase-php-upload-limit-apache.md](apache/increase-php-upload-limit-apache.md) | Upload limits |
| [create-multiple-ips-aws.md](apache/create-multiple-ips-aws.md) | Multiple IPs on AWS |

### centos
| Note | What it covers |
| --- | --- |
| [laravel-nextjs-apache-centos-deploy.md](centos/laravel-nextjs-apache-centos-deploy.md) | Laravel API + Next.js on CentOS / RHEL 8 with httpd: what differs from Ubuntu + nginx (`apache` user, `conf.d`, certbot's `-le-ssl.conf` is the live site, SELinux booleans and `storage` labels, systemd queue worker instead of Supervisor, `next build --webpack` because of glibc 2.28, vault repos), full setup and a troubleshooting table ("Index of /" instead of the app, 503 from SELinux) |
| [laravel-nextjs-apache-centos-redeploy.md](centos/laravel-nextjs-apache-centos-redeploy.md) | Every code change: API (down, pull, composer, migrate, optimize, chown, up, restart worker) and web (pull, `npm ci`, `next build --webpack`, pm2 restart), checks, `git pull` refusals, rolling back |

### clodex
| Note | What it covers |
| --- | --- |
| [README.md](clodex/README.md) | How clodex, the VS Code launcher and the proxy server fit together |
| [update-procedure.md](clodex/update-procedure.md) | Checking that the extension, CLI and clodex versions match, and updating both the VS Code extension and the terminal CLI (stop the proxy → update → patch → rebuild the launcher → start the proxy → reload) |
| [troubleshooting.md](clodex/troubleshooting.md) | `EBUSY`/`EPERM` on install, missing models, `encrypted content could not be verified` |

### claude-code
| Note | What it covers |
| --- | --- |
| [vscode-tab-stopped-responding.md](claude-code/vscode-tab-stopped-responding.md) | "Claude Code stopped responding in this tab" / the 60 s start-up timeout: reading the logs (extension host unresponsive, event-loop stalls, MCP servers), checking RAM and the page file, and the fixes (shorter chats, free disk, dead MCP servers) |

### other
| Folder | Notes |
| --- | --- |
| [ci-cd-pipelines/](ci-cd-pipelines/) | Next.js CI/CD to a VPS |
| [git/](git/) | Switching HTTPS token auth to SSH; [`git pull` refusing after `chmod -R`](git/git-pull-refuses-after-chmod-file-mode.md) (file mode changes, `core.fileMode false`) |
| [sql/](sql/) | Export/import a database via File Browser |
| [supabase/](supabase/) | Supabase crash course |
| [reactnative-expo/](reactnative-expo/) | EAS builds (cloud, local, Docker, WSL), Expo dev on WSL |
| [aws/](aws/) | Domain registration |
| [cfp/](cfp/) | Apache to Nginx migration on the CFP servers |

## How to use

```sh
git clone https://github.com/bloodykheeng/notes.git
cd notes
```

Open the note for the job, replace the domains/paths at the top, follow it in order.
