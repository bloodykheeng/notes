# Notes

Technical notes and solutions for deployments, server configuration and recurring fixes.
Every note is written to be followed top to bottom and copy-pasted.

## Start here

- **[ai-agents/who-i-am-and-how-i-work.md](ai-agents/who-i-am-and-how-i-work.md)**: give this to any AI coding agent
  first. Who I am, how I talk (fast, screenshots, "what's your view"), how we decide and record decisions, how a
  project is set up (workspace, CLAUDE.md, plan, decisions, handoffs, docs as we go, commit only when asked), the
  quality bar, and every rule I have had to repeat (lists, confirms, no hidden filters, back keeps the list's place,
  card arrows, reports with records, data rules). Then the build or deploy playbook below.
- **[ai-agents/deploy-with-an-ai-agent.md](ai-agents/deploy-with-an-ai-agent.md)**: give this to the AI agent
  before any deploy or redeploy. How we work (agent writes the commands, I run them and paste the output back),
  read-only checks first, loops over environments, paste-safe blocks, secrets, and the
  `docs/deployment.md` + `docs/redeployment.md` it must leave in the code repo.
- **[ai-agents/deploy-yourself-over-ssh.md](ai-agents/deploy-yourself-over-ssh.md)**: the other mode, when I put
  the SSH login in a txt file and the agent runs the whole deploy itself (password helper, secrets moved file to
  file, restoring from a hosting backup, DNS checks, clean-up).
- **[nginx/laravel-nextjs-two-environments-deploy.md](nginx/laravel-nextjs-two-environments-deploy.md)**:
  full build of a fresh Ubuntu VPS for a Laravel API (queue workers, scheduler, Reverb) plus
  a Next.js frontend (pm2), two environments on one box, Nginx + Let's Encrypt. Includes the
  redeploy procedure and a troubleshooting table.
- **[centos/laravel-nextjs-apache-centos-deploy.md](centos/laravel-nextjs-apache-centos-deploy.md)**:
  the same stack on a CentOS / RHEL 8 box with Apache (httpd) and SELinux Enforcing, plus its
  [redeploy procedure](centos/laravel-nextjs-apache-centos-redeploy.md).
- **[project-inbox/](project-inbox/README.md)**: lessons one project left for another, one folder per receiving
  project (`project-inbox/alpha/`, `sds/`, `letshego/`...). Inside a project, say "check coding-notes for lessons":
  its agent applies what holds, then moves the note into that project for history.

## Read before you touch

**For the AI agent.** Before you give a view, write a plan or change code on any of these, open the note on the
right. Not after I remind you: if it is written here, you are expected to know it.

| If the work touches... | Read first |
| --- | --- |
| Any project at all | [who-i-am-and-how-i-work.md](ai-agents/who-i-am-and-how-i-work.md), then the project's own `CLAUDE.md` and `docs/decisions.md` |
| **Any screen change** (before saying it is done: run the app, take pictures, look at them) | [look-at-your-own-screens.md](ai-agents/look-at-your-own-screens.md), and the design skills in [agent-skills.md](ai-agents/agent-skills.md) |
| **Navigation, record pages, a client/patient pipeline, details dialogs, selects** (web, Flutter, React Native) | [navigation-and-client-pipeline.md](ai-agents/navigation-and-client-pipeline.md) |
| Installing an agent skill from the internet | [agent-skills.md](ai-agents/agent-skills.md) (read it all, check the scripts, SOURCE.txt) |
| **Dates, times, timezones, "today", reports by day or month, MySQL timestamps** (session timezone pin!) | [dates-and-timezones-with-an-ai-agent.md](ai-agents/dates-and-timezones-with-an-ai-agent.md) |
| Models, migrations, statuses, foreign keys | [enums-and-migrations.md](ai-agents/enums-and-migrations.md), and the FK rules in [build-laravel-nextjs-app-with-an-ai-agent.md](ai-agents/build-laravel-nextjs-app-with-an-ai-agent.md) |
| Editable data that other records use (prices, schedules, standards) | [master-copy-and-snapshots.md](ai-agents/master-copy-and-snapshots.md) |
| Audit trail | [audit-trail-with-snapshot-schemas.md](ai-agents/audit-trail-with-snapshot-schemas.md) |
| A Laravel + Next.js build: roles and permissions, lists, dialogs, Reverb, notifications, reports | [build-laravel-nextjs-app-with-an-ai-agent.md](ai-agents/build-laravel-nextjs-app-with-an-ai-agent.md), [letshego-field-sales-lessons.md](ai-agents/letshego-field-sales-lessons.md) |
| **A Flutter build fails**, or setting up a PC to build Flutter (Windows, Android, build_runner) | [flutter/flutter-build-troubleshooting.md](flutter/flutter-build-troubleshooting.md) |
| **A Flutter app** (Windows desktop or Android), offline sync, device time, installers, push | [flutter/flutter-engineering-standards-for-agents.md](flutter/flutter-engineering-standards-for-agents.md) (how to work, structure, stack, quality gate), [flutter/flutter-offline-desktop-and-mobile.md](flutter/flutter-offline-desktop-and-mobile.md), [flutter/flutter-phones-dialogs-navigation-lessons.md](flutter/flutter-phones-dialogs-navigation-lessons.md) (phones: status bar, Back, dialogs, pickers, forms), [mrdrs-lessons.md](ai-agents/mrdrs-lessons.md) |
| **A client says "offline"**, or users with unreliable internet | [offline-and-online-explaining-to-clients.md](ai-agents/offline-and-online-explaining-to-clients.md) |
| A system for many sites, offline, several users per site | [mrdrs-lessons.md](ai-agents/mrdrs-lessons.md), [offline-and-online-explaining-to-clients.md](ai-agents/offline-and-online-explaining-to-clients.md) |
| Firebase push (web or Android) | [nextjs/firebase-messaging-sw-and-nextauth-google.md](nextjs/firebase-messaging-sw-and-nextauth-google.md), [nextjs/firebase-push-token-errors-check-versions-first.md](nextjs/firebase-push-token-errors-check-versions-first.md) |
| Reverb on a server | [nginx/laravel-reverb-production-nginx.md](nginx/laravel-reverb-production-nginx.md) |
| Setting up a workspace with app repos inside it (git colours in VS Code) | [vscode/git-colours-for-nested-repos.md](vscode/git-colours-for-nested-repos.md) |
| A deploy or redeploy | [deploy-with-an-ai-agent.md](ai-agents/deploy-with-an-ai-agent.md) or [deploy-yourself-over-ssh.md](ai-agents/deploy-yourself-over-ssh.md) |
| Security patching PHP / Node | [security/patch-php-node-with-an-ai-agent.md](security/patch-php-node-with-an-ai-agent.md) |
| Publishing app builds (apk, Windows installer) for download, Cloudflare R2 | [cloudflare/r2-publish-app-builds-with-an-agent.md](cloudflare/r2-publish-app-builds-with-an-agent.md) |

## Contents

### ai-agents
| Note | What it covers |
| --- | --- |
| [who-i-am-and-how-i-work.md](ai-agents/who-i-am-and-how-i-work.md) | Read first, every project: me, how to read my messages, how decisions are made and recorded, what decides (and what does not), project setup, the "professional not vibe-coded" bar, the rules I keep repeating (data, API, lists, dialogs, forms, dashboards, reports, maps, mobile), and how to work in a session |
| [offline-and-online-explaining-to-clients.md](ai-agents/offline-and-online-explaining-to-clients.md) | "We want it fully offline": what clients really mean (work must not stop, data costs money, distrust of the internet), the connectivity spectrum (fully offline, local-first, offline-capable with sync, online-first with fallback, online-only), classifying every feature into offline OK / offline confirmed by the server / online only, the hidden costs of offline, anti-patterns, a plain-words script for the client, and a checklist |
| [look-at-your-own-screens.md](ai-agents/look-at-your-own-screens.md) | The screenshot runner: a test that drives the real app (Flutter integration_test, Playwright for Next.js / Laravel), saves a picture of every main screen and changed flow in light and dark, and the loop the agent follows (run, look, fix, re-run) before telling me a screen is done. Credentials from the command line, pictures git-ignored, walk without saving, pitfalls (physical pixels, apps that never go idle, window plugins in tests) |
| [navigation-and-client-pipeline.md](ai-agents/navigation-and-client-pipeline.md) | How people move around a business app and how to build it in Next.js, Flutter and React Native: one list per kind of person with status tabs, View opens the person, the work done from their page (current step + buttons), chips and a context bar, the sidebar keeping its section, details that read like details (accordions, first open), dialogs with headers, selects that open in place, 12-hour dates, and a checklist |
| [agent-skills.md](ai-agents/agent-skills.md) | The agent skills installed on my PC and what each is for (ui-ux-design-pro, ui-ux-pro-max, frontend-design, web-design-guidelines, react-best-practices, find-skills), project rules winning over a skill, and installing one safely (own folder, read everything, grep the scripts for network and exec, copy only what it needs, SOURCE.txt) |
| [mrdrs-lessons.md](ai-agents/mrdrs-lessons.md) | MRDRS (Nov 2026): a facility records + DHIS2 app that grew from one user to many facilities without a plan. The mistakes (no deployment design, shared SQLite file, single tenant, report-first screens, unsafe credential handling, roles built by hand, phone layouts on PCs, 2 s polling) with the rule now, what was kept, and the two-track way out |
| [letshego-field-sales-lessons.md](ai-agents/letshego-field-sales-lessons.md) | The Letshego Field Sales story (Oct–Nov 2026): prototype first, the workspace, the foundation copied from GiftNest, decisions as they were made (client vs customer, stage read never stored, drop as a flag, reach by assignment), a table of the mistakes an agent made with my correction and the rule now, the patterns worth reusing (list memory for Back, card arrows, report scope, timeline from facts) and the process lessons |
| [build-laravel-nextjs-app-with-an-ai-agent.md](ai-agents/build-laravel-nextjs-app-with-an-ai-agent.md) | The playbook for an AI agent building a Laravel + Next.js business app with me: how we work (facts first, search best practice online, root cause, tests and docs with every change), folders, backend conventions (ULIDs, serial counters, enums, money, UTC, migrations, files), the maker/checker approval engine, roles and permissions, sessions and the portal cache bug, lists (pagination, URL state, 3-state sort, chips, one filter form, skeletons, typed exports), dialogs and forms, help text, documents, audit trail, Reverb realtime, reports, packages, testing and docs |
| [dates-and-timezones-with-an-ai-agent.md](ai-agents/dates-and-timezones-with-an-ai-agent.md) | Dates done right in Laravel + Next.js + a mobile app: store and query UTC, "a day" from one business timezone, bare date = whole day and a time = exact; the column kinds (UTC instant vs calendar date); the traps that were live in production (Laravel binds a Carbon without converting its timezone, an end date that drops the last day, `whereDate`/`DATE()` on UTC columns, browser `new Date("YYYY-MM-DD")`, pickers that discard the time, MySQL TIMESTAMP and the DB session timezone, uncast datetime columns, Carbon's month overflow, caches); how to check a whole app (golden master, a `DB::pretend()` guard test, a real-data detector, tallies of dashboard figures against the lists they link to, static sweeps); and the method mistakes an agent made on NICE, so the next one does not |
| [master-copy-and-snapshots.md](ai-agents/master-copy-and-snapshots.md) | Editable data and its history: one master copy edited directly, facts frozen on the records that use them (SDS standards per inspection period; NICE's journey plan schedule, first built with versioned rows and rebuilt), live identities referenced directly, no keys to mutable rows, changes applied forward, upload modes and ownership so mistakes clean up in one step, and a checklist |
| [audit-trail-with-snapshot-schemas.md](ai-agents/audit-trail-with-snapshot-schemas.md) | The audit trail engine from SDS and Alpha: what an entry must answer, `HasSnapshotSchema` on every model, `SnapshotPresenter` (values resolved and frozen at write time), the queued `LogActivityJob`, the module enum, the generic view dialog, the rules that make it trustworthy, how to upgrade an app that logs sentences, and what NICE's upgrade found (secrets in SMS entries, noise, dumps, broken calls) and the mistakes an agent made |
| [enums-and-migrations.md](ai-agents/enums-and-migrations.md) | Every fixed list as a PHP enum cast on the model (strings in the database, values listed in the migration comment, web mirrors), reading and writing cast values, and migrations people can read (every column written out, grouped and spaced, explanations above, no date columns, edit in place only before a deploy), with the mistakes an agent made |
| [deploy-with-an-ai-agent.md](ai-agents/deploy-with-an-ai-agent.md) | The playbook for an AI agent deploying a Laravel + Next.js project with me: study the repo first, read-only server checks, env files made locally (fresh secrets, never in git), rounds of short verified blocks, `for` loops over environments, nothing interactive mid-paste, backups and asking before destructive steps, stop-all/restart-all on small boxes, secrets never echoed, and writing `docs/deployment.md` + `docs/redeployment.md` (plus a handoff) into the repo |
| [deploy-yourself-over-ssh.md](ai-agents/deploy-yourself-over-ssh.md) | When the agent has the SSH login (in a txt on my machine) and runs the deploy itself: folder layout, reading secrets without leaking them (masks, file-to-file), the `SSH_ASKPASS` password helper for Windows OpenSSH (and asking before installing a key), the order of work for restoring a site from a hosting backup (baked API URLs, `env()` vs `config:cache`, MariaDB dump's sandbox line, `tar --force-local`), DNS traps (no zone = REFUSED, Contabo's template MX, empty Name = root, RDAP for the registrar), Windows traps, clean-up and reporting (Elevate Pesa, Nov 2026) |

### cloudflare
| Note | What it covers |
| --- | --- |
| [r2-publish-app-builds-with-an-agent.md](cloudflare/r2-publish-app-builds-with-an-agent.md) | App builds handed out from an R2 bucket: one folder per app and environment, fixed file names and a `release.json` the download page reads; the bucket-scoped API token (Object Read & Write) saved outside the repos; the agent uploads with curl's `--aws-sigv4` and a temporary config so keys never show; content types, `no-cache` on `release.json`, upload order, checks from outside; checking what server a build talks to before release; SmartScreen and code signing |

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
| [laravel-production-cache-permission-error-fix.md](nginx/laravel-production-cache-permission-error-fix.md) | `file_put_contents ... storage/framework/cache` (root vs www-data); Passport refusing `oauth-*.key` at 664 after a storage sweep (`chmod 600`) |
| [install-ssl-on-nginx.md](nginx/install-ssl-on-nginx.md) | Certbot |
| [increase-php-upload-limit-nginx.md](nginx/increase-php-upload-limit-nginx.md) | Upload limits |
| [install-filebrowser-nginx.md](nginx/install-filebrowser-nginx.md) | File Browser behind Nginx |
| [postgres-pgvector-laravel-deploy.md](nginx/postgres-pgvector-laravel-deploy.md) | Postgres + pgvector for Laravel |

### integrations
| Note | What it covers |
| --- | --- |
| [google-workspace-mail-firebase-oauth-recaptcha.md](integrations/google-workspace-mail-firebase-oauth-recaptcha.md) | Where every Google-side env value comes from: Workspace SMTP + App Password, Firebase config and service account, OAuth client, reCAPTCHA, and the walls (2SV, org policy, project quota) |
| [free-email-with-maildog.md](integrations/free-email-with-maildog.md) | Free email for a client's domain: the options compared (MailDog, Brevo, Zoho free has no SMTP, forwarding, why not self-host FastPanel on the app's VPS), MailDog domain + the four DNS records on Contabo (empty Name = root, edit the template MX, one SPF), mailbox = SMTP login, the Laravel `.env` (quote a `#` password), the tinker test and port check, troubleshooting |

### nextjs
| Note | What it covers |
| --- | --- |
| [firebase-messaging-sw-and-nextauth-google.md](nextjs/firebase-messaging-sw-and-nextauth-google.md) | One Firebase service worker for every environment (config via query string) and Sign in with Google: NextAuth v4 for the handshake, the API verifies Google's ID token (never a posted email), with tests |
| [firebase-push-token-errors-check-versions-first.md](nextjs/firebase-push-token-errors-check-versions-first.md) | `VersionError (1) < (2)` and `messaging/permission-blocked` when getting a push token: Firebase 13 raised its IndexedDB to v2, `localhost:3000` storage is shared by every project, how to compare versions in five minutes, the guard that drops a newer copy, and why restarting never helps |
| [leaflet-map-backgrounds.md](nextjs/leaflet-map-backgrounds.md) | One set of map backgrounds on every Leaflet map (OpenStreetMap, Google streets / satellite / satellite with labels `lyrs=y` / terrain, Esri Dark and Grey with labels), the theme's default and the remembered pick; traps: `lyrs=h` is labels only, a stray TileLayer under the switcher, waiting for tiles in a test (`complete`, not the loaded class) |
| [leaflet-plan-vs-visits-map.md](nextjs/leaflet-plan-vs-visits-map.md) | One person's plan against their visits on a map over a date range: pins by status, a millipede per day with a start flag to fold it, follow the dialog's period and filters (a 31-day cap showed an empty month), fullscreen and print like the other maps |
| [leaflet-map-rotation.md](nextjs/leaflet-map-rotation.md) | Turning a react-leaflet map like Google Maps with leaflet-rotate: Shift + scroll, Ctrl + drag, a two-finger twist, a compass; the `globalThis.L` trap with a bundler, types for 0.6, print north up, labels turning with raster tiles, and a Playwright check |
| [primereact-tailwind-dark-mode-missing-classes.md](nextjs/primereact-tailwind-dark-mode-missing-classes.md) | PrimeReact's Tailwind preset + Tailwind 4: white-on-white selected options, date-picker days that look disabled, no "today" circle. Why (node_modules is never scanned) and the CSS that fixes it |
| [laravel-validation-error-toasts-without-duplicates.md](nextjs/laravel-validation-error-toasts-without-duplicates.md) | The same Laravel 422 error shown in two toasts: why (Laravel's `message` repeats the first field error) and the hook logic that shows each thing once |
| [dashboard-sidebar-groups.md](nextjs/dashboard-sidebar-groups.md) | A dashboard sidebar people can scan: a few small headings, related pages folded under one parent, children on a guide line, one lit row, gates that drop empty parents and groups, the collapsed rail, and the Playwright check that the folded menu fits a laptop screen (SDS, GiftNest, NICE) |
| [primereact-inputnumber-zod-form-traps.md](nextjs/primereact-inputnumber-zod-form-traps.md) | InputNumber `onChange` vs `onValueChange`, `min`/`max` that only clamp the screen, `z.coerce` and 0 defaults letting empty fields through, AutoComplete text vs a picked item; plus an audit grep |
| [primereact-datatable-pinned-actions.md](nextjs/primereact-datatable-pinned-actions.md) | Row buttons always in view on a wide list: the Actions column frozen right, its cell inheriting the row's colour (striped, hovered) with a hairline, unpinned in the phone card layout, action tooltips on the left so the arrow points at the button, and fewer columns by default (Elevate Pesa) |

### linux
| Note | What it covers |
| --- | --- |
| [jobs-and-queues.md](linux/jobs-and-queues.md) | Laravel scheduler (cron) and queue workers (Supervisor) |
| [laravel-production-logs-and-debug.md](linux/laravel-production-logs-and-debug.md) | `APP_DEBUG=false`, daily logs kept 30 days (`LOG_STACK=daily`, `LOG_DAILY_DAYS`), business refusals kept out of the log (`ShouldntReport`, not `report()` false), the one-box fix for a live server with backup and `config:show` checks, and reading daily logs by their entry headers |
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

### android
| Note | What it covers |
| --- | --- |
| [samsung-s10-custom-rom-yurom.md](android/samsung-s10-custom-rom-yurom.md) | Galaxy S10 (SM-G973F) from ExtremeROM to YuROM Ultra (Android 17): official TWRP bootloop, the installer's busybox and exact-vendor-size checks patched (hash chain rebuilt), f2fs to ext4 data, installing and verifying over adb, the 5-6 restart first boot, Magisk root, a custom first-screen logo in `up_param`, why the QMG boot animation is mis-scaled, and a Magisk module that clears `/data/apex/active` after 3 failed boots; the missing region (CSC/OMC) fixed (optics/prism mounts + Magisk props, `mdc.system.path`) to bring back call recording, decoding Samsung's encrypted `cscfeature.xml`, and the QS mobile-data tile traced to telephony publishing active data sub -1 |

### flutter
| Note | What it covers |
| --- | --- |
| [flutter-build-troubleshooting.md](flutter/flutter-build-troubleshooting.md) | Every Flutter build error hit setting up a fresh Windows build PC, with cause and fix: font_awesome vs Flutter 3.50 (`IconData` final, analyze silent), NuGet empty sources (primarySources), missing ATL (`atlstr.h`), two Visual Studio installs (Flutter picks one; env var ignored), installer exit 5007, build_runner 6 GB, lost `@DataClassName` after a split, drift NOT NULL warnings, the template counter test; and the check order (baseline analyze, test, build per target) |
| [flutter-engineering-standards-for-agents.md](flutter/flutter-engineering-standards-for-agents.md) | The bar for any agent on our Flutter apps ("don't gamble"): facts and official docs before patterns, `flutter pub add` at latest, spike risky plugins, root cause not trial and error, one way of doing each thing; feature-first structure with thin screens; the stack (Riverpod 3, go_router, dio, freezed, drift, flavours); one sync engine and one permission service; desktop-first screens; the quality gate (format, analyze, test, real device); repo hygiene; the anti-patterns seen on MRDRS |
| [flutter-offline-desktop-and-mobile.md](flutter/flutter-offline-desktop-and-mobile.md) | Flutter apps that run on Windows PCs and Android phones, offline, syncing to a Laravel API: local store + outbox + server (never a shared SQLite file), ids made on the device (UUID v7), dates on the device (UTC on the wire, a fixed-offset business day, a wrong clock/zone banner), device enrolment and tokens, Reverb from Dart, Firebase (no FCM on Windows), desktop-first layouts, permissions in one place, installers and in-app updates, Android release, phone data costs, and what went wrong on MRDRS |
| [flutter-phones-dialogs-navigation-lessons.md](flutter/flutter-phones-dialogs-navigation-lessons.md) | A desktop-first Flutter app on phones, the lessons with root causes: test on a real phone over adb; edge-to-edge status bar; an environment strip that pushes the screen down; Android Back with a shell and nested Navigator (page, section trail, previous section, Dashboard, exit); stay signed in; a sidebar that reveals the open section and marks its parent; one first-party picker for touch; maximizable dialogs (full screen on phones); Wrap-in-Row and other layout traps; never parse a displayed date in onSaved; derived rows that take the source id; plugins still on the v1 embedding |

### architecture
| Note | What it covers |
| --- | --- |
| [mrdrs-track-a-architecture.md](architecture/mrdrs-track-a-architecture.md) | MRDRS Track A in plain words, for the team: one cloud server and one database for 40+ facilities, devices enrolled with a code, sync (what stays, what is new), Reverb signals, offline vs online per feature, ready-made roles, notifications by platform, dates and ids, the file structure of the API, the Flutter app and the console, how we work together |

### vscode
| Note | What it covers |
| --- | --- |
| [git-colours-for-nested-repos.md](vscode/git-colours-for-nested-repos.md) | No git colours or Source Control when I open the root folder above `<project>-workspace/`: VS Code scans one level deep; a root `.vscode/settings.json` with `git.scanRepositories` listing each app repo fixes it (Elevate Pesa, MRDRS). The agent creates it at workspace setup |

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
