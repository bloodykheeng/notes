# Google services for a Laravel + Next.js app: where every env value comes from

How to create the Google-side pieces an app needs and which `.env` key each value fills:
Workspace mailbox for SMTP, Firebase Cloud Messaging, Sign in with Google (OAuth),
reCAPTCHA v3. Includes every wall hit while doing it (2-Step Verification, org policy,
project quota).

Code side (service worker, NextAuth) is in
[../nextjs/firebase-messaging-sw-and-nextauth-google.md](../nextjs/firebase-messaging-sw-and-nextauth-google.md).

---

## 0. One Google project per environment

Create **two** Firebase projects, `<App> Test` and `<App>`, and put each environment's OAuth
client and reCAPTCHA in that environment's project. Local dev shares the Test project.

Why not one project with two web apps: the Firebase service-account key on the test server
would be able to push to production users, and one OAuth consent screen has one publish
state (test should stay unpublished, allow-listed users only). Two projects are free.

| | Test project | Production project |
| --- | --- | --- |
| Used by | localhost, `test.example.com` | `app.example.com` |
| OAuth audience | Testing (allow-listed emails) | Published |
| reCAPTCHA domains | `localhost`, `example.com` | `example.com` |

Which account creates them matters (see the gotchas at the end): a plain Gmail account is
the least friction; a Workspace user's project lands inside the company's Google Cloud
organisation, whose default policy blocks service-account keys.

---

## 1. Mail: Google Workspace mailbox over SMTP

Use when the client's domain MX is `smtp.google.com` (`nslookup -type=MX example.com`).

1. Workspace admin creates the mailbox (admin.google.com > Directory > Users), e.g.
   `noreply@example.com`. Sign in once and accept the terms.
2. Google refuses normal passwords from apps. You need an **App Password**, which requires
   **2-Step Verification** on that user:
   - If the user sees "2-Step Verification is disabled for your account", the admin must
     allow it first: Admin console > Security > Authentication > 2-Step Verification >
     "Allow users to turn on 2-Step Verification" for the user's org unit (enforcement Off).
   - As the mailbox: Security & sign-in > 2-Step Verification > turn on (phone).
   - https://myaccount.google.com/apppasswords > name it > Create. 16 characters, shown once.
3. Laravel `.env` (Laravel 12+ reads `MAIL_SCHEME`; `MAIL_ENCRYPTION` is ignored):

```env
MAIL_MAILER=smtp
MAIL_SCHEME=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=noreply@example.com
MAIL_PASSWORD="<16-char app password>"
MAIL_FROM_ADDRESS=noreply@example.com
MAIL_FROM_NAME="${APP_NAME}"
```

Port 587 (STARTTLS), not 465: some ISPs block 465 outbound (`Test-NetConnection
smtp.gmail.com -Port 465` to check). Limit: 2,000 mails per day per mailbox.

4. Deliverability: check the domain has SPF (`TXT @` = `v=spf1 include:_spf.google.com ~all`)
   and DKIM on (Admin console > Apps > Google Workspace > Gmail > Authenticate email).
   Without them Google-sent mail still lands in spam.
5. Mailbox avatar is "managed by your organization": the admin sets it at Directory > Users >
   the user > photo. Use a square image; Gmail crops it to a circle.

Server-only alternative, no password: Workspace **SMTP relay** (`smtp-relay.gmail.com:587`,
IP allow-list of the server) at Admin console > Apps > Google Workspace > Gmail > Routing.

---

## 2. Firebase Cloud Messaging (web push)

Per environment:

1. https://console.firebase.google.com > Create a project > name > Google Analytics off.
2. Project settings > General > Your apps > `</>` (web) > nickname > Register app. The
   `firebaseConfig` block maps 1:1 to the frontend env:

| firebaseConfig | Next.js `.env` |
| --- | --- |
| `apiKey` | `NEXT_PUBLIC_FIREBASE_API_KEY` |
| `authDomain` | `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` |
| `projectId` | `NEXT_PUBLIC_FIREBASE_PROJECT_ID` |
| `storageBucket` | `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` |
| `messagingSenderId` | `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` |
| `appId` | `NEXT_PUBLIC_FIREBASE_APP_ID` |
| `measurementId` | `NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID` |

3. Project settings > Cloud Messaging > Web Push certificates > Generate key pair >
   `NEXT_PUBLIC_FIREBASE_VAPIDKEY` (optional; the SDK falls back to a default pair).
4. Project settings > Service accounts > Generate new private key. The JSON maps 1:1 to the
   Laravel env, all 11 keys:

| JSON | Laravel `.env` |
| --- | --- |
| `type` | `FIREBASE_TYPE` |
| `project_id` | `FIREBASE_PROJECT_ID` |
| `private_key_id` | `FIREBASE_PRIVATE_KEY_ID` |
| `private_key` | `FIREBASE_PRIVATE_KEY` |
| `client_email` | `FIREBASE_CLIENT_EMAIL` |
| `client_id` | `FIREBASE_CLIENT_ID` |
| `auth_uri` | `FIREBASE_AUTH_URI` |
| `token_uri` | `FIREBASE_TOKEN_URI` |
| `auth_provider_x509_cert_url` | `FIREBASE_AUTH_PROVIDER_X509_CERT_URL` |
| `client_x509_cert_url` | `FIREBASE_CLIENT_X509_CERT_URL` |
| `universe_domain` | `FIREBASE_UNIVERSE_DOMAIN` |

`FIREBASE_PRIVATE_KEY` must be **one double-quoted line** with the `\n` escapes exactly as in
the JSON. Split into real lines, `php artisan` dies with "The environment file is invalid".

5. Laravel: read these through `config/services.php` (`'firebase' => [ 'project_id' =>
   env('FIREBASE_PROJECT_ID'), ... ]`) and `config('services.firebase')` in the service class,
   never `env()` in app code: `env()` returns `null` once `php artisan config:cache` runs on the
   server and push dies silently.

---

## 3. Sign in with Google (OAuth client)

Inside the Firebase project of the same environment (Firebase projects are Google Cloud
projects). Check the project selector at the top before every step.

1. https://console.cloud.google.com/auth/overview > Get started: app name, support email,
   audience **External**, contact email > Create.
2. Branding: logo, home page, **Authorised domains** = the site's root domain.
3. Audience:
   - Test project: stay in **Testing**, add testers' Google accounts under Test users.
   - Production project: **Publish app**.
4. Clients > Create client > Web application:
   - Authorised JavaScript origins: `http://localhost:3000`, `https://test.example.com`
     (prod: `https://app.example.com`)
   - Authorised redirect URIs: each origin + the callback path. NextAuth v4:
     `/api/auth/callback/google`.
   - Create > copy Client ID and secret (secret shown once; download the JSON).

```env
GOOGLE_CLIENT_ID=<client id>.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=<secret>
```

A client ID starts with the project **number**, a quick way to see which project it belongs
to. `redirect_uri_mismatch` at sign-in = the current `NEXTAUTH_URL` is not on the client.

---

## 4. reCAPTCHA v3

1. https://www.google.com/recaptcha/admin/create (pick the classic key if it pushes
   Enterprise). Label, type **Score based (v3)**, domains (`localhost` + root domain for
   test, root domain only for prod; subdomains are covered). Under "Google Cloud Platform"
   pick the matching Firebase project.
2. Site key to the frontend, secret to Laravel:

```env
# Next.js
NEXT_PUBLIC_RECAPTCHA_SITE_KEY=<site key>
# Laravel
RECAPTCHA_SECRET_KEY=<secret key>
```

---

## 5. Handing projects to the client later

No env change, only ownership:

- Firebase > Project settings > Users and permissions > Add member > client account > Owner
  (they must accept the emailed invite).
- reCAPTCHA > gear > Owners > add the client account.
- OAuth consent screen > support and contact email > the client account.
- Then downgrade or remove your own account.

---

## Gotchas met while doing this

| Wall | What happened | Way through |
| --- | --- | --- |
| "The setting you are looking for is not available for your account" on App passwords | 2-Step Verification off for that Workspace user | Admin allows 2SV for the org unit, user turns it on, then App passwords appears |
| 535-5.7.8 `Username and Password not accepted` | Login password used, or App Password made in another Google account | Only an App Password works; generate it while the avatar shows the right account |
| SMTP `Connection could not be established ... :465` | ISP blocks 465 | Port 587 with `MAIL_SCHEME=smtp` |
| Firebase "Key creation is not allowed on this service account" | Project created by a Workspace user sits in the company org; `iam.disableServiceAccountKeyCreation` is enforced by default | Org Policy Administrator overrides the policy for that project, or create the project on a Gmail account and add the client as Owner later |
| "You have reached your project quota" | Gmail accounts start with about 10 projects | Google Cloud > IAM & Admin > Quotas > request more (approved within minutes) |
| Deleting a project to free a slot | Pending deletion still counts for 30 days | Request quota or use another account |
| Two Google accounts in one browser | OAuth client created in the wrong project/account | Use `/u/N/` URLs and check the project selector before each step |
