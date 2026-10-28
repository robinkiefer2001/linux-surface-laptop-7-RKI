# Linux on the Surface Laptop 7 (ARM)

Community notes for getting Linux working on the Surface Laptop 7 (ARM).

**Disclaimer:** Surface Laptop 7 Linux support is highly experimental. Many of these fixes are quite hacky and may cause unpredictable behavior. Use at your own risk.

## What's working

| **Feature**               | **Working?** | **Notes**                                                                                                                                                  |
|---------------------------|:------------:|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| NVMe                      |       ✅      |                                                                                                                                                            |
| Graphics                  |       ✅      | Requires firmware                                                                                          |
| Backlight                 |       ✅      |                                                                                                                                                    |
| USB                       |   ✅  | USB-A and USB-C ports are working. Have not tested USB over Surface Connector              |
| USB-C display output      |       ✅      |        [#11](https://github.com/bryce-hoehn/linux-surface-laptop-7/issues/11)                                                                                                                                                    |
| Wi-Fi                     |       ✅      | Requires firmware and either the ELLX prebuilt kernel or the manual rfkill patch below        |
| Bluetooth                 |       ✅      |  Requires firmware [#6](https://github.com/giantdwarf17/linux-surface-laptop-7/issues/6)                                                                                                                                                          |
| Audio                     |       ✅      |  [#2](https://github.com/giantdwarf17/linux-surface-laptop-7/issues/2) |
| Touchscreen               |   ⚠️ Fixed (experimental)   | 15-inch only, ELLX `7.3.0-rc3+`: touch and multi-finger gestures work with the [device-tree overlay](patches/001-touch/fix/romulus15-touchscreen.dtso) — see [Touchscreen](#touchscreen-15-inch-laptop-7-only-experimental) / `./surface.sh touch-test`. Pen untested. [#13](https://github.com/bryce-hoehn/linux-surface-laptop-7/issues/13) |
| Touchpad               |       ✅     |     Works with the ELLX `iptsd` package - custom calibration may be required, see below.             |
| Keyboard             |       ✅      |                                                                                        |
| Lid switch/suspend        |       ✅      | https://github.com/giantdwarf17/linux-surface-laptop-7/issues/7#issuecomment-2750000739                                                                                                                                      |
| Camera |       ✅      |    Requires device tree patches [0004](patches/outgoing/ov02c10/0004-OV02C10-camera-device-tree.patch) & [0005](patches/outgoing/ov02c10/0005-OV02C10-camera-metadata.patch) - [#4](https://github.com/giantdwarf17/linux-surface-laptop-7/issues/4) — see [config/OV02C10-camera](config/OV02C10-camera/) for libcamera tuning |
| RTC |  ✅  | https://github.com/giantdwarf17/linux-surface-laptop-7/issues/8 |

> [!WARNING]
> Without installing the firmware, many hardware components will be broken!

## Quick Start

Tested working on the [Ubuntu concept image](https://people.canonical.com/~platform/images/ubuntu-concept/) for Ubuntu 26.04. The easiest setup right now is the community-maintained ELLX prebuilts for the Surface Laptop 7 X1 Elite at <https://public.hgci.org/software/ELLX/>. They are built for Ubuntu 26.04 and should also work on Debian.

These prebuilts currently cover:
* Wi-Fi
* Bluetooth
* Trackpad
* GPU

> [!IMPORTANT]
> Keep your Windows install if you can. If the trackpad gets into a bad state, the easiest known reset right now is to reboot into Windows and then back into Linux.

Steps:
* Allocate disk partition space for dual booting (**highly recommended** with bleeding-edge unsupported Linux).
* Install [Ventoy](https://www.ventoy.net/en/download.html) to a USB and drag and drop the ISO afterwards. Ventoy is required to enable keyboard support in GRUB.
* Reboot, select the ISO with Ventoy, and install Ubuntu.
* Download the latest matching kernel packages from `https://public.hgci.org/software/ELLX/kernels/`. At the time of writing these are:
  * `kernels/7.0.0-rc4-11/linux-headers-7.0.0-rc4+_7.0.0~rc4-gc0ce08f6e526-42_arm64.deb`
  * `kernels/7.0.0-rc4-11/linux-image-7.0.0-rc4+_7.0.0~rc4-gc0ce08f6e526-42_arm64.deb`
* Install the kernel packages:

```bash
sudo apt install ./*.deb
```

* Install the firmware from `surface-laptop-7-firmware/proprietary-firmware.tar.gz`:

```bash
sudo tar -xf proprietary-firmware.tar.gz -C /
```

* Install `iptsd` from [iptsd/alex-lentz_iptsd/iptsd_3.1.0-1_arm64.deb](https://public.hgci.org/software/ELLX/iptsd/alex-lentz_iptsd/) and reboot.
* Calibrate the trackpad: 
```bash
# Take note of your hidraw# - most commonly hidraw1
sudo ./etc/scripts/iptsd-foreach -t touchpad -- echo {}
# Calibrate using the hidraw# from the previous command
sudo iptsd-calibrate /dev/<YOUR_HIDRAW#>
# Track your finger _everywhere_ around the trackpad. Ctrl+C -> configuration should be saved.
```
* Lastly, if you notice any problems recovering the trackpad or graphics after resuming from sleep, check out these useful fixes [here](https://public.hgci.org/software/ELLX/fixes/)

Sources:
* ELLX kernel source: <https://github.com/ProgrammerIn-wonderland/ELLX-Kernel>
* ELLX prebuilts / fixes: <https://public.hgci.org/software/ELLX>
* `iptsd` source: <https://github.com/alex-lentz/iptsd>

> [!NOTE]
> Trackpad support is much better with this setup, but suspend can still leave it in a broken state.

## Helper Script (`surface.sh`)

All scripts, patches, and tests in this repository can be run from one entry
point in the repository root. The individual scripts it calls are kept in
[scripts/](scripts) and in each patch folder under [patches/](patches):

```bash
./surface.sh help
```

| Command | What it does | Script it runs |
| --- | --- | --- |
| `wifi-board` | Repack `board-2.bin` for WiFi | [scripts/fix-board-2-wifi.sh](scripts/fix-board-2-wifi.sh) |
| `firmware` | Extract DSP/GPU firmware from the Microsoft MSI | [scripts/romulus-firmware-extract.sh](scripts/romulus-firmware-extract.sh) |
| `touch-status` | Show what touchscreen parts are installed | built in |
| `touch-check` | Build and validate the touchscreen overlay offline | both touchscreen scripts, `--check` |
| `touch-test` | Add a separate EXPERIMENTAL GRUB entry | [test-touchscreen.sh](patches/001-touch/test/test-touchscreen.sh) `--install` |
| `touch-reset` | Delete the test GRUB entry and test DTB | [test-touchscreen.sh](patches/001-touch/test/test-touchscreen.sh) `--reset` |
| `touch-install` | Apply the touchscreen fix to normal boot | [install-touchscreen.sh](patches/001-touch/fix/install-touchscreen.sh) `--install` |
| `touch-restore` | Restore the original normal-boot DTB | [install-touchscreen.sh](patches/001-touch/fix/install-touchscreen.sh) `--restore` |
| `touch-remove-all` | Remove everything touchscreen-related | `--restore` (if patched), then `--reset` |

Commands that change the system ask for `sudo`. None of them reboot automatically.
The individual scripts can still be run directly.

## Manual Wifi / Kernel Setup

If you want to build the kernel yourself instead of using the ELLX prebuilts, the older manual notes are below.

* Download and automatically repack board-2.bin by running [fix-board-2-wifi.sh](scripts/fix-board-2-wifi.sh)

```bash
./surface.sh wifi-board
```

* Build kernel and install [0001-wifi-rfkill-hack.patch](https://github.com/bryce-hoehn/linux-surface-laptop-7/blob/main/patches/0001-wifi-rfkill-hack.patch)
```bash
# Install dependencies
sudo apt install git build-essential fakeroot devscripts equivs flex bison bc libssl-dev libelf-dev dwarves dpkg-dev debhelper rsync kmod cpio fakeroot

# Clone most recent concept kernel
git clone git.launchpad.net/~ubuntu-concept/ubuntu/+source/linux/+git/resolute -b qcom-x1e-7.0
patch -p1 < 0001-wifi-rfkill-hack.patch

# Build & install
fakeroot .debian/rules binary-qcom.x1e
sudo apt install ./*.deb 
sudo reboot
```

## Tips & Tricks

### Touchscreen (15-inch Laptop 7 only, experimental)

> [!WARNING]
> **Status: fixed, but experimental.** The touchscreen works with a
> device-tree overlay on the 15-inch Laptop 7 (Romulus15) with the ELLX
> `7.3.0-rc3+` kernel. It is not upstream, has only been tested on one
> machine, and may not match every board/SKU. Test it with the separate GRUB
> entry first and keep recovery media available.

| | |
| --- | --- |
| Works | Finger touch, multi-finger gestures (tested 2026-10-28 on one 15-inch Laptop 7) |
| Not working / untested | Rnote finger drawing (needs an app setting, see below), active pen, suspend/resume, repeat-boot reliability, 13-inch model, other kernels |
| Fix | [patches/001-touch/fix/romulus15-touchscreen.dtso](patches/001-touch/fix/romulus15-touchscreen.dtso) + [install-touchscreen.sh](patches/001-touch/fix/install-touchscreen.sh) |
| Test | [patches/001-touch/test/test-touchscreen.sh](patches/001-touch/test/test-touchscreen.sh) |
| Tracking issue | [#13](https://github.com/bryce-hoehn/linux-surface-laptop-7/issues/13) |

If you test it, please report in the tracking issue with your model/SKU and
the output of `grep -H . /sys/bus/hid/devices/*/uevent` while booted with the
fix, so the result can be confirmed on more machines.

**Quick way: try it once in GRUB, then keep it or remove all of it.**
Run from the repository root:

```bash
./surface.sh touch-check        # offline build + validation, changes nothing
./surface.sh touch-test         # adds a separate EXPERIMENTAL GRUB entry
# Reboot and pick "EXPERIMENTAL: Laptop 7 touchscreen" in GRUB.
# Your normal entry is unchanged and still boots without the patch.
```

Then choose one:

```bash
# It worked: make it the normal default boot, then delete the test entry.
./surface.sh touch-install      # type APPLY
./surface.sh touch-reset

# It did not work, or you want to remove everything again:
./surface.sh touch-remove-all   # restores the original DTB and deletes the test entry
./surface.sh touch-status       # confirm: "original", no test entry, no test DTB
```

`touch-remove-all` is safe to run at any stage. It restores the original
normal-boot DTB if the fix was installed, deletes the test GRUB entry
(including its 15-second menu timeout) and the test DTB, and leaves the
original backup file in place. The sections below describe each step in detail.

For finger drawing in Rnote, enable **Draw With Touch Input** in its canvas
menu; upstream defaults this option to off. This is separate from kernel
touchscreen support and does not establish active-pen support.

The overlay is
[romulus15-touchscreen.dtso](patches/001-touch/fix/romulus15-touchscreen.dtso).
The permanent fix lives in [patches/001-touch/fix](patches/001-touch/fix); the
separate GRUB test tooling lives in [patches/001-touch/test](patches/001-touch/test).
It targets the ELLX `7.3.0-rc3+` kernel built from `557ed1f34df3`, whose
QSPI controller and HID-over-SPI drivers already operate the touchpad.
It is not a generic mainline overlay.

The published [Romulus ACPI tables](https://github.com/aarch64-laptops/build/tree/master/misc/microsoft-surface-romulus13_and_15)
describe a `GTCH` SPI touchscreen variant on `SP11` (`spi10`, address
`0xa88000`), with interrupt GPIO 51, reset GPIO 48, and power-enable GPIO 64.
The overlay enables that controller and its GPI DMA provider, adds the quad
data pinmux, and describes the touchscreen separately from the touchpad.
The quad data pins and QSPI transaction parameters follow the matching
kernel's pinctrl definitions and existing touchpad implementation; they
were confirmed only by the touch test reported above.

**How it was found:** Linux boots this machine with a device tree, so its own
ACPI tables could not be read (`/dev/mem` is locked down). The values above come
from the published tables, and the approach follows the Surface Pro 11
touchscreen work, which uses the same HID-over-SPI/QSPI design (see
References below).

**Limitations:** the published tables select different touchscreen variants
by board/SKU. Wiring has not been verified against this machine's own tables.
The installed HID driver also uses touchpad-specific reset/power sequencing
and forces a 20 MHz bus clock. Its suitability for the touchscreen, including
reset timing and report processing, remains unverified. No supply voltage is
specified because the ACPI GPIO power control does not establish one.

To compile and merge a candidate **without installing or applying it**, run
from the repository root on the matching 15-inch system:

```bash
mkdir -p /tmp/romulus15-touchscreen-test
dtc -@ -I dts -O dtb \
  -o /tmp/romulus15-touchscreen-test/touchscreen.dtbo \
  patches/001-touch/fix/romulus15-touchscreen.dtso
fdtoverlay \
  -i /boot/dtbs/7.3.0-rc3+/qcom/x1e80100-microsoft-romulus15.dtb \
  -o /tmp/romulus15-touchscreen-test/romulus15-touchscreen.dtb \
  /tmp/romulus15-touchscreen-test/touchscreen.dtbo
fdtget /tmp/romulus15-touchscreen-test/romulus15-touchscreen.dtb \
  /soc@0/geniqup@ac0000/spi@a88000 status
```

The last command should print `okay`; this verifies the merged description,
**not hardware operation**. Do not copy it over `/boot/dtb` by hand or apply
it live; use the GRUB test entry below first, then `install-touchscreen.sh`
(which keeps a backup) to make it the default. A successful test enumerates a
new touchscreen HID device (not just the `045E:0C77` touchpad), produces real
screen-touch input, and leaves the keyboard and touchpad functional.

#### Preparing a selectable GRUB test entry

[test-touchscreen.sh](patches/001-touch/test/test-touchscreen.sh) automates the offline build and
adds a separate, manually selected test entry for this exact kernel and
15-inch model. Run from the repository root:

```bash
bash patches/001-touch/test/test-touchscreen.sh --check
sudo bash patches/001-touch/test/test-touchscreen.sh --install
```

Installation requires typing `INSTALL` after the risk warning. It creates
`/boot/romulus15-touchscreen-test.dtb` and `/boot/grub/custom.cfg`, shows the
GRUB menu for 15 seconds, and leaves the normal default entry unchanged.
It refuses to overwrite an existing custom configuration, does not run
`update-grub` or `flash-kernel`, and never reboots automatically.
Keep recovery media available and save your work before restarting.
Select **EXPERIMENTAL: Laptop 7 touchscreen** to test; choose the normal entry
on the next restart if the experiment fails.

To reset the test, deleting the separate GRUB test entry and test DTB:

```bash
sudo bash patches/001-touch/test/test-touchscreen.sh --reset
```

Reset deletes the managed custom configuration, including its menu timeout
override, and `/boot/romulus15-touchscreen-test.dtb`. It is safe to repeat and
never changes the normal boot DTB; `--remove` is kept as an alias. Do not add unrelated entries to that file while using this script.
The experimental hardware limitations above still apply; successful script
validation is not a successful hardware test.

#### Applying the tested overlay to normal boot

After successfully testing the experimental entry, use
[install-touchscreen.sh](patches/001-touch/fix/install-touchscreen.sh) to apply the overlay to
the DTB already used by the normal GRUB entries:

```bash
bash patches/001-touch/fix/install-touchscreen.sh --check
sudo bash patches/001-touch/fix/install-touchscreen.sh --install
```

The script requires the exact original `7.3.0-rc3+` Romulus15 DTB and verifies
that rebuilding the overlay produces the boot-tested DTB byte for byte, using its embedded SHA-256
checksum. It therefore works before or after resetting the test entry. Installation requires
typing `APPLY`. It saves the original alongside the DTB with suffix
`.before-touchscreen`, then atomically replaces the normal DTB. It leaves
GRUB configuration and DTB symlinks unchanged; do not run `update-grub`.
Select the normal entry on the next boot. It does not reboot automatically.

**Unlike the separate test entry, this also changes the tree used by normal
and recovery entries referencing that same DTB.** Keep recovery media
available. To restore from a working Linux boot:

```bash
sudo bash patches/001-touch/fix/install-touchscreen.sh --restore
```

If the system cannot boot, use recovery media to mount the root filesystem
and copy
`/boot/dtbs/7.3.0-rc3+/qcom/x1e80100-microsoft-romulus15.dtb.before-touchscreen`
over the corresponding `.dtb` on that mounted filesystem.
Kernel/package updates may overwrite this patch; the script deliberately
does not patch future kernels automatically.

#### Touchscreen references

* Tracking issue for this repository: [bryce-hoehn/linux-surface-laptop-7#13](https://github.com/bryce-hoehn/linux-surface-laptop-7/issues/13)
* Surface Pro 11 touchscreen investigation (same `GTCH` HID-over-SPI design): [dwhinham/archiso-aarch64-sp11#10](https://github.com/dwhinham/archiso-aarch64-sp11/issues/10)
* Surface Pro 11 Ubuntu QSPI / HID-over-SPI work: <https://github.com/denisix/ubuntu-surface-pro-11>
* Published Romulus 13/15 ACPI tables (`SSDT.dsl` `GTCH`, `DSDT.dsl` `SP11`): <https://github.com/aarch64-laptops/build/tree/master/misc/microsoft-surface-romulus13_and_15>
* ELLX kernel (provides the QSPI and `spi_hid` drivers used here): <https://github.com/ProgrammerIn-wonderland/ELLX-Kernel>
* Linux device-tree overlay docs: <https://docs.kernel.org/devicetree/overlay-notes.html>
* Rnote (for the touch-drawing setting): <https://github.com/flxzt/rnote>

### Factory MAC Addresses
Both the ELLX prebuilds and the manual WiFi/Bluetooth instructions currently make up a random MAC address on each boot. This breaks DHCP reservations and may cause your device to sign out of the WiFi network each boot. An experimental patch is available [here](https://github.com/valeronm/sl7-mac).

### Fixing Firefox crashes
If you experience any issues with Firefox, try switching from snap to the .deb or flatpak package.

# Credits
* [dwhinham](https://github.com/dwhinham) for readme template and patches from Surface Pro 11: https://github.com/dwhinham/linux-surface-pro-11
* [qzed](https://github.com/qzed) from linux-surface: https://github.com/linux-surface/linux-surface
* [ProgrammerIn-wonderland](https://github.com/ProgrammerIn-wonderland) for the precompiled ELLX kernel builds
* [valeronm](https://github.com/valeronm) for the MAC address patch: https://github.com/valeronm/sl7-mac
* [dwhinham](https://github.com/dwhinham) and [denisix](https://github.com/denisix) for the Surface Pro 11 touchscreen research the Laptop 7 touchscreen overlay builds on
* [aarch64-laptops](https://github.com/aarch64-laptops/build) for the published Romulus ACPI tables
* all of our contributors / maintainers
* everyone from the ubuntu-concept issue thread: https://bugs.launchpad.net/ubuntu-concept/+bug/2084951
