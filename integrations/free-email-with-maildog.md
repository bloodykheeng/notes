# Free email for a client's domain with MailDog (send from the app + an inbox)

How Elevate Pesa got working email again (Nov 2026) after its Hostinger mailbox went with the Hostinger account:
the Laravel app sends password resets and order mail to anyone (Gmail, Yahoo...), and `admin@<domain>` has a
web inbox, for free, without running a mail server. Follow it top to bottom for the next client.

---

## 1. Pick the right kind of email first

| Option | Sending from the app | An inbox at `@domain` | Cost | Verdict |
| --- | --- | --- | --- | --- |
| **MailDog** free | yes, SMTP | yes, web inbox (200 MB) | $0: 4,000/month, 150/day, 1 domain | what we used |
| Brevo free | yes, SMTP, 300/day | no | $0 | good sender if no inbox is needed |
| Zoho Mail Forever Free | **no** (no SMTP/IMAP/POP on the free plan) | yes, web + app, 5 users | $0 | inbox only; Mail Lite (~$1/user) adds SMTP |
| Forwarding (ImprovMX, Cloudflare Email Routing) | no | forwards to an existing Gmail | $0 | receiving only |
| Google Workspace | yes | yes | ~$7/user/month | best, not free |
| Self-hosted (FastPanel, Mailcow, aaPanel on the VPS) | yes | yes | $0 + your time | **no**, see below |

**Why not self-host on the app's VPS.** Panels like FastPanel install their own nginx, Apache, PHP and MySQL: on a
box already running the app they take over or break it (they want an empty server). Even on its own server, a new
VPS IP (Contabo ranges have been on blocklists) lands in spam until warmed up, and you become the mail admin
(spam filtering, updates, storage, blocklists). And mail on the app's box dies with the box, which is exactly what
happened when Hostinger deleted the VPS.

Check the free limits against real traffic: Elevate Pesa had ~680 orders in 16 months, far below 150 mails a day.

---

## 2. MailDog setup

1. Sign up at `maildog.io` with the client's own Gmail (the account belongs to them).
2. **Domains → Add Domain** → the domain. MailDog shows four records, e.g.:

   | Type | Name | Value |
   | --- | --- | --- |
   | MX | `@` | `10 mail.maildog.io` |
   | TXT | `@` | `v=spf1 include:spf.maildog.io ~all` |
   | CNAME | `<selector>._domainkey` | `<selector>._domainkey.<domain>.dkim.maildog.io` |
   | TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:postmaster@mail.maildog.io` |

   The selector (`md687609e0` for Elevate Pesa) is per domain: copy yours from the panel.
3. Add them at whoever hosts the DNS (section 3), then **Refresh DNS** until MX, SPF, DKIM and DMARC are green.
4. **Mailboxes → Create** `admin@<domain>` (or `no-reply@`). Its address and password **are the SMTP login**;
   there is no separate API key. The eye/copy icons beside the dots give the password. Store it in the
   project's `deployment/<provider>/` txt, never in a repo.
5. **Sending Setup** shows the SMTP values: host `mail.maildog.io`, port `587` (STARTTLS, recommended) or `465`
   (SSL), username = the full mailbox address. Port 25 is not available for sending.

---

## 3. Adding the records on Contabo DNS (and the mistakes we made)

Network → DNS Management → the zone → **Add a new resource record**.

- **Name empty = the domain itself.** For the MX and SPF leave Name empty (`@` if it insists). Typing the
  priority `10` into Name creates a record for `10.<domain>`.
- **Edit the existing MX, do not add a second one.** Contabo's zone template already has
  `MX 10 mail.<domain>` pointing at the VPS. Two MX with the same priority split incoming mail between MailDog and a
  server that cannot receive it. Pencil on that row → Data `mail.maildog.io`, Priority `10`.
- Only **one SPF TXT** may exist at the root. If one is there (another sender), merge:
  `v=spf1 include:spf.maildog.io include:other ~all`.
- CNAME and DMARC names are relative: `md687609e0._domainkey`, `_dmarc` (Contabo shows them with the domain
  appended afterwards). Paste values exactly: no quotes, no trailing spaces.
- The wildcard `*` A record does not clash with these.
- If the client already reads email at the domain somewhere else, **keep that provider's MX** and use MailDog for
  sending only (SPF then lists both).

Check from your machine before trusting the panel:

```bash
for q in "MX example.com" "TXT example.com" "CNAME md687609e0._domainkey.example.com" "TXT _dmarc.example.com"; do
  set -- $q; echo "$1 $2: $(curl -s "https://dns.google/resolve?name=$2&type=$1" | grep -o '"data":"[^"]*"')"
done
nslookup -type=MX example.com ns1.contabo.net       # the authoritative answer
```

The TXT lookup on the DKIM name should come back with `v=DKIM1; ... p=MII...` through the CNAME.

---

## 4. Laravel `.env`

```
MAIL_MAILER=smtp
MAIL_HOST=mail.maildog.io
MAIL_PORT=587
MAIL_USERNAME=admin@example.com
MAIL_PASSWORD="the-mailbox-password"
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="admin@example.com"
MAIL_FROM_NAME="${APP_NAME}"
```

- **Quote the password.** One starting with `#` is otherwise a comment and the value is empty.
- Make `MAIL_FROM_ADDRESS` the mailbox you log in with (the panel's sample says `noreply@`, which has no mailbox).
- Edit the env file on your machine, upload it, back up the live one first:

  ```bash
  cp /var/www/api.example.com/.env /root/api.example.com.env.bak-$(date +%F-%H%M)
  install -o root -g www-data -m 640 /root/new-api.env /var/www/api.example.com/.env
  ```

- With a cached config run `sudo -u www-data php artisan config:cache` (or `optimize`) afterwards; without one the
  next request reads the file.

Check on the server:

```bash
cd /var/www/api.example.com
sudo -u www-data php artisan config:show mail.mailers.smtp | grep -vi password
timeout 8 bash -c "</dev/tcp/mail.maildog.io/587" && echo "587 open" || echo "587 BLOCKED"
sudo -u www-data HOME=/tmp php artisan tinker --execute='Illuminate\Support\Facades\Mail::raw("Test from the server via MailDog.", fn ($m) => $m->to("someone@gmail.com")->subject("Test email")); echo "SENT OK\n";'
```

`SENT OK` means MailDog accepted it. MailDog queues and delivers about one per second per receiving domain, so
allow a minute; follow it in **Email Logs**. Then check the recipient's inbox and spam folder.

---

## 5. Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| Records stay red in MailDog | not live yet (check with `dns.google`), a typo, the priority typed into Name, or a second MX/SPF |
| `Connection could not be established` / timeout | outbound 587 blocked by the host or ufw egress rules: test with `/dev/tcp`, try 465 + `ssl` |
| `535 Authentication failed` | username not the full address, password with `#` unquoted, or a password from a different mailbox |
| `553`/`550` sender rejected | `MAIL_FROM_ADDRESS` is not a mailbox on a verified domain |
| `450 try again later` | over 1,000/min per mailbox; Laravel's mailer retries, nothing is lost |
| Arrives in spam | DKIM/SPF not green yet, or new domain reputation: send a few real mails, keep `p=none` until it settles |
| Daily limit reached | free plan is 150/day: Basic ($15/month) or add Brevo for bulk mail |
