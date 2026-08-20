# Kernel-Lamu

Custom Android GKI kernel (`android15-6.6`, GKI 2.0 mixed-build) for the MT6768
"Lamu" device family — built and tested on the Motorola moto g05.

## What's customized here

- **Root**: [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) (`gki-android15-6.6-dev`
  branch) replacing KernelSU-Next, compiled directly into the kernel image
  (`CONFIG_KSU=y`).
- **SUSFS** v2.2.0 integrated on top (susfs4ksu kernel-side patch), for hiding
  root artifacts from apps that check for it.
- **Networking**: BBR set as the *active default* TCP congestion control
  (`CONFIG_DEFAULT_BBR=y`), not just compiled in.
- **ZRAM**: lz4 as the primary compressor, zstd for idle/cold-page
  recompression (better ratio, worth the extra CPU cost since those pages are
  already infrequently touched).
- **CPU idle**: trimmed to the `teo` governor only (`menu` disabled).
- `CONFIG_LOCALVERSION="-Madruga"`.

This repo is the ACK/GKI kernel side only. The device-tree / vendor module
source (touch-boost tuning, MediaTek drivers, build scripts) lives in the
companion repo, [motorola-lamu-device-modules-6.6](https://github.com/OWLXS/motorola-lamu-device-modules-6.6).

Published as a single squashed commit (no upstream ACK history) to keep the
repo lightweight — see `COPYING` for the kernel's own licensing.

## How to build

This is a **GKI mixed-build** kernel built with Google's Kleaf (Bazel-based
kernel build system) — it is not buildable standalone from this repo alone.

1. Get the rest of the workspace via `repo` tool, using the manifest this
   device tree is actually based on:

   ```bash
   mkdir lamu-kernel && cd lamu-kernel
   repo init -u https://github.com/LineageOS/android_kernel_motorola_lamu_manifest.git -b lineage-23.2
   repo sync -c -j$(nproc)
   ```

   This pulls in everything the build needs: `build/kernel`,
   `build/bazel_mgk_rules`, the `prebuilts/` toolchain, `external/` mirrors,
   etc.

2. Replace the stock `kernel-6.6/` and `kernel_device_modules-6.6/` that
   `repo sync` checked out with **this repo** and
   **[motorola-lamu-device-modules-6.6](https://github.com/OWLXS/motorola-lamu-device-modules-6.6)**
   respectively.

3. From the **root of that workspace**, run:

   ```bash
   ./build_mgk_64_k66.sh
   ```

   This is the entry point used throughout development. Internally it:
   - Calls `kernel_device_modules-6.6/build.sh`, which invokes
     `tools/bazel build/run` against the `mgk_64_k66` target with
     `--config=stamp --lto=thin` (see that repo for the exact `KLEAF_ARGS`).
   - Copies the resulting `Image*`, `.ko` modules, `dtbo.img`, `.dtb` and
     merged UAPI headers into `out/mgk_64_k66/dist/`.

4. Package the result into something flashable. Two options that have been
   tested end-to-end on this device:
   - **magiskboot**: unpack a stock `boot.img` for this device, swap in the
     new `Image` (concatenated with the device's `.dtb` and gzip'd — this
     device's `boot` partition has no separate dtb section), repack.
   - **AnyKernel3**: works too, since this device's `boot` partition has
     `RAMDISK_SZ=0` (ramdisk lives in the separate `init_boot` partition,
     standard GKI header-v4 split) — use `split_boot`/`flash_boot` instead
     of the template defaults, and name the kernel file `Image.gz-dtb`
     (Image + dtb appended, gzip'd) so `ak3-core.sh` picks it up correctly.

## Root manager signature

If you're using ReSukiSU's manager app and see a "not the official manager"
warning, make sure you installed an **official release** build (not a PR/CI
debug build) — the kernel checks the manager APK's signing certificate
against a hardcoded hash (`manager/manager_sign.h`), which only matches the
real release signing key.
