# Publishing app builds to Cloudflare R2 with an AI agent

**For the AI agent.** How we hand out app builds (Android `.apk`, Windows installer, a `release.json`) from a
Cloudflare R2 bucket, and how you upload a new release yourself instead of asking me to drag files into the dashboard.
Written from a real release (Nov 2026). No secrets here: the keys live in a text file on my PC, read file to file.

## 1. How the bucket is laid out

- One bucket for all our apps (`apps`), one folder per app and environment: `<app>/production/`, `<app>/test/`.
- **Fixed file names**, so a release only replaces files and no web page changes: `<app>-android.apk`,
  `<app>-windows-setup.exe`, `release.json` (`{"version", "date", "notes"}`). The web dashboard's download page reads
  `release.json` itself (fetch, so the bucket's CORS allows GET/HEAD from the dashboard's domains) and shows a button
  only for files that exist.
- Public access: the bucket's **Public Development URL** (`https://pub-<id>.r2.dev`) while we test. It is
  rate-limited and has no caching; connect a **custom domain** to the bucket before real traffic.

## 2. The keys (once, by me)

R2 overview page (not the bucket's settings) > **API** > **Manage API tokens** > **Create Account API token**:

- Permissions **Object Read & Write**, **only the one bucket**, never Admin.
- The page shows the Access Key ID, Secret Access Key and the S3 endpoint
  (`https://<account id>.r2.cloudflarestorage.com`) **once**. I save the whole page as a text file in the project's
  `deployment/` folder (outside every repo). You read the values from that file by line number; you never print them,
  never paste them into chat, docs or commits.

## 3. Uploading (you)

No AWS CLI needed: curl 7.75+ signs S3 requests itself (`--aws-sigv4`). Keep the keys off the command line with a
temporary curl config file in your scratchpad, deleted at the end:

```bash
KEYS="<path to the saved token page>"
AK=$(sed -n '<line of Access Key ID>p' "$KEYS" | tr -d '\r\n ')
SK=$(sed -n '<line of Secret Access Key>p' "$KEYS" | tr -d '\r\n ')
EP=$(sed -n '<line of the endpoint>p' "$KEYS" | tr -d '\r\n ')
CFG="$SCRATCH/r2.curl"; umask 077; printf 'user = "%s:%s"\n' "$AK" "$SK" > "$CFG"; trap 'rm -f "$CFG"' EXIT

put() {  # file  key-in-bucket  content-type  [cache-control]
  curl -sS -K "$CFG" --aws-sigv4 "aws:amz:auto:s3" -o /dev/null -w "%{http_code}  $2\n" -X PUT \
    -H "Content-Type: $3" ${4:+-H "Cache-Control: $4"} --upload-file "$1" "$EP/<bucket>/$2"
}
put builds/test/app-android.apk        app/test/app-android.apk        application/vnd.android.package-archive
put builds/test/app-windows-setup.exe  app/test/app-windows-setup.exe  application/vnd.microsoft.portable-executable
put builds/test/release.json           app/test/release.json           application/json no-cache
```

- Region is `auto` (`aws:amz:auto:s3`); the path is `<endpoint>/<bucket>/<key>`.
- **`release.json` with `Cache-Control: no-cache`**, so the download page shows the new version at once.
- Upload the big files first and `release.json` last: the page never announces a version whose files are not there yet.
- 200 on every PUT, then **check from outside**: `curl -sI <public url>/<key>` (200, the same Content-Length as the
  local file) and `curl -s <public url>/app/<env>/release.json`.
- If curl prints anything that could contain a key, mask it before it reaches the chat (`sed -E 's/[A-Za-z0-9]{32,}/<masked>/g'`).

## 4. Before uploading a build

- Check what server a build talks to: an `.apk` is a zip; `lib/arm64-v8a/libapp.so` holds the Dart strings, so a test
  build must contain the test API host and must not contain anything it should not (an old hard-coded third-party
  address, for example).
- Bump the version in `pubspec.yaml`, write `release.json` (version, date, notes in plain words for the users).
- Windows: an unsigned installer shows SmartScreen ("Windows protected your PC"). Only a code-signing certificate in
  the company's name removes the "Unknown publisher" (an OV certificate, paid yearly; Microsoft's cheap cloud signing is
  limited to a few countries). Until then the download page says "More info, then Run anyway".

## 5. Don't

- Don't give the token Admin rights or access to every bucket.
- Don't put the endpoint plus keys in the repo, the app, or a `.env` that ships. The app only ever sees public URLs.
- Don't delete a folder to "clean up" before uploading: replace the files in place (the links never change).
