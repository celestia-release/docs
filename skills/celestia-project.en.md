---
name: celestia-project
description: >-
  Celestia Project / BSP layer (project/, system/<SoC>/) development and
  troubleshooting: project/<proj>/config.mk is the single source of truth
  (CONFIG_SOC_PLATFORM/CONFIG_BOARD/CONFIG_APP_*/CONFIG_FW_*/CONFIG_HW_* →
  generates config.h), assembling anypi_nova (AnyCloud39AV200/ipc_main),
  anypi_nova_dev (AnyCloud39AV200/nova_main), anypi_luna
  (AnyCloud39AV100_V2/240x240/luna_main), anypi_luna_dev (AV100_V2
  V2.04/luna_dev_main), and anypi_mira_dev (Sky39AV130 V1.01/aos/mira_main);
  product selection: --product-name → cache → interactive project/* pick.
  BSP includes kernel/DTS/rootfs overlay/insmod boot scripts/burntool. Partition
  tuning focus: two env_*.cfg files (mtdparts/kernel_offset/kernel_size/boot_normal)
  → regenerate env_*.img → burn download_to_nand and DTS bootargs aligned in
  three places; APP burn=`0x82E000`; hybrid /usr+SD install. Vendor tree is
  read-only by default. After config.mk changes use build.sh -k or full build.
  Use when debugging build options, SoC/board, DTS, driver load, image packaging,
  flash partition overflow, and on-board bring-up.
---

<p align="right"><b>English</b> | <a href="celestia-project.md">中文</a></p>

# Project / BSP layer skill

> Docs: `docs/quick-start/bsp-and-products.md`. Overview: [celestia](celestia.en.md).
> On-board flash and serial observation: [celestia-scripts](celestia-scripts.en.md) and [celestia-debug](celestia-debug.en.md) (the “wearable on-board” section here is board-level essentials).

## Assembly model

The same Celestia sources assemble different SoC/board/form-factor images via different `project/<proj>/`. Project has three main jobs: **pick platform** (`CONFIG_SOC_PLATFORM`), **pick board** (`CONFIG_BOARD`), **ship board resources** (DTS/rootfs/boot scripts). `Application/Framework/System Core/hardware/include` are shared across SoCs; SoC switch cost is limited to HAL implementation + BSP.

```
project/
├── anypi_nova/      config.mk + include/config.h(auto-generated)
│                    + system/AnyCloud39AV200/V1.04/board/PDB_MBD_AK3918AV200_ACTCAM/...
├── anypi_nova_dev/  config.mk + ... + system/AnyCloud39AV200/V1.04/board/EVB_MBD_AK3918AV200/...
├── anypi_luna/      config.mk + ... + system/AnyCloud39AV100_V2/V2.05/board/PDB_MBD_KM01W_ACTCAM/... (also keeps V2.04 legacy overlay)
├── anypi_luna_dev/  config.mk + ... + system/AnyCloud39AV100_V2/V2.04/board/EVB_MBD_KM01A/...
└── anypi_mira_dev/      config.mk + setup_toolchain.sh + system/Sky39AV130/V1.01/board/EVB_CBD_AK3918AV130/...
system/
├── AnyCloud39AV200/      # vendor BSP (may include V1.03/V1.04 PDK)—read-only by default, no board/ overlay
├── AnyCloud39AV100_V2/   # same (V2.04 / V2.05)
├── Sky39AV130/           # same (V1.01, Mira)
└── core/                 # see celestia-system-core (not BSP)
```

⚠️ **Board overlay exists only under `project/<proj>/system/<SoC>/<SDK>/board/<Board>/`**; top-level `system/<SoC>/<SDK>/` has **no** `board/` (that is vendor PDK payload). The assembly diagram above follows this layout.

## config.mk — single source of truth (SSOT)

`build.sh` parses all `CONFIG_*` in `project/<proj>/config.mk` and generates `project/<proj>/include/config.h`. Core variables for five products:

| Variable | anypi_nova | anypi_nova_dev | anypi_luna | anypi_luna_dev | anypi_mira_dev |
|------|------------|----------------|------------|----------------|------------|
| `CONFIG_SOC_PLATFORM` | AnyCloud39AV200 | AnyCloud39AV200 | AnyCloud39AV100_V2 | AnyCloud39AV100_V2 | Sky39AV130 |
| `CONFIG_SOC_SDK_VERSION` | V1.04 | V1.04 | V2.05 | V2.04 | V1.01 |
| `CONFIG_OS_NAME` | linux | linux | linux | linux | aos |
| `CONFIG_BOARD` | PDB_MBD_AK3918AV200_ACTCAM | EVB_MBD_AK3918AV200 | PDB_MBD_KM01W_ACTCAM | EVB_MBD_KM01A | EVB_CBD_AK3918AV130 |
| `CONFIG_APP_MAIN_TARGET` | ipc_main | nova_main | luna_main | luna_dev_main | mira_main |
| `CONFIG_APP_PACKAGES_*` | TABLET=y | TABLET=y | WEARABLE=y | IPCAMERA=y | IPCAMERA=y |
| `CONFIG_WIFI_TYPE` | atbm6162 | — (board-level) | aic8800 | atbm6032 | — |

> anypi_mira_dev uses `CONFIG_OS_NAME=aos`: `build.sh` automatically sources `project/anypi_mira_dev/setup_toolchain.sh` (alios `arm-none-eabi-` toolchain). Full Linux→AOS port flow (POSIX compat, static link, kernel patches): [celestia-aos-port](celestia-aos-port.en.md).

Also includes `CONFIG_APP_*` (App switches), `CONFIG_FW_*` (service switches), `CONFIG_HW_*` (HAL switches), `CONFIG_THIRD_PARTY_*`, `CONFIG_WIFI_TYPE`/`CONFIG_SENSOR_NAME` (board-level). When changing switches, **always check other products’ config.mk** to avoid cross-impact.

> Note: luna board overlay `board/<Board>/config.mk` defines `CONFIG_DTB_NAME` (e.g. `EVB_CBDM_KM01A_V1.0.0`); product top-level `config.mk` no longer defines it. See next section for DTB selection.

## BSP essentials

- Toolchain by platform (≠ by product name): nova (AV200) uses `riscv64-anyka-linux-musl-` (RISC-V/musl); luna family (AV100_V2) uses `arm-anycloud-linux-uclibcgnueabi-`; mira (Sky39AV130, aos) gets `arm-none-eabi-` from `project/anypi_mira_dev/setup_toolchain.sh`.
- DTB selection: nova family auto-picked by `package.sh` from FASTSYS/LCD/DMIC; luna from board overlay/pack scripts (no longer via `CONFIG_DTB_NAME`). See `docs/quick-start/bsp-and-products.md §6.5`.
- rootfs overlay order: SOC base → `board/<Board>/rootfs/` → `patch/rootfs/` → (FASTSYS) `patch_fastsys/rootfs/`; later layers override earlier.
- Boot scripts differ by product: **tablet family uses `app_01_start.sh`** (`insmod` driver list + launch main); **wearable uses `app.sh` + `app_02_load.sh`** (not `app_01_start.sh`), see “wearable on-board”.

## Prohibited / cautions

- ⚠️ **Vendor BSP under `system/<SoC>/<SDK>/`** (kernel, rootfs, platform SDK) is **read-only by default**; do not edit chip vendor tree for debug—prefer HAL or upper-layer workarounds. When driver load/DTS must change, **only edit `project/<proj>/system/.../board/` overlay and insmod list in boot scripts**.
- Changing `config.mk` is a configuration change; it must not replace code-layer fixes.

## Troubleshooting: Project layer

1. Only when symptoms are “build options/platform/board/driver load/packaging”: verify `config.mk` variables, `config.h` generation, DTB matches board, boot scripts (nova `app_01_start.sh` / luna `app.sh`) insmod order and `.ko` completeness.
2. Missing driver devices (`/dev/video0`, etc.) → confirm insmod order and ko paths, then [celestia-hardware](celestia-hardware.en.md) for HAL device open.
3. New SoC/board: `docs/quick-start/bsp-and-products.md §9` eight-step flow + [celestia-hardware](celestia-hardware.en.md) “new SoC”.

## Wearable on-board (board essentials)

For **PDB_MBD_KM01W_ACTCAM** (AnyCloud39AV100_V2). After build/package, images and burn tools live in `out/anypi_luna/burntool/`; flash and observe via serial. Luna board must-know items only:

- **Serial console**: USB serial, `115200` 8N1; login prompt `[root@anyka ~]#`; boot log and `app.sh` execution visible.
- **Network**: WiFi interface is `vnet0` (not `wlan0`). KM01W WiFi and TF share SDIO; `ak_mci.ko` loaded by `wifi_driver.sh`.
- **Boot scripts**: `/etc/config/app.sh` runs drivers/WiFi at boot (**does not auto-start app**); `app_02_load.sh` loads ST7789 + backlight, called by `luna_main` before UI. Debug: manually `killall luna_main; /usr/sbin/luna_main` without re-running `app.sh`.
- **Display**: KM01W has no GPU, software fb refresh (PARTIAL mode); `/dev/fb0` expects 240×240, `lv_conf.h` uses `LV_COLOR_DEPTH 16`, `LV_MEM_SIZE` ~512KB. Related changes in `project/anypi_luna/{include/lv_conf.h,config.mk,.../board/etc/config}`.

## Board overlay contents

Path: `project/<proj>/system/<SoC>/<SDK>/board/<Board>/` (`CONFIG_BOARD` = directory name).

| Subdir / file | Role |
|---|---|
| `os/kernel/.../dts/*.dtsi` | Board DTS (DMA reserved, pinmux, IO, LCD/sensor) |
| `rootfs/` | Board overlay: `fw_*/hal_*/app_*` under `utils/etc/config/` + `app.sh` / `app_01_start.sh` |
| `defconfig` | kernel defconfig |
| `burntool/` | flash tool and partition config |
| `config.mk` (board) | DTB name, partitions, flash layout; included by top-level `config.mk` |

⚠️ board and patch DTS must be updated together (patch overrides board); top vendor tree is read-only.

## New product (specify BOARD + UI)

When opening a new product **must explicitly provide**:

1. `CONFIG_BOARD=<BoardName>` (= overlay directory name)
2. **UI form**: ipcamera / wearable / tablet (drives `CONFIG_APP_MAIN_TARGET` and `CONFIG_APP_PACKAGES_*`)
3. Single-app placement (`IOTCamera` form → `packages/ipcamera/`; with display → `packages/tablet` / `packages/wearable`)

Naming suggestion: `<form>_<carrier>_<chip_or_module>`, e.g. `PDB_MBD_KM01W_ACTCAM`.

| UI | MAIN_TARGET | GUI | Single-app dir | Mount |
|---|---|---|---|---|
| IOTCamera | `luna_dev_main` (or `CONFIG_APP_MAIN_TARGET`) | `IPCAMERA=y`; tablet/wearable=`n` | `packages/ipcamera/<id>/` SO | `settings.json`: `launch_mode=ipcamera` (also `headless`) + `module_start` keepalive |
| wearable | `luna_main` | `WEARABLE=y` | `packages/wearable/<id>/` | `installed_apps` |
| tablet | `nova_main` (nova_dev also `nova_main`) | `TABLET=y` | `packages/tablet/<id>/` | `installed_apps` |

Steps:

1. `cp -r` baseline `project/<base>/` → `project/<new>/`
2. Top `config.mk`: `CONFIG_SOC_*` + **`CONFIG_BOARD`** + **MAIN/GUI** + capability switches
3. New board: create `board/<BoardName>/` and complete board checklist; reuse: verify SDK directory
4. Land single app per form (IOTCamera → application skill “IOTCamera single app”)
5. **config.mk trim** → `-k/-f/-a/-p` → on-board

Board handling doc (required checklist; details [bsp-and-products](../quick-start/bsp-and-products.en.md) §1/§6):

| Item | Path / action |
|---|---|
| DTS (board+patch sync) | `board/<Board>/os/.../dts/` + `patch/os/.../dts/` |
| rootfs config | `board/<Board>/rootfs/utils/etc/config/` (`fw_*`/`hal_*`/`app_*`) |
| Boot script | nova: `app_01_start.sh`; luna family: `app.sh` etc.; **binary launched = MAIN_TARGET** |
| Board `config.mk` | DTB name, partitions, flash; included by top level |
| defconfig / burntool | kernel options, flash partitions |

AI worksheet: [roles.md §2](../quick-start/roles.en.md#2-新产品移植开发者) (includes table C IOTCamera single app).

## config.mk trim (required after port)

Goal: image only builds capabilities this product uses. Against `installed_apps.json` / `settings.json` / actual App calls, set **unused** switches to `n` (do not delete rows—for easy re-enable).

| Category | Trim principle |
|---|---|
| `CONFIG_APP_PACKAGES_*` | tablet/wearable mutually exclusive; `IOTCamera` form both `n`, main program `luna_dev_main` |
| `CONFIG_APP_*` App/cloud/protocol | Apps/cloud/`RTSP`/`NEBULA`/`UVC` not in `installed_apps` / not wired → `n` |
| `CONFIG_FW_*` | Services with no App/gateway consumer (e.g. media_player) → `n` |
| `CONFIG_HW_*` | No such peripheral on board or App does not call → `n` (keep camera etc. essentials) |
| `CONFIG_THIRD_PARTY_*` | Turn off if unused; size-sensitive: prefer off ffmpeg static link, mbedTLS swap, etc. |

After trim must: correct `config.h` → rebuild affected layers → on-board confirm no regression. When changing switches **do not** sync-edit other products’ `config.mk`.

## New board (same product, new board)

1. `cp -r` nearest board → `board/<NewBoard>/`
2. Top `config.mk` set `CONFIG_BOARD=<NewBoard>`
3. Per board table: DTS / rootfs / boot script / board config / defconfig
4. `./build.sh -k -p --product-name <p>` → on-board via celestia-debug

AI worksheet: [roles.md new product](../quick-start/roles.en.md#2-新产品移植开发者).

## Anyka burntool partition tuning (Nova 16MB SPL fastsys)

On 16MB NOR **maximize APP (`usr.sqsh4`)** first: measure real images then edit tables. CONFIG only needs to fit `usr.jffs2` (+ margin).

**Three places must match** (burn only without env/DTS → `unknown-block` / wrong root; env changed without regenerating `.img` → old table still flashed):

1. **burn** `config_*.txt` → `download_to_nand*` offset/size
2. **env** `env_*.cfg` → `mtdparts` + `kernel_offset`/`kernel_size` + `boot_normal` `mtdblockN`, then generate `.img`
3. **DTS** `chosen/bootargs` → same `mtdparts` (old DTB ROOTFS as `4096K` → panic)

### Nova files to change (SSOT in project overlay)

Root: `project/anypi_nova/system/AnyCloud39AV200/V1.04/`

| Role | File | What to change |
|------|------|--------|
| env (short name, same content as long name) | `patch_fastsys/tools/envtool/env_av200_256MB_action_fastsys.cfg` | see “env keys” table below |
| env (burn filename) | `patch_fastsys/tools/envtool/env_av200_256MB_action_camera_spinor_spl_fastsys.cfg` | **same content as short cfg** (currently same hash) |
| env image | `patch_fastsys/tools/{envtool,burntool}/env_av200_256MB_action_camera_spinor_spl_fastsys.img` | **regenerate** from cfg then copy |
| burn | `patch_fastsys/tools/burntool/config_av200_256MB_action_camera_spinor_spl_fastsys.txt` | `download_to_nand6..9`; ENV/ENVBK filenames must match `.img` |
| burn copy | `patch_fastsys/tools/envtool/config_av200_256MB_action_camera_spinor_spl_fastsys.txt` | same as burntool (UTF-16) |
| DTS | `board/PDB_MBD_AK3918AV200_ACTCAM/os/kernel/.../EVB_CBDM_KM02X_ACTION_FASTSYS_V1.0.0.dts` | `mtdparts=...` + `root=/dev/mtdblock6` in `bootargs` |
| hybrid pack | `patch/scripts/build_rootfs.sh` (sync `system/.../scripts/`) | `/usr` vs SD `install/` placement |

After edits: `out/anypi_nova/burntool/` must carry new `.img` / burn config / `.dtb` (`-k`/`-p` or manual copy). If vendor `system/.../tools/` still referenced by package, sync there too.

> Non-fastsys `EVB_CBDM_KM02X_ACTION_V1.0.1.dts` (UBOOT layout, `mtdblock5`) **do not** mix with fastsys table.

### Which env keys to change

Partition-related keys in `env_*.cfg` (edit both files):

| Key | Current value | Notes |
|----|--------|------|
| `kernel_offset` | `0x52000` | = KERNEL start = after DTB |
| `kernel_size` | `0x380000` | = burn KERNEL size = mtdparts `3584K` |
| `dtb_offset` / `dtb_size` | `0x42000` / `0x10000` | fixed header, usually unchanged |
| `mtdparts` | full string below | **replace entire string**; must match DTS `bootargs` embedded string |
| `boot_normal` | `root=/dev/mtdblock6` … `${mtdparts}` | fastsys: ROOTFS = **mtdblock6** (SPL=0…APP=8) |
| `read_kernel` | uses `${kernel_offset}` `${kernel_size}` | after size change follows vars automatically |

Current full `mtdparts` (in env written as `mtdparts mtdparts=spi0.0:...`):

```
mtdparts=spi0.0:128K@0x0(SPL),128K@0x20000(OPENSBI),4K@0x40000(ENV),4K@0x41000(ENVBK),64K@0x42000(DTB),3584K@0x52000(KERNEL),3584K@0x3D2000(ROOTFS),512K@0x752000(CONFIG),8368K@0x7D2000(APP)
```

### Must regenerate `.img` after env edit

```bash
ENVDIR=project/anypi_nova/system/AnyCloud39AV200/V1.04/patch_fastsys/tools/envtool
cd "$ENVDIR"
# After both cfg files match:
CFG=env_av200_256MB_action_camera_spinor_spl_fastsys.cfg
IMG=env_av200_256MB_action_camera_spinor_spl_fastsys.img
sed -i 's/\r$//' "$CFG" fw_env.config
tr '\000' '\377' < /dev/zero | dd of=./env.img bs=1024 count=4 status=none
./fw_setenv -s "$CFG"
./fw_printenv | grep -E 'mtdparts|kernel_offset|kernel_size'
cp -f env.img "$IMG"
cp -f "$IMG" ../burntool/"$IMG"
# If package still reads system/ tools, copy there too
```

ENV/ENVBK in burn must reference **same name**:

```
download_to_nand3 = env_av200_256MB_action_camera_spinor_spl_fastsys.img, ENV, 0x40000, 0x1000, 0
download_to_nand4 = env_av200_256MB_action_camera_spinor_spl_fastsys.img, ENVBK, 0x41000, 0x1000, 0
```

### burn `download_to_nand` (aligned with env)

| Partition | offset | size (burn) | mtdparts label |
|------|--------|--------------|---------------|
| KERNEL | `0x52000` | `0x380000` (3584K) | `3584K@0x52000` |
| ROOTFS | `0x3D2000` | `0x380000` (3584K) | `3584K@0x3D2000` |
| CONFIG | `0x752000` | `0x80000` (512K) | `512K@0x752000` |
| APP | `0x7D2000` | `0x82E000` (**8577024**) | `8368K@0x7D2000` |

```
download_to_nand6 = uImage, KERNEL, 0x52000, 0x380000, 0
download_to_nand7 = root.sqsh4, ROOTFS, 0x3D2000, 0x380000, 0
download_to_nand8 = usr.jffs2, CONFIG, 0x752000, 0x80000, 0
download_to_nand9 = usr.sqsh4, APP, 0x7D2000, 0x82E000, 0
```

> burn APP size `0x82E000` = `0x1000000 − 0x7D2000` (fill 16MB tail). mtdparts label `8368K` (8KB smaller) coexists with burn; **offsets must be same set in all three places**, and `usr.sqsh4` ≤ burn size.

**UTF-16**: above `config_*.txt` are **UTF-16 LE + BOM**. Writing UTF-8 → burn tool parse fails (falls back to `DDR3_PHY0_720MHZ.bin`). After edit confirm still UTF-16 LE with `file`. Copy burn to local NTFS; do not run Burn Tool directly on WSL/`Z:`/Samba.

### Tuning flow (when changing partitions again)

1. Measure `out/anypi_nova/burntool/` `uImage` / `root.sqsh4` / `usr.jffs2` / `usr.sqsh4`, leave ≥88–256KB each; overflow → `Page:N more than end page`.
2. Do not shrink fixed header (SPL/OPENSBI/ENV/ENVBK/DTB) arbitrarily; on overflow move from partitions with margin (often CONFIG); **do not** compress APP burn size below `usr.sqsh4`.
3. Edit **both** `env_*.cfg` `mtdparts` / `kernel_*` / `boot_normal` → regenerate `.img` → copy to burntool.
4. Edit burn `download_to_nand*` (and envtool same-name txt), keep UTF-16.
5. Edit DTS `bootargs` `mtdparts` + `root=` → `-k` new `.dtb`.
6. `-p` or copy to `out/.../burntool/`, before burn verify: contiguous layout, APP tail=16MB, burn≡env≡DTS, ENV filename, four image types fit.

### hybrid: `/usr` (flash APP) vs `/mnt/install` (SD)

Pack SSOT: `prepare_install_pack_and_exclude_list` (`patch/scripts/build_rootfs.sh`).

| Placement | Contents |
|------|------|
| flash `/usr` → `usr.sqsh4` | `lib/`, `gui/`, `ipc_main`, `apps/setting`, `sbin/`, `tips/` |
| SD `install/` | other tablet apps, `model/`, external `.so`; **not** in `download_to_nand` |

If APP grows: trim `/usr/lib` or move more to `install`, not squeeze ROOTFS. Nova voice SW decode enlarging `libmultimedia.so`: use `LINK_MODE=decode` + `DECODE_ONLY` (see [celestia-third-party](celestia-third-party.en.md)), after `-p` confirm `usr.sqsh4` ≤ `0x82E000`.

Incremental update on running Nova board without Burn Tool. USB app replaces `/mnt/install/apps/<id>` (backup `/mnt/install/.app_bak/<id>`). User partition: write new `usr.sqsh4` via `/bin/mtd_debug erase` + `mtd_debug write` to `"APP"` partition in `/proc/mtd` then `reboot`. Write length must not exceed partition; fastsys on-device APP is `0x8AC000`, if larger than burntool text length use `/proc/mtd` as truth. Full steps for commands and new/edit apps: [celestia-studio](celestia-studio.en.md) and [celestia-scripts](celestia-scripts.en.md) “command download to dev board”.

## FAQ

| Symptom | Root cause | Fix |
|---|---|---|
| config.mk change affects another product | `CONFIG_*` cross-talk | Check other products’ config.mk |
| overlay not applied | edited top vendor tree | only edit `project/<p>/system/.../board/<Board>/` |
| rootfs overlay overridden | `patch/rootfs` over `board/rootfs` | adjust file location per overlay order |
| DTB mismatch | wrong DTB selection | nova: `package.sh` FASTSYS/LCD/DMIC; luna: overlay; do not use deprecated `CONFIG_DTB_NAME` |
| boot `unknown-block(31,6)` / no root mount | DTS/env `mtdparts` vs burn mismatch, or cfg changed **without** regenerating `.img` | align both `env_*.cfg` → `fw_setenv` `.img` → sync burn/DTS; reflash ENV+DTB |
| wrong main at boot | script binary ≠ `CONFIG_APP_MAIN_TARGET` | align boot script with MAIN_TARGET (nova→`ipc_main`, nova_dev→`nova_main`, luna→`luna_main`, luna_dev→`luna_dev_main`, mira→`mira_main`) |
| SDK version vs overlay | `CONFIG_SOC_SDK_VERSION` ≠ overlay dir | align version |
| toolchain error | toolchain by product name not platform | AV200 `riscv64-…` / AV100_V2 `arm-…` |
| bad `config.h` | config.mk syntax / missing product | verify config.mk + `project/<p>/` |
| missing features after trim | turned off in-use `CONFIG_*` | re-enable per settings.json/installed_apps and serial log |
| burn `Page more than end page` (ROOTFS) | `root.sqsh4` > ROOTFS burn size | expand ROOTFS (sync env `mtdparts`+burn+DTS) or shrink rootfs |
| burn `Page more than end page` (APP, part=`0x82E000`) | `usr.sqsh4` > APP; or burn size not filling tail | confirm APP=`0x82E000`; trim `/usr` or `DECODE_ONLY`; see section above |
| burn looks for `DDR3_PHY0_720MHZ.bin` | burn config UTF-8 / run on network drive | restore UTF-16 LE; copy to local NTFS |
| ENV still old partition after flash | only changed `.cfg` / short cfg, burn still references old `.img` | regenerate `env_av200_256MB_action_camera_spinor_spl_fastsys.img` and copy to burntool |
| call no video / `VIDEO_DECODER_SW_STUB` | `FFMPEG_LINK_MODE=none` | Nova use `decode` + DECODE_ONLY; see [celestia-third-party](celestia-third-party.en.md) |

## After coding → build (required)

```bash
cd ~/celestia
./build.sh -k --product-name anypi_nova   # kernel / BSP only (system/<SoC>/<SDK>)
./build.sh -p --product-name anypi_nova   # package only (rootfs/images, skip compile)
./build.sh    --product-name anypi_luna        # after config.mk change prefer full + package verify
```

After `config.mk` changes run at least affected layers (e.g. `CONFIG_FW_*` → `-f`, `CONFIG_APP_*` → `-a`), or full build to verify `config.h` and packable images. `config.mk not found` → confirm `--product-name` / cache / interactive pick points to existing `project/<proj>/`.