---
name: celestia-scripts
description: >-
  Celestia build and scripts layer (scripts/, build.sh): build.sh layered entry (-s system-core /
  -t third-party / -h hardware / -f framework / -a application / -k kernel / -p package / -c clean,
  --product-name, --jobs) and dependency order. Use when compiling, pushing to board via commands or
  burntool, serial bring-up and logs, or changing/debugging build scripts.
---

<p align="right"><b>English</b> | <a href="celestia-scripts.md">中文</a></p>

# Scripts / build layer skill

> Docs: `docs/quick-start/bsp-and-products.md §4`.
> Luna bring-up: [celestia-project](celestia-project.en.md) “wearable bring-up”. Overview: [celestia](celestia.en.md).

## Where to run (important)

Build and package in a **Linux build environment** (WSL or native Linux), project path `~/celestia`.
**Do not run build.sh from Windows paths** (e.g. PowerShell `Z:\...`).
Images land in `out/<product>/burntool/` for full Burn Tool flash. For Nova daily app or `/usr` updates, use on-device commands — see “Push to board without burntool” and [celestia-studio](celestia-studio.en.md).

## build.sh — layered build entry

`build.sh` at repo root (same as `scripts/build.sh`). No layer flag = full build + package.

| flag | long option | Action |
|------|--------|------|
| `-s` | `--system-core` | system/core |
| `-t` | `--third-party [module]` | third_party (optional single module, e.g. `-t lvgl`) |
| `-h` | `--hardware` | hardware HAL |
| `-f` | `--framework` | framework |
| `-a` | `--application` | application |
| `-k` | `--kernel` | SoC platform / kernel (`system/<SoC>/<SDK>`) |
| `-p` | `--package` | rootfs / image packaging |
| `-c` | `--clean` | clean selected layers (all if no layer flag) |

Other options: `--product-name anypi_nova|anypi_nova_dev|anypi_luna|anypi_luna_dev|anypi_mira_dev` (interactive + cache to `out/.build_product_name` if omitted), `--jobs N`, `--log-level DEBUG|INFO|ERROR`, `--build-level DEBUG|RELEASE`, `--release [path]`, `--copy-build-src` (minimal source copy to `out/<product>/release-src/`, **no compile**). Toolchains by platform: nova (AV200) `riscv64-anyka-linux-musl-`, luna (AV100_V2) `arm-anycloud-linux-uclibcgnueabi-`, mira (Sky39AV130, aos) `arm-none-eabi-` via `project/anypi_mira_dev/setup_toolchain.sh` (auto-sourced when `CONFIG_OS_NAME=aos`).

Build order: `① third_party → ② system/core → ③ hardware → ④ framework → ⑤ application → ⑥ package`. `build_framework`/`build_system_core` self-check `liblvgl.a`/`cJSON.h` and auto-build third_party if missing. Combine layers in order: `./build.sh -h -f -a`.

```bash
cd ~/celestia
./build.sh                                         # interactive product + cache
./build.sh --product-name anypi_luna               # full luna
./build.sh -f -a --jobs 4 --product-name anypi_luna
./build.sh -c -f                                   # clean then framework only
./build.sh --copy-build-src --product-name anypi_luna
```

After package, full flash and serial loop: [celestia-project](celestia-project.en.md) and [celestia-debug](celestia-debug.en.md). App-only or APP partition updates use commands below, not burntool.

## Push to board without burntool

On Nova (`anypi_nova`) with studio listening on TCP 7777, use sync tool or serial/`EXEC`. Storage must be locally mounted: `/mnt/install` must exist. In USB storage mode the SD is exported to the PC and those commands fail.

**USB tablet apps** (except `setting`) — replace directory, no flash write:

```sh
# Package at /tmp/sync_push.tar.gz: app.json, res/, lib/*.so
DEST=/mnt/install/apps/<id>
rm -rf /tmp/sync_pkg "$DEST.bak"
mkdir -p /tmp/sync_pkg /mnt/install/.app_bak
tar -xzf /tmp/sync_push.tar.gz -C /tmp/sync_pkg
BAK=/mnt/install/.app_bak/<id>
rm -rf "$BAK"
[ -d "$DEST" ] && mv "$DEST" "$BAK"
mv /tmp/sync_pkg "$DEST"
chmod -R a+rX "$DEST"
```

Backup under `/mnt/install/.app_bak/`, not `apps/<id>.bak` (would register a second app). Relaunch UI: `killall nova_main ipc_main ipc_home`, then `/usr/sbin/start_ipc_main.sh --bg`.

**User area** `/usr` (includes `apps/setting` and `/usr/lib/libstudio.so`) is squashfs. After rebuilding `usr.sqsh4`, write the partition named `"APP"` in `/proc/mtd`:

```sh
grep '"APP"' /proc/mtd
# fastsys example: mtd8  size 0x008ac000
mtd_debug erase /dev/mtd8 0 <partition bytes>
mtd_debug write /dev/mtd8 0 <image bytes> /tmp/usr.sqsh4
sync
reboot
```

Do not write if image exceeds partition. Trust `/proc/mtd` size (example `0x8AC000`), not larger burntool text `0x8AE000`. Do not truncate-overwrite mapped `/usr/lib/*.so`. Full create/edit/push: [celestia-studio](celestia-studio.en.md).

## Minimal source release

`scripts/release_src.sh` copies only enabled modules per `project/<product>/config.mk` to `out/<product>/release-src/`, strips disabled modules, other SoC `hal/`, other products, `.git`/build artifacts. Same as `build.sh --copy-build-src` (invokes it, skips compile).

```bash
bash scripts/release_src.sh --product-name anypi_luna                 # no vendor BSP by default
bash scripts/release_src.sh --product-name anypi_luna --with-vendor-bsp
bash scripts/release_src.sh --product-name anypi_luna,anypi_nova,anypi_mira_dev --dest out/release-open
bash scripts/release_src.sh --product-name anypi_nova --with-cloud
```

> Vendor BSP (`system/<SoC>/<SDK>/`) not released by default; place before build or use `--with-vendor-bsp`.
> rootfs path derivation is inlined in `application/` and `hardware/input/evdev/` Makefiles; `scripts/` is now `build.sh` + `release_src.sh` + license/readme.

## Troubleshooting / script changes

1. On compile failure, read build.sh output for layer and command; **source bug** → layer skill; **script/path** → this layer.
2. Changing `build.sh` is build-logic change: keep layer flag semantics and dependency order; run `--help` and rebuild affected layers.
3. On-device PASS/FAIL: [celestia-debug](celestia-debug.en.md) (preview ≥20s key logs).

## Common build issues

| Symptom / log | Root cause | Fix |
|---|---|---|
| Source change not on board | stale `out/` / wrong product | `-c` rebuild; confirm `--product-name` |
| build.sh on Windows path | not Linux build env | use WSL/`~/celestia` |
| Missing lvgl/cjson link | third_party not built | `-t` or let self-check run |
| Single layer still links old .o | dependent layers not cleaned | `-c -h -f -a` in dependency order |
| package missing files | overlay/switches | check config.mk + board rootfs |
| APP flash `Page more than end page` | `usr.sqsh4` > burn APP (Nova=`0x82E000`) | `-p` then `stat out/<p>/burntool/usr.sqsh4`; trim FFmpeg/`install/`; [celestia-project](celestia-project.en.md) §partitions |

Runtime: [celestia-debug](celestia-debug.en.md) **classes D/F**.

## Build after coding (required)

After script changes, **run** the command for real:

```bash
cd ~/celestia
./build.sh --help
./build.sh -a --product-name anypi_luna            # 0 errors on affected layer
```

Deliver only when scripts pass in the build environment and outputs match expectations.