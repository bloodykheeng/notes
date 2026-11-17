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
| "Process system isn't responding" right after boot | heavy first minutes (load 15+) | tap Wait; only worry if it repeats |

## Lessons

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
- **YuROM's GitHub has Issues turned off** (no Issues tab, `/issues/new` gives 404), so bug reports go to the XDA thread. Make one BBCode post with a `[SPOILER]` per finding (`github_issues/xda_post_bbcode.txt`); pre-filled `/issues/new?title=..&body=..` links only work where Issues are on.
