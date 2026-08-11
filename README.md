# SpacemiT Android BSP (Banana Pi F3, MusePi Pro, Muse Pico-ITX)

Multi-repo manifests for the BayLibre Android 17 BSP on SpacemiT RISC-V
SoCs.

| Board | SoC | AOSP product | Output |
|---|---|---|---|
| Banana Pi F3 | K1 (X60, in-order RVA22) | `aosp_bananapi_f3` | `out/target/product/k1/` |
| MusePi Pro | K1 (X60, in-order RVA22) | `aosp_bananapi_f3` | `out/target/product/k1/` |
| Muse Pico-ITX | K3 (X100, out-of-order RVA23) | `aosp_k3_pico_itx` | `out/target/product/k3/` |

**The two K1 boards share one product and one image.** `aosp_bananapi_f3`
and `flash_bpi_f3.sh` are named after the F3 for historical reasons but
cover the MusePi Pro just as well: both device trees are packed into the
multi-DTB image and the right one is picked at boot through `adtb_idx`, so
there is nothing to choose at build or flash time. See
`aosp/device/spacemit/k1/README.md` for the per-board details.

The **bootloader is the exception** — it is built per board, from its own
config (`spacemit-k1.yaml` for the F3, `spacemit-musepi-pro.yaml` for the
MusePi Pro), and staged into its own `vendor/spacemit/BOARD/bootloader/`
directory.

One kernel serves both SoCs — a single `Image`, the union of the K1 and K3
modules, and the three device tree blobs. Nothing is gated at compile time;
drivers bind at runtime through their `of_match` tables.

## Manifests in this repo

| File | Tree | Base |
|---|---|---|
| `default.xml` | AOSP | `android17-release` (Google) |
| `kernel-spacemit.xml` | Android Common Kernel | `main-kernel` (Google), `common` on `android-mainline-spacemit` |
| `bootloader-spacemit.xml` | Bootloader chains | BayLibre forks of pi-u-boot / pi-opensbi + build-bootloaders |

`default.xml` and `kernel-spacemit.xml` inherit the upstream Google
manifests and apply BayLibre overrides at the bottom of each file.
`bootloader-spacemit.xml` is a small standalone manifest with only
BayLibre forks. See *Overrides* below.

Android 17 work lives on the **`android-17`** branch of this repo.

## Expected layout

```
spacemit/
  aosp/         AOSP source        (default.xml, required)
  kernel/       kernel source      (kernel-spacemit.xml, optional — only to rebuild the kernel)
  mesa/         Mesa source        (BayLibre/mesa @ android-17-pvr-support, optional)
  bootloaders/  bootloader source  (bootloader-spacemit.xml, optional)
```

The AOSP tree already ships:

- Mesa userspace prebuilts under `device/spacemit/k1/mesa/lib64/`
- kernel prebuilts under `device/spacemit/kernel/mainline/` — one set,
  consumed by both products
- bootloader prebuilts under `vendor/spacemit/{k1,musepi-pro,k3}/bootloader/`

so a plain `m` from `aosp/` produces flashable images on its own. Cloning
`kernel/`, `mesa/` and `bootloaders/` is only needed to test local changes.

## Prerequisites

Standard AOSP host setup (`repo`, `git`, JDK, ~300 GB free disk).

To rebuild Mesa locally, also install `meson`, `ninja`, `cmake`,
`llvm-config-15` or newer (with C++ libs), `patchelf` and Python 3.

## Sync the source

Required — AOSP tree:

```sh
mkdir -p spacemit && cd spacemit
mkdir aosp && (cd aosp && repo init -u git@github.com:BayLibre/android_manifest.git -b android-17 -m default.xml && repo sync -j$(nproc))
```

Optional — to rebuild the kernel:

```sh
mkdir kernel && (cd kernel && repo init -u git@github.com:BayLibre/android_manifest.git -b android-17 -m kernel-spacemit.xml && repo sync -j$(nproc))
```

Optional — to rebuild Mesa:

```sh
git clone -b android-17-pvr-support git@github.com:BayLibre/mesa.git mesa
```

The cross-build helper (`build-mesa-powervr.sh`) and the meson cross file
(`android-riscv64`) live at the top of the Mesa fork — nothing to install
separately.

Optional — to rebuild the bootloader (FSBL, OpenSBI, U-Boot):

```sh
mkdir bootloaders && (cd bootloaders && repo init -u git@github.com:BayLibre/android_manifest.git -b android-17 -m bootloader-spacemit.xml && repo sync -j$(nproc))
```

The riscv64 cross-toolchain is pulled from Bootlin on first build by
`build-bootloaders/utils.sh` — no manual toolchain install needed.

> The K3 bootloader chain is built from upstream U-Boot 2026.07 and
> upstream OpenSBI, which are **not published yet** and therefore not
> declared in the manifest. They are provisioned by hand; a fresh sync
> will not rebuild the K3 chain until those forks exist.

## Build

### AOSP

```sh
cd aosp && source build/envsetup.sh
lunch aosp_bananapi_f3-trunk_staging-userdebug   # K1: Banana Pi F3 *and* MusePi Pro
lunch aosp_k3_pico_itx-trunk_staging-userdebug   # K3: Muse Pico-ITX
m
```

There is no separate MusePi Pro product: one `aosp_bananapi_f3` build
serves both K1 boards. Only the bootloader differs, see below.

### Rebuild the kernel (optional)

One build produces the `Image`, the union of the K1 and K3 modules and the
three device tree blobs, and overwrites the shared prebuilts:

```sh
cd kernel && tools/bazel run --config=fast //devices/spacemit/spacemit_soc:spacemit_k1x_dist -- --destdir=../aosp/device/spacemit/kernel/mainline/
```

Re-run `m` from `aosp/` afterwards to repackage the images.

### Rebuild Mesa (optional)

Cross-builds Mesa for Android RISC-V and overwrites the prebuilts in
`aosp/device/spacemit/k1/mesa/lib64/`. Includes a
`patchelf --set-soname libgbm_mesa.so` step so the Soong wrapper finds the
prebuilt at runtime.

```sh
cd mesa && ./build-mesa-powervr.sh                  # sibling aosp/ tree
cd mesa && ./build-mesa-powervr.sh --aosp=../aosp   # or point it somewhere else
```

The first run builds the native helpers `mesa_clc` and `pco_clc` under
`/tmp/mesa-compiler` (a few minutes); later runs can reuse them with
`--skip-native`. Re-run `m` from `aosp/` afterwards.

### Rebuild the bootloader (optional)

`release_android.sh` wraps `build_all.sh` + `prepare_android_img.sh` +
`commit-binaries.sh`, builds the chain and stages the outputs straight into
the AOSP `vendor/spacemit/BOARD/bootloader/` tree. One config per board:

```sh
cd bootloaders/build-bootloaders
./release_android.sh --aosp=../../aosp --config=config/boards/spacemit-k1.yaml
./release_android.sh --aosp=../../aosp --config=config/boards/spacemit-musepi-pro.yaml
./release_android.sh --aosp=../../aosp --config=config/boards/spacemit-k3.yaml
```

Without `--config` every board config is released. Options:

```
--aosp=PATH    AOSP root path (required)
--commit       commit the new binaries into the AOSP-side vendor repo
--config=FILE  release a single board config (default: all)
--mode=MODE    [release|debug|factory] build only one mode
--no-build     skip rebuild, just re-stage existing artefacts
--silent       silence build command output
```

Re-run `m` from `aosp/` afterwards to repackage the images.

#### Low-level scripts

To drive individual steps yourself, e.g. iterating on U-Boot only:

| Script | Purpose |
|---|---|
| `setup_android.sh --aosp=PATH --branch=BR` | prepare AOSP-side branches for the binary commits |
| `build_all.sh spacemit-k1` | build every component (FSBL/OpenSBI/U-Boot) for one board |
| `build_opensbi.sh` / `build_uboot.sh` | build a single component |
| `prepare_android_img.sh --config=config/boards/spacemit-k1.yaml --mode=debug` | assemble `out/` into the layout the flasher expects |
| `commit-binaries.sh --from-repo=PATH --to-repo=PATH --to-project=device/spacemit/k1` | commit refreshed binaries into the AOSP-side repo |
| `secure.sh` | (placeholder) signing hook for production builds |

## Flash the board

`m` deposits the flash script, the bootloader prebuilts and the Android
images together in the product output directory. Run the script from there.

| Board | Directory | Script |
|---|---|---|
| Banana Pi F3 | `out/target/product/k1/` | `flash_bpi_f3.sh` |
| MusePi Pro | `out/target/product/k1/` | `flash_bpi_f3.sh` (same script and images) |
| Muse Pico-ITX | `out/target/product/k3/` | `flash_pico_itx.sh` |

Flashing a MusePi Pro uses the same script and the same images as the F3;
make sure the bootloader staged in `vendor/spacemit/musepi-pro/bootloader/`
is the one that was built for it.

Both take the same modes:

```sh
./flash_bpi_f3.sh                # default: bootloader + Android via fastboot
./flash_bpi_f3.sh --bootloader   # bootloader only
./flash_bpi_f3.sh --android      # Android images only
./flash_bpi_f3.sh --dfu          # full DFU flash from BROM (first-time setup)
./flash_bpi_f3.sh --dfu --wipe   # first-time DFU + clean userdata
```

`flash_bpi_f3.sh` also takes `--no-avb` to disable AVB verification during
bring-up. `flash_pico_itx.sh` takes `--android-media emmc|ufs`, since the
Pico-ITX may ship Android on either.

For the fastboot modes, run `fastboot usb 0` on the U-Boot console first so
the board exposes its fastboot interface.

### Putting the board in DFU mode (first-time setup)

Both SoCs expose the SpacemiT BROM USB download interface (USB ID
`361c:1001`) in DFU mode. This is required for the first flash on fresh
storage, and to recover from a bricked bootloader.

1. Disconnect all power and USB cables from the board.
2. Connect a USB-C data cable from the host to the OTG / USB-data port.
3. Press and hold the **BOOT** button (sometimes labelled `U-BOOT` or `KEY`).
4. Apply power while holding the button.
5. Release the button after about 2 seconds.
6. Check the host:

```sh
lsusb | grep 361c:1001
```

## Overrides

Compared to the upstream Google manifests, this BSP overrides:

| Manifest | Project | Source | Notes |
|---|---|---|---|
| `default.xml` | `external/minigbm` | `BayLibre/android_external_minigbm` @ `android-16` | gbm_mesa backend + dma_heap driver + runtime backend selection on top of upstream cros_gralloc |
| `default.xml` | `build/soong` | `BayLibre/android_build_soong` @ `android-16` | adds the SpacemiT riscv64 arch variants (`x60` for K1, `x100` for K3, cc + rust) |
| `default.xml` | `hardware/realtek` | `BayLibre/android_hardware_realtek` @ `android-17` | Realtek WiFi HAL (`libwifi-hal-rtk`), pruned from the DC-DeepComputing import (added) |
| `default.xml` | `device/spacemit/common` | `BayLibre/android_device_spacemit_common` @ `android-16` | board-agnostic SpacemiT bits (added) |
| `default.xml` | `device/spacemit/k1` | `BayLibre/android_device_spacemit_k1` @ `android-16` | K1 SoC + board overlays, Mesa prebuilts, audio/Bluetooth/WiFi HAL configs, SELinux vendor policy (added) |
| `default.xml` | `device/spacemit/kernel` | `BayLibre/android_device_spacemit_kernel` @ `android-16` | kernel prebuilts (`Image`, `.ko`, `.dtb`) — one set for both SoCs (added) |
| `default.xml` | `vendor/spacemit` | `BayLibre/android_vendor_spacemit` @ `android-16` | per-board makefiles, firmware blobs, bootloader prebuilts (added) |
| `kernel-spacemit.xml` | `common` | `BayLibre/android_kernel_common` @ `android-mainline-spacemit` | kernel/common + the SpacemiT K1 and K3 commits (drivers, dts, configs) |
| `kernel-spacemit.xml` | `devices/spacemit` | `BayLibre/android_kernel_device_spacemit` @ `android-mainline-spacemit` | Kleaf `kernel_build` target under `spacemit_soc/`, one image for K1 and K3 (replaces the Pixel `raviole` device tree) |
| `kernel-spacemit.xml` | (all other Google upstream) | pinned SHAs | frozen against upstream drift (rust-toolchain, clang, gcc, build-tools, libcap, …) |
| `bootloader-spacemit.xml` | `build-bootloaders` | `BayLibre/android_bootloader_build` @ `spacemit-android-16` | build orchestration scripts, extended with SpacemiT K1 and K3 support |
| `bootloader-spacemit.xml` | `pi-u-boot` | `BayLibre/pi-u-boot` @ `v2022.10-k1` | K1 U-Boot port (BPI-SINOVOIP + Android boot / AVB / fastboot enablement) |
| `bootloader-spacemit.xml` | `pi-opensbi` | `BayLibre/pi-opensbi` @ `v1.3-k1` | K1 OpenSBI port (defensive fork — no local patches) |

`device/spacemit/k3` and `device/spacemit/k3-kernel` are not repositories
yet; the K3 port carries them as plain directories. They will be declared
here once `android_device_spacemit_k3` exists.
