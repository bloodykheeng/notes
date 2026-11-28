# Notifications in a Flutter app on Windows and Android: Android push, Windows tray (2026)

How a clinic records app (Flutter on Windows PCs and Android phones, Laravel API, Reverb for live sync) alerts people:
a lab result, a bill, a message from the facility. Built and checked Nov 2026. Live sync itself:
[realtime-reverb-from-flutter.md](realtime-reverb-from-flutter.md).

## Why the two platforms differ

| | Android | Windows (a Flutter .exe) |
| --- | --- | --- |
| Push when the app is closed | Firebase Cloud Messaging (`firebase_messaging` 17.x: Android, iOS, macOS, web) | No Firebase client. Microsoft's WNS needs the app as an MSIX package, a multi-tenant Entra app registration, a mapping request mailed to Microsoft (weekly turnaround), a fresh channel URI each launch. `windows_push` 0.1 wraps it (new, unproven). |
| What we did | FCM push, targeted per person | Keep the app running in the tray (it starts with Windows), live sync through Reverb, Windows notifications from the app |
| Next.js comparison | n/a | A web app gets push on Windows because the browser (Chrome, Edge) runs a service worker and talks to the browser's push service. A Flutter exe has no browser. |

## Android push (Firebase)

**Server (Laravel):**

- The device sends `PUT devices/me/push {token, user_id, audience[]}` at start, after sign-in and sign-out, every few
  minutes, and when Firebase renews the token (the client sends only when something changed). `audience` = the role
  names the signed-in person receives, computed by the app with the same function its bell uses, so push and bell
  agree. Columns on `devices`: `push_token` (hidden from every JSON), `push_user_id`, `push_audience` (json),
  `push_token_at`.
- A new notification row (`created` model event, after commit) queues a job: the facility's devices with a token, not
  revoked, not the sender's device, filtered by audience (facility: all; role: the name in `push_audience`; user: that
  `push_user_id`).
- Send with **FCM HTTP v1**: `POST https://fcm.googleapis.com/v1/projects/{project}/messages:send`, a Google access
  token from the service account (`google/auth` `ServiceAccountCredentials`, scope `firebase.messaging`, cached 50
  minutes). The legacy server key API is gone. `UNREGISTERED` / 404: forget the token.
- The key file path comes from `config('services.firebase.credentials')` (env `FIREBASE_CREDENTIALS`), never `env()`
  in code: with cached config `env()` returns null and push silently stops.
- Test with `Http::fake`; pre-fill the cached access token so the test signs nothing (generating an RSA key in a test
  failed on Windows PHP without an OpenSSL config).

**App:**

- Firebase settings from the build, like a `.env`, not from the source or a committed `google-services.json`:
  `flutter build apk --dart-define-from-file=<outside the repo>/test/app-build.json`, a small JSON generated from that
  project's `google-services.json` (api key, app id, sender id, project id). Read with
  `String.fromEnvironment(...)`; `Firebase.initializeApp(options: FirebaseOptions(...))`. A build without them has push
  off, nothing else changes. One file per server (test and production use different Firebase projects).
- `requestPermission()` (Android 13+ asks once); `getToken()`; `onTokenRefresh`.
- **Status bar and shade:** a notification channel created natively in `MainActivity.onCreate`
  (`NotificationChannel("mrdrs_alerts", ..., IMPORTANCE_HIGH)`, heads-up with sound); the server sends
  `android.notification.channel_id`. A **white vector icon** (`res/drawable/ic_stat_*.xml`, shape only) set as
  `com.google.firebase.messaging.default_notification_icon`, with a colour: the full-colour launcher icon shows as a
  white square. Manifest: `POST_NOTIFICATIONS`.
- App open: Android does not show notification messages itself; `onMessage` triggers a sync and the in-app bell shows
  it. App in the background or closed: the system shows it; a tap opens the app (`onMessageOpenedApp`).

## Windows: the tray, Laragon style

Packages: `tray_manager` **0.5.x** (0.7 moved to a brand-new native layer, `nativeapi`, written over FFI; pick the
proven line for field PCs), `window_manager` (already used for the custom title bar), `launch_at_startup`,
`local_notifier` (or `flutter_local_notifications` 23.x, which now supports Windows toasts).

- A tray icon (`.ico` from assets) with a tooltip showing the sync state ("MRDRS: synced", "offline, work is kept").
- Left click: open the window. Right click: a menu: **Open MRDRS, Sync now, [x] Start with Windows, [x] Close button
  keeps MRDRS in the tray, [ ] Minimise to the tray, Exit MRDRS.** Choices are saved (shared preferences).
- `windowManager.setPreventClose(true)` and a `WindowListener`: `onWindowClose` hides to the tray (or exits when that
  option is off); `onWindowMinimize` hides when "Minimise to the tray" is on. Hide = `hide()` +
  `setSkipTaskbar(true)`; show = `show()`, `setSkipTaskbar(false)`, `focus()`. Exit = `setPreventClose(false)`, destroy
  the tray, `destroy()` the window.
- The first time it hides, a Windows notification says it is still running and where (people think closing quits).
- Start with Windows on by default (a facility PC comes back connected after a power cut), started with `--hidden`, so
  it opens in the tray: `main(List<String> args)` passes the args; skip `show()` at start when hidden.
- No tray available (a locked-down PC): catch, turn prevent-close off, behave as a normal window.
- **One copy at a time** (`windows_single_instance`, a named pipe, first thing in `main`): an app hidden in the tray is
  invisible, so people double-click the shortcut again; a second copy would open the same database and run a second
  sync loop. The second launch hands over to the running one, which comes to the front (`onSecondWindow` shows the
  window and puts it back on the taskbar), and exits. One pipe name for every build that shares the data folder (our
  test and production builds do).
- Checked on a laptop: the menu, the tooltip with the sync state, the X hiding the window with the process still
  running, Start with Windows registered as `uhfmris.exe --hidden` under `HKCU...CurrentVersionRun`.
- **Rebuilding while it sits in the tray fails** (`MSB3073 ... INSTALL.vcxproj`, exit code 1): the X no longer stops
  the app, so the old exe in `buildwindowsdunnerRelease` is locked. Quit it from the tray menu (or
  `Stop-Process -Name <exe>`) before `flutter build windows`. The same goes for installers: close it before updating.

## Situations to walk on real devices

| Situation | Android | Windows |
| --- | --- | --- |
| App closed | Push shows in the shade | Closed to the tray: still connected, notifications show; exited: nothing until reopened (sync catches up) |
| Phone locked or asleep | High-priority push wakes the shade | n/a |
| Signed out | Only facility-wide notifications (no person, no roles) | Same, the bell is per person |
| Another person signs in | The server is told at once; their roles from then on | Same |
| Notifications refused (Android 13+) | Nothing in the shade; the bell still works | n/a |
| No Google Play services | Push off, the app works | n/a |
| Device revoked | Its token stops getting pushes (revoked devices are skipped) | Live sync refused |

## Checklist

- [ ] FCM HTTP v1 with a service account from config (not `env()` in code); stale tokens forgotten; tests with
      `Http::fake`.
- [ ] Targeting matches the app's own bell rule; never the sender's device; never a revoked one.
- [ ] Firebase settings from `--dart-define-from-file` outside the repo, one per server.
- [ ] Channel created natively, white status-bar icon, `POST_NOTIFICATIONS`.
- [ ] Windows: tray menu with the options, close and minimise behaviour, start with Windows hidden, a first-time
      notice, a fallback when there is no tray.
- [ ] One copy at a time: a second launch brings the running one forward.
