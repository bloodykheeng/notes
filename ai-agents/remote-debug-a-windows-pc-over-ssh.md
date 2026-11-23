# Debugging someone's Windows PC over SSH (same Wi-Fi)

A user's Windows laptop misbehaves with our app ("installs but cannot reach the server", "the window runs off the
screen"), the person is next to me on the same Wi-Fi, and screenshots back and forth are too slow. Windows 10 and 11
ship an SSH server: turn it on for an hour with a temporary account, and the agent on my PC can run commands there,
copy files both ways, install builds, open the app **on the person's own screen**, take screenshots, even click and
type, then everything is removed.

Written from the MRDRS case (Nov 2026): a Flutter Windows app said "Could not reach the server" on a colleague's
Windows 10 laptop. Three guesses (certificates, firewall, installer) were wrong; ten minutes on the machine found the
real cause, an old server address saved in the app's settings. See section 7.

Two parts: **A** is the quick human guide (what I do, what the person does). **B** is for the AI agent: how to drive
it, the traps, and a diagnostics checklist.

---

# A. Quick guide (for me)

## A1. On the other laptop (the person does this, about 5 minutes)

Open **PowerShell as Administrator**: right-click the Start button > **Windows PowerShell (Admin)**.
Paste with **Ctrl+V** (or Shift+Insert, or the window icon > Edit > Paste). The clipboard does not cross computers:
send the commands to that laptop first (WhatsApp Web or email opened **on it**). Paste **one line at a time**.

```powershell
# 1. Windows' own SSH server (downloads from Windows Update: "Operation running" with a progress bar, 1 to 3 minutes)
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# 2. Start it now and at every boot, and let it through the firewall
Set-Service sshd -StartupType Automatic; Start-Service sshd; Get-Service sshd      # must say Running
New-NetFirewallRule -Name sshd-support -DisplayName "OpenSSH (support)" -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22

# 3. A temporary admin account for the agent (asks for a password; nothing shows while typing)
$p = Read-Host -AsSecureString "Password"; New-LocalUser -Name support-temp -Password $p; Add-LocalGroupMember -Group Administrators -Member support-temp

# 4. Its address on the Wi-Fi (the 192.168.x.x one)
ipconfig | findstr IPv4
```

Then I put the IP and the password in a text file in the project's `deployment/` folder (never in the chat) and tell
the agent where it is.

## A2. What I tell the agent

> Read `coding-notes/ai-agents/remote-debug-a-windows-pc-over-ssh.md`. The laptop's IP and the support account's
> password are in `<path to txt>` (line 1 IP, line 2 password). Check `<the problem>`; show me screenshots.

The person keeps the laptop **on, awake and signed in** (the agent opens apps on the signed-in desktop), and should
not type or move the mouse while the agent clicks and types on their screen.

## A3. When finished: remove everything (PowerShell as Administrator, on that laptop)

```powershell
# The account, the firewall rule and the SSH server
Remove-LocalUser -Name support-temp; Remove-NetFirewallRule -Name sshd-support; Stop-Service sshd; Set-Service sshd -StartupType Disabled

# What the agent left: its scheduled tasks, its working folder, the deleted account's profile folder
Get-ScheduledTask SUPPORT-* | Unregister-ScheduledTask -Confirm:$false
Remove-Item C:\Users\Public\support, C:\Users\support-temp -Recurse -Force -ErrorAction SilentlyContinue
```

The agent then confirms from its side that port 22 is closed and the login fails. The working folder holds
screenshots of the person's desktop (their WhatsApp may be on them): always delete it.

## A4. Common hiccups

| What you see | What it means / do |
| --- | --- |
| Right-click does not paste | Ctrl+V, or Shift+Insert; in Windows Terminal Ctrl+Shift+V. Check you copied **on that laptop**. |
| "Access is denied" | PowerShell is not "as Administrator". |
| `Start-Service sshd` red error "cannot start" | It is set to Disabled (often after running the clean-up by mistake): run `Set-Service sshd -StartupType Automatic` first, then start. |
| `Add-WindowsCapability` fails | The PC cannot reach Windows Update (office policy, WSUS). Use the zip from github.com/PowerShell/Win32-OpenSSH (`install-sshd.ps1`). |
| The agent says "port 22 closed" | sshd not running, or the Wi-Fi isolates devices (guest networks do): use a phone hotspot both PCs join. |

---

# B. For the AI agent

## B1. Ground rules

- **Consent and scope.** Only a machine whose owner is present and agreed, for the problem named. Read-only first;
  ask before anything that changes the machine beyond our app (installing other software, changing system settings).
- **Secrets.** The password comes in a file; read it file to file (`sed -n '2p' file > "$SP/pw"`), never echo it. If
  the user pastes it in chat anyway, use it, do not repeat it, and remind them to remove the account afterwards.
- **The person's data.** Look only at what the problem needs (our app's folders, logs, settings). Screenshots show
  their whole desktop: keep them in the scratchpad, show only what matters, delete them at the end.
- **Clean up and verify** (A3), and say what you could not remove once your access was gone.

## B2. Connecting from Git Bash on my PC (no sshpass on Windows)

OpenSSH 8.4+ reads the password from a helper (`SSH_ASKPASS`), as in
[deploy-yourself-over-ssh.md](deploy-yourself-over-ssh.md):

```bash
SP="<scratchpad>"; F="<path to the txt>"
umask 077
sed -n '1p' "$F" | tr -d '\r\n ' > "$SP/ip"; sed -n '2p' "$F" | tr -d '\r\n' > "$SP/pw"
printf '#!/bin/sh\ncat "%s/pw"\n' "$SP" > "$SP/ap.sh"
IP=$(cat "$SP/ip")
printf '#!/bin/sh\nSSH_ASKPASS="%s/ap.sh" SSH_ASKPASS_REQUIRE=force DISPLAY=:0 exec ssh -o StrictHostKeyChecking=accept-new -o PubkeyAuthentication=no -o PreferredAuthentications=password,keyboard-interactive -o ConnectTimeout=10 support-temp@%s "$@"\n' "$SP" "$IP" > "$SP/l.sh"
printf '#!/bin/sh\nSSH_ASKPASS="%s/ap.sh" SSH_ASKPASS_REQUIRE=force DISPLAY=:0 exec scp -o StrictHostKeyChecking=accept-new -o PubkeyAuthentication=no -o PreferredAuthentications=password,keyboard-interactive "$@"\n' "$SP" > "$SP/lcp.sh"
chmod 700 "$SP"/*.sh

timeout 5 bash -c "</dev/tcp/$IP/22" && echo open || echo closed     # ping is usually blocked; test the port
"$SP/l.sh" "powershell -NoProfile -Command \"(Get-CimInstance Win32_OperatingSystem).Caption\""
```

The remote shell is **cmd.exe**; every command goes through `powershell -NoProfile -Command "..."` or a script.

## B3. Running scripts: copy the file, run it with -File

Piping a script into `powershell -Command -` runs it in interactive mode: it echoes `>>` prompts and often runs
nothing. Write the `.ps1` locally, copy it, run it:

```bash
"$SP/lcp.sh" "$SP/check.ps1" support-temp@$IP:check.ps1            # lands in C:\Users\support-temp\
"$SP/l.sh" "powershell -NoProfile -ExecutionPolicy Bypass -File check.ps1"
```

In the script, end tables with `| Format-Table | Out-String` (or `Format-List`) or nothing prints over SSH.

## B4. Files both ways

```bash
"$SP/lcp.sh" ./build/MyApp-Setup.exe support-temp@$IP:C:/Users/Public/support/        # upload
"$SP/lcp.sh" support-temp@$IP:C:/Users/Public/support/install.log "$SP/"              # download
"$SP/lcp.sh" -r support-temp@$IP:"C:/Users/<them>/AppData/Roaming/<app>" "$SP/appdata" # a folder
```

Use forward slashes in scp paths. A shared working folder (`C:\Users\Public\support`) is readable both by the
support account and by tasks running as the person. Downloading on the laptop itself: `curl.exe -sS -L -o <file> <url>`
(`-sS`, not `-s`, or a failure is silent).

## B5. Installing our build with a log

```powershell
Get-Process myapp -ErrorAction SilentlyContinue | Stop-Process -Force
$p = Start-Process 'C:\Users\Public\support\MyApp-Setup.exe' -ArgumentList '/VERYSILENT','/SUPPRESSMSGBOXES','/NORESTART','/LOG=C:\Users\Public\support\install.log' -Wait -PassThru
"exit $($p.ExitCode)"
(Get-Item 'C:\Program Files\MyApp\myapp.exe').VersionInfo.FileVersion
```

(Inno Setup flags; the SSH session of an Administrators member is elevated, so a per-machine installer runs.) Inno
keeps a file's old date when it is unchanged, so check the version, not the date.

## B6. Opening an app on the person's screen

An SSH session has no desktop (it is a service session): a GUI app started from it runs invisibly. Start it **in the
signed-in user's session** with a scheduled task:

```powershell
$who = (Get-CimInstance Win32_ComputerSystem).UserName          # e.g. LAPTOP\Jane
$s = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries   # see the trap below
$p = New-ScheduledTaskPrincipal -UserId $who -LogonType Interactive
$a = New-ScheduledTaskAction -Execute 'C:\Program Files\MyApp\myapp.exe'
Register-ScheduledTask -TaskName 'SUPPORT-open' -Action $a -Principal $p -Settings $s -Force | Out-Null
Start-ScheduledTask 'SUPPORT-open'
```

**Trap:** a task's default settings refuse to start on battery. On a laptop off the charger `Start-ScheduledTask`
returns, `LastTaskResult` says 0, and nothing runs. Always pass the battery settings. Test with a task that writes
`%USERNAME%` to a file before blaming the app.

## B7. Screenshots of their screen

The same trick runs a capture script as the person. Make it **DPI aware**, or a 150 % screen is captured cropped:

```powershell
# C:\Users\Public\support\shot.ps1
Add-Type -AssemblyName System.Windows.Forms, System.Drawing
Add-Type -TypeDefinition 'using System.Runtime.InteropServices; public class Dpi { [DllImport("user32.dll")] public static extern bool SetProcessDPIAware(); }'
[void][Dpi]::SetProcessDPIAware()
$b = [System.Windows.Forms.SystemInformation]::VirtualScreen
$bmp = New-Object System.Drawing.Bitmap $b.Width, $b.Height
[System.Drawing.Graphics]::FromImage($bmp).CopyFromScreen($b.Left, $b.Top, 0, 0, $bmp.Size)
$bmp.Save('C:\Users\Public\support\screen.png', [System.Drawing.Imaging.ImageFormat]::Png)
```

Register it as `SUPPORT-shot` (action `powershell.exe -NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File
C:\Users\Public\support\shot.ps1`), start it, wait 3 to 5 seconds, scp the PNG back, and look at it.

## B8. Clicking and typing on their screen

`WScript.Shell.AppActivate` + `SendKeys` from a background task usually fails: Windows does not let a background
process bring a window to the front, so the keys go nowhere. Click the field first (a click focuses it), then type:

```powershell
Add-Type -TypeDefinition @'
using System; using System.Runtime.InteropServices;
public class M {
  [DllImport("user32.dll")] public static extern bool SetProcessDPIAware();
  [DllImport("user32.dll")] public static extern bool SetCursorPos(int x, int y);
  [DllImport("user32.dll")] public static extern void mouse_event(int f, int x, int y, int d, int e);
  public static void Click(int x, int y) { SetCursorPos(x, y); mouse_event(2,0,0,0,0); mouse_event(4,0,0,0,0); }
}
'@
[void][M]::SetProcessDPIAware()          # then the coordinates are the screenshot's pixels
$w = New-Object -ComObject WScript.Shell
[M]::Click(1850, 522); Start-Sleep -Milliseconds 400; $w.SendKeys('^a'); $w.SendKeys('FAC99')
```

Read the coordinates off the last screenshot (DPI-aware capture and DPI-aware clicks use the same pixels). Prove a
connection without anyone's real account: type a code and user that do not exist; "wrong code or password" means the
app reached the server, "could not reach the server" means it did not.

## B9. Diagnostics checklist ("the app cannot reach the server")

Run these before guessing; each rules out a cause.

| Check | Command | Rules out |
| --- | --- | --- |
| Windows edition, build, 32/64-bit | `Get-CimInstance Win32_OperatingSystem \| Select Caption, Version, OSArchitecture` | 32-bit, very old builds |
| Clock and zone | `Get-Date -Format o; (Get-TimeZone).Id` | certificate "not yet valid" / "expired" |
| What is installed | `HKLM:\...\Uninstall\*` DisplayName / DisplayVersion; exe `VersionInfo` | "they still run the old version" (they often do) |
| Proxy | `netsh winhttp show proxy`; per user `HKU\<sid>\...\Internet Settings` ProxyEnable / ProxyServer / AutoConfigURL | a proxy only the browser knows |
| Firewall | `Get-NetFirewallProfile \| Select Name, DefaultOutboundAction`; rules naming the app | outbound blocks |
| Antivirus | `Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntiVirusProduct` | a third-party AV filtering |
| Windows' own HTTPS to our server | `Invoke-WebRequest -UseBasicParsing https://<api>/up` | DNS, routing, the server |
| DNS and hosts | `Resolve-DnsName <api>`; `drivers\etc\hosts` | a hosts override |
| Root certificate present | `Cert:\LocalMachine\Root` (and the registry: `AuthRoot` = fetched on demand) | missing CA root |
| **The app's own settings** | its `shared_preferences.json` (Flutter: `%APPDATA%\<company>\<app>\`), any saved server URL | **a stale saved address** |
| App crashes | `Get-WinEvent` Application log, Level 2, message naming the exe | crash at start |

Note: `Invoke-WebRequest` itself makes Windows fetch a missing root into `AuthRoot`, so check the root store
**before** testing HTTPS if the root is a suspect.

## B10. Reporting

Say what was found, with the screenshot that proves it; what was changed on the machine (and backed up first, e.g.
`settings.json.before-support`); what the build fix is; and which leftovers the person must remove after access ends.

---

# 7. The MRDRS case (what this note came from)

- **Symptom:** the Flutter Windows app installed but every sign-in said "Could not reach the server"; the browser and
  other apps worked; a release that shipped the CA roots did not help.
- **Found in ten minutes on the machine:** Windows 10 Pro 22H2 64-bit, correct clock, no proxy, outbound allowed,
  Defender only, Windows' HTTPS to the API 200, root present. The person still ran the older build (installer cached).
  Opening the app on their screen and typing a fake account reproduced the error on the new build too, which ruled
  out certificates (that build names certificate errors). The app's `shared_preferences.json` held
  `sync_base_url_override = http://uhfmrisapi.test/api/`: a developer's local address saved by an old build, which
  survives reinstalling and overrode the built-in server. Removing it: the app got the server's "wrong code or
  password" answer at once.
- **Fixed in the app:** release builds drop a saved server address that is not HTTPS (or is `.test`, `.local`,
  `localhost`, an IP); the error names the server it tried; the window opens at a size that fits the screen (it ran
  off a laptop's screen and hid the title-bar buttons); the Windows exe carries the real version (an old project
  without the `FLUTTER_VERSION` defines showed "1.0.0").
- **Lesson:** a catch-all `catch (_) { "could not reach the server" }` cost a release. Name the cause class
  (TLS, timeout, DNS/socket) and the server address in the message; then a screenshot alone would have shown
  `uhfmrisapi.test`.
