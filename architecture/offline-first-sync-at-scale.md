# Offline-first sync at scale (devices that keep their own copy of the data)

How to build an app that works without internet by keeping records on the device (SQLite) and syncing with one
server for many sites, without crashing the server, filling the phone or doubling records. From a clinic records
app (Flutter + drift on Windows and Android, Laravel + MySQL server, 40+ facilities), Nov 2026. The numbers below
are from a load test with a fake facility holding two years of a busy health centre.

## When offline-first is the right call

- The work must go on without internet (rural clinics, field work, power cuts). Then the device holds the data it
  needs and syncs when it can. If the work can wait for the network, stay online-only: it is far simpler.
- Each device holds **only its own site's data**, never everyone's. The server holds all sites.

## The rules that keep it safe

1. **Ids are made on the device** (UUID; v7 sorts by time) and every save on both sides is "insert or update by id".
   A record sent twice, a pull repeated after a cut connection, a pass killed by a power cut: all overwrite, nothing
   doubles. Test it: count rows on server and device, cut the network and kill the app mid-download, count again.
2. **Newest edit wins** on a device-stamped edit time (`edited_at`), checked by the server, so an old copy never
   overwrites a newer one.
3. **Deletes are soft** with tombstones (`deleted_at` and a `/deleted` endpoint), so other devices learn about them.
4. **The pull cursor is the server's clock** (`server_time` from a cheap "what changed" probe), never the device's,
   or a device whose clock runs ahead silently misses rows.
5. **A "what changed since" probe first**: one request lists the tables with changes, so an idle sync costs one
   request instead of one per table.

## Pull in pages, never a whole table in one reply

One reply per table works with test data and falls over in the field. Load test (30,000 patients, 60,000 visits,
120,000 bill items):

| Pull | Result |
| --- | --- |
| Patients, one reply with photos | the server ran out of its 2 GB memory and failed after 42 s |
| Bill items, one reply | 54 MB in one response: the phone must hold all of it at once |
| Pages of 2,000 rows | every table came through; about 1 s a page; memory flat |

How:

- The client asks `?limit=2000`; the server orders by `(updated_at, id)` and answers with a header
  `X-Sync-Next: <updated_at>|<id>` when more follow; the client asks again with `&after=` that value (keyset paging,
  not `OFFSET`, which gets slower page by page and skips rows when data changes underneath).
- Many rows share one `updated_at` (bulk imports, one transaction): the cursor needs the id as a tie-breaker, and a
  test with groups of rows on the same instant across a page boundary.
- **Check the query plan.** With a soft-delete filter MySQL chose the `(facility_id, deleted_at)` index and sorted
  every remaining row for each page (2 s pages, getting slower). Forcing the `(facility_id, updated_at)` index fixed
  it. Run `EXPLAIN` on the page query with a realistic amount of data.
- The client saves each page in **one database transaction** before asking for the next (hundreds of times faster
  than row by row on a phone) and remembers the page reached, so a cut connection carries on from there.
- Old clients that do not ask for pages still get everything: add parameters, never change the default.
- Page size: small pages (500) pay the per-request cost too often; 2,000 rows is about 0.5 to 1.5 MB of JSON.

## Keep big things out of the sync

- **Photos and files are not rows.** Store them as files on the server, send only a fingerprint (a short hash) in
  the pull, and fetch one picture when the record is opened (and in the background when the fingerprint changed).
  Photos inline were most of the first download (2 MB for 6 test patients; gigabytes for a real site).
- **Compress the API's JSON** at the web server (gzip/deflate for `application/json`): 5 to 10 times smaller, no
  app change.
- **History the device does not need** (old visits) can stay on the server and load when a record is opened online.
  Keep what the offline work and the device's own reports need.

## The first sign-in on a new device

- Check the person with the server **before** anything downloads (the server issues the device token only then).
- Download only what sign-in needs first (staff, roles, permissions, settings), let the person in, and bring the rest
  in the background with a window that shows a real percentage, what is coming down and the megabytes so far; it says
  "waiting for the internet" when the connection drops and comes back when the app is reopened before it finished.
- Never a spinner or a bar sliding back and forth for something that takes minutes.

## Security of the device copy

- The device keeps staff passwords only as bcrypt hashes (needed to sign in offline), never plain text; a stolen
  copy can still be attacked offline, so enforce decent passwords.
- Patient records on a device are the real exposure: **encrypt the device database** (SQLCipher) with a key kept in
  the platform's secure storage (Windows DPAPI, Android Keystore), and let an administrator revoke a lost device on
  the server (it stops syncing at once).

## Checklist before go-live

- [ ] Same id twice never doubles (test with counts before and after).
- [ ] Network cut and app killed mid-download: resumes, no duplicates.
- [ ] Load test with two years of fake data: no table in one reply; page times flat; `EXPLAIN` checked.
- [ ] Photos and files outside the pulls; API JSON compressed.
- [ ] First sign-in quick, with a real progress window.
- [ ] Device database encrypted; lost devices can be revoked.
