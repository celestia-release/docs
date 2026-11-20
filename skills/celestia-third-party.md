---
name: celestia-third-party
description: >-
  Celestia 第三方库层（third_party/）：跨 SoC 共用的外部依赖，由 build.sh -t 交叉编译，CONFIG_THIRD_PARTY_*
  在 project/<proj>/config.mk 中开关。含 lvgl(GUI)、ffmpeg(音视频)、openssl/mbedtls(TLS/加密)、
  curl/libwebsockets(网络)、cjson/iniparser/minIni(配置解析)、freetype/jpeg/jpegdec/png/zlib(字体图像)、
  tslib(触摸)、iconv(编码)、zxing(二维码)、asr(语音)、awtk/jerryscript 等。含 FFmpeg DECODE_ONLY
  （Nova 16MB NOR 通话 H.264）。构建顺序最先（system/core、framework 依赖它）。
  当新增/升级第三方库、排查库链接/版本/交叉编译错误、APP 分区因 libav* 膨胀时使用。
---

<p align="right"><a href="celestia-third-party.en.md">English</a> | <b>中文</b></p>

# Third-Party 层 skill

> 版本清单：`docs/quick-start/bsp-and-products.md §3.5`。总览见 [celestia](celestia.md)。

## 定位

`third_party/` 是跨 SoC 共用的外部依赖，**构建链最底层**（`① third_party` 在 system/core / framework / application 之前）。各库经 `build.sh -t` 用项目工具链交叉编译，产物（`.a`/`.so`/头）供上层链接。开关在 `project/<proj>/config.mk` 的 `CONFIG_THIRD_PARTY_*`。

```
third_party/
├── Makefile  vendor_rules.mk        # 统一构建规则
├── lvgl ffmpeg openssl mbedtls curl libwebsockets
├── cjson iniparser minIni
├── freetype jpeg jpegdec libpng zlib tslib iconv
├── zxing asr awtk jerryscript
```

主要库、用途与在用版本（节选）：lvgl 9.3.0（GUI，framework/ui + 应用层依赖）、ffmpeg 7.0（多媒体；Nova 通话用 DECODE_ONLY）、mbedtls 2.28.10 / openssl 3.1.5（加密/TLS）、curl 8.3.0 / libwebsockets 4.4（网络）、cjson 1.7.15（`system/core` 自检依赖其头 + framework `liblvgl.a` 链接）、iniparser（vendored/内建，版本未标注）、minIni（INI 解析）、freetype 2.12.0 / jpeg 9e / jpegdec / libpng 1.6.43 / zlib 1.3.1（字体图像）、tslib 1.22（触摸）、iconv 1.15（编码）、zxing（二维码）、asr（语音）、awtk / jerryscript。

## FFmpeg 与 16MB NOR（Nova）

| `config.mk` | 效果 |
|-------------|------|
| `CONFIG_THIRD_PARTY_FFMPEG_MINIMAL=y` | 常用 demux/decoder/protocol，仍偏大 |
| `CONFIG_THIRD_PARTY_FFMPEG_DECODE_ONLY=y` | **覆盖** MINIMAL：仅 `h264` parser/decoder，无 avformat/network；`--enable-small`/`-Os`/`--disable-rvv` |
| `CONFIG_FRAMEWORK_MULTIMEDIA_FFMPEG_LINK_MODE` | `none`=stub（通话无画面）；`decode`=链 SW 解码、无 file demux；`normal`/`whole`=更大 |

Nova 通话 LCD 下行（无 MPI VDEC）：`DECODE_ONLY=y` + `LINK_MODE=decode`。改后必须 `-t`（至少 ffmpeg）+ `-f` + `-p`，确认 `out/.../burntool/usr.sqsh4` ≤ APP（`0x82E000`）。细节见 [celestia-project](celestia-project.md) §分区。

## 关键依赖关系（决定排障方向）

- `system/core` 依赖 cjson 头；`framework` 依赖 lvgl(`liblvgl.a`) + cjson。`build.sh` 在建 framework/system-core 时会自检这些产物，缺失自动先 `build_third_party()`。
- 因此上层报“缺 `cJSON.h` / `liblvgl.a` / 某符号未定义”时，**先确认对应 third_party 库已成功构建**，再回上层。

## 排障 / 升级时

1. 库链接/找不到头/版本不符 → 本层：确认该库已 `-t <module>` 构建出产物、`CONFIG_THIRD_PARTY_<X>` 已开、工具链前缀正确（nova（AV200）`riscv64-anyka-linux-musl-`，luna（AV100_V2）`arm-anycloud-linux-uclibcgnueabi-`）。
2. 升级/新增库：放入 `third_party/<lib>/`，按 `vendor_rules.mk` 接入 Makefile，在 `config.mk` 加 `CONFIG_THIRD_PARTY_<X>=y`，单独 `-t <lib>` 编过后再让上层链接。
3. 升级库可能改变 ABI/头 → 必须回归依赖该库的上层（framework/application）能否编过与运行。
4. ⚠️ 第三方库通常是 vendored 源码，改动应优先通过本项目的 patch/构建参数，而非散改库源码。

## 编码完成 → 编译（必做）

```bash
cd ~/celestia
./build.sh -t lvgl --product-name anypi_luna     # 单库
./build.sh -t --product-name anypi_luna          # 全部 third_party
# 升级库后回归依赖方：
./build.sh -t -s -f -a --product-name anypi_luna
```

解决全部交叉编译错误，并确认**依赖该库的上层仍能编过/链接通过**再交付。nova 与 luna 工具链不同（RISC-V/musl vs ARM/uclibc），受影响时相关产品都验证。
