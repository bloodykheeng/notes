# Firebase push token errors in Next.js: check versions first

`VersionError: The requested version (1) is less than the existing version (2)`, `messaging/permission-blocked`,
and how to find the cause in minutes instead of restarting servers. Met on NICE (09/11/2026, firebase 12.1.0);
applies to any Next.js app using `firebase/messaging`. The service-worker pattern itself (config in the worker's
URL): [firebase-messaging-sw-and-nextauth-google.md](firebase-messaging-sw-and-nextauth-google.md).

---

## 1. What happened

The dev overlay showed `Console VersionError: The requested version (1) is less than the existing version (2).` on
every page, and the terminal `[browser] Could not get a notification token: VersionError ...`. Stopping and
restarting `npm run dev` changed nothing.

- **Restarting the server cannot fix it.** The error is about data stored **in the browser** (IndexedDB), not the
  server. A reload or a restart reads the same stored database.
- **Firebase 13 raised its messaging database to version 2.** `@firebase/messaging` stores the push token in an
  IndexedDB database called `firebase-messaging-database`. Firebase 12 opens it at version 1; Firebase 13
  (`@firebase/messaging` 0.13) opens it at version 2.
- **IndexedDB never opens a database at a lower version than the one stored.** It throws `VersionError`.
- **`localhost:3000` is one origin for every project.** IndexedDB, localStorage, cookies and service workers
  belong to the origin (scheme + host + port), not the project. Another app on the same port that ran Firebase 13
  (Alpha, SDS, GiftNest, a tutorial...) left version 2 behind, and the Firebase 12 app then could not open it.
  In production each app has its own domain, so the usual way to meet this there is a browser that ran a
  **newer build of the same app**, then an older one (a rollback).
- **Firebase keeps the failure for the whole page load.** `getDbPromise()` stores the first `openDB()` promise,
  even a rejected one, so catching the error and calling `getToken()` again fails the same way. The fix has to
  run **before** Firebase opens the database.

## 2. Check versions first (the five-minute diagnosis)

When an error names versions, compare the versions before changing any code.

**a. What the browser holds.** DevTools > Application > Storage > IndexedDB: each database shows its version. Or
in the console:

```js
(await indexedDB.databases()).map((d) => `${d.name} v${d.version}`)
// ['firebase-messaging-database v2', 'firebase-installations-database v1', 'firebase-heartbeat-database v1']
```

**b. What your installed SDK opens.**

```bash
node -e "console.log(require('firebase/package.json').version)"          # 12.1.0
grep -rhoE "const (DB_VERSION|DATABASE_VERSION) = [0-9]+|const (DB_NAME|DATABASE_NAME) = '[^']+'" \
  node_modules/@firebase/{app,installations,messaging}/dist/esm/index.esm.js
# DATABASE_NAME = 'firebase-messaging-database'  DATABASE_VERSION = 1
```

**c. What the service worker opens.** The worker loads its own copy from gstatic (`importScripts(".../11.3.1/...")`):

```bash
curl -s https://www.gstatic.com/firebasejs/11.3.1/firebase-messaging-compat.js \
  | grep -oE '"firebase-[a-z-]*database",[a-zA-Z]+=[0-9]+'
# "firebase-messaging-database",We=1
```

**d. What the other projects on the same port use.**

```bash
for d in ../alpha-nextjs ../sds-nextjs; do node -e "console.log('$d', require('$d/node_modules/firebase/package.json').version)"; done
```

**e. What the latest release opens** (did a newer version raise it?):

```bash
v=$(npm view @firebase/messaging version)
curl -sL "https://unpkg.com/@firebase/messaging@$v/dist/esm/index.esm.js" | grep -oE "const DATABASE_VERSION = [0-9]+"
# @firebase/messaging 0.13.3: const DATABASE_VERSION = 2
```

The browser holds a higher version than anything in (b), (c) or (d)? Then a newer build than yours wrote it, as in
(e).

## 3. The fix: drop a newer copy before Firebase opens it

In the hook that calls `getToken()` (`hooks/useFCMToken.ts`), after `isSupported()` and before `getMessaging()`:

```ts
/** The version of firebase-messaging-database the installed SDK opens (firebase 12: 1; 13: 2). Check on a major upgrade. */
const MESSAGING_DB = { name: "firebase-messaging-database", version: 1 };

async function dropNewerMessagingDatabase(): Promise<void> {
  if (typeof indexedDB === "undefined" || typeof indexedDB.databases !== "function") return; // older browsers
  const existing = (await indexedDB.databases()).find((db) => db.name === MESSAGING_DB.name);
  if (!existing?.version || existing.version <= MESSAGING_DB.version) return;

  await new Promise<void>((resolve) => {
    const request = indexedDB.deleteDatabase(MESSAGING_DB.name);
    request.onsuccess = () => resolve();
    request.onerror = () => resolve();
    request.onblocked = () => resolve(); // another tab holds it: the delete waits, Firebase's open queues behind it
  });
}
```

Deleting it is safe: it only caches the push token. Firebase makes a new token, and the app saves it to the API
as it does at every sign-in.

Do **not** "probe" by opening the database yourself at version 1: on a missing database that creates it
**without** Firebase's object store, and Firebase's own open then skips its upgrade and breaks.

The other way out is upgrading the app to the newer Firebase (page package **and** the worker's gstatic version
together). That flips the problem onto the apps still on the older version, so the guard is worth keeping anyway.

## 4. `messaging/permission-blocked` is a choice, not an error

`FirebaseError: Messaging: The notification permission was not granted and blocked instead.` means notifications
are blocked for the site (Chrome's site settings, or Windows notifications for Chrome). The person chose that.

- Go by `Notification.permission`, the value `getToken()` itself checks, not by what `requestPermission()`
  returned: Chrome answered "granted" while still blocking. Ask only while it is `"default"`:

  ```ts
  if (Notification.permission === "default") await Notification.requestPermission();
  if (Notification.permission !== "granted") return; // blocked or dismissed: nothing to report
  ```

- Catch `error.code === "messaging/permission-blocked"`, record "denied", log nothing.
- To get push on your own machine: the icon left of the address > Notifications > Allow.

## 5. `console.error` becomes an "Issue" in Next's dev overlay

Next.js 15/16 shows every `console.error` in the browser as an **Issue** in the dev overlay, and forwards browser
logs to the terminal as `[browser] ...`. A failure the page works fine without (no push token) is a
`console.warn`, not a `console.error`. Changing `console.log` to `console.error` is exactly what made this
long-silent failure suddenly appear on every page.

## 6. Checking it in a browser (Playwright)

```ts
await context.grantPermissions(["notifications"]);
await page.goto("/dashboard");
// Plant what Firebase 13 leaves behind, then reload.
await page.evaluate(() => new Promise<void>((resolve) => {
  indexedDB.deleteDatabase("firebase-messaging-database").onsuccess = () => {
    const open = indexedDB.open("firebase-messaging-database", 2);
    open.onupgradeneeded = () => open.result.createObjectStore("firebase-messaging-store");
    open.onsuccess = () => { open.result.close(); resolve(); };
  };
}));
await page.reload();
// Headless Chromium has no push service, so Firebase does not recreate it: check that v2 is gone and no VersionError.
```

Blocked notifications: `context.clearPermissions()` (headless then answers "denied"); expect no console output.

## 7. Lessons

1. **An error that names versions: compare the versions first** (section 2), before restarting or changing code.
2. **Restarting the dev server never clears browser storage.** IndexedDB, localStorage, cookies and service
   workers outlive it. DevTools > Application > Clear site data is the manual reset.
3. **Every project on `localhost:3000` shares that storage.** One app's leftovers can break another. Suspect this
   whenever an error only happens on your machine.
4. **A library that caches a rejected promise cannot be retried.** Fix the cause before its first call.
5. **Pin the number the guard depends on, with a comment saying when to revisit it** (`MESSAGING_DB.version`).
