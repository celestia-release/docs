---
name: celestia-third-party
description: >-
  Celestia third-party layer (third_party/): cross-SoC external dependencies, cross-built via
  build.sh -t, toggled by CONFIG_THIRD_PARTY_* in project/<proj>/config.mk. Includes lvgl (GUI),
  ffmpeg (AV), openssl/mbedtls (TLS/crypto), curl/libwebsockets (network), cjson/iniparser/minIni
  (config), freetype/jpeg/jpegdec/png/zlib (fonts/images), tslib (touch), iconv, zxing, asr, awtk/jerryscript.
  FFmpeg DECODE_ONLY for Nova 16MB NOR H.264 calls. Built first (system/core, framework depend on it).
  Use when adding/upgrading libs, link/version/cross-compile errors, or APP partition bloated by libav*.
---

<p align="right"><b>English</b> | <a href="celestia-third-party.md">中文</a></p>

# Third-Party layer skill

> Version list: `docs/quick-start/bsp-and-products.md §3.5`. Overview: [celestia](celestia.en.md).

## Role

`third_party/` holds **cross-SoC external deps**, **bottom of the build chain** (`① third_party` before system/core / framework / application). Each lib is cross-built via `build.sh -t` with the product toolchain; artifacts (`.a`/`.so`/headers) link into upper layers. Switches in `project/<proj>/config.mk` as `CONFIG_THIRD_PARTY_*`.

```
third_party/
├── Makefile  vendor_rules.mk        # unified build rules
├── lvgl ffmpeg openssl mbedtls curl libwebsockets
├── cjson iniparser minIni
├── freetype jpeg jpegdec libpng zlib tslib iconv
├── zxing asr awtk jerryscript
```

Major libs, purpose, versions in use (excerpt): lvgl 9.3.0 (GUI, framework/ui + apps), ffmpeg 7.0 (multimedia; Nova calls use DECODE_ONLY), mbedtls 2.28.10 / openssl 3.1.5 (crypto/TLS), curl 8.3.0 / libwebsockets 4.4 (network), cjson 1.7.15 (system/core header check + framework `liblvgl.a` link), iniparser (vendored/bundled, version N/A), minIni (INI), freetype 2.12.0 / jpeg 9e / jpegdec / libpng 1.6.43 / zlib 1.3.1 (fonts/images), tslib 1.22 (touch), iconv 1.15, zxing, asr, awtk / jerryscript.

## FFmpeg and 16MB NOR (Nova)

| `config.mk` | Effect |
|-------------|------|
| `CONFIG_THIRD_PARTY_FFMPEG_MINIMAL=y` | Common demux/decoder/protocol, still large |
| `CONFIG_THIRD_PARTY_FFMPEG_DECODE_ONLY=y` | **Overrides** MINIMAL: `h264` parser/decoder only, no avformat/network; `--enable-small`/`-Os`/`--disable-rvv` |
| `CONFIG_FRAMEWORK_MULTIMEDIA_FFMPEG_LINK_MODE` | `none`=stub (no video in calls); `decode`=SW decode, no file demux; `normal`/`whole`=larger |

Nova call LCD downlink (no MPI VDEC): `DECODE_ONLY=y` + `LINK_MODE=decode`. After changes: `-t` (at least ffmpeg) + `-f` + `-p`, confirm `out/.../burntool/usr.sqsh4` ≤ APP (`0x82E000`). Details: [celestia-project](celestia-project.en.md) §partitions.

## Key dependencies (drives triage)

- `system/core` needs cjson headers; `framework` needs lvgl (`liblvgl.a`) + cjson. `build.sh` self-checks when building framework/system-core and auto-runs `build_third_party()` if missing.
- Upper-layer “missing `cJSON.h` / `liblvgl.a` / undefined symbol” → **confirm third_party built first**, then return to upper layer.

## Troubleshooting / upgrades

1. Link/header/version issues → this layer: `-t <module>` built, `CONFIG_THIRD_PARTY_<X>` on, toolchain prefix correct (nova AV200 `riscv64-anyka-linux-musl-`, luna AV100_V2 `arm-anycloud-linux-uclibcgnueabi-`).
2. Upgrade/add lib: place under `third_party/<lib>/`, wire Makefile per `vendor_rules.mk`, add `CONFIG_THIRD_PARTY_<X>=y` in `config.mk`, `-t <lib>` then upper layers.
3. Upgrades may change ABI/headers → regression-build framework/application.
4. ⚠️ Libraries are usually vendored; prefer project patches/build flags over scattered upstream edits.

## Build after coding (required)

```bash
cd ~/celestia
./build.sh -t lvgl --product-name anypi_luna     # single lib
./build.sh -t --product-name anypi_luna          # all third_party
# After lib upgrade, regression:
./build.sh -t -s -f -a --product-name anypi_luna
```

Fix all cross-compile errors; confirm **consumers still link/build** before delivery. nova vs luna toolchains differ (RISC-V/musl vs ARM/uclibc) — verify affected products.