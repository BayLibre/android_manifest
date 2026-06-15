# SpacemiT K1 Android BSP (Banana Pi F3 + MusePi Pro)

Multi-repo manifests for the BayLibre Android 16 BSP targeting the
SpacemiT K1 (RISC-V), on the **Banana Pi F3** and **MusePi Pro** boards
(one `k1` device, both DTBs in a multi-DTB image picked at boot via
`adtb_idx`; see `aosp/device/spacemit/k1/README.md` for the per-board details).

## Manifests in this repo

| File | Tree | Base |
|---|---|---|
| `default.xml`    | AOSP                  | `android16-qpr2-release` (Google) |
| `kernel.xml`     | Android Common Kernel | `main-kernel` (Google), kernel/common branch `android-mainline-riscv64` |
| `bootloader.xml` | Bootloader chain      | BayLibre forks of pi-u-boot / pi-opensbi + build-bootloaders |

`default.xml` and `kernel.xml` inherit the upstream Google manifests
and apply BayLibre overrides at the bottom of each file.
`bootloader.xml` is a small standalone manifest with only BayLibre
forks. See *Overrides* below.

## Expected layout

```
spacemit/
  aosp/        AOSP source         (synced from default.xml, required)
  kernel/      kernel source       (synced from kernel.xml, optional — only to rebuild the kernel)
  mesa/        Mesa source         (BayLibre/mesa @ android-pvr-support, optional — only to rebuild Mesa)
  bootloader/  bootloader source   (synced from bootloader.xml, optional — only to rebuild the bootloader)
```

The AOSP tree already ships:

- Mesa userspace prebuilts under `device/spacemit/k1/mesa/lib64/`
- kernel prebuilts under `device/spacemit/k1-kernel/mainline/`
- bootloader prebuilts under `vendor/spacemit/k1/bootloader/`

so a plain `m` from `aosp/` produces flashable images on its own.
Cloning `kernel/`, `mesa/` and `bootloader/` is only required when
you have local changes you want to test.

## Prerequisites

Standard AOSP host setup (`repo`, `git`, JDK, ~300 GB free disk).

If you plan to rebuild Mesa locally, also install:

- `meson`, `ninja`, `cmake`
- `llvm-config-15` or newer (with C++ libs)
- `patchelf`
- Python 3

## Sync the source

Required — AOSP tree:

```sh
mkdir -p spacemit && cd spacemit
mkdir aosp && (cd aosp && repo init -u git@github.com:BayLibre/android_manifest.git -b android-16-spacemit -m default.xml && repo sync -j$(nproc))
```

Optional — only if you plan to rebuild the kernel locally:

```sh
mkdir kernel && (cd kernel && repo init -u git@github.com:BayLibre/android_manifest.git -b android-16-spacemit -m kernel.xml && repo sync -j$(nproc))
```

Optional — only if you plan to rebuild Mesa locally:

```sh
git clone -b android-pvr-support git@github.com:BayLibre/mesa.git mesa
```

The cross-build helper (`build-mesa-powervr.sh`) and the meson
cross file (`android-riscv64`) live at the top of the Mesa fork on
the `android-pvr-support` branch — nothing to install separately.

Optional — only if you plan to rebuild the bootloader (FSBL,
OpenSBI, U-Boot) locally:

```sh
mkdir bootloader && (cd bootloader && repo init -u git@github.com:BayLibre/android_manifest.git -b android-16-spacemit -m bootloader.xml && repo sync -j$(nproc))
```

The riscv64 cross-toolchain is pulled from Bootlin on first build
by `build-bootloaders/utils.sh` (`download_riscv64_toolchain`) —
no manual toolchain install needed.

## Build

### AOSP (required)

```sh
cd aosp && source build/envsetup.sh && lunch aosp_bananapi_f3-trunk_staging-userdebug && m
```

Output images land in `out/target/product/k1/`.

### Rebuild Mesa (optional)

Cross-builds Mesa for Android RISC-V and overwrites the prebuilts
in `aosp/device/spacemit/k1/mesa/lib64/`. Includes a
`patchelf --set-soname libgbm_mesa.so` step so the Soong wrapper
finds the prebuilt at runtime.

```sh
cd mesa && ./build-mesa-powervr.sh
```

The first run builds the native helpers `mesa_clc` and `pco_clc`
under `/tmp/mesa-compiler` (a few minutes). Subsequent runs can
skip them with `--skip-native`. Re-run `m` from `aosp/` afterwards
to repackage the images.

### Rebuild the kernel (optional)

Builds the GKI kernel, the SpacemiT K1 modules and the Banana Pi F3
device tree blob, and overwrites the kernel prebuilts in
`aosp/device/spacemit/k1-kernel/mainline/`.

```sh
cd kernel && tools/bazel run --config=fast //devices/spacemit/bananapi_f3:spacemit_k1x_dist -- --destdir=../aosp/device/spacemit/k1-kernel/mainline/
```

Re-run `m` from `aosp/` afterwards to repackage the images.

### Rebuild the bootloader (optional)

Builds the bootloader chain (FSBL → OpenSBI → U-Boot) for the
Banana Pi F3 and bundles the artefacts (`FSBL.bin`, `fw_dynamic.itb`,
`u-boot.itb`, `env.bin`, `partition_android.json`) for `flash_bpi_f3.sh`.

The recommended entry point is `release_android.sh`, which wraps
`build_all.sh` + `prepare_android_img.sh` + `commit-binaries.sh`
into a single command that builds the chain and stages the outputs
directly into the AOSP `vendor/spacemit/k1/bootloader/` tree:

```sh
cd bootloader/build-bootloaders && ./release_android.sh --aosp=../../aosp
```

First run downloads the Bootlin riscv64 toolchain into
`bootloader/toolchains/` automatically (no manual install).

Options on `release_android.sh`:

```
--aosp=PATH    AOSP root path (required)
--commit       commit the new binaries into the AOSP-side vendor repo
               (calls commit-binaries.sh under the hood)
--config=FILE  release a single board config file (default: all configs)
--mode=MODE    [release|debug|factory] build only one mode
--no-build     skip rebuild, just re-stage existing artefacts
--silent       silence build command output
```

For a debug build that just refreshes binaries without committing:

```sh
./release_android.sh --aosp=../../aosp --mode=debug
```

Re-run `m` from `aosp/` afterwards to repackage the images.

#### Low-level scripts

If you need to drive individual steps yourself (e.g. iterating on
U-Boot only without re-running the full release pipeline):

| Script | Purpose |
|---|---|
| `setup_android.sh --aosp=PATH --branch=BR` | initial setup: prepares AOSP-side branches for the binary commits |
| `build_all.sh spacemit-k1` | build every component (FSBL/OpenSBI/U-Boot) for the K1 board |
| `build_opensbi.sh` / `build_uboot.sh` | build a single component |
| `prepare_android_img.sh --config=config/boards/spacemit-k1.yaml --mode=debug` | assemble the artefacts under `out/` into the layout `flash_bpi_f3.sh` expects |
| `commit-binaries.sh --from-repo=PATH --to-repo=PATH --to-project=device/spacemit/k1` | commit refreshed binaries into the AOSP-side repo with a sensible message |
| `secure.sh` | (placeholder) signing hook for production builds |

## Flash the board

After `m`, the AOSP build deposits `flash_bpi_f3.sh`, the bootloader
prebuilts (`FSBL.bin`, `fw_dynamic.itb`, `u-boot.itb`, `env.bin`,
`partition_android.json`) and the Android images (`boot.img`,
`super.img`, `vbmeta*.img`, …) all together into
`out/target/product/k1/`. Run the script from there:

```sh
cd aosp/out/target/product/k1
```

### Putting the board in DFU mode (first-time setup)

The board exposes the SpacemiT BROM USB download interface (USB ID
`361c:1001`) when in DFU mode. This is required for the first-time
flash on a fresh eMMC, and to recover from a bricked bootloader.

1. Disconnect all power and USB cables from the board.
2. Connect a USB-C data cable from the host to the F3 OTG / USB-data port.
3. Press and hold the **BOOT** button on the board (sometimes labelled `U-BOOT` or `KEY`).
4. Apply power while holding the button — second USB-C / barrel jack, or a powered USB-C host port.
5. Release the BOOT button after about 2 seconds.
6. Verify on the host that the board is in BROM mode:

```sh
lsusb | grep 361c:1001
```

The line `Bus xxx Device xxx: ID 361c:1001 ...` confirms DFU mode.

### Flash modes

From `out/target/product/k1/`:

```sh
./flash_bpi_f3.sh                # default: bootloader + Android via fastboot (board in U-Boot)
./flash_bpi_f3.sh --bootloader   # bootloader only
./flash_bpi_f3.sh --android      # Android images only
./flash_bpi_f3.sh --dfu          # full DFU flash from BROM (first-time setup)
./flash_bpi_f3.sh --dfu --wipe   # first-time DFU + clean userdata
./flash_bpi_f3.sh --no-avb       # disable AVB verification (bringup/development)
```

For the fastboot modes, run `fastboot usb 0` on the U-Boot console
first so the board exposes its fastboot interface to the host.

## Overrides

Compared to the upstream Google manifests, this BSP overrides:

| Manifest | Project | Source | Notes |
|---|---|---|---|
| `default.xml` | `external/minigbm` | `BayLibre/android_external_minigbm` @ `android-16` | gbm_mesa backend + dma_heap driver + runtime backend selection on top of upstream cros_gralloc |
| `default.xml` | `device/spacemit/common` | `BayLibre/android_device_spacemit_common` @ `android-16` | board-agnostic SpacemiT bits (added) |
| `default.xml` | `device/spacemit/k1` | `BayLibre/android_device_spacemit_k1` @ `android-16` | K1 SoC + Banana Pi F3 board overlay, Mesa userspace prebuilts, audio/Bluetooth/Wi-Fi HAL configs, SELinux vendor policy (added) |
| `default.xml` | `device/spacemit/k1-kernel` | `BayLibre/android_device_spacemit_k1_kernel` @ `android-16` | Kleaf-built kernel prebuilts (`Image`, `.ko` modules, `.dtb`) consumed by the AOSP build (added) |
| `default.xml` | `vendor/spacemit` | `BayLibre/android_vendor_spacemit` @ `android-16` | `k1/k1.mk`, firmware blobs, bootloader prebuilts (added) |
| `default.xml` | `build/soong` | `BayLibre/android_build_soong` @ `android-16` | adds the Spacemit X60 riscv64 arch variant (`x60` `-march`/`-mcpu`/`-mtune`, cc + rust) (added) |
| `kernel.xml` | `common` | `BayLibre/android_kernel_common` @ `android-mainline-spacemit` | kernel/common + ~35 SpacemiT K1 ANDROID: commits (drivers, dts, configs) |
| `kernel.xml` | `devices/spacemit` | `BayLibre/android_kernel_device_spacemit` @ `android-mainline-riscv64` | Banana Pi F3 Kleaf `kernel_build` target (replaces the Pixel `raviole` device tree) |
| `kernel.xml` | (all other Google upstream) | pinned SHAs | Frozen against upstream drift (rust-toolchain, clang, gcc, build-tools, libcap, etc.) |
| `bootloader.xml` | `build-bootloaders` | `BayLibre/android_bootloader_build` @ `spacemit-android-16` | Build orchestration scripts (forked from BayLibre GitLab TI Android, extended with SpacemiT K1 support) |
| `bootloader.xml` | `pi-u-boot` | `BayLibre/pi-u-boot` @ `v2022.10-k1` | U-Boot port (forked from BPI-SINOVOIP + Android boot / AVB / fastboot enablement) |
| `bootloader.xml` | `pi-opensbi` | `BayLibre/pi-opensbi` @ `v1.3-k1` | OpenSBI port (forked from BPI-SINOVOIP, defensive fork — no local patches) |
