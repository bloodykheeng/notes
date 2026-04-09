# EAS Local Build via Docker (Windows)

Build your Expo app locally using Docker — no WSL Android SDK setup needed.
Drop the config files into any project and run one command to get an APK.

> ✅ **Confirmed working.** Successfully built a 112 MB APK on Docker Desktop Windows
> using `network_mode: host` + `_JAVA_OPTIONS=-Djava.net.preferIPv4Stack=true` + updated CA certs.

## Key Fixes That Made It Work

| Problem                                                              | Fix                                                                              |
| -------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Docker DNS broken                                                    | `network_mode: host` — container shares host network stack                       |
| Java/Gradle tries IPv6 (broken on Docker Desktop Windows)            | `_JAVA_OPTIONS=-Djava.net.preferIPv4Stack=true`                                  |
| TLS handshake failure on Maven Central                               | Updated CA certs in `Dockerfile.eas` via `apt-get install ca-certificates`       |
| `docker build --no-cache` fails (re-pulls base image via broken DNS) | Use `docker compose build` (no `--no-cache`) — base image already cached locally |

---

## Why Docker?

| Approach         | Dev Server (`expo start`) | Local Build (`eas build --local`) |
| ---------------- | ------------------------- | --------------------------------- |
| Windows terminal | ✅ Works perfectly        | ❌ NDK/SDK not available          |
| WSL              | ❌ Fast refresh broken    | ✅ Works (complex setup)          |
| **Docker**       | ❌ Not needed             | ✅ Works, cached, portable        |

Docker gives you the Linux Android SDK in a container — no WSL SDK setup,
no node_modules conflicts, no NDK issues. Everything is cached in Docker volumes.

---

## Requirements

- **Docker Desktop for Windows** — download from [docker.com](https://www.docker.com/products/docker-desktop/)
- **WSL2 backend enabled** in Docker Desktop (see below)
- An **Expo access token** from expo.dev (for EAS authentication inside container)

### Enable WSL2 Backend in Docker Desktop

1. Open **Docker Desktop**
2. Click the **gear icon** (top right) → **Settings**
3. Click **General**
4. Check **"Use the WSL 2 based engine"**
5. Click **Apply & Restart**

> This is usually enabled by default on Windows 11. If it's already checked, skip this step.

---

## Fix: Docker Can't Resolve DNS (`no such host`)

If you see an error like:

```
dial tcp: lookup registry-1.docker.io: no such host
```

This happens because Docker runs containers inside a lightweight Linux VM with its own
network stack. The VM doesn't always inherit Windows DNS correctly — especially after
switching WiFi, connecting a VPN, or restarting. Since it has no fallback, all DNS
lookups inside containers fail even though your browser works fine.

**Fix — hardcode Google DNS in Docker Desktop:**

1. Open **Docker Desktop**
2. Go to **Settings → Docker Engine**
3. Add `"dns"` to the JSON config:

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

4. Click **Apply & Restart**

Also flush Windows DNS cache in PowerShell:

```powershell
ipconfig /flushdns
```

Then restart Docker Desktop and retry your build command.

> **Why this works:** Instead of relying on whatever DNS the VM inherited from Windows
> at startup, Docker now always uses Google's public DNS (`8.8.8.8`) — which is stable
> and doesn't change when your network does.

### If DNS still fails after daemon.json fix

Windows Firewall may be blocking Docker's VM from reaching `8.8.8.8` on UDP/TCP port 53.
Run this once in **PowerShell as Administrator**:

```powershell
netsh advfirewall firewall add rule name="Docker DNS" dir=out action=allow protocol=UDP remoteport=53
netsh advfirewall firewall add rule name="Docker DNS TCP" dir=out action=allow protocol=TCP remoteport=53
```

This allows Docker's VM to reach external DNS servers. If this works, you won't need
the `extra_hosts` workarounds in `docker-compose.eas.yml` anymore.

---

## Files to Add to Your Project

### 1. `Dockerfile.eas`

```dockerfile
# EAS Local Build Image
# Based on the official React Native Android image which has:
#   Java, Android SDK, NDK, CMake, Gradle all pre-installed
FROM reactnativecommunity/react-native-android:latest

# Update CA certificates — the base image ships with outdated certs that cause
# TLS handshake failures against Maven Central and other HTTPS hosts in Docker Desktop Windows
RUN apt-get update -qq && apt-get install -y --no-install-recommends ca-certificates && update-ca-certificates

# Install EAS CLI globally (cached in image layer)
RUN npm install -g eas-cli@latest

WORKDIR /app
```

### 2. `docker-compose.eas.yml`

```yaml
services:
  eas-build:
    build:
      context: .
      dockerfile: Dockerfile.eas

    # Use host network — container shares Windows/WSL2 network stack directly.
    # This fixes all DNS issues without any extra config or IP pinning.
    network_mode: host

    volumes:
      # Mount project into container
      - .:/app

      # Isolate node_modules inside container (Linux binaries, not Windows)
      # This volume persists — npm install is fast on subsequent runs
      - node_modules:/app/node_modules

      # Cache Gradle downloads and build outputs (~1-2GB, only downloaded once)
      - gradle_cache:/root/.gradle

      # Cache Android SDK extras (avoids re-downloading NDK components)
      - android_cache:/root/.android

      # Cache npm global packages
      - npm_cache:/root/.npm

    # Fix Java/Gradle DNS on Docker Desktop Windows:
    # Java tries IPv6 first by default — IPv6 is broken in Docker Desktop on Windows.
    # This forces Java to use IPv4, fixing all Gradle/SDK download DNS failures.
    environment:
      - _JAVA_OPTIONS=-Djava.net.preferIPv4Stack=true -Dhttps.protocols=TLSv1.2,TLSv1.3 -Djdk.tls.client.protocols=TLSv1.2,TLSv1.3

    # Load all env vars from .env.local (Supabase keys, Firebase paths, EXPO_TOKEN, etc.)
    env_file:
      - .env.local

    # Default profile is "preview" — override by passing arg:
    #   docker compose ... run --rm eas-build production
    entrypoint: >
      bash -c "
        echo '--- Installing dependencies ---' &&
        npm install &&
        echo '--- Starting EAS local build ---' &&
        for attempt in 1 2 3; do
          eas build --local --platform android --profile $${1:-preview} --non-interactive && break
          echo \"--- Attempt $$attempt failed, retrying in 5s... ---\" &&
          sleep 5
        done &&
        echo '--- Build complete. APK is in your project folder. ---'
      "
    command: preview

volumes:
  # node_modules: auto-named per project (e.g. nwt-servicehub_node_modules)
  # Docker Compose prefixes with folder name automatically — no conflicts across projects
  node_modules:

  # gradle/android/npm caches are shared across all projects (saves re-downloading)
  gradle_cache:
    name: eas_gradle_cache
  android_cache:
    name: eas_android_cache
  npm_cache:
    name: eas_npm_cache
```

### 3. `.dockerignore`

```
node_modules
.expo
dist
web-build
android
ios
build-*.apk
build-*/
.git
*.log
```

---

## One-Time Setup

### Step 1 — Get your Expo access token

1. Go to [expo.dev](https://expo.dev)
2. Click your profile → **Settings** → **Access Tokens**
3. Create a new token and copy it

### Step 2 — Add token to `.env.local`

Open your `.env.local` file and add:

```
EXPO_TOKEN=your_token_here
```

> This file is already in `.gitignore` so it won't be committed.

### Step 3 — Add the 3 files to your project

Copy `Dockerfile.eas`, `docker-compose.eas.yml`, and `.dockerignore` into the root of your project.

### Step 4 — Add to `.gitignore` (optional but recommended)

```
# Docker
.dockerignore
```

---

## Running a Build

### First run (slow — downloads image + all caches, ~10–20 min)

```powershell
docker compose -f docker-compose.eas.yml run --rm eas-build
```

This will:

1. Pull the `reactnativecommunity/react-native-android` image (~4GB)
2. Install EAS CLI inside the image
3. Run `npm install` inside the container
4. Run `eas build --local --platform android --profile preview`
5. Output the APK to your project folder on Windows

### Subsequent runs (fast — all caches reused)

```powershell
docker compose -f docker-compose.eas.yml run --rm eas-build
```

Same command — Docker reuses the existing image and cached volumes.
Only your code changes are rebuilt.

### Build a specific profile

```powershell
# preview build (default)
docker compose -f docker-compose.eas.yml run --rm eas-build preview

# production build
docker compose -f docker-compose.eas.yml run --rm eas-build production

# development build
docker compose -f docker-compose.eas.yml run --rm eas-build development
```

---

## Output

The APK appears in your project root on Windows when the build completes:

```
build-1234567890.apk
```

---

## What Gets Cached (Docker Volumes)

| Volume          | Name                               | Contents                          | Approx Size |
| --------------- | ---------------------------------- | --------------------------------- | ----------- |
| `node_modules`  | auto: `[folder-name]_node_modules` | npm packages (Linux binaries)     | ~500 MB     |
| `gradle_cache`  | `eas_gradle_cache` (shared)        | Gradle + all Android dependencies | ~2 GB       |
| `android_cache` | `eas_android_cache` (shared)       | NDK / SDK extras                  | ~500 MB     |
| `npm_cache`     | `eas_npm_cache` (shared)           | npm global cache                  | ~200 MB     |

- `node_modules` is **per-project** — Docker Compose auto-names it using your folder name, so projects never clash
- `gradle_cache`, `android_cache`, `npm_cache` are **shared across all projects** — Gradle and NDK are only downloaded once ever

---

## Using This in a New Project

Copy just these 3 files into the new project root:

```
Dockerfile.eas
docker-compose.eas.yml
.dockerignore
```

**No changes needed.** Docker Compose automatically names the `node_modules` volume
using your project folder name — so it's already unique per project with no config required.

Everything else (Gradle cache, Android cache) is already shared across projects.

---

## Clearing the Cache

If you get strange build errors, clear the volumes and rebuild from scratch:

```powershell
# Remove all cached volumes
docker volume rm nwt-servicehub_node_modules eas_gradle_cache eas_android_cache eas_npm_cache

# Rebuild the Docker image from scratch
docker compose -f docker-compose.eas.yml build --no-cache

# Then run normally
docker compose -f docker-compose.eas.yml run --rm eas-build
```

---

## Recommended Workflow Summary

```
Write code in VSCode (Windows)
        ↓
npx expo start         ← Windows terminal  (fast refresh works)
        ↓
Test on phone/emulator
        ↓
docker compose -f docker-compose.eas.yml run --rm eas-build    ← when ready for APK
        ↓
build-XXXX.apk appears in project folder
```

---

## Quick Reference

| Command                                                                  | What it does                      |
| ------------------------------------------------------------------------ | --------------------------------- |
| `docker compose -f docker-compose.eas.yml run --rm eas-build`            | Build preview APK (default)       |
| `docker compose -f docker-compose.eas.yml run --rm eas-build production` | Build production APK              |
| `docker compose -f docker-compose.eas.yml build --no-cache`              | Rebuild Docker image from scratch |
| `docker volume ls`                                                       | List all cached volumes           |
| `docker volume rm eas_gradle_cache`                                      | Clear Gradle cache                |
