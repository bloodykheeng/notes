# Running Expo Dev Server in WSL (Windows)

Guide for running `npx expo start` in WSL and connecting your phone via Expo Go.

---

## The Problem

WSL2 runs on a private virtual network — by default your phone cannot reach it,
even if it's on the same WiFi. You need to configure WSL to mirror Windows networking.

---

## 1. Enable Mirrored Networking in WSL2

Run this in WSL to create/update `.wslconfig` on Windows:

```bash
echo '[wsl2]
networkingMode=mirrored

[experimental]
hostAddressLoopback=true' > /mnt/c/Users/YOUR_WINDOWS_USERNAME/.wslconfig
```

> Note: `hostAddressLoopback` must be under `[experimental]`, not `[wsl2]`.
> Otherwise WSL shows: `Unknown key 'wsl2.hostAddressLoopback'` on startup.
> Also using `>` (overwrite) not `>>` (append) to avoid duplicates.

> Replace `YOUR_WINDOWS_USERNAME` with your actual Windows username (e.g. `bk`).

Then restart WSL from **PowerShell on Windows**:

```powershell
wsl --shutdown
```

Reopen WSL. This makes WSL share the same network interfaces as Windows,
so your phone can connect using your Windows IP.

---

## 2. Allow Expo Port Through Windows Firewall

Run this in **PowerShell as Administrator** to allow your phone to reach the Metro bundler:

```powershell
netsh advfirewall firewall add rule name="Expo Metro" dir=in action=allow protocol=TCP localport=8082
```

> If Expo uses a different port (e.g. 8081, 8083), change `localport` accordingly.
> The port is shown in the terminal when you run `npx expo start`.

---

## 3. Make Sure Phone is on Same WiFi

Your phone must be connected to the **same WiFi network** as your Windows PC.
If they're on different networks the QR code will not work.

---

## 4. Start Expo

Navigate to your project in WSL:

```bash
cd /mnt/d/coding/your-project-folder
```

Install dependencies from WSL (required if originally installed on Windows):

```bash
rm -rf node_modules
npm install
```

Start the dev server:

```bash
npx expo start
```

You should see your **Windows IP** in the output:

```
› Metro waiting on exp://192.168.1.7:8082
› Scan the QR code above with Expo Go (Android) or the Camera app (iOS)
```

Scan the QR code with Expo Go on your phone.

---

## 5. Test the Connection

If the QR scan fails, test the connection manually by opening a browser on your phone and visiting:

```
http://YOUR_WINDOWS_IP:8082
```

If the page loads, the network connection is working and the issue is with the QR scan.

---

## WSL Limitation: Code Changes Not Reflecting on Device

> **This is a known architectural limitation of WSL2 — there is no workaround.**

When your project lives on a Windows drive (`/mnt/d/...`) and Metro runs inside WSL,
file change events are **never delivered** to Linux. WSL uses inotify for filesystem
watching, but Windows NTFS changes on `/mnt/` drives do not trigger inotify — so Metro
never knows a file changed and never hot-reloads.

Workarounds like `CHOKIDAR_USEPOLLING=true`, `useWatchman: false`, or
`WATCHMAN_DISABLE_FILESYSTEMS_WATCH=1` do **not** help because the polling/watchman
layer still depends on the underlying filesystem events that WSL never receives.

### Recommended workflow

| Task | Terminal to use |
|---|---|
| `npx expo start` (dev server) | **Windows terminal** (CMD, PowerShell, or Git Bash) |
| `eas build --local` | **WSL** (needs Linux Android SDK) |

Running `expo start` from a **Windows terminal** (not WSL) means Metro runs natively
on Windows where file changes are detected immediately — fast refresh works normally.

```powershell
# In Windows terminal (CMD / PowerShell / Git Bash on Windows)
cd D:\coding\your-project-folder
npx expo start
```

Keep WSL only for local EAS builds where you need the Linux Android SDK toolchain.

---

## Fallback — Tunnel Mode

If nothing else works, use tunnel mode (works across different networks, no firewall config needed):

```bash
npx expo start --tunnel
```

> Tunnel mode is slower than direct connection but always works.

---

## Quick Reference

| Issue | Fix |
|---|---|
| WSL IP shown instead of Windows IP | Enable mirrored networking (Step 1) |
| Phone can't connect | Allow port through firewall (Step 2) |
| QR scan fails | Open URL manually in phone browser to test |
| Different networks | Use `--tunnel` mode |
| node_modules issues | Delete and reinstall from WSL |
| Code changes not reflecting | Run `expo start` from Windows terminal, not WSL |

---

## Summary of All Commands

```bash
# 1. Enable mirrored networking (run once)
echo '[wsl2]
networkingMode=mirrored

[experimental]
hostAddressLoopback=true' > /mnt/c/Users/bk/.wslconfig
```

```powershell
# 2. Restart WSL (PowerShell on Windows)
wsl --shutdown
```

```powershell
# 3. Allow firewall port (PowerShell as Administrator)
netsh advfirewall firewall add rule name="Expo Metro" dir=in action=allow protocol=TCP localport=8082
```

```powershell
# 4. Start dev server — run from Windows terminal, NOT WSL
cd D:\coding\your-project-folder
npx expo start
```
