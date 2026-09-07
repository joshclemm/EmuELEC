# Wired Switch Pro Controller: Amlogic-ng / Linux 4.9

The implementation change is `CONFIG_HID_NINTENDO=m` in
`linux/linux.aarch64.conf`. The pinned kernel already contains the early
`hid-nintendo` backport, including USB initialization and input parsing.
No new driver, compatibility shim, Kconfig entry, Makefile entry, kernel
upgrade, bootloader change, or device-tree change is necessary.

**Status:** loaded and tested on the actual EmuELEC 4.8 box. `evtest` recorded
real face-button, shoulder/trigger, Start/Select, Home/Capture, and joystick
press/release/movement events under `nintendo`. The user confirmed this Intec
panel has one joystick and no mode switch; it sends `ABS_X/ABS_Y`, not separate
D-pad or right-stick inputs. Those additional driver paths remain physically
unverified. The installed kernel is unchanged. The tested module can also be
installed persistently on this box using the optional runtime setup below.

## Source and patch selection

At EmuELEC revision `4f68ecf4784e2c076689e302d21f5f2ca8c1b788`:

| Selection | Repository location / value |
| --- | --- |
| Target | `PROJECT=Amlogic-ce DEVICE=Amlogic-ng ARCH=aarch64 DISTRO=EmuELEC` |
| Kernel family | `projects/Amlogic-ce/devices/Amlogic-ng/options`: `LINUX="amlogic-4.9"`, kernel architecture `arm64` |
| Kernel package | `projects/Amlogic-ce/packages/linux/package.mk` (project package overrides `packages/linux`) |
| Pinned revision | `ab03043a776354e02be86d06007751f652680c00` in `CoreELEC/linux-amlogic` |
| Archive | `https://github.com/CoreELEC/linux-amlogic/archive/ab03043a776354e02be86d06007751f652680c00.tar.gz` |
| SHA-256 | `c75bbd36d0dc92b21d75ce82dd8e0a4fefac9cf4b62f5d21c33eb7d12ca5bc2a` (verified) |
| Archive's kernel version | `4.9.269` |
| Package branch metadata | `amlogic-4.9-20`; the archive commit, not a moving branch, selects the source |
| Config | `projects/Amlogic-ce/devices/Amlogic-ng/linux/linux.aarch64.conf`, selected by `kernel_config_path()` in `config/functions` |

`scripts/unpack` runs the package's `post_unpack()` (including its exFAT source
overlay), then applies package, architecture/family/version, project, and device
patch directories in its declared order, then runs `post_patch()` to prepare the
config. For this checkout, the existing patch files are three package-root
patches followed by 29 patches in
`projects/Amlogic-ce/devices/Amlogic-ng/patches/linux/`. None adds or changes
`hid-nintendo.c`. `PKG_PATCH_DIRS="$LINUX"` also selects family directories when
present. `config/options` includes the device config directory in
`LINUX_DEPENDS`, so this config change participates in build invalidation.

The inspected device reports EmuELEC 4.8, build ID
`5a3e0877546318dd70cd001563e57bfb78328a6b`, and kernel
`4.9.269 #1 SMP PREEMPT Sun Oct 19 11:17:54 CST 2025`, compiled with GCC 12.2.0.
That release uses the same kernel package and kernel patches as this checkout.
Its live `/proc/config.gz` was used for the temporary module build. For another
box, establish its installed build identity and ABI before deploying.

## Existing implementation and references

- [CoreELEC introduction, `c5541684ca60b7aca40ae3c4dda12f7b4468e9a8`](https://github.com/CoreELEC/linux-amlogic/commit/c5541684ca60b7aca40ae3c4dda12f7b4468e9a8):
  imports Daniel Ogorchock's December 2019 driver and adds the HID core's special
  driver entries. Its `drivers/hid/hid-nintendo.c` is byte-for-byte identical to
  [Android's import, `f1aac226ceb002a65216a89b067c6fbb58b7db36`](https://android.googlesource.com/kernel/common/+/f1aac226ceb002a65216a89b067c6fbb58b7db36/).
  The original submission is
  [20191230012720.2368987-2](https://lore.kernel.org/linux-input/20191230012720.2368987-2-djogorchock@gmail.com/).
  This is the earliest suitable kernel implementation in the inspected vendor
  history and is already integrated into the exact source being built.
- [Mainline introduction, `2af16c1f846bd60240745bbd3afa13d5f040c61a`](https://github.com/torvalds/linux/commit/2af16c1f846bd60240745bbd3afa13d5f040c61a):
  the initial driver is still relatively small; notably, it changes the Pro
  Controller D-pad to hat axes. Subsequent patches add the optional features
  present in [Linux 5.16's driver](https://github.com/torvalds/linux/blob/v5.16/drivers/hid/hid-nintendo.c).
  Those patches are not imported here.
- [Peter Rankin's `joycon-linux-kernel`, `4bfa20239b4b5c3fde83b5f83c9ceeaf5a3a3cee`](https://gitlab.com/pjranki/joycon-linux-kernel/-/tree/4bfa20239b4b5c3fde83b5f83c9ceeaf5a3a3cee):
  inspected `drivers/hid/hid-joycon.c`; it matches Bluetooth Joy-Cons and does
  not provide the required USB Pro Controller path.
- [FrotBot / TheWaveWarden `SwitchProConLinuxUSB`, `6249d4b98876c4a174b796f86c490913f0053980`](https://github.com/TheWaveWarden/SwitchProConLinuxUSB/blob/6249d4b98876c4a174b796f86c490913f0053980/src/procon.hpp):
  inspected the USB handshake in `src/procon.hpp` and the project's history.
  Its first revision, `6eaf7725311510b8379cccebaf1a70b0c2d948c7`, only opens
  devices. The later implementation uses userspace hidapi/uinput; it is protocol
  reference material, not a kernel module to transplant.
- [Nintendo Switch reverse engineering: USB-HID-Notes.md](https://github.com/dekuNukem/Nintendo_Switch_Reverse_Engineering/blob/master/USB-HID-Notes.md):
  documents the USB command sequence and the timeout-disable command.

The existing probe opens HID I/O and enables reports during probe with
`hid_device_io_start()`. It sends `80 02` (handshake), `80 03` (3 Mbit baud rate),
`80 02` (second handshake), then `80 04` (disable USB timeout; no reply expected).
It requests factory stick calibration, falls back to defaults on read failure,
and sends subcommand `03` with argument `30` to select full input reports. It
also sets a steady player-light pattern. Its parser handles buttons and sticks
in reports `21`, `30`, and `31`; report `30` is used for ordinary full input.

The early driver already uses APIs available in this vendor 4.9 tree:
`hid_hw_output_report()`, `hid_device_io_start()`, `hid_field_extract()` (exported
GPL symbol), `devm_input_allocate_device()`, standard input reporting, mutexes,
and wait queues. No API incompatibility or shim was needed for compilation.
There is no reason to port the modern LED, power-supply, rumble, or IMU code.
The existing driver's Bluetooth matches remain as supplied by CoreELEC;
enabling this module also enables those existing Nintendo paths. Other
controller drivers and IDs are unchanged.

`drivers/hid/Kconfig` already defines `HID_NINTENDO` as a tristate depending on
`HID`, and `drivers/hid/Makefile` already builds `hid-nintendo.o` for it.
`hid-ids.h` defines `057e:2009`. `nintendo_hid_devices[]` provides the USB module
alias. In a kernel built with this config change,
`drivers/hid/hid-core.c` includes Nintendo in `hid_have_special_driver[]` under
`IS_ENABLED(CONFIG_HID_NINTENDO)`, keeping `hid-generic` from claiming it during
normal enumeration. The normal modules installation/depmod step enables
autoloading. Keep `hid.ignore_special_drivers` at its default `0`.

## Fastest test build: use the original prepared kernel build

Use a Debian/Ubuntu build host with the prerequisites in the repository's root
README (the documented Docker environments are in `tools/docker/README.md`).
Install the host's `kmod` package if `modinfo` is unavailable.
Use Bash, a regular user, and the EmuELEC checkout/toolchain that produced the
installed kernel. First record the box's identity, replacing the address:

```sh
export EE_HOST=192.168.1.123
ssh "root@$EE_HOST" 'uname -a; cat /etc/os-release; cat /etc/release; cat /proc/version' > emuelec-runtime.txt
ssh "root@$EE_HOST" 'zcat /proc/config.gz' > emuelec-runtime.config
```

`CONFIG_MODVERSIONS=y` requires the matching kernel's `Module.symvers`, generated
headers, configuration, source/patch set, and compatible toolchain. Matching
`uname -r` or vermagic alone is insufficient. `modules_prepare` does **not**
generate the required symbol CRCs. Do not use `insmod -f`, `modprobe --force`,
or disable version checks. Preserve the original build before cleaning anything.

With that prepared build available, compile only this existing source as an
external module. This keeps the original kernel `.config` (Nintendo disabled)
and `Module.symvers` intact, while the repository config supplies permanent
integration for a later image. From the matching EmuELEC repository root:

```bash
export PROJECT=Amlogic-ce DEVICE=Amlogic-ng ARCH=aarch64 DISTRO=EmuELEC
# Set this to the ORIGINAL prepared kernel build, not an unprepared source tree.
export EE_KDIR=/absolute/path/to/original/prepared/linux-build
export EE_MODDIR="$(mktemp -d /tmp/emuelec-nintendo-module.XXXXXX)"
bash <<'BUILD_MODULE'
set -e
. config/options linux
test -s "$EE_KDIR/.config"
test -s "$EE_KDIR/include/generated/autoconf.h"
test -s "$EE_KDIR/Module.symvers"
cp "$EE_KDIR/drivers/hid/hid-nintendo.c" "$EE_KDIR/drivers/hid/hid-ids.h" "$EE_MODDIR/"
printf '%s\n' 'obj-m += hid-nintendo.o' > "$EE_MODDIR/Makefile"
kernel_make -C "$EE_KDIR" M="$EE_MODDIR" modules
modinfo "$EE_MODDIR/hid-nintendo.ko"
BUILD_MODULE
export EE_MODULE="$EE_MODDIR/hid-nintendo.ko"
```

The driver source above must be the identified vendor backport. For another
installed release, establish its provenance before substituting source. Future
iterations can rerun the same `kernel_make ... M=... modules` command after
updating the staged driver. Make lasting fixes in the device's kernel patch
directory, then stage the patched source for the temporary build.

If the original build artifacts are unavailable, reconstruct them from the
installed release's exact checkout/config using `scripts/build linux`. This
builds the kernel/modules and dependencies, including toolchain and initramfs,
without building a full EmuELEC image. A cold toolchain build is still expensive.
Do not assume this current checkout's output can be loaded into an older image.

For a separate integration build of this checkout, run from its repository root:

```bash
export PROJECT=Amlogic-ce DEVICE=Amlogic-ng ARCH=aarch64 DISTRO=EmuELEC
export AUTOREMOVE=no
# In this separate build only: discards cached linux artifacts, not sources.
# Needed if linux was previously built: post_patch can restore the old .image/.config.
scripts/clean linux
scripts/build linux
bash <<'SHOW_MODULE'
set -e
. config/options linux
test "$(sed -n '/^CONFIG_HID_NINTENDO=/p' "$PKG_BUILD/.config")" = CONFIG_HID_NINTENDO=m
printf 'Module: %s/drivers/hid/hid-nintendo.ko\n' "$PKG_BUILD"
SHOW_MODULE
```

On an already prepared integration build, the single in-tree target is
`kernel_make -C "$PKG_BUILD" drivers/hid/hid-nintendo.ko` after sourcing
`config/options linux`. It does not rebuild the complete image.

## Temporary deployment and binding on the existing kernel

Copy the ABI-matched module from the build host:

```sh
scp "$EE_MODULE" "root@$EE_HOST:/storage/hid-nintendo-test.ko"
ssh "root@$EE_HOST"
```

Close running games. For an unambiguous test, connect only one `057e:2009`
controller. On the box, check the module and load it:

```sh
uname -r
modinfo /storage/hid-nintendo-test.ko
insmod /storage/hid-nintendo-test.ko
```

Stop on a load error and inspect `dmesg`; resolve the build/CRC mismatch before
continuing. The kernel built with Nintendo disabled still lets `hid-generic`
claim the device, even after the module is loaded. Rebind this HID device only:

```sh
set -- /sys/bus/hid/devices/0003:057E:2009.*
if [ "$#" -ne 1 ] || [ ! -d "$1" ]; then
    echo 'Expected exactly one USB 057e:2009 HID device; inspect /sys/bus/hid/devices.'
else
    EE_HID=${1##*/}
    EE_DRIVER=$(readlink "$1/driver")
    case "${EE_DRIVER##*/}" in
        hid-generic)
            printf '%s' "$EE_HID" > /sys/bus/hid/drivers/hid-generic/unbind
            printf '%s' "$EE_HID" > /sys/bus/hid/drivers/nintendo/bind
            ;;
        nintendo) echo 'Already bound to nintendo' ;;
        '') printf '%s' "$EE_HID" > /sys/bus/hid/drivers/nintendo/bind ;;
        *) echo "Unexpected driver: $EE_DRIVER; inspect before rebinding" ;;
    esac
    readlink "/sys/bus/hid/devices/$EE_HID/driver"
fi
dmesg | tail -80
```

The driver link must end in `/nintendo`; new log messages should name `nintendo`
and `Nintendo Switch Pro Controller`. Historical `hid-generic` lines can remain
in `dmesg`. No USB-level `usbhid` unbind, global blacklist, new USB ID, reboot,
or boot partition write is required. Replugging on the old kernel can allow
`hid-generic` to win again; repeat this explicit rebind for each test connection.

## Required hardware acceptance

Run `evtest` on the box and choose **Nintendo Switch Pro Controller**. Record
the selected event device and test it directly, for example:

```sh
evtest
# Replace eventN with the device selected above; press/release all controls.
evtest /dev/input/eventN | tee /storage/nintendo-evtest.txt
```

This early driver reports the following; the D-pad uses keys, not hat axes:

| Physical input | Required evtest events |
| --- | --- |
| A / B / X / Y | `BTN_EAST` / `BTN_SOUTH` / `BTN_NORTH` / `BTN_WEST`, both press and release |
| D-pad | `BTN_DPAD_UP`, `BTN_DPAD_DOWN`, `BTN_DPAD_LEFT`, `BTN_DPAD_RIGHT`; include diagonals |
| L / R / ZL / ZR | `BTN_TL` / `BTN_TR` / `BTN_TL2` / `BTN_TR2` |
| Minus / Plus | `BTN_SELECT` / `BTN_START` |
| Stick clicks | `BTN_THUMBL` / `BTN_THUMBR`, if the device provides them |
| Left / right stick | `ABS_X`, `ABS_Y` / `ABS_RX`, `ABS_RY`, range -32767..32767 |

Check that axes return near zero, reach both directions, and that buttons
release without sticking. If the arcade controller has a D-pad/left-stick/
right-stick mode switch, test each mode it actually provides. An absent
physical analog control cannot prove analog operation; document that limit.
Leave the controller idle for at least a minute, then test again to check USB
timeout handling. Unplug/replug, rebind on the old kernel, and repeat. Verify
another previously working controller still produces its normal events, then
map the Nintendo input in EmulationStation/RetroArch if necessary.

If initialization fails or evtest stays silent, capture diagnostics instead of
building an image. With the target config's dynamic debug enabled:

```sh
grep -q ' /sys/kernel/debug debugfs ' /proc/mounts || mount -t debugfs debugfs /sys/kernel/debug
echo 'module hid_nintendo +p' > /sys/kernel/debug/dynamic_debug/control
# Unplug/replug and repeat the HID rebind, then exercise the controls.
dmesg > /storage/nintendo-dmesg.txt
cat /proc/bus/input/devices > /storage/nintendo-input-devices.txt
echo 'module hid_nintendo -p' > /sys/kernel/debug/dynamic_debug/control
```

`Failed to set baudrate`, `Failed handshake`, or `Failed to set report mode`
points to protocol initialization. Calibration warnings call for checking
centering and range. A clone sharing `057e:2009` is not proof of protocol
compatibility; any further patch must follow the observed failure.

## Rollback

If the persistent setup below was installed, disable it using that section's
rollback steps first. For the original temporary test, unplug the controller,
then on the box:

```sh
rmmod hid_nintendo
rm /storage/hid-nintendo-test.ko
```

Replug: the original kernel will bind `hid-generic` again. If `rmmod` reports
the module is in use, close evtest/games and unplug any other Nintendo devices
using this module; do not force removal. Rebooting also restores the original
runtime when no persistent setup has been installed. Saved diagnostic
logs can remain in `/storage`.

To abandon the source change, restore just the config line to
`# CONFIG_HID_NINTENDO is not set` and rebuild any future kernel from that
configuration. The temporary test never replaces the installed kernel.

## Validation record

- Verified archive SHA-256 and kernel version; reviewed driver history,
  initialization, input mapping, module aliases, and HID binding behavior.
- Applied all 32 selected kernel patches in order to the archive in a disposable
  Ubuntu 20.04 ARM64 container. The unrelated exFAT overlay was not needed for
  this isolated HID compilation check; this was not a complete package build.
- Used the target config with `CONFIG_HID_NINTENDO=m`, then ARM64
  `olddefconfig`, `modules_prepare`, and `drivers/hid/hid-nintendo.ko`.
  GCC 9.4 compiled and linked it without a Nintendo source change or shim.
- `modinfo` reports `hid:b0003g*v0000057Ep00002009` and
  `4.9.269 SMP preempt mod_unload modversions aarch64`.
- That first isolated GCC 9.4 compile-check module is **not a deployment
  artifact**. A separate GCC 12.2 build was then prepared from the same pinned
  kernel, all 32 patches, and the package's checksum-verified exFAT overlay.
  Every resulting kernel config setting matches the device's live config.
- Since the original full `Module.symvers` was unavailable, compiled the 14
  kernel objects exporting the 32 required symbols, including `module_layout`.
  All 32 generated CRCs independently match metadata read from installed HID
  modules. Passed that generated subset via `KBUILD_EXTRA_SYMBOLS`, then verified
  the final module embeds exactly those 32 checks. No version checks were
  disabled or bypassed, and no CRC was substituted to conceal a mismatch.
- The final module loaded with normal `insmod`, and the selected controller
  bound to `nintendo` without initialization errors. Kernel taint remained
  unchanged at its pre-test value of 4096. The existing Xbox controller
  retained the same input device (`event6`); its physical controls were not
  exercised during this test.
- The local test artifact and ABI evidence are in the ignored
  `build.nintendo-module-test/` directory. The module SHA-256 is
  `10d30f2d0f1282434a8258704f891e55a26776efe79adaae6b8655a5584366c9`.
  It is copied to `/storage/hid-nintendo-test.ko` on the tested box.
- Recorded 359 input changes in the initial hardware capture: all four face
  buttons, four shoulder/trigger buttons, Start/Select, Home/Capture, and both
  joystick axes. Every observed button returned to zero. Both axes reached
  -32767 and +32767. The final neutral values were X=2463 and Y=-858; applications
  needing a perfectly centered analog input may need a deadzone/calibration.
  The driver's existing factory-calibration handling is unchanged.
- The user confirmed one joystick with no D-pad/right-stick mode switch. No
  physical D-pad, right-stick, or stick-click tests are claimed. Map this panel
  using its two joystick axes in EmulationStation/RetroArch.
- A physical USB unplug/replug was observed. The new HID instance initialized
  and bound successfully after the explicit rebind required by the old kernel.
  No physical input changes were captured in the second window; the recorded
  button/movement acceptance results are from the first capture. Both captures
  have ended, releasing their temporary input grabs.
- The initial test used no full-image build, boot-file change, or persistent
  module-loading rule. See the later persistent setup for the subsequent
  installation on the same box.


## Retained local test build and full-image command

On the development workstation, the prepared GCC 12.2 test container is
`emuelec-nintendo-work` (stopped after testing). Its `/kernel` contains the exact prepared source and
runtime config, and `/module` contains the existing driver. The independently
verified subset of kernel symbol versions is mounted at
`/work/verified-kernel-symbols.symvers`. From this repository root, the fastest
repeat compilation in that retained container is:

```sh
docker start emuelec-nintendo-work
docker exec emuelec-nintendo-work make -C /kernel ARCH=arm64 M=/module KBUILD_EXTRA_SYMBOLS=/work/verified-kernel-symbols.symvers modules
docker cp emuelec-nintendo-work:/module/hid-nintendo.ko build.nintendo-module-test/hid-nintendo.ko
```

The generated symbol table contains only this driver's dependencies. It is
not a replacement for a complete kernel `Module.symvers` for arbitrary drivers.
The test artifacts include the live config, the generated and installed symbol
versions, the comparison script, and input logs. For a fresh build environment,
use the original-build/kernel-package commands above.

The module has now passed actual button and joystick testing on this panel.
To integrate it in a future image, use the documented build host and, after
ensuring the Linux package picks up the new config as described above, run:

```sh
PROJECT=Amlogic-ce DEVICE=Amlogic-ng ARCH=aarch64 DISTRO=EmuELEC make image
```

This command has not been run as part of the temporary test. The rebuilt
kernel's existing Nintendo special-driver entries should handle automatic
binding; verify that behavior on the resulting image. A new image also needs
its own hardware checks before replacing a known-working installation.

## EmulationStation mapping on the tested Intec panel

After the driver test, pressing any button opened Configure Input because
EmulationStation had no saved profile for the newly exposed device. SDL 2.32.10
reports it as `Nintendo Switch Pro Controller`, GUID
`030056fb7e0500000920000011010000`, with four axes, 18 buttons, and no hats.
EmuELEC disables EmulationStation's fallback that would automatically create a
profile from SDL's game-controller database, so a saved ES profile is needed.

Added one device profile to `/storage/.emulationstation/es_input.cfg` on the
tested box, using axes 0/1 for this panel's directions and left analog input.
The buttons are A=1, B=0, X=2, Y=3, L/R=5/6, ZL/ZR=7/8, Select=9, Start=10,
and Home=11 as the hotkey. No nonexistent right stick or stick clicks were
assigned. All existing profiles were verified unchanged. EmulationStation was
restarted without rebooting or unloading the driver.

The saved backup is
`/storage/.config/emulationstation/es_input.cfg.before-nintendo-20260905`.
The standalone added profile is included at
`nintendo-usb-runtime/emulationstation-intec-profile.xml`. To reuse it, stop
EmulationStation, back up `es_input.cfg`, and add this `inputConfig` element
inside its existing `inputList` before restarting the frontend. Preserve the
other profiles and the `inputAction` element. This is a user profile for this
one-stick panel, not a new global Nintendo default.
The profile persists across boots. Without the optional persistent setup below,
the kernel module still requires the load/bind steps described above.

A second Intec panel was tested simultaneously on USB port 1-1.2 while the
first remained connected on 1-1.3. The second initially bound to `hid-generic`
and produced a connection toast without usable input. Explicitly rebinding
only its HID device (`0003:057E:2009.0006` for that connection) to `nintendo`
initialized it successfully. Both devices reported the same SDL GUID and
matched the saved profile. A nonexclusive `evtest` capture on the second
panel's `event6` recorded ABS_Y movement reaching -32767 and 32767, and
BTN_SOUTH press/release values 1/0. The user confirmed that the second panel
navigated the menu. This verifies simultaneous device initialization and menu
input on the second panel; two-player gameplay has not been tested.

## Persistent setup on the existing EmuELEC 4.8 installation

The saved EmulationStation mapping already persists. The remaining work is to
make the tested module discoverable by `modprobe` and rebind each new wired HID
device from `hid-generic`. EmuELEC provides both mechanisms without changing
the installed kernel or boot files:

- `packages/sysutils/busybox/scripts/kernel-overlays-setup` reads overlay paths
  from `/storage/.cache/kernel-overlays/*.conf` and runs `depmod`. The initramfs
  invokes it after mounting `/storage`, before starting systemd.
- `/etc/udev/rules.d` points to `/storage/.config/udev.rules.d` on this image.
  The existing `80-drivers.rules` loads modules by device alias, and
  `systemd-udev-trigger.service` sends add events for already connected devices
  during boot. The additional rule also handles later physical connections.
  See the [systemd 252 udev reference](https://github.com/systemd/systemd/blob/v252/man/udev.xml).

The optional files in `nintendo-usb-runtime/` implement only the rebind needed
by the old kernel. The helper loads the module before unbinding, skips devices
already using Nintendo or another specialized driver, allows two seconds for
the panel to finish starting, and attempts to restore `hid-generic` if Nintendo
initialization fails. It matches only USB
`057e:2009`; it does not blacklist another driver or change player assignments.
These files are not automatically included in rebuilt images. A rebuilt kernel
with `CONFIG_HID_NINTENDO=m` should use its normal driver selection instead.

For the exact tested 4.9.269 runtime, from the repository root on the build host:

```sh
EE_HOST=192.168.1.123 # Replace with your EmuELEC box's address.
EE_RUNTIME=projects/Amlogic-ce/devices/Amlogic-ng/nintendo-usb-runtime
EE_MODULE=build.nintendo-module-test/hid-nintendo.ko
ssh "root@$EE_HOST" 'test "$(uname -r)" = 4.9.269 && mkdir -p /storage/.config/nintendo-usb/lib/modules/4.9.269/extra'
scp "$EE_MODULE" "root@$EE_HOST:/storage/.config/nintendo-usb/lib/modules/4.9.269/extra/hid-nintendo.ko"
scp "$EE_RUNTIME/bind-controller" "$EE_RUNTIME/99-nintendo-usb.rules" "root@$EE_HOST:/storage/.config/nintendo-usb/"
ssh "root@$EE_HOST" 'sh -s' <<'INSTALL_NINTENDO'
set -e
chmod 0755 /storage/.config/nintendo-usb/bind-controller
printf '%s\n' /storage/.config/nintendo-usb > /storage/.cache/kernel-overlays/50-nintendo-usb.conf
kernel-overlays-setup
modinfo -n hid_nintendo
modprobe hid_nintendo
cp /storage/.config/nintendo-usb/99-nintendo-usb.rules /storage/.config/udev.rules.d/99-nintendo-usb.rules
udevadm control --reload-rules
INSTALL_NINTENDO
```

This installs the already ABI-verified module, not a module built for an
arbitrary 4.9.269 kernel. Keep normal module version checks enabled. Remove this
runtime overlay before an EmuELEC image upgrade; the replacement image should
provide its own matching driver.

Physically unplug/replug a panel and check the automatic path on the box:

```sh
udevadm settle --timeout=15
set -- /sys/bus/hid/devices/0003:057E:2009.*
test -d "$1" || exit 1
for EE_HID in "$@"; do
    EE_DRIVER=$(readlink "$EE_HID/driver")
    test "${EE_DRIVER##*/}" = nintendo || exit 1
done
journalctl -t nintendo-usb -n 10 --no-pager
```

Then confirm menu input and real `evtest` events. Also check a reboot with
controllers attached and a connection after boot with no controllers attached.
Replaying an add event on an already initialized controller verifies rule
selection and idempotence, but does not prove a fresh handshake or cold boot.

On this box, the persistent module resolved through both `modprobe` and its HID
alias, and the installed rule passed `udevadm test`. A physical reconnect of
the first panel initialized automatically. The second panel timed out when
initialized immediately after enumeration, then succeeded on a later manual
retry. The helper's two-second startup delay was added after that observation;
a subsequent reboot with both panels attached logged successful automatic
initialization for both devices, which appeared as `event3`/`js0` and
`event4`/`js1` using `nintendo`. Physical reconnection with the delay and
validation of every control after reboot are still pending; subsequent gameplay
input was confirmed as described below.

A later Configure Input dialog report occurred while both devices were already
bound to `nintendo` and still exactly matched the saved GUID/name. Refreshing
EmulationStation with temporary debug logging recorded both as `Added known
joystick`, using the existing profile on `event5` and `event6`. No profile rewrite
was needed. The log excerpt is `/storage/.config/nintendo-usb/menu-profile-check.log`.
The user subsequently confirmed both panels worked in the menu.

## RetroArch game controls and single-button shortcuts

The initial menu-only profile did not create a RetroArch autoconfiguration file.
An accidentally launched Flycast game exposed that missing step: RetroArch used
`/tmp/joypads`, which contained no Nintendo profile, so the gamepad exit hotkey
was unavailable. The game was closed with RetroArch's handled SIGINT signal.

The optional `nintendo-usb-runtime/Nintendo Switch Pro Controller.cfg` supplies
the same tested button/axis mapping for RetroArch's `udev` input driver. It sets
Home (button 11) as exit and Capture (4) as menu toggle, with the controller's
hotkey modifier unset. On the controller assigned to Player 1, press Home once
to return to EmulationStation, or press and release Capture to open RetroArch's
menu. Plus/Start and X remain ordinary game buttons. These replace the initial
Home + Start and Home + X shortcuts.

To install from the repository root on the build host:

```sh
EE_HOST=192.168.1.123 # Replace with your EmuELEC box's address.
scp 'projects/Amlogic-ce/devices/Amlogic-ng/nintendo-usb-runtime/Nintendo Switch Pro Controller.cfg' "root@$EE_HOST:/storage/.config/nintendo-usb/retroarch-profile.cfg"
ssh "root@$EE_HOST" 'cp /storage/.config/nintendo-usb/retroarch-profile.cfg "/tmp/joypads/Nintendo Switch Pro Controller.cfg"'
```

With RetroArch closed, back up `/storage/.config/retroarch/retroarch.cfg` and set
these two entries in that existing file:

```ini
input_exit_emulator_btn = "11"
quit_press_twice = "false"
```

The explicit global Exit binding is needed on the tested RetroArch 1.21.0
(`bfa603828d`) with its existing F1 keyboard hotkey. Its modifier filtering checks
the explicitly configured joypad Exit binding when allowing a standalone
controller hotkey alongside the keyboard modifier. Capture's autoconfigured menu
button has a separate path that works without a controller modifier. The
keyboard hotkey and other keyboard shortcuts are preserved. The global Exit
binding is for this Nintendo-panel setup; reset it to `nul` to use other
controllers' autoconfigured exit buttons. Disabling `quit_press_twice` applies
to RetroArch generally.

EmulationStation's profile remains separate. Running Configure Input again can
regenerate RetroArch's controller file; reapply this profile afterward to retain
the single-button shortcuts.

A later input failure exposed an incorrectly saved Configure Input result in
both ES and RetroArch: A was assigned to a joystick axis and physical button 1
became the hotkey modifier. Restored only this controller's ES entry and
RetroArch profile from the verified files, backed up the incorrect configuration,
and removed its stale temporary wizard file. Both repaired profiles survived an
EmulationStation restart. The global Home/quit settings were preserved; physical
gameplay was subsequently confirmed in the arcade Super Off Road: R accelerates
and B operates the game's action button. Its MAME 2003-Plus default maps the
pedal to R and button 1 to B, so A being unused in this game is expected.

On this image `/tmp/joypads` is an overlay with `/storage/joypads` as its writable
upper directory. Writing through `/tmp/joypads` was verified to persist the same
file in `/storage/joypads`. Other controller profiles were preserved. Only the
two global settings above were changed for the single-button setup. A separate
one-frame RetroArch menu run with null video/audio and
config saving disabled confirmed both panels were automatically configured.
Actual in-game use of the single-button shortcuts still needs a physical check.

## Game loading splashes

The custom artwork and controls guide is saved as
[`nintendo-usb-runtime/splash/arcade/offroad.png`](nintendo-usb-runtime/splash/arcade/offroad.png).
It shows joystick steering, R acceleration, B nitro, Minus for coins, and the
Player 1 Home/Capture shortcuts configured above. The image is designed for the
tested 1280x1024 display and this panel's mapping.

Installed it at `/storage/roms/splash/arcade/offroad.png`, matching the ROM's
`offroad.zip` filename. The existing splash script selects this per-game file
before platform or default artwork; no launcher or global setting change was
needed. Verified the transferred checksum, decoded the PNG, and exercised the
installed script's selection logic without displaying over the running game.
It takes effect on the next launch. Remove only this PNG to restore the existing
splash fallback.

The eight-second delay now applies only to the four custom game guides. The
splash script reads EmuELEC's per-game duration setting only when it selects
that ROM's own splash file. Platform and default artwork keep the global timing;
removing a custom image also restores the fallback timing. Arcade aliases such
as Neo Geo share the `arcade` prefix used by the splash directory.

The tested box's `/storage/.config/emuelec/configs/emuelec.conf` uses:

```ini
ee_splash_loading_duration=0
arcade["offroad.zip"].ee_splash_loading_duration=8
arcade["wjammers.zip"].ee_splash_loading_duration=8
arcade["tapper.zip"].ee_splash_loading_duration=8
atomiswave["dolphin.zip"].ee_splash_loading_duration=8
```

For the existing read-only system image, the installed splash script was copied
to `/storage/.config/emuelec/scripts/show_splash.sh` with the same duration lookup
change. EmuELEC's existing PATH selects this persistent override immediately;
no restart is needed. The repository's main splash script includes the change
for future builds. Remove the runtime override after installing a build that
includes it, so subsequent system script updates take effect. The previous
configuration and original script were backed up before installation.
Verified the actual settings lookup and splash selection with playback and sleep
mocked: all four guides use eight seconds, ordinary games use zero added delay,
Neo Geo finds the shared Arcade guide, missing artwork restores fallback timing,
and exit/blank screens keep their own behavior.

The companion Windjammers artwork is
[`nintendo-usb-runtime/splash/arcade/wjammers.png`](nintendo-usb-runtime/splash/arcade/wjammers.png),
installed at `/storage/roms/splash/arcade/wjammers.png`. Its guide uses the
configured FinalBurn Neo default layout: joystick movement/aim, B throw/dash,
A lob, Minus for coins, Plus to start, and the same Player 1 shortcuts.
These are the panel's labels; the original Neo Geo button A maps to this
panel's B. See [FBNeo's input mapping](https://github.com/libretro/FBNeo/blob/master/src/burner/libretro/retro_input.cpp).
Verified the PNG decode, transferred checksum, eight-second duration setting,
and selection for both Arcade and Neo Geo copies of `wjammers.zip` (the splash
script normalizes both systems to `arcade`). No game was interrupted for the
installation; the splash takes effect on its next launch.

The original Tapper uses
[`nintendo-usb-runtime/splash/arcade/tapper.png`](nintendo-usb-runtime/splash/arcade/tapper.png),
installed at `/storage/roms/splash/arcade/tapper.png`. Its guide shows up/down
to change bars, left/right to move along a bar, B held to fill and released to
serve, Minus for coins, Plus to start, and the Player 1 Home/Capture shortcuts.
The default MAME 2003-Plus layout maps Tapper's button 1 to the panel's B.
Verified the PNG decode, transferred checksum, selection for `tapper.zip`, and
the existing eight-second setting without interrupting a game. The separate
Root Beer Tapper (`rbtapper.zip`) keeps its existing splash fallback.

The Dolphin Blue guide is
[`nintendo-usb-runtime/splash/atomiswave/dolphin.png`](nintendo-usb-runtime/splash/atomiswave/dolphin.png),
installed at `/storage/roms/splash/atomiswave/dolphin.png` for `dolphin.zip`.
Flycast maps Shoot to RetroPad B, Jump to A, and Special to Y. The tested box's
pre-existing Player 1 overrides swap physical X/Y relative to its Nintendo
autoconfiguration, so the guide explicitly shows physical X for Player 1's
Special and physical Y for Player 2's Special. Those mappings were preserved.
The remaining entries show movement/aim, Minus for coins, Plus to start, and
the Player 1 Home/Capture shortcuts. The button translation was checked against
[the pinned Flycast source](https://github.com/flyinghead/flycast/blob/bf2bd7efed41e9f3367a764c2d90fcaa9c38a1f9/shell/libretro/libretro.cpp).
Verified the PNG decode, transferred checksum, selection for the Atomiswave
ROM, and the existing eight-second setting without interrupting a game.

## Disable the persistent runtime setup

To disable the persistent setup, run on the box:

```sh
rm -f /storage/.config/udev.rules.d/99-nintendo-usb.rules
rm -f /storage/.cache/kernel-overlays/50-nintendo-usb.conf
udevadm control --reload-rules
reboot
```

The reboot clears the loaded module and runtime overlay links. The module copy
and helper can remain in `/storage/.config/nintendo-usb` as inactive artifacts;
remove that directory later if desired. The saved menu mapping is independent
and remains available.
