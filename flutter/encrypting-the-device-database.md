# Encrypting the device database (Flutter + drift, Windows and Android)

An offline-first app keeps real records on laptops and phones. A stolen device, a copied app folder or a backup on a
USB stick then holds readable personal data. This is how we encrypted the local SQLite database of a clinic records
app (drift, ~60 tables, devices already in the field with plain databases), Nov 2026.

## Which library

- `sqlcipher_flutter_libs` is **end of life** (`0.7.0+eol`). Its last working release only fits sqlite3 2.x. Do not
  start new work on it.
- `package:sqlite3` 3.x builds SQLite itself through Dart build hooks and can bundle an encrypting build instead.
  Pick it in `pubspec.yaml`:

  ```yaml
  dependencies:
    drift: ^2.35.0      # drift 2.3x uses sqlite3 3.x
    sqlite3: ^3.9.0     # drop sqlite3_flutter_libs (end of life too)
  dev_dependencies:
    drift_dev: ^2.35.0

  hooks:
    user_defines:
      sqlite3:
        source: sqlite3mc   # SQLite3MultipleCiphers; `sqlcipher` also exists
  ```

- We chose **SQLite3MultipleCiphers** (`sqlite3mc`): newer SQLite than the bundled SQLCipher build, a permissive
  licence, no OpenSSL, ChaCha20-Poly1305 by default, and it encrypts an existing plain file in place
  (`PRAGMA rekey`). The binaries are downloaded at build time from the package's GitHub release and checked against
  hashes in the package, so the build machine needs internet.
- Upgrading drift from 2.20 to 2.35 resolved with no conflicts and regenerated the same files (we already used
  modular generation). Run the whole test suite after it.

## The key

- One random 256-bit key **per database file** (`Random.secure()`), stored only in `flutter_secure_storage`
  (Windows: DPAPI-protected; Android: Keystore). Never in a file, a log, the server or the source.
- Every connection's **first statement** is the key: `PRAGMA hexkey = '<64 hex>';` (drift: `NativeDatabase(file,
  setup: (db) => db.execute(...))`, before `journal_mode` and the rest).
- A removed account/site deletes its key with its file.

## Converting devices that already have a plain database

The first start after the update must convert, and it must never lock anyone out:

1. Detect a plain file by its first 16 bytes, `SQLite format 3\0` (an encrypted file starts with noise).
2. Open it plainly, `PRAGMA wal_checkpoint(TRUNCATE)` and `PRAGMA journal_mode = DELETE` (pages still in the WAL
   would otherwise be left plain), and count the rows of every table.
3. Copy the file aside, then `PRAGMA hexrekey = '<key>'`.
4. Reopen with the key, count again, and check the header is no longer plain. Equal: delete the copy. Anything else:
   put the copy back and carry on **unencrypted** this time (next start tries again).
5. A file that does not open with the key held (the key lost with a reset Windows profile, say) is renamed aside
   (`.unreadable-<time>`) and the app starts empty and downloads again from the server. Unsent changes in it are lost,
   so sync often.

Also check `PRAGMA cipher` answers: a build that silently loaded plain SQLite would ignore the key pragmas and leave
the data readable while you believe it is encrypted.

## Measured cost (60 MB database, two years of a busy site, laptop)

| Step | Time |
| --- | --- |
| Convert once | 2.6 s |
| Open, plain | 3 to 7 ms |
| Open, encrypted (key derivation) | 110 to 145 ms |
| Two full table scans | about the same as plain |

Phones are 3 to 5 times slower: still fine for a one-off conversion and a start-up.

## Things that change once encrypted

- **No going back:** an older app version (plain SQLite) cannot open the file. Ship encryption as its own release
  and never offer the previous build again.
- **Backups:** copies of the file are encrypted with the same key, which is what you want (a USB backup is no longer
  readable). A restore that checked for the `SQLite format 3` header must also accept a file that opens with the key.
- **Tests and tools:** tests that open their own files without a key still work (the encrypting build reads plain
  files). A DB browser needs the key and SQLite3MultipleCiphers support to open a device file.
- Keep the server-side controls too: a lost device is revoked on the server so it stops syncing at once.

## Checklist

- [ ] Encrypting build selected; `PRAGMA cipher` answers in a test.
- [ ] Key per file in secure storage; key pragma first on every connection.
- [ ] Conversion test with rows still in the WAL; no personal text readable in the converted file.
- [ ] Wrong key refused; lost key handled (set aside, re-download), never a crash.
- [ ] Restore accepts encrypted backups.
- [ ] Release notes: no downgrade after this version.
