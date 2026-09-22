# Quest 1 (monterey) — OrangeFox recovery + true 90Hz

One `boot.img` that boots stock Android normally and OrangeFox recovery
on **Volume Up + Power**, plus a flashable zip that unlocks genuine 90Hz
(not a Magisk-module workaround — patches the real system partition
directly, keeps SELinux **enforcing**, WiFi works normally).

## Downloads

- **boot.img** (flash with `fastboot`): https://github.com/TheCez/quest1-90hz-orangefox/releases/download/v1.0/quest1_90hz_boot.img
- **True 90Hz zip** (flash from recovery): https://github.com/TheCez/quest1-90hz-orangefox/releases/download/v1.0/quest1_true90hz.zip
- **Decrypt fix zip** (flash from recovery, see Part 2): https://github.com/TheCez/quest1-90hz-orangefox/releases/download/v1.0/disable_fbe_fstab_monterey.zip
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
- **Figure out which slot you actually use day to day before you start:**
  ```
  fastboot getvar current-slot
  ```
  For most people that's `b`, but not everyone — some setups run daily on
  `a`. Whatever `current-slot` shows you *right now, before touching
  anything*, is your **active/daily-use slot**. The other one is your
  **inactive slot**. Part 1 below always targets the inactive slot first,
  no matter which letter that turns out to be for you.
- **Nothing on your active/daily-use slot is touched until Part 2.**

---

## Using OrangeFox with hardware buttons (no touchscreen)

Quest 1 has no touchscreen, so this build has a hardware-button
navigation mode built in.

**Activate/toggle it:** hold **Volume Up + Volume Down together for ~3
seconds**. A highlight box appears around the focused element — that
means it's on.

**Once active:**
- **Volume Up / Volume Down** → move focus between buttons/items
- **Power** → select / activate the focused item

Hold **Volume Up + Volume Down** again for ~3 seconds to toggle it back
off.

**Sliders (swipe-to-confirm, e.g. installing a zip or wiping data):**
- Focus the slider — **red** boundary = focused, not active yet.
- Press **Power** — boundary turns **green** = slider is now active.
- Press **Volume Up** 3 times — swipes it across (confirms the action).
- Press **Power** again to release it.

---

## Part 1 — Test on your inactive slot first (zero risk to daily use)

The inactive slot isn't what you boot day to day — testing here can't
break normal use even if something goes wrong. Substitute `a`/`b` below
for whichever one is actually *not* your `current-slot` from above.

```
adb reboot bootloader
fastboot getvar current-slot                # confirms which is active
fastboot --set-active=<inactive slot letter>
fastboot flash boot_<inactive slot letter> quest1_90hz_boot.img
```

**Test recovery:** power off completely, power on holding **Volume Up +
Power**. Should land in OrangeFox. ❌ If not — stop, don't touch your
active slot.

**Test normal boot:** power off, power on normally (no buttons). Meta
logo should appear and at least blink, then continue into stock Android.
❌ If it hangs at the logo — stop, don't touch your active slot.

✅ Only continue if both worked.

---

## Part 2 — Install on your active/daily-use slot

```
adb reboot bootloader
fastboot --set-active=<active slot letter>
fastboot getvar current-slot        # confirm it matches your active slot
```

**Back up your current boot image on that slot first:**
```
adb reboot
adb root
adb shell dd if=/dev/block/bootdevice/by-name/boot_<active slot letter> of=/sdcard/boot_backup.img
adb pull /sdcard/boot_backup.img
```
(If `adb root` refuses — "cannot run as root in production builds" — boot
into recovery instead, Volume Up + Power, and run the same `dd` command
there.)

**Flash and reboot:**
```
adb reboot bootloader
fastboot getvar current-slot        # confirm it's still your active slot
fastboot flash boot_<active slot letter> quest1_90hz_boot.img
```

Boot into recovery (Volume Up + Power), then push and install the zips:
```
adb push quest1_true90hz.zip /sdcard/
adb push disable_fbe_fstab_monterey.zip /sdcard/
adb push Magisk-v30.7.zip /sdcard/          # optional, only if you want root
```

In OrangeFox, in this order:
1. **Install** → `quest1_true90hz.zip` → swipe to confirm.
2. **Install** → `disable_fbe_fstab_monterey.zip` → swipe to confirm.
   **Required** — without this, stock Android's own encryption setup
   (FBE) re-encrypts `/data` on its own the next time it boots, which is
   not what you want. This zip patches the real system partition's fstab
   so it doesn't.
3. **Wipe → Format Data / Factory Reset** — this actually removes the
   existing encryption footer from `/data` so the fstab patch above
   sticks. **This erases everything currently on `/data` (apps, saved
   data) on this slot** — back up anything you care about first if this
   isn't a fresh setup.
4. Optionally **Install** → `Magisk-v30.7.zip` → swipe to confirm.

Reboot to system (plain power-on, no buttons). Done.

## Verify it worked

```
adb shell "logcat -d | grep -oE 'FPS=[0-9]+/[0-9]+'"     # expect n/90
adb shell "cat /sys/fs/selinux/enforce"                   # expect 1 (still enforcing)
adb shell "ip addr show wlan0"                            # expect a real inet address
adb shell "su -c id"                                       # if you installed Magisk: uid=0(root)
```

## If something goes wrong

- **Inactive-slot test failed:** your active/daily-use slot was never
  touched — headset is completely unaffected.
- **Something breaks on your active slot:** restore your backup:
  ```
  adb reboot bootloader
  fastboot getvar current-slot   # confirm it's your active slot
  fastboot flash boot_<active slot letter> boot_backup.img
  fastboot reboot
  ```

## What's actually in the 90Hz zip

Byte-patches the composer HAL, VrDriver.apk, and xrspd helper (unlocks
the 90Hz rate), sets `device_props.json`, adds one missing SELinux allow
rule so WiFi doesn't break under enforcing mode, and installs an
automatic boot-time service that resyncs the panel's pixel clock so
there's no tearing/slicing — no manual "sleep and wake" needed.

## What's actually in the decrypt fix zip

Removes the `fileencryption=ice` flag from the real `fstab.monterey` on
whichever slot's system partition is currently active, so vold stops
setting up file-based encryption on `/data` on that slot. It only edits
the fstab — it doesn't touch `/data` itself, which is why Part 2 also has
you run a Factory Reset: that's what actually strips the existing
encryption footer so the patched fstab can take effect cleanly on the
next boot. Needs to be flashed separately on each slot you use it on,
since `/system` (and its fstab) is per-slot on this device.

---

⚠️ **Do this at your own risk.** Unlocking, flashing, and modifying
system partitions can brick your device if a step is skipped or done out
of order. Backups exist for a reason — use them. Not responsible for
bricked headsets.
