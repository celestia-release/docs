<p align="right"><b>English</b> | <a href="directory-map.md">中文</a></p>

# Directory Map

> Answers two questions: **what each directory is responsible for** and **where new files belong**.
> Terminology follows the [Glossary](../glossary.en.md); layering contracts and the runtime model are in [Architecture Overview](overview.en.md).

## 1. Top-Level Directory Responsibilities

```text
celestia/                        # Source root
├── build.sh -> scripts/build.sh # Sole build entry point
├── application/                 # ① Application layer
├── framework/                   # ② Framework layer
├── hardware/                    # ③ Hardware adaptation layer
├── system/core/                 # ④ System foundation libraries
├── project/<product>/           # ⑤ Product engineering (config + overlay)
├── scripts/                     # Build scripts
├── third_party/                 # Third-party components
└── docs/                        # Documentation (this tree)
```

| Directory | Layer | Responsibility | What must not appear |
|:----------|:------|:---------------|:---------------------|
| `application/` | ① | Main program assembly, GUI app packages, system services, cloud integration | Direct `/dev/*` access, includes of private HAL headers |
| `framework/` | ② | Multimedia, network, device, notification, storage, runtime, gateway protocols | Product branches, `#ifdef CONFIG_SOC_*`, chip-private headers |
| `hardware/` | ③ | Peripherals and media capabilities per SoC, unified ops upward | Business logic, GUI, references to upper-layer symbols |
| `system/core/` | ④ | Logging, config, threads, timers, error codes, memory, crypto, containers, etc. | Any dependency on ①②③ |
| `project/` | ⑤ + config | One `config.mk` and BSP overlay, rootfs config, packaging scripts per product | Shared business code |
| `scripts/` | Build | `build.sh` body, packaging, toolchain injection | Runtime logic |
| `third_party/` | Dependencies | Third-party library sources and build rules | Business code, product switches |
| `docs/` | Documentation | This documentation set | Code |

## 2. Key Subdirectories

### 2.1 `application/`

| Path | Description |
|:-----|:------------|
| `app_main/` | Main program: startup, config load, service registration, SO module load, form-factor branching |
| `packages/tablet/` | **Tablet-class (`TABLET`)** desktop and built-in apps (`ipc_camera` `camera` `setting` `clock` `file_manager` `audio_algo` `volc_engine`) |
| `packages/wearable/` | **Wearable-class (`WEARABLE`)** UI (launcher, camera, dashcam, power_keyd) |
| `packages/ipcamera/` | **`IOTCamera`** form-factor business SOs (`ipc_cam`), export `module_init` / `module_start` |
| `packages/common/` | GUI components shared by tablet / wearable |
| `services/` | Always-on services unrelated to UI (`weather`, `uvc`), linked static |
| `cloud/` | Cloud platform adapters (`aliyun`, `volc_engine`), hot-pluggable SOs |
| `tests/` | Application-layer self-tests |

### 2.2 `framework/`

| Path | Description |
|:-----|:------------|
| `services/multimedia/` | Preview / capture / record / playback / record-audio / push-pull streaming; subdirs `camera` (`device` `core` `capture` `af` `ai` `osd`), `media` (`record` `play`), `av_codec`, `audio` |
| `services/network/` | Wi-Fi, Ethernet, cellular; credential encrypt/decrypt |
| `services/device/` | Backlight, PTZ, night vision, LED, power |
| `services/notification/` | Keys, alarms, input subsystem |
| `services/storage/` | Flash / TF card, space and mount |
| `gateway/rtsp_protocol/` | RTSP server |
| `gateway/nebula_protocol/` | Nebula signaling and intercom |
| `runtime/` | bootstrap, console, security, OTA, watchdog, power_button |
| `ui/` | LVGL porting layer (`graphics` / `lvgl` / `awtk`) |
| `tests/` | Framework-layer self-tests |

### 2.3 `hardware/`

| Path | Description |
|:-----|:------------|
| `include/` | **Public HAL headers (sole cross-layer contract)**: `audio.h` `camera.h` `display.h` `wifi.h` `gpio.h` `hardware.h` … |
| `audio/` `camera/` `display/` `input/` `network/` `osd/` `gpu/` | Capability modules; implementations split under `hal/<full SoC name>/` (e.g. `camera/hal/AnyCloud39AV200/`) |
| `vision/isp/` | ISP tuning (per-SoC impl under `hal/`) |
| `vision/motion_detect/` | Motion detection |
| `driver/` | gpio / adc / pwm / pwd / led / ir / wdt / efuse / blkdev / mmc / motor |
| `soc_platform/` | Platform abstraction (boot, memory, clocks, etc.) |
| `hardware.c` | Implementation of `hw_get_module()` / `hw_device_open()` |
| `tests/` | Adaptation-layer self-tests |

### 2.4 `system/core/`

`log` `config` `thread` `time` `watchdog` `memory` `str` `file` `crypto` `geometry`
`image_convert` `include/`

> Threads uniformly use `thread_create(&id, fn, param, name, stack_size, priority)`; default stack 100KB.

### 2.5 `project/<product>/`

| Path | Description |
|:-----|:------------|
| `config.mk` | **Sole product configuration entry** |
| `include/` | Hand-written product headers, not generated: `app_globle_config.h`, `app_product_config.h`, `lv_conf.h` |
| `system/<SoC>/<SDK>/board/<BOARD>/` | Board overlay: `config.mk` (DTB, partitions), `rootfs/` (`settings.json`, `fw_*.ini`, startup scripts) |
| `system/<SoC>/<SDK>/patch/` | SDK source patches |
| `system/<SoC>/<SDK>/patch_fastsys/` | Fast-boot patches and tools (only when `CONFIG_CELESTIA_FASTSYS=y`) |
| `package.sh` / `build_soc_system.sh` | Packaging and system image build |
| `setup_toolchain.sh` | AOS product toolchain injection (`anypi_mira_dev`) |

> `config.h` **generated** from `config.mk` is not in the source tree; it is written to
> `out/<product>/build/project/include/config.h`.

### 2.6 `third_party/`

`lvgl` `awtk` `ffmpeg` `mbedtls` `openssl` `curl` `libwebsockets` `cjson` `iniparser`
`minIni` `freetype` `jpeg` `jpegdec` `libpng` `zlib` `iconv` `tslib` `zxing` `jerryscript` `asr`

> Built uniformly via `third_party/Makefile` + `vendor_rules.mk`, per `CONFIG_THIRD_PARTY_*` and
> `CONFIG_THIRD_PARTY_TARGET_ARCH` (`arm` / `riscv`). **Do not** embed product logic inside library sources.

## 3. Where to Put New Files (Decision Table)

| What you add | Put it here | Rebuild main binary? |
|:-------------|:------------|:---------------------|
| Tablet-class (`TABLET`) app | `application/packages/tablet/<app>/` | No (SO) |
| Wearable-class (`WEARABLE`) app | `application/packages/wearable/<app>/` | No (SO) |
| IOTCamera business module | `application/packages/ipcamera/<name>/` | No (SO) |
| Cloud platform | `application/cloud/<vendor>/` | No (SO) |
| Always-on service | `application/services/<svc>/` | Yes (static link) |
| Framework capability | `framework/services/<svc>/` (with `include/*.h`) | Yes |
| External protocol | `framework/gateway/<proto>/` | Yes |
| HAL impl for new SoC | `hardware/<module>/hal/<full SoC name>/` | Yes |
| Public HAL API | `hardware/include/*.h` | Yes |
| Foundation library capability | `system/core/<module>/` | Yes |
| New product engineering | `project/<product>/` (`config.mk` + `system/<SoC>/<SDK>/board/<BOARD>/`) | Yes |
| New third-party library | `third_party/<lib>/` + `third_party/Makefile` | Yes |
| Documentation | `docs/` (structure in [README](../README.en.md)) | — |

## 4. Naming Conventions Quick Reference

| Object | Convention | Example |
|:-------|:-------------|:--------|
| Product directory | `anypi_<codename>[_dev]` | `anypi_nova_dev` |
| Board type | `<EVB\|PDB>_<MBD\|CBD>_<chip/module>[_<form>]` | `EVB_CBD_AK3918AV130` |
| GUI package directory | Named by **form factor**, not product | `tablet` / `wearable` / `ipcamera` |
| Config macro | `CONFIG_<domain>_<module>_<switch>` | `CONFIG_FW_MULTIMEDIA_CAMERA_OSD` |
| Framework public header | `framework/**/include/<svc>.h` | `framework/services/network/include/network.h` |
| SO module entry | `module_init()` / `module_start()` | `application/packages/ipcamera/ipc_cam/` |

Full rules (with good / bad examples) in [glossary §11](../glossary.en.md#11-identifier-naming-conventions).

## 5. Related Documentation

- [Architecture Overview](overview.en.md) — layering, contracts, runtime, startup, build
- [Product Form Factor](form-factor.en.md) — criteria, startup branching, selection
- [Glossary and naming](../glossary.en.md) — `AOS` / `IOTCamera` / `AI_GLASSES` / `TABLET` / `WEARABLE` / EVB / PDK
- [Download source](../quick-start/download.en.md) — obtain source and SDK placement
- [BSP and new products](../quick-start/bsp-and-products.en.md) — add board / add SoC
- [Conventions](../guide/conventions.en.md) — unified doc and code conventions