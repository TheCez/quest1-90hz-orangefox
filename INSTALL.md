# Quest 1 (monterey) — OrangeFox recovery + true 90Hz

One `boot.img` that boots stock Android normally and OrangeFox recovery
on **Volume Up + Power**, plus a flashable zip that unlocks genuine 90Hz
(not a Magisk-module workaround — patches the real system partition
directly, keeps SELinux **enforcing**, WiFi works normally).

## Downloads

- **boot.img** (flash with `fastboot`): https://github.com/TheCez/quest1-90hz-orangefox/releases/download/v1.0/quest1_90hz_boot.img
- **True 90Hz zip** (flash from recovery): https://github.com/TheCez/quest1-90hz-orangefox/releases/download/v1.0/quest1_true90hz.zip
- **Magisk v30.7** (optional, for root): https://github.com/TheCez/quest1-90hz-orangefox/releases/download/v1.0/Magisk-v30.7.zip

All releases: https://github.com/TheCez/quest1-90hz-orangefox/releases

## Requirements

- Bootloader already unlocked.
- `fastboot`/`adb` working on your PC, headset connected via USB.
- **Check your firmware version first:**
  ```
  adb shell getprop ro.build.version.incremental
  ```
  Must read `49845030443200410`. On a different version, the 90Hz zip's
  own safety checks make it abort cleanly without changing anything — not
  dangerous, just won't apply until patched for that build.
- **Nothing on slot B (your daily-use slot) is touched until Part 2.**

---

## Part 1 — Test on slot A first (zero risk to daily use)

Slot A isn't your daily-use slot — testing here can't break normal use
even if something goes wrong.

```
adb reboot bootloader
fastboot getvar current-slot
fastboot --set-active=a
fastboot flash boot_a quest1_90hz_boot.img
```

**Test recovery:** power off completely, power on holding **Volume Up +
Power**. Should land in OrangeFox. ❌ If not — stop, don't touch slot B.

**Test normal boot:** power off, power on normally (no buttons). Meta
logo should appear and at least blink, then continue into stock Android.
❌ If it hangs at the logo — stop, don't touch slot B.

✅ Only continue if both worked.

---

## Part 2 — Install on slot B (your daily-use slot)

```
adb reboot bootloader
fastboot --set-active=b
fastboot getvar current-slot        # confirm it says b
```

**Back up your current boot_b first:**
```
adb reboot
adb root
adb shell dd if=/dev/block/bootdevice/by-name/boot_b of=/sdcard/boot_b_backup.img
adb pull /sdcard/boot_b_backup.img
```
(If `adb root` refuses — "cannot run as root in production builds" — boot
into recovery instead, Volume Up + Power, and run the same `dd` command
there.)

**Flash and reboot:**
```
adb reboot bootloader
fastboot getvar current-slot        # confirm it still says b
fastboot flash boot_b quest1_90hz_boot.img
```

Boot into recovery (Volume Up + Power), then push and install the zips:
```
adb push quest1_true90hz.zip /sdcard/
adb push Magisk-v30.7.zip /sdcard/          # optional, only if you want root
```

In OrangeFox: **Install** → `quest1_true90hz.zip` → swipe to confirm.
Optionally **Install** again → `Magisk-v30.7.zip` → swipe to confirm.

Reboot to system (plain power-on, no buttons). Done.

## Verify it worked

```
adb shell "logcat -d | grep -oE 'FPS=[0-9]+/[0-9]+'"     # expect n/90
adb shell "cat /sys/fs/selinux/enforce"                   # expect 1 (still enforcing)
adb shell "ip addr show wlan0"                            # expect a real inet address
adb shell "su -c id"                                       # if you installed Magisk: uid=0(root)
```

## If something goes wrong

- **Slot A test failed:** your daily-use slot (B) was never touched —
  headset is completely unaffected.
- **Something breaks on slot B:** restore your backup:
  ```
  adb reboot bootloader
  fastboot getvar current-slot   # confirm it says b
  fastboot flash boot_b boot_b_backup.img
  fastboot reboot
  ```

## What's actually in the 90Hz zip

Byte-patches the composer HAL, VrDriver.apk, and xrspd helper (unlocks
the 90Hz rate), sets `device_props.json`, adds one missing SELinux allow
rule so WiFi doesn't break under enforcing mode, and installs an
automatic boot-time service that resyncs the panel's pixel clock so
there's no tearing/slicing — no manual "sleep and wake" needed.
