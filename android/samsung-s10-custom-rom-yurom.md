# Galaxy S10 (SM-G973F): ExtremeROM to YuROM Ultra, custom boot logo, bootloop guard

How the Exynos Galaxy S10 went from ExtremeROM Nexus to **YuROM Ultra V35 R34 (One UI 9, Android 17)**
in November 2026. Every wall we hit is here, with the cause and the fix, plus the custom boot logo and a
Magisk module that ends the "battery died, now it bootloops" problem.

Everything from that job (ROM zips, patched installer, backups, logo, modules) is on the PC in
`D:\OS\S10 custom rom\`.

---

## The device and the words

| Thing | Value / meaning |
| --- | --- |
| Phone | SM-G973F/DS (Exynos 9820), codename `beyond1lte` / `beyond1` |
| Bootloader | `G973FXXSHHWI1` (revision **H**: the 5th character after `G973FXX` is S/U, then the revision letter) |
| Odin | Samsung's PC flasher. `.tar` files go in **AP** (BL/CP/CSC only for full firmware) |
| TWRP | Custom recovery. Boot into it with **Vol Up + Bixby + Power** |
| Download mode | **Vol Down + Bixby** and plug in the USB cable (or from TWRP: Reboot > Download) |
| Force restart | Hold **Vol Down + Power** |
| vbmeta | Verified-boot metadata. Custom ROMs flash a "verification disabled" one |
| `up_param` | 8 MB partition: a GNU tar of the bootloader's JPEGs (`logo.jpg` is the first screen) |
| `/data/apex/active` | Updated core Android modules from Play system updates |

Read the bootloader revision without Settings: in TWRP > Advanced > Terminal (or `adb shell`)
run `getprop ro.boot.bootloader`.

---

## Final working recipe (short version)

1. Odin, AP = **ExtremeROM TWRP** (`ExtremeROM_Nexus_G973_TWRP_OdinPack_v2.tar`), then boot straight into TWRP.
2. Make `/data` **ext4** (YuROM refuses f2fs), wipe it.
3. Put the 3 `DATA_xof3.zip` files and the **patched** installer in the same folder on the phone.
4. Run the installer (TWRP Install, or over adb, see below). Wait for "readback passed" and "hotfix r7 installed".
5. Reboot. **The first boot restarts 5-6 times before setup. Wait.** Later boots are normal.

---

## Wall 1: the official TWRP (twrp.me) bootloops

**Symptom:** Samsung logo > TWRP splash > the phone turns off, in a loop. ExtremeROM's TWRP works fine.

**Not the cause:** vbmeta. If vbmeta were wrong, the bootloader would refuse before TWRP ("only official
released binaries"). Seeing the TWRP splash means the recovery kernel booted and TWRP itself crashed.
The `vbmeta.tar` from twrp.me was already "verification disabled" (flags byte at offset 120 = `2`):

```bash
tar -xf vbmeta.tar && xxd -s 120 -l 4 vbmeta.img   # 0000 0002 = verification disabled
```

**Cause:** official TWRP 3.7.0_9-1 (Feb 2023) does not cope with this device after ExtremeROM
(repartitioned, f2fs data, newer boot pieces). **Fix:** use the ROM community's own TWRP
(ExtremeXT TWRP 3.7.1_12). Do not burn hours on the official one.

## Wall 2: "The compatible recovery BusyBox is required"

**Symptom:** the YuROM installer aborts with `The compatible recovery BusyBox is required. Updater process ended with ERROR: 1`.

**Cause:** the first line of the installer's `update-binary` is `[ -x /sbin/busybox ] || abort ...`.
Official TWRP has `/sbin/busybox`. ExtremeXT TWRP has **no busybox at all**, but has every tool the
installer uses (`unzip` via ziptool, `sha256sum`/`dd`/`mktemp` via toybox, `awk`, `sh`).

How to check what a recovery image ships without flashing it: the ramdisk is LZMA (`]\x00\x00\x80`)
or lz4; unpack it and list the cpio entries (a short Python script did this, `lzma.FORMAT_ALONE`).

**Fix:** patch only `META-INF/com/google/android/update-binary` so it falls back to the plain commands:

```sh
BB=; [ -x /sbin/busybox ] && BB=/sbin/busybox
for t in sh unzip sha256sum; do [ -n "$BB" ] || command -v $t >/dev/null 2>&1 || abort "Recovery is missing $t."; done
$BB unzip -p "$SELF" artisan/install.sh > "$WORK/install.sh" || abort '...'
actual="$($BB sha256sum "$WORK/install.sh")" || abort '...'
$BB sh "$WORK/install.sh" "${1:-3}" "$OUTFD" "$SELF"
```

The installer verifies the hash of `install.sh` (left untouched) and of the DATA zips, not of
`update-binary`, and TWRP does not check zip signatures by default, so the patched zip installs.
Check the DATA zips on the PC first with `sha256sum`. The expected hashes are inside `install.sh`.

## Wall 3: "failed to unmount /vendor" and the real error behind it

**Symptom:** `failed to unmount /vendor (Device or resource busy)` then error 1.

**Not the cause:** /vendor. That is TWRP noise, and nothing held it (checked over adb:
`/proc/mounts`, `/proc/*/maps`). **Cause:** `/data` was **f2fs** (ExtremeROM's format) and the
installer requires **ext4**:

```sh
awk -v data=/data '$2 == data && $3 == "ext4" { found=1 } END { exit !found }' /proc/mounts || abort '...ext4 /data...'
```

TWRP "Format Data" keeps the current filesystem. Use **Wipe > Advanced Wipe > Data > Repair or
Change File System > Change File System > EXT4**, or over adb from TWRP (`sda34` = userdata here):

```sh
umount /sdcard; umount /data; grep sda34 /proc/mounts     # must print nothing
mke2fs -F -t ext4 -b 4096 /dev/block/sda34
e2fsdroid -e -S /file_contexts -a /data /dev/block/sda34
mount -t ext4 /dev/block/sda34 /data
mkdir -p /data/media/0 && chown -R 1023:1023 /data/media && chmod 770 /data/media /data/media/0
chcon -R u:object_r:media_rw_data_file:s0 /data/media
```

Only `lost+found` and `media` may exist in `/data` afterwards, or the installer takes the "upgrade" path
and refuses.

## Wall 4: ExtremeROM repartitioned the phone

```sh
for p in vbmeta system vendor dtb dtbo boot; do echo "$p $(cat /sys/class/block/$(basename $(readlink -f /dev/block/by-name/$p))/size)"; done
```

| Partition | Stock | After ExtremeROM |
| --- | --- | --- |
| system | ~11.7M sectors | 16384000 (8000 MiB) |
| vendor | 2252800 (1100 MiB) | 3072000 (1500 MiB) |

YuROM's `install.sh` accepts system/vendor **at least** the minimum (fine). But every hotfix part inside
`artisan/hotfix-r7.zip` (Hotfix_r7, Bluetooth_Fix_r2, SyncTile_Fix_r1, the S10e ones) checks
`[ "$sectors" = "$SECTORS" ]` = exact stock size, and would refuse. Without the hotfix: dropped calls,
wrong resolution, Chrome restarting the phone.

**Fix:** change the check to `-ge` in each part, then rebuild the hash chain bottom-up:
1. patched part zip > new sha256 > replace the old hash in the bundle's `update-binary` (`PARTS=` list)
2. repacked `hotfix-r7.zip` > new sha256 > replace `CALL_FIX_SHA256=` in the installer's `update-binary`
3. verify the whole chain in a script before using it (each hash matches the bytes it names).

Saved as `YuROM_R34_G973F_INSTALLER_ExtremeTWRP_v2.zip` (busybox fallback + size fix). The alternative
is to flash full stock firmware with `CSC_` (not `HOME_CSC`), which restores stock partition sizes.

## Installing over adb from TWRP (no tapping, full log)

```bash
export MSYS_NO_PATHCONV=1      # Git Bash: stop it rewriting /data/... into C:/Program Files/Git/data/...
adb push DATA_1of3.zip DATA_2of3.zip DATA_3of3.zip INSTALLER_v2.zip /data/media/0/
adb shell 'Z=/data/media/0/INSTALLER_v2.zip; unzip -p $Z META-INF/com/google/android/update-binary > /tmp/ub;
  (sh /tmp/ub 3 1 $Z; echo "EXIT=$?") > /tmp/install.log 2>&1 </dev/null &'
```

- `3 1 zip` = API version, output fd 1, zip path, the same arguments TWRP passes.
- The log gets partly overwritten (the script reopens `/proc/self/fd/1`), so read it with
  `cat /tmp/install.log | tr '\0' '\n'`. The last lines plus `EXIT=0` are what matter.
- In a `while read` loop, use `adb shell -n` or adb eats the loop's input.

Verify what was written against the ROM's own manifest (`artisan/chunks.tsv`):

```bash
grep -E "^(boot|dtb|dtbo|vbmeta)\b" chunks.tsv | while read part off bytes sha entry; do
  got=$(adb shell -n "head -c $bytes /dev/block/by-name/$part | sha256sum" </dev/null | cut -d' ' -f1)
  [ "$got" = "$sha" ] && echo "$part OK" || echo "$part MISMATCH"; done
```

## First boot: 5-6 restarts is normal (for this build)

Samsung logo > "powered by Android" > animation > black > again, 5-6 times, then setup appears.
It is not a bootloop. Every restart showed up as Windows' USB "connect" sound on the laptop, so that
sound is a way to count them. Afterwards:

```bash
adb shell getprop persist.sys.boot.reason.history   # "reboot,..." = requested restarts, not crashes
adb shell dumpsys dropbox | grep -E "^[0-9]{4}-"     # crash entries; *_KP = kernel panic
```

One of them was a panic forced from userspace (`sysrq_handle_crash <- write_sysrq_trigger`), i.e. a
first-boot step deliberately resetting. A second boot is one clean boot.

**Debugging a phone that loops before the "Allow USB debugging" prompt:** the Android logs are
FBE-encrypted and TWRP could not decrypt them. `/proc/last_kmsg` after a *forced* restart is a cold
boot and only has the bootloader log. Changing `ro.adb.secure=0` in `/vendor/build.prop` did **not**
work on this user build (still "unauthorized"). If needed again: copy the PC's `adbkey.pub` into
`/data/misc/adb/adb_keys` from an init `.rc` on vendor, or just wait, since here it was the normal first boot.

## Magisk

YuROM ships Magisk. Open the stub app > "upgrade to full Magisk" > reboot. The **Install** button on
Magisk's home screen is always there (re-patch/update). It does not mean it is not installed. Root for adb:
run `adb shell su -c id` and tap Grant for "Shell". Use `su -mm -c` to remount `/vendor` rw.

## Custom first screen ("bloodykheeng" logo)

The first screen (SAMSUNG Galaxy, Secured by Knox) is `logo.jpg` inside `up_param`
(1440x3120, baseline JPEG). Replace it, keep everything else byte for byte:

```bash
adb shell su -c 'dd if=/dev/block/by-name/up_param of=/data/local/tmp/up_param.img'   # back up first!
adb pull /data/local/tmp/up_param.img
```

1. Make a 1440x3120 JPEG (PIL, Montserrat Black, red gradient + glow), `progressive=False`, `subsampling=2`.
2. Rebuild the tar with Python `tarfile`, **same format (gnu), same member order and metadata**, only
   `logo.jpg` swapped, padded with zeros to exactly 8388608 bytes. Check every other member is identical.
3. Write it and verify:
   ```bash
   adb push up_param_new.img /data/local/tmp/
   adb shell su -c 'dd if=/data/local/tmp/up_param_new.img of=/dev/block/by-name/up_param bs=4096 conv=fsync; sha256sum /dev/block/by-name/up_param'
   ```
Restore = the same `dd` with the backup. A missing or bad image only makes the bootloader skip it
(it already logs `find fail!(booting_warning.jpg)` and carries on).

## Boot animation: why it is drawn too big, and why we left it

The animation after the logo is Samsung's own `bootsamsung.qmg` + `bootsamsungloop.qmg`
(`shutdown.qmg` for power off). The bootanimation binary links only `libQmageDecoder`, so **QMG only**,
no `bootanimation.zip`. There is no public QMG encoder. Samsung converts PNGs for Knox business
customers.

The log explains the bad scaling (the ROM is ported from a 1080x2340 Galaxy):

```bash
adb shell logcat -d | grep '!@BootAnimation' | grep -E 'resolution|Load'
#  <<MAIN>> RESOURCE resolution w = 1080, h = 2340
#  DEVICE DISPLAY resolution w = 1440, h = 3040
```

Overlaying the loop with the main file via a Magisk module made it worse (MAIN is the "Samsung Galaxy"
part and is mis-scaled too), so it was reverted. Hiding the animation was rejected, because a black screen looks like a crash.

**Fixed by the dev's BootLogo Fix r1** (release `v35-r34-fixes1` on github.com/yuefdev/YuROM-Ultra). It puts
QHD (1440x3120) boot/loop/shutdown QMGs **and the bootanimation binary from a QHD One UI 9 phone** in
`/vendor/etc/s10-bootlogo/`, and bind-mounts them over `/system/media` and `/system/bin/bootanimation` at
`post-fs` (`/vendor/etc/init/10-s10-bootlogo.rc`, plus one odm_sepolicy rule `allow init bootanim_exec (file (mounton))`).
A watcher restarts the animation once after the switch to FHD+. Known: the logo plays twice.
It has the same exact vendor-size check, so on the repartitioned phone: copy the zip, change line 98
`[ "$sectors" = "$SECTORS" ]` to `[ "$sectors" -ge "$SECTORS" ] 2>/dev/null`, and rebuild the zip with **fresh
ZipInfo headers** (reusing the original ZipInfo objects gave "Bad magic number for file header" on read-back).
Verify with Python plus `unzip -t`, then run it in TWRP over adb:
```bash
adb reboot recovery; adb push fix.zip /tmp/bootlogo.zip
adb shell 'unzip -p /tmp/bootlogo.zip META-INF/com/google/android/update-binary > /tmp/ub; sh /tmp/ub 3 1 /tmp/bootlogo.zip; echo EXIT=$?'
```
Result: `RESOURCE resolution w = 1440, h = 3120` on both runs (1440x3040 then 1080x2280), centred, no denials.
(TWRP's toybox `grep` does not understand `\|`, so use `grep -E 'a|b'`.)

## Bootloop after the battery dies: the APEX guard module

On ExtremeROM, if the battery died with an app open, the next boot looped on the Samsung logo and the
only fix was TWRP > delete the contents of `/data/apex/active`. Cause: a half-written Play system update
(APEX) and no checkpoint/rollback on this device.

Magisk module `apex_bootloop_guard` (`D:\OS\S10 custom rom\magisk_modules\`):

`post-fs-data.sh` (every boot, early):
```sh
D=/data/adb/apexguard; LIMIT=3; mkdir -p "$D"
C=$(cat "$D/count" 2>/dev/null); case "$C" in ''|*[!0-9]*) C=0 ;; esac
C=$((C + 1)); echo "$C" > "$D/count"; echo "$(date '+%F %T') boot attempt $C" >> "$D/log.txt"
if [ "$C" -ge "$LIMIT" ]; then
    if [ -n "$(ls -A /data/apex/active 2>/dev/null)" ]; then
        rm -rf "$D/quarantine"; mkdir -p "$D/quarantine"
        mv /data/apex/active/* "$D/quarantine/" && echo "$(date '+%F %T') BOOTLOOP: moved apex/active" >> "$D/log.txt"
    fi
    echo 0 > "$D/count"
fi
```
`service.sh` (boot finished and stayed up 60 s = good boot):
```sh
until [ "$(getprop sys.boot_completed)" = 1 ]; do sleep 5; done
sleep 60; echo 0 > /data/adb/apexguard/count
```
Install: `adb push apex_bootloop_guard.zip /data/local/tmp/ && adb shell su -c 'magisk --install-module /data/local/tmp/apex_bootloop_guard.zip'`, reboot,
then `su -c cat /data/adb/apexguard/log.txt` shows `boot attempt 1` ... `boot OK`.
Remove a module: `su -c touch /data/adb/modules/<id>/remove` and reboot.

## No region (CSC/OMC) loaded: call recording missing (fixed)

**Symptom:** no call recording (ExtremeROM had it), `getprop ro.csc.sales_code` empty, `/optics` and
`/prism` are empty folders, `SemCscFeature: files does not exist from null` in logcat.

**Cause:** YuROM never mounts the `optics` (carrier configs) and `prism` (OMC etc) partitions. Stock mounts
them in first stage. `init.rc` has `import /prism/etc/init/init.rc`, but that runs before `/prism` exists.
The partitions still held ExtremeROM's OMC (`SAOMC_SM-G973F_OXM_SWC_15_0010`, region from
`/efs/imei/mps_code.dat` = SWC).

**Fix, part 1: mount them** from the vendor fstab (`mount_all` runs at `fs`, before init's builtin
`omc_update` at `post-fs`). Add after the efs line of `/vendor/etc/fstab.exynos9820`:
```
/dev/block/by-name/prism	/prism	ext4	ro,nosuid,nodev,noatime	wait,nofail
/dev/block/by-name/optics	/optics	ext4	ro,nosuid,nodev,noatime	wait,nofail
```
and `/vendor/etc/init/09-yurom-omc-prism.rc`:
```
on fs
    mount ext4 /dev/block/by-name/omr /omr nosuid nodev noatime noauto_da_alloc,discard,journal_checksum,data=ordered
```
- `nofail` = a mount failure never stops boot.
- **`/vendor/etc` is a read-only overlayfs** (vendor_overlay), so `remount,rw /vendor` is not enough. Write
  through a second mount of the block device:
  ```sh
  su -mm -c 'mount -o remount,rw /vendor; mkdir -p /mnt/vrw; mount -t ext4 -o rw /dev/block/by-name/vendor /mnt/vrw
  cat /data/local/tmp/fstab.exynos9820 > /mnt/vrw/etc/fstab.exynos9820     # cat > keeps inode + SELinux label
  umount /mnt/vrw; mount -o remount,ro /vendor'
  ```
  New files under `/vendor/etc/init` need `chcon u:object_r:vendor_configs_file:s0`.
- `setprop ro.omc.*` / `chown` in a vendor `.rc` are refused: vendor rc runs as **vendor_init**, which may not
  set `csc_prop` (`avc: denied { set } ... vendor_init ... csc_prop`).

**Fix, part 2: set the props from Magisk.** After part 1, `omc_update` logs `SALES_CODE : SWC` and
`OMC_PATH : /optics/configs/carriers/single/SWC/conf`, but the props stay empty. Module `omc_region_fix`,
`post-fs-data.sh` (runs before zygote) with `resetprop -n`:
```sh
CODE=$(cat /efs/imei/mps_code.dat | tr -d '[:space:]'); CONF=/optics/configs/carriers/single/$CODE/conf
resetprop -n ro.csc.sales_code $CODE;      resetprop -n ro.csc.country_code SWITZERLAND
resetprop -n ro.csc.countryiso_code CH;    resetprop -n ril.sales_code $CODE
resetprop -n mdc.sys.omc_salescode $CODE;  resetprop -n persist.sys.omc_path $CONF
resetprop -n persist.sys.omc_etcpath /prism/etc/carriers/single/$CODE
resetprop -n mdc.system.path $CONF/system  # <- the one SemCscFeature needs ("from null" before it)
resetprop -n mdc.system.nw_path $CONF/system; resetprop -n mdc.matched_code $CODE
resetprop -n persist.omc.sales_code $CODE; resetprop -n ro.omc.multi_csc OXM   # (+ the rest in the module)
```
Do **not** set `ro.omc.changetype DATA_RESET_ON` / `ro.simbased.changetype`: with a foreign SIM the phone
could switch CSC and **factory reset itself**. Result: `SemCscFeature: Encoded`, and call recording is back.

**How the property names were found:** pull `/system/framework/framework.jar`, scan its `classes*.dex` for
printable strings with Python (`re.finditer(rb'[\x20-\x7e]{4,160}', dex)`), grep `mdc\.|persist\.sys\.omc|ro\.csc`.

### Reading Samsung's encrypted `cscfeature.xml`
OMC text files (`cscfeature.xml`, `customer_carrier_feature.json`) are encoded. Algorithm from
fei-ke/OmcTextDecoder ("decompiled from csc.apk"): **decode** = for each byte i, rotate left by
`SHIFTS[i % 256]`, XOR with `SALTS[i % 256]`, then **gunzip**. **Encode** = gzip(level 9), XOR salt, rotate right.
The Python port with both 256-entry tables is in `D:\OS\S10 custom rom\omc_fix\omc_codec.py`
(`python -I omc_codec.py decode cscfeature.xml out.xml`). Gzip's CRC proves the tables are right.
The decoded SWC file showed ExtremeROM's edit:
```xml
<!-- Added by unica/mods/csc/customize.sh -->
<CscFeature_VoiceCall_ConfigRecording>RecordingAllowed</CscFeature_VoiceCall_ConfigRecording>
```

## Quick panel Mobile data tile does nothing (ROM telephony bug, not fixed)

Tap is registered (`SystemUIAnalytics ... MobileData, icon tap`), nothing else happens. The chain:
1. SystemUI `MobileDataTileUserActionInteractor.handleClick` starts with
   `mobileConnectionsRepository.getActiveMobileDataRepository()`; null means a silent return.
   `dumpsys activity service com.android.systemui/.SystemUIService` shows `Current active data subId=-1`.
2. `dumpsys telephony.registry`: `mActiveDataSubId=-1`. `dumpsys isub`: `activeDataSubId=1`.
3. `dumpsys activity service com.android.phone/com.android.phone.TelephonyDebugService` (PhoneSwitcher section):
   `DDS change success, PhoneId : 0` then `notifyPreferredDataSubIdChanged to -1`, `mPreferredDataSubId=-1`.
   Samsung's `PhoneSwitcherForMultiSim` maps phone 0 to subId -1 when publishing (single SIM in a DSDS model).
   An airplane-mode cycle does not fix it.
**Fixed with a root workaround** (Magisk module `data_tile_fix`): tell the registry the real value through the
same binder call the phone service should make. On this build `ITelephonyRegistry.notifyActiveDataSubIdChanged`
is transaction **30** (read from `TRANSACTION_notifyActiveDataSubIdChanged` in framework.jar's dex static values):
```sh
su -c 'service call telephony.registry 30 i32 1'   # 1 = settings get global multi_sim_data_call
dumpsys telephony.registry | grep mActiveDataSubId  # -> 1, SystemUI: "Current active data subId=1", tile works
```
`service.sh` waits for boot_completed, **only runs if `ro.build.display.id` equals the tested build** (transaction
numbers change between builds), then every 30 s re-publishes `multi_sim_data_call` whenever the registry
disagrees (airplane mode makes PhoneSwitcher publish -1 again). It gives up after 3 failed sets.
Log: `/data/adb/data_tile_fix.log` (`active data sub -1 -> 1` after each boot).

**Reading compiled code without a decompiler** (jadx download was blocked): a ~100-line Python dex walker that,
per method of a class, lists `invoke-*` targets, field refs and `const-string`s (instruction widths by opcode,
skipping switch/array payloads). That was enough to see the null-check-and-return in `handleClick`.
Script: `dexcalls.py` (kept with the S10 files).

### Tile "slow to turn on" with Wi-Fi off: a 5 s system_server freeze (fixed 2026-11-19)

Symptom: Wi-Fi on, the data tile toggles instantly. Wi-Fi off: **off** is instant, **on** takes 3-5 s before the
tile colours up. Measuring first (don't guess):
- Tapping with `input tap` and timing: tile state changed in ~0.1 s, data connected in ~0.4 s. So telephony was fine.
- The user's own taps with `logcat -c` before: SystemUI's log (`Tile.MobileDataTile ... state=2`) was also fast,
  **but** right before every "on" tap: `W/Looper (system_server): Slow dispatch took 5015ms main ...
  LoadedApk$ReceiverDispatcher` and `Slow delivery took 4426ms android.ui ... GestureDetector`. The tap itself
  sat in a queue for 5 s. The freeze always started right after the "off" (= no network at all, since Wi-Fi was off).
- Stack of the frozen main thread: `kill -3 <system_server pid>` writes `/data/anr/trace_00` (`debuggerd -j` from a
  Magisk shell fails: `Failed to write tombstoned output: Permission denied`, SELinux won't let system_server write
  to the su pipe). The main thread was in
  `RealTimeTokenService$2.onReceive` -> JNI `RealTimeTokenManager::getActiveTokenNumber` (`librtts_client.so`) ->
  `AServiceManager_getService` -> `usleep` loop.
- `RealTimeTokenService` (Samsung, `services.jar`) listens to `CONNECTIVITY_CHANGE`, `BOOT_COMPLETED`, `TIME_SET`;
  when the active network is not connected it asks the HAL
  `vendor.samsung.hardware.security.rtts.ISehRealTimeToken/default` for the token count. The S10 vendor has no such
  HAL (same class of problem as the missing `ISehEngmode`), and libbinder's `getService` polls 5 s before returning
  NULL, on system_server's main thread. SystemServer starts it with a hard-coded check (`"36"` >= 34), no prop to turn
  it off; patching services.jar risks a bootloop.

**Fix: Magisk module `rtts_stub`.** A 6 KB NDK program (`rtts_stub_src/rtts_stub.c`) that `dlopen`s
`libbinder_ndk.so`, defines a class with descriptor `vendor.samsung.hardware.security.rtts.ISehRealTimeToken`,
`AServiceManager_addService(binder, ".../default")`, `ABinderProcess_joinThreadPool()`, and answers every
transaction with `UNKNOWN_TRANSACTION`. The lookup is now instant; the client logs
`getActiveTokenNumber error[0]` after ~1 ms (it failed before too, just 5 s later). No "Slow dispatch" any more.
- `AServiceManager_addService` / `ABinderProcess_*` are not in the NDK stubs: get them with `dlsym`.
- servicemanager only demands a VINTF manifest entry for binders with VINTF stability; a plain local binder can be
  registered under a HAL-style name.
- `sepolicy.rule`: `allow system_server magisk binder { call transfer }`, `allow system_server default_android_service
  service_manager find`, `allow servicemanager magisk binder { call transfer }` (+ dir/file/process getattr so it
  can read the caller). Live testing first with `magiskpolicy --live "<rules>"`.
- Built with NDK 30: `aarch64-linux-android29-clang -O2 -s -o rtts_stub rtts_stub.c -ldl`.
Same freeze hit every full network loss (airplane mode, leaving Wi-Fi with data off), not only the tile.
Also `data_tile_fix` v2: checks every 2 s for 3 min after boot (was: wait 20 s, then every 30 s), because the
PhoneSwitcher -1 bug returns at every boot and the tile ignores taps until the fix lands.

## Galaxy Store: "not compatible with your phone's operating system"

Gallery's Photo Editor, Galaxy AI language packs / voice styles, Samsung Semantic Core all fail. The ROM ships
these as **stubs** (e.g. `/system/priv-app/OfflineLanguageModel_stub/`) or not at all (Photo Editor,
Samsung TTS `com.samsung.SMT`), and the real app comes from Galaxy Store, which matches on **model + API level**.
The public update-check endpoint shows it:
```
https://vas.samsungapps.com/stub/stubUpdateCheck.as?appId=<pkg>&versionCode=0&deviceId=<model>&mcc=..&mnc=..&csc=..&sdkVer=<api>&pd=0
```
| deviceId / sdkVer | Photo Editor answer |
| --- | --- |
| SM-G973F / 37 (this phone) | resultCode 0 "Couldn't find your app which matches ... device, api level" |
| SM-S938B / 37 | resultCode 2, 3.9.09.16 |
| SM-G973F / 31 (real S10 firmware) | resultCode 2, 3.0.26.51 |

The S10 officially ended at Android 12, so "S10 + Android 17" matches nothing. The offline language model is
`resultCode 0 "Application is not approved as stub"` for every model (not delivered through the stub channel),
so the ROM has to bundle the full app. Options: the ROM bundles the apps, or an APK the user gets from a
trusted source is installed with `adb install`.

## Hide the gesture hint (bottom pill)

Settings > Display > Navigation bar has no "Gesture hint" switch. **That is stock Samsung, not a ROM bug:** Samsung removed it in One UI 6.1, and the official way is **Good Lock > NavStar**. Without Good Lock, the setting still works if SystemUI is restarted:
```sh
adb shell settings put global navigation_bar_gesture_hint 0
adb shell su -c 'kill $(pidof com.android.systemui)'   # SystemUI restarts at once and the pill is gone (persists across reboots)
```
- **Don't** switch the overlay by hand (NavStar manages those) to `com.samsung.internal.systemui.navbar.gestural_no_hint` or `sec_gestural_no_hint`:
  on this ROM the system rejects them and falls back to **three-button** (`navigation_mode=0`). Restore with
  `cmd overlay enable-exclusive --category com.android.internal.systemui.navbar.gestural`.
- **Don't** use `pkill -f com.android.systemui` inside `adb shell '...'`: `-f` matches the shell's own command line and kills it.
- Undo: `settings put global navigation_bar_gesture_hint 1`, then restart SystemUI again.
- **The proper way (works):** Galaxy Store > Good Lock > NavStar > **Swipe gestures** (bottom) > turn on **Enable extra gesture settings** (the other options are greyed out unless Buttons is chosen, which is normal). Then Settings > Display > Navigation bar > Swipe gestures > **More options** shows the **Gesture hint** switch.
- (Retired) **On-phone toggle, no PC:** Magisk module `gesture_hint_toggle` with an `action.sh` (Magisk 28+ shows an
  **Action (▶) button** on the module in the Magisk app and runs it as root). The script flips
  `navigation_bar_gesture_hint` 0/1, prints SHOWN/HIDDEN, then `kill $(pidof com.android.systemui)`. It refuses if
  `navigation_mode` is not 2. Tip: after `magisk --install-module`, also copy `action.sh` into
  `/data/adb/modules/<id>/` so the button works before the next reboot. **Removed:** killing SystemUI while Magisk's action screen is open left a black screen (back did nothing; power gave the lock screen). NavStar is the right tool, so don't kill SystemUI from an app UI.

## Screen goes black in sunlight (won't wake until turned face-down)

Cause: **Accidental touch protection** (`settings get system screen_off_pocket` = 1) keeps the screen off while the
proximity sensor reads "covered". Sunlight's infrared blinds the S10's `TMD4910` proximity sensor, so it reads "near".
Notifications and Bluetooth keep working; turning the phone face-down "uncovers" it. Fix: Settings > Display >
Accidental touch protection **off** (`settings put system screen_off_pocket 0`).
Note for the dev: on this port `android.sensor.proximity` (type 8) is a Samsung **"Palm Proximity Sensor"**, and the
real TMD4910 is only `com.samsung.sensor.physical_proximity`.

## Screen suddenly goes almost black while scrolling (HDR videos): fixed

Symptom: while scrolling Instagram/YouTube, the screen dims to near-black for a moment, then recovers. The
brightness slider looks unchanged (with adaptive brightness on, it shows your preference, not the live level).
Not the light sensor (`lux=52` steady). Logcat `DisplayPowerController` shows a switch each time an HDR video is on screen:
```
Animating brightness: target=498(1.95), sdrTarget=125(0.49), reason=automatic[125] -> hdr[498]
BrightnessEvent: brt=0.4896(87.0%)(hdr=1.9526), nits=198.2(hdr=976.3) ... reason=automatic [ hdr ]
```
while the panel reports `HdrCapabilities{mSupportedHdrTypes=[2, 3, 4], mMaxLuminance=400.0 ...}`. The donor's HDR
brightness targets 976 nits on a panel declaring 400, so Android dims the SDR UI to "make room" and everything goes dark.
**Fix (disable HDR output, reversible):**
```sh
adb shell settings put global are_user_disabled_hdr_formats_allowed 0
adb shell cmd display set-user-disabled-hdr-types 1 2 3 4   # DolbyVision, HDR10, HLG, HDR10+
adb shell cmd display get-user-disabled-hdr-types           # -> [1, 2, 3, 4]
# undo: cmd display set-user-disabled-hdr-types   (no args), settings delete global are_user_disabled_hdr_formats_allowed
```
Then fully restart the app. Result: 0 `-> hdr[` switches in 40 s of scrolling reels (several before).
(`settings put global user_disabled_hdr_formats` alone did nothing live. Use the `cmd display` command.)

**That only covers HDR *video*.** Instagram reels kept triggering `-> hdr[501]` (Ultra HDR / extended-brightness
content is not an HDR video type). Also `cmd display set-hdr-conversion-mode 3 -1` (force SDR) does nothing on the
built-in panel. **Real fix: a static RRO overlay** (Magisk module `hdr_dimming_fix`) that makes the HDR nits equal the
SDR nits. The ratio becomes 1x, so there is nothing to dim:
- read the donor map: `cmd overlay lookup android android:array/config_sdrHdrMapSdrNits` (502 values: 1..500, 1500,
  2500) and `...config_sdrHdrMapHdrNits` (199 -> 199.282 ... -> 1500)
- overlay `res/values/arrays.xml`: `<array name="config_sdrHdrMapHdrNits">` = the SDR list
- manifest `<overlay android:targetPackage="android" android:isStatic="true" android:priority="999"/>`
- build with the Android SDK on the PC: `aapt2 compile --dir res -o res.zip`, `aapt2 link -I platforms/android-36/android.jar
  --manifest AndroidManifest.xml -o unsigned.apk res.zip`, `zipalign -p 4`, `apksigner sign` (own keystore via keytool)
- Magisk module path `system/product/overlay/S10HdrFix/S10HdrFix.apk` (preinstalled = may overlay private
  framework resources). Reboot, then check with `cmd overlay lookup ... | sed -n 199p` -> `199.0`.
Confirmed: both reels that went dark stay normal, and `hbmMode=off` while Instagram plays them.
(`cmd overlay fabricate` from the shell says "must be root", and it can't do arrays anyway.)

## Screen goes dark in direct sun (sunlight HBM drives the panel out of range)

What the user saw with the brightness bar open in the sun: the bar **climbs, then the screen suddenly goes dark**;
facing away, the screen lights up again **as the bar comes back down**. So the dark state lines up with the
**top of the range**, the sunlight HBM above 1.0, and not with low brightness.
Numbers: forced `cmd display set-brightness 1.0` gives `/sys/class/backlight/panel/brightness` 25500,
`actual_brightness` 420 (fine). Driver `max_brightness` = 48600. The donor's curve goes to 2450 nits
(`config_screenBrightnessNits`, backlight up to 1275), far past what the S10 driver takes, so in HBM the panel
goes dark instead of brighter. (At lux 1435 behind a privacy guard HBM did not start; in strong sun it does.)
**First attempt was wrong:** lowering `config_autoBrightnessHbmAmbientLux` to 1000 made HBM start *more* often.
**Fix:** HBM effectively off in the same RRO, `<integer name="config_autoBrightnessHbmAmbientLux">100000</integer>`
(-> `mHbmData=HBM{minLux: 100000.0`). Max is then 1.0 (~420 nits): not dazzling, but never black.
**Verified in the sun (2026-11-18):** the bar rises to max and stays readable, with no blackout, and eases down facing away. One UI "Extra brightness" (`secure screen_extra_brightness`) made no difference.

## Bluetooth pairings deleted at every reboot: FIXED (Samsung root check in libbluetooth_jni)

Symptom: after a reboot the headset (dual-connection oraimo) never auto-connects and must be paired again.
`dumpsys bluetooth_manager` shows `Bonded devices: 0`, `bredr_linkkey_known:F`, and the boot log has
`bt_btif_storage: btif_in_fetch_bonded_devices: enc key isn't available. remove device` + `bt_paired_device_deleted`.
What it is not: a startup race (Bluetooth kept off until after PIN unlock still deleted the bond), and not Common
Criteria (`BluetoothKeystoreService: init isCommonCriteriaMode: false`). Also don't touch `security.mdf`
(`vold` reads it). What it is: on **rooted/modified Samsung** builds, the native library stores the LinkKey with its
own encryption (15-char placeholder in bt_config.conf) and refuses to restore it, the known "rooted Samsung loses BT
pairings" problem. **Credit:** the fix comes from the Magisk module [3arthur6/BluetoothLibraryPatcher](https://github.com/3arthur6/BluetoothLibraryPatcher) (`hexpatch.sh` holds the per-API hex patterns), found through the XDA thread [Bluetooth pairing lost after every reboot](https://xdaforums.com/t/bluetooth-pairing-lost-after-every-reboot.3682172/) (post #14 by UniX1803 links the module). The module itself refuses Android 17 (no API 37 entry) and patches the APEX copy, while YuROM uses a vendor bind-mounted copy, so we applied its API 36 pattern by hand.
The patcher has no Android 17 entry, but its **API 36 pattern matches YuROM's lib exactly once**:
```
libbluetooth_jni.so  pre : 00122a0140395f01086b00020054   (cmp w10,w8 ; b.eq)
                     post: 00122a0140395f01086bde030014   (cmp w10,w8 ; b  -> skip the check)
```
YuROM uses `/vendor/lib64/s10-r8/libbluetooth_jni.so`, bind-mounted over the APEX copy by Bluetooth_Fix r2. So patch
that file in place: back it up, change exactly one aligned instruction (3 bytes at 0x7abae8), and write it through a
second vendor mount (`cat >` keeps the inode and the `system_lib_file` label). Bluetooth must be off while writing. Reboot.
Result: pair once, reboot, `Bonded devices: 1`, and no `enc key isn't available` in the log.
Undo: write back `/data/adb/bt_patch_backup/libbluetooth_jni.ORIGINAL.so` (sha 5a1ee6c3…; patched e4180f32…).

## Face recognition, and the 5 s lag before "Confirm PIN"

Face unlock does not work on R34. The dev said on XDA (post #687) that "a major update is coming very soon + Face ID".

**The PIN-screen lag is not face unlock (root cause found):** `ConfirmLockPassword` took **5.66 s** to show
(`performCreate,5056`). In onCreate, Samsung's `EngineeringModeManager` loads `lib.engmodejni.samsung.so`, which
asks for the AIDL HAL `vendor.samsung.hardware.security.engmode.ISehEngmode/default`. It does not exist on the
S10 vendor, so libbinder logs `Waiting for service ...` and gives up after its fixed **5 s** getService poll
(`didn't start. Returning NULL`). It happens on every screen that asks for the PIN first.
How it was measured:
```sh
adb shell 'logcat -d -b main,system,events -v threadtime -t 20000' | grep -E "START u0.*ConfirmLock|wm_on_create_called.*ConfirmLock|Displayed .*ConfirmLock"
# then read that PID's main-thread lines between START and onCreate
```
`am start -W -a android.app.action.CONFIRM_DEVICE_CREDENTIAL` only times the trampoline (70 ms). Look at the real
`ConfirmLockPassword$InternalActivity` launch time instead. No safe local fix: the framework picks the slow JNI path
whenever the library *loads*, and its fallback ("em library is disabled, will use service") needs the library gone,
which risks uncaught UnsatisfiedLinkError crashes. Reported to the dev.

## Firmware APKs: "App not installed" = INSTALL_FAILED_REJECTED_BY_BUILDTYPE (fixed for Photo Editor)

The Photo Editor APK from a mirror site (`samsung-photo-editor-3-8-11-30.apk`) failed with "App not installed". `pm install` from
`/data/local/tmp` (system_server can't read `/sdcard` files: "no access to read file context fuse") gives the real error:
```
Failure [-3001: INSTALL_FAILED_REJECTED_BY_BUILDTYPE]
```
Searching services.jar for the constant -3001 (`const/16 vX, #-3001`) finds a single method:
`com.android.server.asks.ASKSManagerService.verifyASKStokenForPackage`. ASKS is Samsung's app-security check. The APK has
`SEC-INF/buildinfo.xml` with `<APKNAME name="PhotoEditor_AIFull.apk"/>` and `<KEYTYPE name="platform"/>`: it is a firmware
system app signed with Samsung's **platform key**, and ASKS will not let that be installed as a user app.

Check whether the key matches the ROM:
```bash
apksigner verify --print-certs PhotoEditor.apk | grep -m1 SHA-256
adb pull /system/framework/framework-res.apk && apksigner verify --print-certs framework-res.apk | grep -m1 SHA-256
# both 34df0e7a... on YuROM (it is built from Samsung firmware, so it keeps Samsung's platform key)
```
Same key, so the APK works as a **system app** through a Magisk module:
```
photo_editor_system/
  module.prop
  system/app/PhotoEditor_AIFull/PhotoEditor_AIFull.apk
  system/app/PhotoEditor_AIFull/lib/arm64/*.so     # the 72 lib/arm64-v8a libs extracted (extractNativeLibs=true)
```
- `/system/app`, not `priv-app`: `ro.control_privapp_permissions=enforce`, and a priv-app with privileged permissions missing
  from a `privapp-permissions-*.xml` allowlist can stop the phone from booting. Platform-signed apps get signature permissions anyway.
- Check its `uses-library` entries exist (`saiv` -> `/system/etc/permissions/saiv.xml`) and `minSdkVersion` (35).
- Result: Gallery > Edit asks for access once, then the full Photo Editor opens. It has no launcher icon (same on stock).

**priv-app vs app:** if the ROM still ships `/system/etc/permissions/privapp-permissions-<pkg>.xml` for a removed app, put the app
back in `/system/priv-app` (where stock had it). A newer APK may request privileged permissions the old allowlist lacks, and with
`ro.control_privapp_permissions=enforce` that stops boot, so the module also ships `privapp-permissions-<pkg>-module.xml` listing
every permission the APK requests (`aapt2 dump permissions`; listing non-privileged ones is harmless). `make_sysapp_module.py ... priv`
does this. Downloading: APKMirror family page -> highest version -> variant page -> `download/?key=` -> `download.php?id=`, with a
browser User-Agent and the previous page as Referer (curl). Search: `/?post_type=app_release&searchtype=app&s=<name>`. Always check
package name, targetSdk and the 34df0e7a cert: the "Digital Wellbeing" there is an old `com.samsung.android.wellbeing` with another key.

APKMirror also hosts several of these Samsung system apps, signed with the same platform key. APKMirror checks signatures, and
a matching `34df0e7a...` proves the file came from Samsung, since only Samsung holds that key. Done the same way (Magisk, `/system/app`):

| App | Package | Result |
| --- | --- | --- |
| Photo Editor 3.8.11.30 | `com.sec.android.mimage.photoretouching` | works (Gallery > Edit) |
| Samsung TTS 3.5.02.31 (voice styles) | `com.samsung.SMT` (`sharedUserId=android.uid.system`) | loads, becomes default TTS |
| Semantic Core 6.0.50.22 (636 MB) | `com.samsung.android.ssco` | loads |
| Intelligence Voice Services 1.3.10.24 | `com.samsung.android.intellivoiceservice` | **fills Settings > Language packs** (see below) |
| Call Services / CallAssistViewer 18.5.00.154 | `com.samsung.android.callassistant` | **opens Text call / Live translate transcripts** from call history |
| Smart View 8.2.28.6 | `com.samsung.android.smartmirroring` | /system/app; Smart View (cast) tile available |
| Samsung Capture 6.2.12 | `com.samsung.android.app.smartcapture` | /system/app; screen recorder + screenshot toolbar (its `uses-native-library` libs are in the ROM) |
| Quick Share 13.9.10.5 | `com.samsung.android.app.sharelive` | **priv-app** (ROM still has its privapp xml; sharedUserId android.uid.sharelive) |
| Modes and Routines 5.1.03.1 | `com.samsung.android.app.routines` | **priv-app** (sharedUserId android.uid.systemui) |
| Link to Windows Service 3.6.01.2 | `com.samsung.android.mdx` | **priv-app** |
| Language Core 4.1.20.28 (3 GB, on-device AI language model) | `com.samsung.android.offline.languagemodel` | **impossible**: `INSTALL_FAILED_MISSING_SHARED_LIBRARY ... libenn_user_lib.so` |

`libenn_user_lib.so` is the Exynos NPU (AI chip) user library (`<uses-native-library required="true">`). The S10 vendor only has
`libenn_dsp_*`, so the Galaxy AI offline language packs cannot run here. Editing the manifest would break Samsung's signature.
The S10 does have an NPU, but its vendor uses Samsung's older **EDEN** stack (`libeden_rt.so`, `libeden_ud_npu.so`, `libnpuc_*`).
Language Core is built for the newer **ENN** stack. Everything Galaxy AI that sits on Language Core goes with it: the language packs list,
translation voice packs, Live Translate, call screening / Call assist (the call settings show Galaxy Store error **9908**).
**Correction: the empty Settings > Language packs screen was not the NPU.** Logcat while opening it:
```
ScsApi@Feature: dump(), com.samsung.android.intellivoiceservice does not exist
[LanguagePack]LanguagePackManager: LanguagePack ConfigInfo is null.
```
SecSettings builds the list from `com.samsung.android.sdk.scs.ai.asr.Environment.getLangpackConfigInfo()`, which calls the
content provider of **Samsung Intelligence Voice Services** (`get_langpack_config` -> `langpack_config_json`). YuROM leaves that
app out. Installing it (APKMirror, platform key, as `/system/app`) makes the full list appear (Arabic, Chinese, Dutch, English
AU/IN/UK/US, Filipino, ...). Installed TTS voices come from `content://com.samsung.SMT.LanguageProvider` (Samsung TTS lists the
installed `com.samsung.SMT.lang_*` voice packs). The ExtremeROM issue #462 hint ("installed TTS pack from Galaxy Store and they
started to be displayed") pointed this way. The voice pack APKs (e.g. `com.samsung.SMT.lang_en_au_m00`) are signed with a
separate Samsung store key and install as normal apps.
Each language pack is a bundle of Samsung apps fetched through the same stub API as Galaxy Store
(`vas.samsungapps.com/stub/stubUpdateCheck.as`), and the server does serve them to SM-G973F on Android 17:
on-device voice input `com.samsung.android.bixby.ondevice.<xx>`, TTS voices `com.samsung.SMT.lang_<xx>_<variant>`,
translation `com.samsung.android.nmt.apps.t2t.languagepack.en<xx>`. German is about 600 MB. If one download hangs, every later
one waits (`[processDownload] wait for task finish`); `am force-stop com.android.settings` clears the queue. The full list
(115 languages) is cached in the voice service's `shared_prefs/common:language_pack_configuration_1.xml`. Languages with no
packages (Swahili, Luganda/Ganda, Kinyarwanda, Chichewa) are hidden. After this: voice typing works, and Text call can be turned on.
Call screening still asks for Language Core and fails with 9908.

**Text call: the caller hears nothing (ROM bug, audioserver crash).** Log during a Text call:
```
SCallUI: TxAudioSynthesizerProcessor - onStart ... sampleRateInHz: 24000
SCallUI: TxAudio AudioTrackHelper - init, make AudioTrack
F/libc (audioserver): Fatal signal 11 (SIGSEGV) ... fault addr 0x90   Cause: null pointer dereference
  #00 libaudiopolicymanagerdefault.so android::AudioPolicyManager::getOutputForAttrInt(...)
  #04 audioserver android::AudioFlinger::createTrack(...)
```
Same on the earpiece without Bluetooth (5 crashes in one call), so it is not a Bluetooth issue. Every typed reply makes audioserver crash and restart (one tombstone each in `/data/tombstones`), so the reply is never
injected into the call and the call-audio capture for transcription breaks too. The S25-based audio policy code and the
S10 vendor `audio_policy_configuration.xml` (which does declare `incall_music_uplink -> Telephony Tx`) do not fit for this
track. Reported to the dev.

Root cause, traced to the instruction:
- InCallUI plays the reply with `AudioAttributes` usage `VOICE_COMMUNICATION` (2), content `SPEECH` (1) and the
  Samsung tag `VOICE_TX` (found by searching the APK's dex for the log string `init, make AudioTrack`).
- In `libaudiopolicymanagerdefault.so` (llvm-objdump from the NDK) the Samsung branch ORs
  `AUDIO_OUTPUT_FLAG_INCALL_MUSIC` (0x10000) and calls `mAvailableOutputDevices.getDevice(AUDIO_DEVICE_OUT_TELEPHONY_TX)`,
  then reads `[x22,#0x90]` without a null check. That is the `fault addr 0x90`.
- The config audioserver loads (`dumpsys media.audio_policy` -> `Config source: /vendor/etc/audio_policy_configuration_sec.xml`)
  has no Telephony Tx.
- Adding Telephony Tx + an `incall_music_uplink` mixPort with a Magisk overlay makes the policy try it, but the S10
  HAL answers `adev_open_output_stream: requested to open un-supported output ... error(-22)`. It only has
  primary / deep buffer / fast / offload / mmap / aux / multichannel playback streams. So playing audio into the call
  uplink is impossible on the S10's audio HAL. The ROM should hide Text call or null-check that lookup; keep it off.

Stopping the crash (the uplink still can't work). The branch is guarded by a tag compare:
```
7a2b4: bl   strcmp            ; strcmp(attr.tags, "VOICE_TX")   string at 0x202d1
7a2b8: cbnz w0, 0x7a418       ; not VOICE_TX -> normal path
       ...                    ; VOICE_TX -> CallAudioManager / TELEPHONY_TX branch -> null deref
```
Patch one instruction so every track takes the normal path: file offset 0x7a2b8 (vaddr == offset in this lib),
`00 0b 00 35` (cbnz) -> `58 00 00 14` (`b 0x7a418`, imm26 = (0x7a418-0x7a2b8)/4 = 0x58). Shipped as a Magisk module replacing
`/system/lib64/libaudiopolicymanagerdefault.so` with `set_perm ... u:object_r:system_lib_file:s0` in customize.sh.
Why the other side then hears **nothing at all** (not even your real voice): in Text call and Live translate,
Samsung mutes your microphone on purpose, so only the synthesized voice goes up, and that voice can't be sent on the S10.
Live translate settings live in `Settings.System` (keys found as strings in InCallUI's dex):
```bash
adb shell settings put system translate_during_calls_mute_my_voice 0      # the other side hears your real voice
adb shell settings put system translate_during_calls_mute_other_voice 0
adb shell settings put system translate_during_calls_my_language en-US    # what YOU speak
adb shell settings put system translate_during_calls_your_language de-DE  # what THEY speak
```
(Check them first: here "my language" had been left on `de-DE`, so speaking English was recognised as German.) Text call has no
unmute option (by design you don't speak), so use a normal call on the S10.
Better translations: Galaxy AI's "Process data only on device" is `settings put system prevent_online_processing 0|1`
(1 = offline models only). Gemini / Google Translate cannot replace call translation: Android only lets system apps
capture call audio, and the uplink limit stays anyway. Use Google Translate's Conversation mode face to face (it also
has Swahili and Luganda, which Samsung's packs don't).

After the reboot audioserver stays up and no tombstones appear. Tested in calls: Live translate and Text call no longer drop the audio; the translated / typed replies are spoken on the S10's own speaker, and the other side hears only the real voice (the HAL limit). Remove the module before updating the ROM.

Reading tombstones: `adb shell su -c 'grep -m1 "#00 pc" /data/tombstones/tombstone_NN'`.

Finding a missing provider: open the screen, `adb logcat | grep -i -E "LanguagePack|ScsApi"`, then walk the class with a dex lister.

ExtremeROM does not bundle Language Core either. Its framework hook (`unica/mods/knoxpatch`, SamsungPropsHooks) only reports
model SM-S721B to a few Samsung apps (Health, Find My Mobile, Routines, ...).

### Recipe: a Samsung button or screen does nothing (missing system app)

YuROM leaves out several Samsung system apps. The symptom is a button that does nothing or an empty screen. Find the
missing package in logcat while tapping it:
```bash
adb logcat -c   # then tap the button
adb logcat -d | grep -E "ActivityNotFoundException|does not exist|Package not found|ScsApi@Feature"
```
Seen on this ROM:
| Symptom | Log line | Missing package (APKMirror name) |
| --- | --- | --- |
| Settings > Language packs empty | `ScsApi@Feature: dump(), com.samsung.android.intellivoiceservice does not exist` | Samsung Intelligence Voice Services |
| Call history: chat/transcript button does nothing | `ActivityNotFoundException ... com.samsung.android.callassistant/...CallAssistViewerActivity` | Call Services (CallAssistViewer) |
| Gallery > Edit: "App not installed" for Photo Editor | `INSTALL_FAILED_REJECTED_BY_BUILDTYPE` | Samsung Photo Editor |

Then: download the newest **Android 17** build from APKMirror, check `apksigner verify --print-certs` shows the ROM's platform
cert (34df0e7a...), check `<uses-native-library required="true">` entries exist on the phone, wrap it as `/system/app/<Name>`
in a Magisk module (ASKS refuses platform-signed APKs as user apps), reboot. Some activities are protected by a Samsung
permission (CallAssistViewer needs `com.samsung.android.callassistant.permission.TEXT_CALL`), so `am start` from adb reports
"does not exist"; test from the app that normally opens it.

Helper used: a small script that copies the APK to `system/app/<Name>/<Name>.apk`, extracts `lib/arm64-v8a/*.so` to
`system/app/<Name>/lib/arm64/`, writes `module.prop` and zips it for `magisk --install-module`.

## Instagram downloads that WhatsApp status refuses ("Couldn't add")

All failing files were **AV1** (`av01`, one VP9). The one that uploaded was **H.264** (`avc1`). Meta encodes most Reels
in AV1, and the S10 has no AV1 hardware decoder (only the software `c2.android.av1-dav1d.decoder`). Check a file's codec
without ffprobe by reading the `stsd` box (`ftyp/mdat/moov > trak > mdia > minf > stbl > stsd`; the first 4 bytes after its
header give `avc1` / `av01` / `vp09` / `hvc1`). Fix in Piko: **Download options > Video variants > 1280x720** gives H.264 and uploads fine; **Download current media** picks the 1080p AV1 copy. Or set an external downloader (YTDLnis, `com.deniscerri.ytdl`, yt-dlp based), or re-encode (`ffmpeg -i in.mp4 -c:v libx264 -c:a copy out.mp4`).

## Playing audio INTO a phone call on the S10 (Exynos 9820 ABOX): the tunnel and the light

Goal: make Text call / Live translate replies reach the other person. Android's way (an `INCALL_MUSIC` output to
`TELEPHONY_TX`) is impossible here: the exynos9820 HAL refuses it (`requested to open un-supported output`). But the call
itself runs inside Samsung's **ABOX** audio DSP, and ABOX routing is plain ALSA mixer controls (card 0, `ABOX ...`), so the
HAL can be bypassed. On 2026-11-19 a 1 kHz tone was played into a live call and the other phone heard it.

### What failed first, and why (so nobody repeats it)

| Try | Why it failed |
| --- | --- |
| Add `Telephony Tx` + `incall_music_uplink` to `audio_policy_configuration_sec.xml` | HAL: `requested to open un-supported output (-22)` |
| Tone into `ABOX VPCMIN_DAI0_B` / `VPCMIN_DAI1`, even replacing `VPCMIN_DAI0_A` | `VPCMIN` is only a side copy for call recording; the voice kept going through |
| `ABOX NSRC0 = SIFS0/1/2` | only NSRC0 was switched; **NSRC1 still carried the mic**, and nothing was playing into the SIFS |
| Codec (Madera CS47L93) mixer `AIF1TX1 Input 2 = Tone Generator / AIF1RX` | that run never happened (script timed out), not concluded |
| Anything inside Text call | Text call calls `setMicrophoneMuted(true)` -> HAL `voice_set_mic_mute` -> `AudioRil setTxMute`, the **modem mutes the whole uplink** |
| tinyplay / tinycap | the ABOX firmware reports no stream position for WDMA5 (and RDMA8 here) during calls: streams time out after 1 s |
| tinypcm reading a file with `-q` (real time) | it drops what it can't play now, hits end of file in milliseconds and exits |

### The light: gsm2sip

Searching "inject audio into call Exynos ABOX" found **[pulpoff/gsm2sip](https://github.com/pulpoff/gsm2sip)** (an Android
GSM-SIP gateway). Its README section ["The Galaxy S10e: past the HAL, through the ABOX"](https://github.com/pulpoff/gsm2sip#the-galaxy-s10e-past-the-hal-through-the-abox)
and `DeviceProfile.kt` (`exynos9820()`) give the working chain, verified on a live call on the S10e (same SoC):
```
agent -> RDMA8 -> SPUS OUT8 -> SIFS2 (24-bit) -> ABOX NSRC0 + NSRC1 (the voice processor's 2 uplink inputs) -> modem
pacer:  ABOX NSRC4 = SIFS2 -> WDMA5 captured and discarded (a SIFS is only clocked while a capture runs through it)
```
Rules found the hard way (theirs, confirmed here): switch **both** NSRC0 and NSRC1; SIFS2 Width 24 (16 buzzes); play ~10 dB
down; never mute the mic through Android (the HAL mutes the whole uplink); WDMA5/RDMA8 need **`abox_pos.ko`**, a small GPL
kernel module that reports the DMA positions the firmware doesn't (source: `tools/abox-pos/abox_pos.c`).

### Working recipe on this phone (YuROM, ArtisanKRNL 4.14.356-openela)

`abox_pos.ko` from gsm2sip loads on ArtisanKRNL (vermagic `4.14.356-openela-rc1 SMP preempt mod_unload modversions aarch64`,
built without symbol CRCs; it checks the vendor struct layout against the device tree and refuses if it doesn't match):
```bash
insmod /data/local/tmp/abox_pos.ko rdma=0x100 wdma=0x60     # dmesg: "abox_pos: watching 20 DMAs"
# during the call:
tinymix -D 0 set "ABOX SIFS2 Width" 24
tinymix -D 0 set "ABOX SPUS OUT8" SIFS2;  tinymix -D 0 set "ABOX SIFS2" "SPUS OUT8"
tinymix -D 0 set "ABOX NSRC4" SIFS2
touch /data/local/tmp/tone_on
( while [ -e /data/local/tmp/tone_on ]; do cat tone1k_16k_mono.raw; done ) |   tinypcm play -D 0 -c 2 -r 16000 -b 16 -p 160 -m -w 3000 -d 8 -n 3 -P 17 &      # RDMA8 + pacer WDMA5
tinymix -D 0 set "ABOX NSRC0" SIFS2;  tinymix -D 0 set "ABOX NSRC1" SIFS2         # the caller now hears the tone
# undo: NSRC0/1 = UAIF0, rm tone_on, kill tinypcm, NSRC4 = RESERVED, SPUS OUT8 = SIFS0, SIFS2 = "SPUS OUT6", Width 16
```
Tools: `tinymix`/`tinyplay`/`tinycap` built from [tinyalsa v2.0.0](https://github.com/tinyalsa/tinyalsa) with the NDK
(`aarch64-linux-android30-clang -static -Iinclude utils/tinymix.c src/*.c -ldl`); `tinypcm` and `abox_pos.ko` from gsm2sip.
Copies + SHA256 in `D:\OS\S10 custom rom\call_audio\`.

### Working around adb dropping during calls

On this phone adb goes offline (sometimes "unauthorized") the moment a call starts, and adb over Wi-Fi was blocked by the
agent's safety check. So every test ran as a **script on the phone**: armed over adb, it waits for the call
(`tinymix get "ABOX NSRC0 Bridge"` turns On), runs the steps on a timer, restores everything and writes a log that is read
after replugging. Keep only one armed copy (a helper lists/stops them; never `pkill -f`, it matches its own shell).

### Text call working: the final, simpler solution (no kernel module needed)

Getting Text call's own read-aloud into the call took four more rounds:

| Try | Result | Lesson |
| --- | --- | --- |
| RDMA0 (HAL playback) moved to SIFS2 (`SPUS OUT0 = SIFS2`) | RDMA0 restarts 6x in 15 ms and gives up; Text call says "wait for current response to be sent" | a HAL stream won't start on a SIFS with no real output behind it (and SIFS2 was clocked at 16 kHz) |
| also moved RDMA4 | **both phones silent** | RDMA4 (2ch 32 kHz, opened at call start) is the **downlink**; never move it |
| pacer capture alone (`tinypcm cap -d 17`) | `read: stalled: no frames moved` | SIFS2 needs something playing into it to be clocked |
| **uplink inputs read SIFS0** (`NSRC0 = NSRC1 = SIFS0`), playback left alone | **replies heard on the other phone** | SIFS0 is the earpiece mix, always clocked in a call; costs an echo, because SIFS0 also holds the downlink |

Final: switch only while a reply is spoken. `/proc/asound/card0/pcm0p/sub0/status` shows `RUNNING` while RDMA0 plays.
Text call mutes the uplink (`setMicrophoneMuted(true)` -> `voice_set_mic_mute` -> `AudioRil setTxMute`); undo it as root with
`service call audio 46 i32 0 s16 com.android.shell i32 0 s16 ""` (IAudioService.setMicrophoneMute; 43 = isMicrophoneMuted.
The numbers are build-specific: read `TRANSACTION_*` from `framework.jar`'s `IAudioService$Stub` static values, and note that
dex `null`/`boolean` static values carry no payload bytes, or every later number is wrong).

Magisk module `textcall_uplink` (service.sh + a static tinymix), loop during a call every 100 ms:
```
RDMA0 RUNNING and mic muted (= Text call)  -> setMicrophoneMute(false); ABOX NSRC0 = NSRC1 = SIFS0
RDMA0 idle for 0.4 s                        -> ABOX NSRC0 = NSRC1 = UAIF0; setMicrophoneMute(true)
```
Gating on "mic muted" keeps normal calls untouched (a normal call never has the mic muted). Verified on 3 calls: Text call
replies read on the other phone, their speech still transcribed, echo only while a reply plays, normal call unchanged.
`abox_pos.ko` and the RDMA8 pacer are not needed for this; they are only for playing your own audio stream into a call.

**v2: Live translate too.** Live translate plays two voices on RDMA0: the outgoing translation (must go up) and the incoming
translation for you (must not). RDMA0 status can't tell them apart, but the in-call app logs every outgoing utterance
(Live translate reuses the Text call speech engine, so both show as `TextCall@`):
```
SCallUI: TxAudioSynthesizerProcessor(TextCall@...) - setSpeakState, SPEAKING   -> uplink = SIFS0, unmute
SCallUI: TxAudioSynthesizerProcessor(TextCall@...) - onDone                     -> uplink = UAIF0, previous mute back
```
The module's `service.sh` now follows `logcat -v brief -T 1 -s SCallUI:I` (read with `read -t 1` so a 45 s watchdog and a
call-ended check still run). Verified: Live translate (my language en-US, theirs de-DE, "Mute my voice" off) sends your German
translation to the other phone and keeps their translation on yours; Text call unchanged. mksh pitfall: `${var##*Name(}`
inside a `case` arm fails with "no closing quote"; avoid `(` in parameter expansions, and always `sh -n` the script on the phone.

**v3/v4: Bluetooth, and five daemons at once (mistakes worth remembering).**
- In a Bluetooth call the HAL sets the uplink to the headset: `incall_wb-bt-sco-headset-in` -> `ABOX NSRC0 = NSRC1 = UAIF3`
  (16-bit, 16 kHz). v1/v2 restored a hard-coded `UAIF0` (phone mic) after every outgoing voice, so the caller heard the phone,
  not the Oraimo headset. v3 saves the real NSRC0/NSRC1 values first and restores those.
- Feeding SIFS0 (24-bit) into a call that started on UAIF3 (16-bit) buzzes (the same width rule gsm2sip documents), and
  setting SIFS0 to 16-bit while speaking broke the headset audio. v4 skips Bluetooth calls (unless the flag file
  `/data/local/tmp/textcall_uplink.bt` exists for testing), so the headset always works; outgoing translation over BT is open.
- `kill <pid>` on a mksh script that sits in `logcat | while read` does **not** stop it (and leaves the pipeline's subshell),
  so each "restart" added another copy: five daemons toggled mute and routing against each other (mic icon flickering,
  silence, buzz, an emptied log). Fix: `kill -9` every PID (parent, `logcat`, subshell) and verify with
  `ps -A -o PID,PPID,NAME,ARGS | grep service.sh`; v4 also kills older copies of itself at start (single instance).
- **Live translate's listener loop.** With both NSRC0 and NSRC1 on SIFS0, Live translate kept "typing" by itself even with the
  phones in different rooms. The HAL opens a "Voice Call TX" capture on PCM 14 = **WDMA2, fed by NSRC1**; Live translate's
  LOCAL (my speech) recognizer listens there, so it heard its own German voice, translated it and spoke again. A/B test:
  injection off -> no ghosts; NSRC0 only -> no ghosts but the caller hears nothing (the modem uses NSRC1). The modem and
  the LOCAL recognizer share NSRC1, so they can't be separated at the NSRC level. The logs tag recognition as
  `executeSttResults(LOCAL<id>)` / `REMOTE<id>`, and the outgoing voice reuses the LOCAL id (`TextCall@<id>`), so v4 skips
  utterances whose id came from LOCAL (flag `/data/local/tmp/textcall_uplink.lt` re-enables). Result: Text call replies go
  to the caller; Live translate works as on stock S10 (one way) without the loop. Open: an AP echo-canceller route
  (`MCD_TXSE*` with `REFEXT4CH`) might remove the voice from the recognizer's tap.
- **v5: the ghost filter (Live translate works both ways, no loop).** Every recognized phrase has an id, and its spoken
  translation reuses it (`executeSttResults(LOCAL<id>)` ... `TextCall@<id>`). The first log line of a new phrase marks when
  recognition started. A phrase that **starts while the daemon is injecting, or within ~1.5 s after**, is the recognizer
  hearing our own translation, so its id goes into `ghost_ids` and its translation is not injected. Since that ghost
  translation then only plays locally, the recognizer doesn't hear it and the loop dies after one step. Real phrases
  (started while not injecting) are forwarded. Tested: each translation reached the second phone, typing stops when
  you stop talking, log showed 2 ghosts blocked; Text call unchanged. Flags: `.nolt` (don't forward Live translate),
  `.off` (forward nothing), `.bt` (also act in Bluetooth calls, still buzzes), `.mode n0|n1|both` (experiments only).
- **v6: Bluetooth works too.** The earlier BT buzz test was contaminated (five daemons). A clean test with format logging:
  `SIFS0 24bit/48000Hz, UAIF3 16bit/16000Hz, UAIF3 SPK=SIFS2, SIFS2 16bit/16000Hz`. In a Bluetooth call the headset is fed
  from **SIFS2**, not SIFS0, so the read-aloud is in SIFS2 and SIFS0 is an unrelated 24-bit mix (feeding that = buzz).
  v6 reads `ABOX UAIF3 SPK` and injects that mix (SIFS2, already 16-bit like the BT uplink): no buzz, Text call and Live
  translate reach the caller over the Oraimo headset, ghost filter still works. Rule: inject the mix that feeds the active
  output (`ABOX UAIF0/1 SPK` for phone, `UAIF3 SPK` for BT), and it must match the uplink source's width. BT is on by
  default; `.nobt` turns it off. WhatsApp/VoIP calls don't use the CP bridge, so the daemon ignores them.
- **v7: don't re-mute after a Live translate voice.** Live translate mutes the mic only while it speaks and unmutes it
  itself. The daemon saw "muted" at the start, unmuted to send the translation, then re-muted 0.3 s after `onDone`,
  i.e. after Live translate had already unmuted, so the mic stayed muted (noticed when the user went back to the call
  screen). v7 restores the mute only for Text call utterances (mic muted for the whole call), not for LOCAL (Live
  translate) ones.
- **v8 (current): the mic still showed muted after Live translate, because the daemon lagged 5-13 s.** In a Bluetooth
  Live translate call the user typed a few messages, then spoke, then pressed back: the mic was muted. Two causes:
  1. **Lag.** The in-call UI logs hundreds of `SCallUI` lines a second. The daemon read every line in mksh (case match
     plus a `tinymix` call on each loop pass), so it fell 5-13 s behind the log. It re-muted (`AudioRil setTxMute` at
     22:50:49) in the middle of a real Live translate phrase, and the same lag made that phrase look like it started
     "while injecting", so the ghost filter dropped it.
  2. **Typed messages in Live translate are not LOCAL.** They get epoch-timestamp ids (`TextCall@1795...`), so v7 saw
     `lt=0` and restored the mute after them, but Live translate keeps the mic open.
  Fixes: let logcat filter (`logcat -v brief -T 1 -s SCallUI:I -e '<regex>'`, regex
  `TxAudioSynthesizerProcessor.*(SPEAKING|onDone|onError|STOP|IDLE)|executeSttResults.LOCAL[0-9]+. - addSttTextCallMessage, even`),
  so the shell only sees a few lines per utterance; set `lt_session=1` on the first LOCAL phrase and never re-mute for
  the rest of that call; reset `lt_session`, `lt_ids`, `ghost_ids` when the call ends. The single-instance cleanup had to
  match the new logcat command line too (`"logcat -v brief -T 1 -s SCallUI:I"*`). Verified 2026-11-19: Bluetooth and
  phone-earpiece Live translate, typed + spoken, mic not muted after pressing back, Text call still re-mutes as Samsung
  intends. Lesson: a shell daemon that follows logcat must filter in logcat (`-e`, tag filters), never in the shell, or it
  drifts behind and every time-based decision (ghost window, mute restore) goes wrong.
- **Headset connected during a call has no call audio, dialer Bluetooth button greyed out.** `dumpsys bluetooth_manager`
  showed A2DP connecting fine but the HEADSET (HFP) profile timing out after 30 s and the retry failing, after which the
  device's "Calls" switch was off (media only). Turning Calls back on logged `HFS-setConnectionPolicy ..., 100` and HFP
  connected. Not the daemon (no log activity). Connect the headset before the call, or check Bluetooth > device > Calls.
- **Leftover experiment flags.** After the NSRC0-only test, the command that removed `textcall_uplink.mode` sat behind a
  `sh -n ... || exit 1` that failed, so it never ran, and Text call silently stayed on NSRC0 (nothing reached the caller).
  After any experiment, list the flag files (`ls /data/local/tmp/textcall_uplink.*`) and check the log's `mode=` field.
Daemon detail: `ps -A -o PID,PPID,S,NAME` shows its `sleep` child; reading `/proc/*/status` PPid from a quoted `su -c` line
missed it and made a healthy loop look stuck.

## Galaxy Store apps that do work

Good Lock + NavStar install from Galaxy Store, so the store does serve some apps to "S10 + Android 17". Photo Editor, Studio and the language packs it does not. Careful: "compass" apps found by web search are third-party (one had ads and a paywall; `com.ais.compassone.galaxy` is region-locked). **Samsung's compass lives in the Tools edge panel** (Edge panels themselves are in One UI Home here); YuROM leaves out other firmware-only apps by design (INSTALLATION_EN.md: "Quick Share, Routines, Edge panels and similar are not included"). `pm list packages` has no edge/cocktail package at all. ExtremeROM shipped them. They need platform signing, so sideloaded APKs usually fail. Ask the dev to include them.

## Screenshots and the compass panel

- **Quick panel tiles (One UI 9):** SystemUI owns `Settings.Secure sysui_qs_tiles` and writes it back ~10 ms after any
  `settings put` (even as root), dropping what you added. Add tiles with the panel's Edit (pencil) button instead.
  Samsung's full lists are in SystemUI.apk strings `sec_quick_settings_tiles_default` / `sec_quick_settings_tiles_stock`
  (`aapt2 dump resources SystemUI.apk`), with a spec-to-package map (e.g. `AllShareCast` = Smart View
  `com.samsung.android.smartmirroring`, `ScreenRecorder` = `com.samsung.android.app.smartcapture`). **Take screenshot** is the
  built-in `ScreenCapture` tile. YuROM R34 lacks the apps behind Smart View, Screen recorder (Samsung Capture), Quick Share,
  Link to Windows, Modes and Routines and Secure Folder, so those tiles never appear; they can be added as Magisk
  /system/app modules like the other firmware APKs (Secure Folder needs Knox, which root has tripped).

- Screenshot: Power + Volume Down. Gesture: **Palm swipe to capture** (`settings get system surface_palm_swipe` = 1),
  a sweep with the little-finger edge of the hand across the whole screen (it is not a 3-finger swipe). If palm detection
  fails on the port, use Good Lock **RegiStar** (back-tap = screenshot) or **One Hand Operation+**.
- Edge panels are part of One UI Home here (`com.sec.android.app.launcher ... edgepanel`), not a separate app. The
  compass is the **Tools** panel (not Weather/Reminder). **Not present on YuROM R34** (only Apps/Weather/Reminder-type panels are listed). Use Google Maps' compass or an ad-free compass app, and ask the dev to include Tools.
- No 3-finger-swipe screenshot on One UI. Closest built-in options: RegiStar back-tap or One Hand Operation+. Third-party 3-finger apps need Accessibility (full screen access), so pick carefully.

## Side key and Bixby key

- **Long-pressing the side key doesn't show the power menu:** not a bug. One UI's default is the AI assistant
  (`global function_key_config_longpress_selected_item=long_press_ai_agent_app`). Fix: Settings > Advanced features >
  Side button > Press and hold > **Power off menu**. Side key + Volume Down always opens the power menu.
- **The Bixby key does nothing:** YuROM (ported from a phone without one) has no Bixby app or setting, and the key is
  unmapped. `getevent -lt` shows it as `/dev/input/event8 (gpio_keys): EV_KEY 02bf` (scancode 703), and gpio_keys uses
  `/system/usr/keylayout/Generic_internal.kl`, which has no `key 703` line. Fix: Magisk module `bixby_key_assist`
  overlays a copy of that file plus `key 703   ASSIST`, so the key opens the **default assistant**. You need
  Gemini/Google installed and set as the digital assistant. The ROM ships none (`cmd role get-role-holders
  android.app.role.ASSISTANT` is empty, and logcat shows `assistantAvailable = false`). **Verified:** with Gemini installed and set as default, the Bixby key opens Gemini. Tried `VOICE_ASSIST` to open it listening: the Google app logs `No activity found for intent` (it no longer has the old voice-search activity), so nothing opens. Keep `ASSIST`.

## The TWRP the dev links

YuROM's README points to a Google Drive with a `TWRP` folder per model: the **official twrp.me 3.7.0_9-1**
(`.img` + `.img.tar`, README_EN.md, OFFICIAL_DOWNLOAD.url). That is the same TWRP that bootlooped here while the
phone still had ExtremeROM's f2fs data and partitions. XDA users say to flash **vbmeta** with it (we did, and it
still looped). The ExtremeXT TWRP plus the patched installer is what worked.

---

## Troubleshooting table

| Symptom | Cause | Fix |
| --- | --- | --- |
| TWRP splash, then off, looping | official TWRP vs this device | ExtremeROM/ExtremeXT TWRP |
| `compatible recovery BusyBox is required` | no `/sbin/busybox` | patched installer (busybox fallback) |
| `failed to unmount /vendor` + error 1 | `/data` is f2fs | change Data to EXT4 |
| `Unexpected vendor partition size` | ExtremeROM repartition | hotfix size check `-ge`, or stock firmware with `CSC_` |
| `A G973F HWC1-base (revision G) or newer bootloader is required` | old bootloader | flash stock G973FXXSGHWC1 or newer |
| `Source image mismatch` | `/data` not empty | format Data first |
| First boot restarts 5-6 times | first-boot setup | wait |
| Loops after battery died | broken `/data/apex/active` | APEX guard module, or TWRP delete its contents |
| adb shows `unauthorized` during a loop | no prompt before the UI | wait for setup; tap Allow + Always |
| `adb pull /data/...` fails in Git Bash | path conversion | `export MSYS_NO_PATHCONV=1` |
| No call recording, `ro.csc.sales_code` empty | optics/prism never mounted, props unset | vendor fstab entries + `omc_region_fix` module |
| `SemCscFeature: files does not exist from null` | `mdc.system.path` unset | set it to `.../<CSC>/conf/system` |
| `can't create /vendor/etc/...: Read-only file system` after remount | `/vendor/etc` is an overlayfs | write via a second mount of the vendor block device |
| QS Mobile data tile does nothing | telephony publishes active data sub -1 | `data_tile_fix` module (`service call telephony.registry 30 i32 1`) |
| Galaxy Store "not compatible with your phone's operating system" | no store listing for S10 + Android 17 | bundle the app / install an APK |
| Voice styles / language packs empty | stub app, Samsung TTS missing | ROM must bundle the full apps |
| ~5 s pause before every Confirm-PIN screen | missing `ISehEngmode` HAL, 5 s getService poll | ROM fix needed (reported) |
| Screen dims to near-black while scrolling | HDR mode: donor targets 976 nits, panel max 400 | RRO `config_sdrHdrMapHdrNits` = SDR map (module `hdr_dimming_fix`) |
| Headset never auto-reconnects after reboot | Samsung root check in libbluetooth_jni deletes bonds | patch `b.eq`->`b` (BluetoothLibraryPatcher API 36 pattern) |
| Screen goes dark in direct sun (bar high) | sunlight HBM sends values beyond the S10 driver range | RRO `config_autoBrightnessHbmAmbientLux` = 100000 (HBM off) |
| Screen black in sunlight, won't wake | Accidental touch protection + IR-blinded proximity sensor | turn off Accidental touch protection |
| "Process system isn't responding" right after boot | heavy first minutes (load 15+) | tap Wait; only worry if it repeats |

## Lessons

### Don't give up early: what it looked like on this phone

Three times something looked "impossible" or "the dev's problem" and wasn't. What worked each time:

| Looked like | First (wrong) conclusion | What found the real answer | Result |
| --- | --- | --- | --- |
| Settings > Language packs empty | "Galaxy AI needs the newer NPU, impossible on S10" | The user found **ExtremeROM issue #462** (S20 Ultra, same symptom, fixed by "installed TTS pack"). That led to logging the Settings screen: `com.samsung.android.intellivoiceservice does not exist` | Installed the missing app, full list, packs download, voice typing works |
| Text call: caller hears nothing, audioserver crashes | "ROM bug, keep Text call off, report it" | No public report, so traced it: the app's dex (what it plays), the tombstone, then `llvm-objdump` on the crashing library at the exact address. Found the `VOICE_TX` branch and a null `TELEPHONY_TX` lookup | Uplink proven impossible by the HAL's own log (`un-supported output`), and a one-instruction patch stopped the crash, so Live translate stays up |
| Text call replies never reach the caller | "S10 audio HAL can't, impossible" | User asked to search what others know: found **gsm2sip** (same SoC, injects into the ABOX DSP); copied its routing + kernel module, and ran every test as an on-phone script because adb drops during calls | A tone played into a live call reached the other phone; then Text call replies themselves (uplink reads SIFS0 while RDMA0 plays, undo the TX mute) |
| Call history transcript button does nothing | (it looked dead) | Logged the tap: `ActivityNotFoundException ... CallAssistViewerActivity` | Installed Call Services, transcripts open |

How to work a stuck problem on a port ROM (in order):
1. **Reproduce while logging** (`adb logcat -c`, do it, `adb logcat -d`), plus `/data/tombstones` for native crashes.
   Grep for `ActivityNotFoundException`, `does not exist`, `Package not found`, `Fatal signal`, `denied`.
2. **Search the symptom and the exact log line** on XDA, GitHub issues of the ROM *and its parents/siblings*
   (ArtisanROM, ExtremeROM, UN1CA, also archived and closed issues), Samsung Community, Stack Overflow.
3. **Compare with a ROM where it worked:** list what it ships (`git/trees/<branch>?recursive=1` via the GitHub API,
   no clone needed), read its patches and props. Missing system apps are the most common cause on ports.
4. **Read the code at the failure:** dex string search for the log message, then the method's calls and constants.
   For native code, the NDK has `llvm-objdump` / `llvm-readelf`.
5. **Fix what can be fixed,** even if the root cause is hardware: a Magisk module (system app, overlay, patched lib),
   always reversible, with the original kept. Write the proof when something is truly impossible.

- **Read the installer before guessing.** Every error was one `abort` line in `update-binary` /
  `install.sh`. Grep the message, read the check, and the cause is right there.
- **Hash the downloads** against the hashes inside the installer before blaming anything else.
- **With adb in recovery, look instead of asking:** `getprop`, `/proc/mounts`, partition sizes,
  `tune2fs -l`, which processes use a mount.
- **Back up a partition before writing it** (`dd` to a file, pull it to the PC), and verify every
  write with `sha256sum` against the source.
- **A "bootloop" on a fresh ROM's first boot may just be setup.** Check `persist.sys.boot.reason.history`
  before tearing things apart.
- Findings for the YuROM devs are in `D:\OS\S10 custom rom\YUROM_GITHUB_REPORT_NOTES.md`.
- **Reported on XDA ([thread 4805709](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/), pages 35-36; post links are `.../post-<id>`, copied from each post's share icon):** [#682](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90822509) main report (installer, CSC/OMC, data tile, Galaxy Store, first boot); #685 gesture hint (later corrected: NavStar); [#691](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90823381) the 5 s Confirm-PIN lag (ISehEngmode); #693 sunlight black screen + Tools panel/Quick Share/Routines request; #694 request to bundle language packs, Samsung TTS, Photo Editor, Studio, Semantic Core; [#695](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90823502) HDR dimming while scrolling (976 vs 400 nits) + adb workaround. [#696](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90823566) Bluetooth pairings lost at reboot; [#697](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90823583) sunlight darkening (HBM out of range) + fix; [#701](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90823652) Bluetooth FIXED (BluetoothLibraryPatcher API 36 pattern). #702 update to #695 (real HDR fix = RRO overlay) + Bixby key unmapped / no assistant + fix. [#707](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90825076) (2026-11-19): missing Samsung apps (Language packs = Intelligence Voice Services, call transcripts = Call Services, TTS), the VOICE_TX audioserver crash patch, and Text call + Live translate reaching the caller through the ABOX uplink (credit gsm2sip). [#709](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90825159): reply to 67IJUSTBIPED (#708, "app not installed"): ASKS -3001 + how to install as a Magisk /system/app module, and a correction to #707 (Live translate stays one-way because of the listener loop). [#711](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90825186): how to build a Magisk /system/app module (by hand and with the Python script), for 67IJUSTBIPED (#710). [#712](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90825193): update to #709, Live translate works both ways (ghost filter); Bluetooth still one-way. [#715](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90825222): reply to DODO51 (in French): missing Samsung apps for voice input / call assist, accidental touch protection. [#716](https://xdaforums.com/t/rom-port-yurom-ultra-v35-one-ui-9-android-17-galaxy-s10e-s10-s10-exynos.4805709/post-90825281) (2026-11-19, `github_issues/xda_post_18_data_tile_freeze_rtts.txt`): the RealTimeTokenService 5 s freeze + `rtts_stub`, data_tile_fix v2, textcall_uplink v8. #714: update, Text call and Live translate also work over a Bluetooth headset (inject the headset mix SIFS2). #713 (DODO51, S10+ G975F): 13 issues incl. Galaxy AI call/message assist, voice input in Settings search, accidental touch protection, auto brightness, double tap to wake, off-charging screen. Other users: #698-#700 (67IJUSTBIPED) battery drain, min CPU stuck at 502 MHz. Another user (#684) confirmed the store problem and reported weak signal with greyed data arrows outdoors (not yet checked here).
- **Repo lineage:** `ArtisanROM/ArtisanROM` (the original, multi-device, UN1CA-based, Issues on) -> `S10Portt/ArtisanROM` (the S10 port, branch `port/exynos9820-s10`) -> `yuefdev/YuROM-Ultra` (YuROM releases). GitHub disables Issues on forks by default, so YuROM bugs go to XDA. Only take a bug upstream to ArtisanROM if it is in shared code, not the S10 port.
- **Writing XDA posts (checklist, learned the hard way):** one BBCode post with a `[SPOILER]` per finding; **every name in the
  credits gets a link** (the repo, the issue `#462`, the tool, and the earlier post `.../post-<id>` from the post's share icon);
  every `#NNN` reference is a link (never invent a post number); hand over the whole post as one copy-paste block (it gets
  pasted over the old one in Edit) and keep a copy in `github_issues/xda_post_*.txt`; after posting, record the post link here.
  Example of a correct credits line (#707): `Thanks to pulpoff ([URL=https://github.com/pulpoff/gsm2sip]gsm2sip[/URL]), the
  ExtremeROM team ([URL=https://github.com/ExtremeXT/ExtremeROM/issues/462]#462[/URL]) and 3arthur6
  ([URL=https://github.com/3arthur6/BluetoothLibraryPatcher]BluetoothLibraryPatcher[/URL], see my [URL=.../post-90823652]earlier post #701[/URL]).`
- **YuROM's GitHub has Issues turned off** (no Issues tab, `/issues/new` gives 404), so bug reports go to the XDA thread. Make one BBCode post with a `[SPOILER]` per finding (`github_issues/xda_post_bbcode.txt`); pre-filled `/issues/new?title=..&body=..` links only work where Issues are on.
