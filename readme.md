# Notes

Technical notes and solutions for deployments, server configuration and recurring fixes.
Every note is written to be followed top to bottom and copy-pasted.

## Start here

- **[nginx/laravel-nextjs-two-environments-deploy.md](nginx/laravel-nextjs-two-environments-deploy.md)**:
  full build of a fresh Ubuntu VPS for a Laravel API (queue workers, scheduler, Reverb) plus
  a Next.js frontend (pm2), two environments on one box, Nginx + Let's Encrypt. Includes the
  redeploy procedure and a troubleshooting table.

## Contents

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

### linux
| Note | What it covers |
| --- | --- |
| [jobs-and-queues.md](linux/jobs-and-queues.md) | Laravel scheduler (cron) and queue workers (Supervisor) |
| [github-ssh-authentication.md](linux/github-ssh-authentication.md) | SSH keys for Git hosting |
| [fail2ban-installation-and-ssh-protection.md](linux/fail2ban-installation-and-ssh-protection.md) | SSH hardening |
| [install-google-authenticator-on-linux.md](linux/install-google-authenticator-on-linux.md) | 2FA on SSH |

### apache
| Note | What it covers |
| --- | --- |
| [apache-laravel-deploy-by-nick.md](apache/apache-laravel-deploy-by-nick.md), [apache-laravel-deploy-by-susan.md](apache/apache-laravel-deploy-by-susan.md) | Laravel on Apache |
| [deploy-nextjs-apache.md](apache/deploy-nextjs-apache.md), [apache-deploy-react.md](apache/apache-deploy-react.md) | Next.js / React on Apache |
| [letsencrypt-apache.md](apache/letsencrypt-apache.md) | SSL on Apache |
| [increase-php-upload-limit-apache.md](apache/increase-php-upload-limit-apache.md) | Upload limits |
| [create-multiple-ips-aws.md](apache/create-multiple-ips-aws.md) | Multiple IPs on AWS |

### other
| Folder | Notes |
| --- | --- |
| [ci-cd-pipelines/](ci-cd-pipelines/) | Next.js CI/CD to a VPS |
| [git/](git/) | Switching HTTPS token auth to SSH |
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
