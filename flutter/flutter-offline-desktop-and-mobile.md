# Flutter apps on Windows PCs and Android phones, offline, synced to a Laravel API

**For the AI agent.** Read this before planning or changing a Flutter app that runs on desktops and/or phones, works
offline and syncs to our Laravel API. It collects what we know from **MRDRS** (a health facility app, Flutter
Windows desktop + Android, Nov 2026: [../ai-agents/mrdrs-lessons.md](../ai-agents/mrdrs-lessons.md)) and the
**Letshego** field app (Flutter Android, built by Marvin from a guide:
`D:\coding\new wave\letshego\letshego-workspace\docs\flutter-app-guide.md`). Also read
[../ai-agents/who-i-am-and-how-i-work.md](../ai-agents/who-i-am-and-how-i-work.md) and
[../ai-agents/dates-and-timezones-with-an-ai-agent.md](../ai-agents/dates-and-timezones-with-an-ai-agent.md).

**How sure each part is:** **Proven** = shipped and used. **Planned** = decided on MRDRS Track A (20 Nov 2026),
not yet proven in the field: update this note when it is, or when it fails.

---

## 1. Offline: a local store, an outbox, a server (Proven)

First decide **which features are offline at all**: [../ai-agents/offline-and-online-explaining-to-clients.md](../ai-agents/offline-and-online-explaining-to-clients.md)
(offline OK / offline confirmed by the server / online only).


- **Each device keeps its own local database** (drift/SQLite) and works with no network for as long as needed.
  Writes mark the row unpushed (a `tag` such as `created`/`updated`/`deleted`, or an outbox table); a sync pass
  pushes them, then pulls what changed.
- **Never share one SQLite file between users** (a network folder, a shared exe): SQLite is one process on one
  machine. MRDRS did this and it locked and crashed.
- **Never a server per site** (Laragon + MySQL on the facility LAN): nobody can install or support it at 40 sites.
  One server we run, reached over the internet; devices sync when they have data. **Say it plainly to the client:
  devices share work only through the server**, so desks on the same Wi-Fi without internet do not see each other's
  new records until the internet returns; each site needs one internet source (a hub inside the app that relays between
  desks on the local Wi-Fi is an idea to explore if outages are long; see Kolibri's peer sync).
- **The sync protocol that worked** (Zed, MRDRS, Jul to Sep 2026):
  - ids made on the device (§2), upsert by id on the server;
  - the pull cursor (`since`) stamped from the **server's clock** (`server_time` returned by the change probe),
    never the device's, or rows are skipped forever when the device clock runs ahead;
  - soft deletes, with a `.../deleted?since=` endpoint returning tombstoned ids;
  - one change probe (`GET sync/changes?since=`) listing only the tables that changed, so an idle pass is one request;
  - last write wins by `updated_at`, the server ignoring an older write;
  - **schema changes additive only, new fields nullable, server before client**, because old app builds keep
    syncing for weeks.
- **Don't poll every 2 seconds** against a cloud server (160 devices × 52 tables = ~4,000 queries/s idle). Push a
  "something changed" signal with Reverb (§5); keep a slow safety poll. (Planned for MRDRS.)
- **Multi-tenant from the first table.** The server stamps the tenant (`facility_id`) from the device's token; the
  client never sends it. **No fixed ids shared by every install** (MRDRS had one "Admin role" id for the whole fleet,
  which would merge tenants): per-tenant seed data comes from the server. (Planned.)

## 2. Ids made on the device (Planned: UUID v7)

- Devices create records offline, so ids are made on the device. Use **UUID v7** (`uuid` package 4.x, `Uuid().v7()`;
  check the package's current API when upgrading): time-ordered like a ULID, so they sort by creation time
  ascending, descending and as text, and insert at the end of the index, while staying in the UUID format Laravel's
  `uuid` rule and `uuid()` columns expect. UUID v4 (random) is what MRDRS had: it works but scatters inserts.
- ULID is fine too (shorter: 26 characters vs 36). In our Laravel-only projects we use ULIDs; with a Flutter client
  making ids, v7 avoided changing every table. Pick one per project and record why.
- An id made offline carries the device's clock: see §3's wrong-clock banner.

## 3. Dates and times on the device (Planned for MRDRS; the wire rules are Proven on NICE)

The server rules are in the dates note. On the device:

1. **On the wire, instants in UTC with `Z`**: `t.toUtc().toIso8601String()`, one helper, every field (NICE trap 14:
   `DateTime.now().toIso8601String()` has no zone and reads 3 hours late). Calendar days by nature (date of birth)
   go as a bare `YYYY-MM-DD`.
2. **From the wire:** `DateTime.parse(...)` gives a UTC instant; `.toLocal()` for display only.
3. **In the local database:** drift stores `DateTime` as a Unix timestamp by default (an instant). Keep it; don't
   switch to text dates casually.
4. **"Today", "this month", a report's day or month boundaries: the business day, not the device's.** `DateTime(year,
   month, 1)` and `DateTime.now().day` follow the device's zone setting, and a fresh Windows install is often not on
   East Africa Time. Use one helper with the fixed offset (Uganda is +3 all year, no daylight saving):
   `startOfDay`, `startOfMonth`, `dayOf(instant)`. Unit-test it at 00:30 and 23:30 Kampala on a month's first and
   last day.
5. **A wrong-clock / wrong-zone banner:** compare the device's clock with `server_time` on each sync (over 5 minutes
   off) and check `DateTime.now().timeZoneOffset` is +3 h; otherwise show "Set this computer/phone to East Africa Time
   and the correct time".
6. **Four times on every synced record, never mixed up:** `created_at` (made on the device; the server must accept and
   keep it, or a record made at 8 am and synced at 3 pm reads "created 3 pm" on every device after the next pull),
   `edited_at` (last edit on a device; "newest edit wins" compares these, device clock with device clock), `synced_at`
   (first reached the server, with the device it came from), `updated_at` (the server's last change, the pull cursor).
   All UTC `dateTime`. Show "Registered 8:12 am · Synced 3:05 pm", "Not synced yet" before it reaches the server; count
   reports by `created_at` or the event's own time. Device times more than a few minutes in the future: clamp to the
   server's time and log, never refuse.
7. **The server side must be pinned too** (`DB_TIMEZONE=+00:00`, dates note trap 7): MRDRS's API had no pin while all
   its sync timestamps were MySQL `TIMESTAMP` columns.

## 4. Devices, sign-in and secrets (Planned)

- **Enrol each device once** with a short code an admin generates; the API returns a **Sanctum token for the device**
  (a `devices` table: tenant, name, platform, app version, last seen, revoked). A lost device is revoked alone; you can
  see which devices are syncing and on which version. Store the token with `flutter_secure_storage`.
- **No secret in a client build** (a login or key compiled into an app can be extracted by anyone holding it).
  Third-party credentials live on the server per tenant; the app calls our API, which calls them.
- **Offline sign-in needs a hash on the device**: bcrypt, never plain text, and never sent to the server in clear.
- Internet is needed once, at enrolment and the first pull; after that the device works offline.

## 5. Live updates: Reverb from Dart (Letshego Proven on Android; Windows Planned)

- Reverb speaks the Pusher protocol. Use a **pure-Dart** client (Letshego's guide names `laravel_reverb`; alternatives
  `pusher_reverb_flutter`, `dart_pusher_channels`). Pure Dart matters on **Windows**, where the official Pusher plugin
  has no build. **Spike it on a Windows build first** (connect, private channel auth with the Bearer token, receive
  an event) before building on it.
- Broadcast a **signal, not the record** (`{tables, server_time}` or Letshego's `RecordsChanged {resource, id}`) on
  the tenant's private channel, `toOthers()` via `X-Socket-ID`; the app then syncs or refetches. Reverb down must only
  make things slower: keep refresh on resume and a slow poll.

## 6. Notifications and background (researched 20 Nov 2026; Android Proven on Letshego, Windows Planned)

- **Firebase Cloud Messaging has no Windows client** (`firebase_messaging` has no Windows implementation). That does
  NOT mean Windows apps can't notify: say which mechanism, not "Windows can't".
- **Windows, the practical way (Teams, Slack, WhatsApp desktop):** keep a resident process. Close-to-tray
  (`window_manager` close handler + `tray_manager` icon and menu), start with Windows (`launch_at_startup`), keep the
  Reverb connection, raise **Windows toasts** yourself (`flutter_local_notifications` has a Windows implementation;
  its PR #2813 fixes a crash when the Windows push service is disabled: check it merged or guard the call). Bonus:
  the app keeps syncing while "closed".
- **Windows, true push to a fully closed app: WNS** (Windows Push Notification Services) through the Windows App SDK.
  Needs **package identity (MSIX)** for background delivery, a **Microsoft Entra ID app registration** (client id +
  secret for the server to get a WNS token), and a plugin such as `windows_push`. Reports of closed-app reliability
  are mixed; choose it only when the tray approach is not enough. MSIX also changes how you ship (vs an Inno Setup
  installer).
- **Android:** Letshego's pattern: **data-only** FCM messages, `push_tokens` per device, a `PushChannel` on the API,
  App Check with a fixed debug token for test builds (Letshego guide §3.4). One Firebase project per environment.
- **One notification class on the API** sends database (the bell) + broadcast (Reverb: desktop toasts, in-app) + FCM
  (phones); each channel off when its credentials are blank, never failing the save.

Sources: [Windows push notifications overview](https://learn.microsoft.com/en-us/windows/apps/develop/notifications/push-notifications/),
[push quickstart](https://learn.microsoft.com/en-us/windows/apps/develop/notifications/push-notifications/push-quickstart),
[windows_push](https://github.com/hinata-platform/windows_push),
[flutter_local_notifications](https://pub.dev/packages/flutter_local_notifications),
[its PR #2813](https://github.com/MaikuB/flutter_local_notifications/pull/2813),
[tray, notifications and close-to-tray in Flutter desktop](https://dev.to/krtirtho/flutter-desktop-mastery-2-system-tray-local-notifications-4edg),
[a developer's closed-app toast experience](https://medium.com/@baravkarpratik5/the-headache-of-windows-toast-notifications-in-flutter-and-the-sane-alternative-i-shipped-7e3d2ea0da38).

## 7. Screens: desktop-first where desks work (Planned)

- MRDRS was built like a phone app and stretched on PCs. On a PC: **data tables** (columns, sort, search, paging,
  export), **list and detail side by side**, forms in **numbered sections, 2 to 3 columns** from 1280 px, dialogs sized
  to their content, keyboard friendly. On a phone: one column, bottom tabs, big touch targets.
- One design system in `lib/theme/` (Material 3, `ColorScheme.fromSeed`, one bundled font, light and dark) and one app
  shell (sidebar / drawer, top bar with tenant, user and a **sync indicator**: All sent · 3 waiting · Offline · Live).
- Check with screenshots at 1536×745 and 1920×1080, light and dark, with realistic data, and on a phone.

## 8. Permissions in one place (Planned)

- Roles come **ready-made with their permissions**, seeded by the server (declarative, SDS's
  `RolesAndPermissionsSeeder`); a user has one role, a free-text **title**, and per-user Grant/Deny switches against the
  role. Nobody builds roles in the field (MRDRS made admins create roles, tick permissions and attach them per user).
- **One function answers "what can this user do"** (MRDRS: `getUserPermissions()`, used by 66 screens): change it,
  not the screens. Don't copy permission-loading code into every screen.

## 9. Building, installing, updating (Planned)

- **Windows:** an installer (Inno Setup script in the repo) with the app, the VC++ redistributable, shortcuts and an
  uninstaller; data in `%APPDATA%` so updates never touch it. Never ship a zipped folder.
- **Android:** release signing from a keystore kept off the repo, `targetSdk` at the current Google Play requirement
  (MRDRS was on 33), a fixed app id.
- **In-app updates from our server:** `GET /api/app/releases/latest?platform=` (version, minimum version, URL, notes);
  optional update offers Update now, a version below the minimum must update before syncing. Windows runs the
  downloaded installer; Android opens the APK. The `upgrader` package only knows the stores.
- **One release script** builds both, uploads, and registers the release. Send `X-App-Version` on every call.

## 10. Phones in the field (Planned)

- Sync at start, on resume (`AppLifecycleState.resumed`), when the network returns (`connectivity_plus`), on Reverb
  events, after each save. Android stops background apps: don't promise sync while closed (WorkManager, 15 min
  minimum, only if asked).
- Mobile data costs money: compress photos on capture (about 100 KB; MRDRS stored base64 photos in a `longText`),
  show progress on the first pull, measure its size in the field test.

## 10a. Photos and files (seen on MRDRS, Nov 2026)

- **Never base64 inside synced rows.** A photo in the row travels with every change to that row and with every first
  sync; base64 adds a third. Keep files out of the data sync.
- **Compress on capture** (longest side ~1,024 px, JPEG ~75%, about 100 KB); the server refuses oversized uploads.
- **Server:** the file on a private disk (or S3-compatible storage) at `<module>/<kind>/YYYY/MM/<hash>.<ext>` (SDS's
  `file-storage-structure.md`: a path made only of things that never change); the row keeps the path, type, size and
  sha256; endpoints scoped to the tenant, never a public URL.
- **App:** upload through the outbox when online; **download a file only when it is shown**, then cache it.

## 11. Repo hygiene

- `.gitignore` from day one: no `db.sqlite`, backups, `analyze_output*.txt` or DLLs in git (MRDRS had all of them,
  possibly with patient data).
- Old packages pile up (`http 0.13`, `intl 0.17`): upgrade on a schedule, one at a time, reading each changelog.
