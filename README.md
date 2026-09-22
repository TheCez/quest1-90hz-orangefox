# Quest 1 (monterey) OrangeFox recovery + true 90Hz

A single `boot.img` for the original Meta/Oculus Quest 1 that boots stock
Android normally and OrangeFox recovery on Volume Up + Power, plus a
flashable zip that unlocks genuine 90Hz on the display (not just a
Magisk-module workaround) while keeping SELinux enforcing.

See [INSTALL.md](INSTALL.md) for full instructions and download links.

Download the files from the [latest release](../../releases/latest).

## Source

These prebuilt images are built from:
- [quest1-monterey-kernel](https://github.com/TheCez/quest1-monterey-kernel) — kernel patches (dual-boot handoff, true 90Hz panel timing)
- [quest1-monterey-devicetree](https://github.com/TheCez/quest1-monterey-devicetree) — OrangeFox device tree + source patches (VR stereo fix, hardware-button nav, decrypt fix, dual-boot handoff)
