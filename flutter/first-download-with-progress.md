# The first download on a new device, with real progress (Flutter + drift)

An offline-first app has to bring a site's records onto a new laptop or phone before anyone can work. This note is
how we built that download in a clinic records app (Flutter on Windows and Android, drift/SQLite, a Laravel sync API),
Nov 2026, after two versions that went wrong. Server side, paging and cursors:
[../architecture/offline-first-sync-at-scale.md](../architecture/offline-first-sync-at-scale.md). Encrypting the file
it fills: [encrypting-the-device-database.md](encrypting-the-device-database.md).

## The rules

1. **The whole first download finishes on the sign-in screen before anyone goes in.** Letting people in after the
   staff tables and downloading the rest in the background looked faster, but someone could open Patients mid-download,
   search for a returning patient who had not arrived yet, find nothing and register them twice. Prices and lab tests
   could be missing from a bill. It happens once per device, usually at installation: a real progress screen makes
   the wait acceptable.
2. **One pass, not two.** A "sign-in step" plus a "background pass" fetched the staff and permission tables twice
   whenever the hand-over between them failed. One full pass has no hand-over to get wrong.
3. **Real progress**: a percentage, the table being fetched in words ("Now: Patients, page 3"), the size so far
   ("1.4 MB downloaded"). Never a spinner or a bar sliding left and right for something that takes minutes.
4. **Never a dead end**: no internet means "Waiting for the internet" and it carries on by itself; the app closed
   halfway reopens on the download; a site's other facilities stay reachable.
5. **Nothing heavy on the screen's thread while it runs** (see the last section): it is what made our requests time
   out, show "Offline" while online, and download tables twice.

## The pieces

### 1. Progress state the screen listens to

```dart
@immutable
class SyncProgressState {
  const SyncProgressState({this.done = 0, this.total = 0, this.label = '', this.running = false, this.failed = false});
  final int done, total;        // tables finished / tables in this pass
  final String label;           // "Patients, page 3"
  final bool running, failed;   // failed: the pass stopped for no internet
  int get percent => total == 0 ? 0 : (done * 100 / total).round();
}

class SyncProgress {
  static final state = ValueNotifier(const SyncProgressState());
  static final bytes = ValueNotifier<int>(0);           // the http client adds each response's length
  static int _total = 0;                                 // kept apart: the screen may reset `state` mid-pass
  static void start(int total) { _total = total; state.value = SyncProgressState(total: total, running: true); }
  static void step(int done, String label) =>
      state.value = SyncProgressState(done: done, total: _total, label: label, running: true);
  static void finish({bool failed = false}) => state.value = SyncProgressState(
      done: failed ? state.value.done : _total, total: _total, label: state.value.label, failed: failed);
}
```

- The sync pass calls `start` with the number of tables it will touch (the server's "what changed" probe lists only
  tables that hold rows for this site, so a new device skips the empty ones), `step` before each table, and
  `notePage(n)` when a big table comes in pages.
- Count bytes in the one HTTP client every sync call goes through (`bytes.value += response.bodyBytes.length`).
- **Show records done of total** ("42 %  ·  1,240 of 3,600 records"): people asked "how much of how much". Have
  the server's first "what changed" answer carry a count per table (`counts: {patients: 1200, ...}`, one indexed
  `COUNT` per table, once per new device, additive so old apps ignore it). The client adds a table's count when it
  finishes and `(page - 1) x page size` while a big table pages in, and falls back to tables done with an older
  server. The total size in MB is not knowable up front; show the size so far.

### 2. The download loop on the sign-in screen

```dart
Future<bool> _downloadAll() async {
  FirstDownload.running = true;                      // the background timer stays off meanwhile
  setState(() { _downloading = true; _paused = false; });
  try {
    if (await _fileIsEmpty()) await SyncPaging.startOver();   // markers of an earlier file would skip rows
    await seedPermissions(db);                       // whatever the pulled rows point at, first
    SyncProgress.bytes.value = 0;
    while (mounted) {
      final result = await SyncService(db).syncAll();         // the same full pass the app runs every few seconds
      if (await FirstDownload.isDone(db)) return true;
      if (result.errors.any((e) => e.kind == SyncErrorKind.auth)) { _showRevoked(); return false; }
      setState(() => _paused = true);                // "Waiting for the internet"
      await Future<void>.delayed(const Duration(seconds: 3));
      if (mounted) setState(() => _paused = false);
    }
    return false;
  } finally {
    FirstDownload.running = false;
    if (mounted) setState(() => _downloading = false);
  }
}
```

- After it returns true, show "Ready, 100 %" for about a second (otherwise a quick download looks like the screen
  skipped), then sign the person in with the username and password they typed (kept in memory only, never stored)
  and go to the home screen. Start the deferred work (seeding reference lists) here.
- **Report only a clean pass to the status chip from here.** A failed attempt on a flaky line, reported right before
  the home screen, showed "Offline" seconds before the background sync said "Synced".
- **"Done" is a flag the pass sets when it went through every table**, not "no errors": a row the server refuses
  (validation) is retried by later passes and must not keep someone on the download screen forever. Make "done" also
  require that the file holds staff, so a new or emptied file is never "done" because of a flag left by an old one.
- On a device from before the flag, accept the old signs too (a stamped "synced up to" cursor).

### 3. Resume, never restart

- **Per table:** while the pass has no cursor yet (a first download), save each finished table's own cursor (the
  server's time from the probe, minus a small overlap). A retry after a dropped connection then asks those tables only
  for changes. A complete pass clears them.
- **Within a table:** pages are saved one transaction each and the next page's marker is remembered, so a cut pull
  continues from its page (see the sync note).
- **The app closed halfway:** the sign-in screen checks `FirstDownload.isDone` at start and runs the download before
  showing the form. Do not remember the person as signed in until the download finished, or the app reopens on
  half-empty screens.
- **A timer that syncs every few seconds** must not run during the first download (`FirstDownload.running`, or while
  a pending sign-in is in memory), or a pass can start before the permission list is seeded. If two calls do meet,
  share one in-flight pass instead of running two.

### 4. The screen

- Title: "Downloading <site>'s records"; text: "This happens once on this device. When it finishes you go straight
  in, with every record here, and the app works without internet from then on."
- A determinate `LinearProgressIndicator(value: percent / 100)`, then `"$percent %  ·  Now: $label"` and
  `"${size(bytes)} downloaded"`.
- Paused: a cloud-off icon in the warning colour, "Waiting for the internet", "The connection dropped. The download
  carries on by itself, from where it stopped, when the internet is back. Nothing is lost or doubled.", and a
  "Switch facility" button when the device has another site.
- Hide the username and password fields while it runs: seeing them again after typing them reads as "it failed".

## Keep the screen's thread free (drift on the main isolate)

`NativeDatabase(file)` runs SQLite **on the main isolate**: every statement blocks the UI and every timer while it
runs. A one-off bulk insert (84,500 places from a bundled JSON file) right after a new file was created froze the app
for several seconds, at the exact moment the download's first request was waiting for its answer. The request's
10-second timeout fired, so:

- the "what changed" probe failed, the pass that needed its server time could not mark tables done, and the next pass
  downloaded them again;
- the status chip said "Offline" while the internet worked, because every failed request was reported as a network
  error.

The server's access log showed the giveaway: **no failed request at all**. When the server saw nothing wrong, look
for the client freezing, not the network.

What fixed it, and the rule:

- **Defer** heavy local work until the first download is done (start it from the download's success).
- **Chunk** big writes: batches of about 2,000 rows with `await Future.delayed(const Duration(milliseconds: 15))`
  between them, so frames and timers run.
- **Make "already done" checks point at the last step** (the last level of the list), so work cut halfway runs again;
  `insertOrIgnore` makes the rerun cheap.
- Bigger fix when needed: `NativeDatabase.createInBackground(file, setup: ...)` runs SQLite on its own isolate (test
  every screen after switching; the `setup` closure runs in that isolate).
- Parse big JSON with `compute(...)`, never on the main isolate.

## Test it like the field

- **A truly fresh device:** remove the site and add it back (make sure that really starts a new file: ours first
  reused the old one until the next restart, so nothing downloaded), sign in, and count each table's pulls in the
  server's access log: every table from the start exactly once.
- **Cut the internet mid-download:** on Android, `adb shell cmd connectivity airplane-mode enable` the moment the new
  file opens (watch the log for its path), check "Waiting for the internet", then `disable`: it must carry on and
  still pull each table once. USB debugging stays connected.
- **Kill the app mid-download** and reopen: the download first, then the form.
- **A slow, flaky line:** time a few requests with `curl -w "connect %{time_connect} total %{time_total}"` before
  blaming the app; ours took 21 s, then 0.5 s, then 3.8 s for the same call.

## Checklist

- [ ] One full pass on the sign-in screen; nobody goes in before it is done.
- [ ] Percentage, table in words, size so far; "Waiting for the internet" and carries on by itself.
- [ ] Finished tables remembered; pages resumable; the app closed halfway reopens on the download.
- [ ] Background timer off during the first download; one in-flight pass shared.
- [ ] Heavy local work deferred and chunked; big JSON parsed off the main isolate.
- [ ] Access log after a fresh sign-in: each table pulled from the start once.
