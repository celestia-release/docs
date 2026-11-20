<p align="right"><b>English</b> | <a href="roles.md">中文</a></p>

# Role-based guide (AI collaboration entry)

> This doc is an **AI pair-programming worksheet**: per-role “code generation worksheets” and prompt templates.
> Layout conventions / API boundaries / build commands / debug detail → matching [skills](../skills/README.en.md); concepts and APIs → [guide](../guide/api-index.en.md); **code wins** when docs disagree.

## 1. How to work with AI

**Fast path**: pick role → copy one line from “Prompt scenarios” → paste the worksheet for end-to-end work → AI reads skills, implements, fills the sheet back → build and flash.

1. **Pick role** → paste the worksheet → ask AI to **advance item by item and fill paths/params/status/verify**
2. AI **reads that row’s primary skill** (`docs/skills/<name>.md`) first, then guide / code as needed
3. After edits **build + flash**: matching `build.sh` flags → flash → serial PASS/FAIL ([celestia-debug](../skills/celestia-debug.en.md))
4. Commits follow [git-gitee](../skills/git-gitee.en.md) (one repo per squash, default branch **`main`** on GitHub)

**Prompt template**: `Follow the worksheet step by step with the matching skill; after edits run -k/-f/-a/-p build, package, and verify on board: <paste table>`

Trigger skills: `/celestia-application` etc. (links in [skills/README](../skills/README.en.md)), or have AI read `docs/skills/<name>.md` directly. When unsure of layer, start with `/celestia`.

| Role | One-liner | Scenarios |
|---|---|---|
| New product | Copy baseline product + change config/board | [§2](#2-new-product-port-developer) |
| Application | New app / streams / cloud | [§3](#3-application-developer) |
| New chip | Chip SDK → per-module HAL + BSP | [§4](#4-new-soc-port-developer) |
| Debug | Paste log → A–G class → close loop | [§5](#5-debug-and-maintenance) |

| skill | Source | Companion guide / docs |
|---|---|---|
| `/celestia` | [celestia.md](../skills/celestia.en.md) | [architecture/overview](../architecture/overview.en.md) |
| `/celestia-application` | [celestia-application.md](../skills/celestia-application.en.md) | [application](../guide/application.en.md)、[framework-api](../guide/framework-api.en.md) |
| `/celestia-framework` | [celestia-framework.md](../skills/celestia-framework.en.md) | [framework](../guide/framework.en.md)、[framework-api](../guide/framework-api.en.md) |
| `/celestia-hardware` | [celestia-hardware.md](../skills/celestia-hardware.en.md) | [hardware-hal](../guide/hardware-hal.en.md)、[hardware-api](../guide/hardware-api.en.md) |
| `/celestia-system-core` | [celestia-system-core.md](../skills/celestia-system-core.en.md) | [system-core](../guide/system-core.en.md)、[system-core-api](../guide/system-core-api.en.md) |
| `/celestia-project` | [celestia-project.md](../skills/celestia-project.en.md) | [bsp-and-products](bsp-and-products.en.md) |
| `/celestia-scripts` | [celestia-scripts.md](../skills/celestia-scripts.en.md) | [build](build.en.md) |
| `/celestia-third-party` | [celestia-third-party.md](../skills/celestia-third-party.en.md) | — |
| `/celestia-debug` | [celestia-debug.md](../skills/celestia-debug.en.md) | [deploy](deploy.en.md) |
| `/celestia-contributing` | [celestia-contributing.md](../skills/celestia-contributing.en.md) | — |
| `/git-gitee` | [git-gitee.md](../skills/git-gitee.en.md) | — |

On-board sequence: [download](download.en.md) → [build](build.en.md) → [deploy](deploy.en.md); architecture [overview](../architecture/overview.en.md).

| Goal | Primary skill | Section |
|---|---|---|
| New product / new board assembly | `/celestia-project` | [§2](#2-new-product-port-developer) |
| Add app / cloud adapter / GUI | `/celestia-application` | [§3](#3-application-developer) |
| New SoC: SDK + per-module HAL + BSP | `/celestia-hardware` (+ `/celestia-system-core` / `/celestia-project`) | [§4](#4-new-soc-port-developer) |
| On-board debug / crash | `/celestia-debug` | [§5](#5-debug-and-maintenance) |

Status: ☐ todo / ⏳ in progress / ✅ done; verification = `PASS` or `FAIL + symptom`. Done only when all ✅ and on-board PASS.

---

## 2. New product port developer

> Assemble a new product / board on an existing SoC. Detail: [celestia-project](../skills/celestia-project.en.md) (**BOARD**, **UI form factor**, **board bring-up**, **config.mk trim**), [bsp-and-products](bsp-and-products.en.md).

**Declare up front when opening a product** (all required):

| Input | Example | Notes |
|---|---|---|
| Baseline product | `anypi_luna` | `cp -r` source |
| New product name | `anypi_luna_dev` | `project/<name>/` |
| **BOARD name** | `EVB_MBD_KM01A` | = `CONFIG_BOARD` = overlay dir; prefer native `system/AnyCloud39AV100_V2` content |
| **UI form factor** | `IOTCamera` (no display) / `AI_GLASSES` (WIP) / `TABLET` / `WEARABLE` | see “UI and single-app choice” |
| Capability list | RTSP, nebula (incl. talk), AI, OSD, **single app=`ipc_cam`** | where single app lives depends on form factor |

### UI and single-app choice (form factor first)

| Product dir | `CONFIG_APP_MAIN_TARGET` (per product `config.mk`) | packages switches | Where to put single app (recommended) | How to attach |
|---|---|---|---|---|
| `anypi_luna_dev` (IOTCamera) | `luna_dev_main` | `PACKAGES_IPCAMERA=y`; wearable/tablet=`n` | **`application/packages/ipcamera/<id>/` SO** | `settings.json`: `launch_mode=ipcamera` (or `headless`) + single `type:so`; `module_start` **blocking keep-alive** |
| `anypi_luna` (`WEARABLE`) | `luna_main` | `PACKAGES_WEARABLE=y` | `application/packages/wearable/<id>/` | `installed_apps` (in settings.json) + launcher |
| `anypi_nova` (`TABLET`) | `ipc_main` | `PACKAGES_TABLET=y` | `application/packages/tablet/<id>/` | same |
| `anypi_nova_dev` | `nova_main` | `PACKAGES_TABLET=y` | same | same |

Main binary name comes from **`project/<product>/config.mk` `CONFIG_APP_MAIN_TARGET`**; if unset, `application/app_main/Makefile` derives from directory name.

**IOTCamera must not use `packages/wearable`**: desktop apps need LVGL; headless needs SO `module_start` keep-alive. Contract in `app_config.c`; business logic can mirror `packages/wearable/dashcam/` (strip LVGL).

**`anypi_luna_dev` (no UI + single app) recommended steps**: ① baseline luna → product + BOARD=`EVB_MBD_KM01A` + **ipcamera**; ② `luna_dev_main`, disable wearable/tablet; ③ **new** `application/packages/ipcamera/ipc_cam/` SO (`module_init`/`module_start` keep-alive; dashcam minus GUI); ④ `app_runtime` loads only this SO; ⑤ **no** `installed_apps`; ⑥ boot script runs `luna_dev_main` → trim config → flash. See **worksheet C**.

**Needs → skill (quick ref)**

| Need | Primary skill |
|---|---|
| Baseline / SoC / **BOARD** / overlay | `/celestia-project` |
| **UI** / main binary / single-app placement | `/celestia-project` + `/celestia-application` |
| IOTCamera single SO (RTSP/nebula/AI) | `/celestia-application` (IOTCamera contract) |
| Board bring-up / config trim | `/celestia-project` |
| HAL / fw / gateway | `/celestia-hardware` · `/celestia-framework` |
| Build / flash / commit | `/celestia-scripts` · `/celestia-debug` · `/git-gitee` |

### Prompt scenarios (copy-paste)

| Scenario | Ask AI |
|---|---|
| **IOTCamera + single app** | `anypi_luna_dev: no UI, single app ipc_cam. luna_dev_main + PACKAGES_IPCAMERA; app in packages/ipcamera/ipc_cam; app_runtime one SO only; worksheet C` |
| Single app with display (wearable) | `WEARABLE + dashcam only: luna_main+PACKAGES_WEARABLE; mirror packages/wearable/dashcam; installed_apps one entry` |
| New product with BOARD | `From anypi_luna build anypi_luna_dev, CONFIG_BOARD=…, UI=ipcamera, worksheets A+C` |
| Reuse old board | `From anypi_nova new product, same BOARD, change form factor and apps only` |
| New BOARD full pass | `Board checklist: DTS(board+patch)/rootfs/boot script/board config/burntool` |
| Trim config.mk | `Against app_runtime set unused CONFIG_* to n, rebuild, flash` |
| Enable RTSP+nebula | `Enable only needed protocol switches, align fw_video for streaming` |
| Wrong main in boot script | `Boot script must launch CONFIG_APP_MAIN_TARGET (e.g. luna_dev_main)` |

### Dialogue samples

**Sample 1 — anypi_luna_dev headless single app (recommended)**

> **You**: `anypi_luna_dev`: **no UI**, one app for capture+RTSP+nebula+AI.
> Confirm: ipcamera; app in `application/packages/ipcamera/ipc_cam` (export module_init/module_start; dashcam logic minus GUI).
> ① New dir+Makefile → .so; ② `module_init`/`module_start`, blocking keep-alive in start; ③ `settings.json` **only** this SO (runtime `so_path`); ④ config.mk disable GUI, enable RTSP/NEBULA/CAMERA/AI/OSD; ⑤ boot script `luna_dev_main`; ⑥ `-a -p` flash: main stays up, stream PASS.
> No desktop `installed_apps`. Fill worksheet C.
>
> **AI should**: worksheets A (form factor) + C; do not build wearable GUI app.

**Sample 2 — BOARD + UI declared**

> **You**: Baseline `anypi_luna` → `anypi_luna_dev`, BOARD=`PDB_MBD_KM02_ACTCAM`, **UI=headless**, single app. Board checklist first, then A+C, trim config, `-k -f -a -p` verify stream.

**Sample 3 — display but one desktop app**

> **You**: Switch to **`WEARABLE`**, dashcam only: `luna_main`+`PACKAGES_WEARABLE=y`; mirror `packages/wearable/dashcam`; `installed_apps` one item; `launch_mode=ui`. Not services/ headless.

**Sample 4 — reuse BOARD, change form only**

> **You**: From `anypi_nova` build `anypi_ipc_lite2`, same BOARD, keep `ipc_main`; disable cloud and UVC, keep RTSP; trim GUI/modules; y/n list then full build and flash.

**Sample 5 — trim config**

> **You**: Streaming works. Against `app_runtime` set unused `CONFIG_APP_*/FW_*/HW_*` to `n`, diff, rebuild, flash; do not touch other products.

### Worksheet A — new product (BOARD + UI required) — anypi_luna_dev prefilled

| # | Item | Output path (fill) | Key params / values (fill) | Primary skill | Status | Verify |
|---|---|---|---|---|---|---|
| 1 | Declare inputs | — | baseline=`anypi_luna` / product=`anypi_luna_dev` / **BOARD=`EVB_MBD_KM01A`** / **UI=ipcamera** / singleApp=`ipc_cam` / RTSP+nebula+AI+OSD | `/celestia-project` | ☑ | |
| 2 | Project skeleton + board | `project/anypi_luna_dev/`、`…/board/EVB_MBD_KM01A/` | `cp -r` from luna; board from KM01W rename, drop LCD boot | `/celestia-project` | ☑ | |
| 3 | Form factor | `project/anypi_luna_dev/config.mk` | `MAIN_TARGET=luna_dev_main`; `PACKAGES_IPCAMERA=y`; wearable/tablet=`n` | `/celestia-project` | ☑ | |
| 4 | **Single app** | `IOTCamera` → **worksheet C** | `packages/ipcamera/ipc_cam` | `/celestia-application` | ☑ | |
| 5 | fw / HAL / gateway | board `fw_*`/`hal_*` + gateway | RTSP/nebula/AI/OSD=`y`; GUI/LVGL=`n` | hw / fw | ☑ | |
| 6 | Trim config + build + flash | `out/anypi_luna_dev/` | `-a -p --product-name anypi_luna_dev`; main stays up, stream PASS | project / scripts / debug | ☐ | pending flash |

### Worksheet B — new board (same product name)

| # | Step | Output path (fill) | Key params / values (fill) | Primary skill | Status | Verify |
|---|---|---|---|---|---|---|
| 1 | Declare and copy board | `…/board/<NewBoard>/` | `CONFIG_BOARD=<NewBoard>` | `/celestia-project` | ☐ | |
| 2 | Board checklist | same | DTS(board+patch), rootfs, boot script, board config, burntool | `/celestia-project` | ☐ | |
| 3 | Select board + build + flash | `config.mk`、`out/<p>/` | main matches script; `-k -p` | project / scripts / debug | ☐ | |

### Worksheet C — IOTCamera single app (anypi_luna_dev typical) — prefilled

> Prerequisite: worksheet A chose **UI=ipcamera**. App **not** in `packages/wearable|tablet`.

| # | Step | Output path (fill) | Key params / values (fill) | Primary skill | Status | Verify |
|---|---|---|---|---|---|---|
| 1 | Confirm choice | — | `luna_dev_main`; `PACKAGES_IPCAMERA=y`; wearable/tablet=`n` | `/celestia-application` | ☑ | |
| 2 | New service SO | `application/packages/ipcamera/ipc_cam/` | Makefile → `libipc_cam_app.so` | `/celestia-application` | ☑ | |
| 3 | Export + keep-alive | `ipc_cam_app.c` | `module_init`/`module_start`; blocking `sleep` in start | `/celestia-application` | ☑ | |
| 4 | Wire into product | `packages/ipcamera/Makefile`、`config.mk` | `CONFIG_APP_PACKAGES_IPCAMERA=y` | application · project | ☑ | |
| 5 | `settings.json` | board overlay `board/EVB_MBD_KM01A/rootfs/utils/etc/config/` | `launch_mode=ipcamera`; only `ipc_cam`; `so_path=/usr/lib/libipc_cam_app.so` | `/celestia-application` | ☑ | |
| 6 | Business wiring | reuse dashcam record/rtsp/storage + nebula/AI | RTSP+nebula+AI+OSD via Framework | fw + application | ☑ | |
| 7 | Boot script | `board/EVB_MBD_KM01A/.../app.sh` | launch **`luna_dev_main`**; no LCD/`installed_apps` | `/celestia-project` | ☑ | |
| 8 | Trim + flash | `out/anypi_luna_dev/` | main stays up; stream ≥20s | scripts / debug | ☐ | pending `-a -p` flash |

Troubleshooting: [celestia-project](../skills/celestia-project.en.md); IOTCamera SO: [celestia-application](../skills/celestia-application.en.md#iotcamera-single-app-recommended-approach).

---

## 3. Application developer

> Add apps / GUI / cloud adapters under `application/` without touching HAL/BSP. Detail: [celestia-application](../skills/celestia-application.en.md), [application guide](../guide/application.en.md).
> **Mirror existing apps**; do not guess APIs from scratch.

### What exists → what to mirror (example library)

| Goal | Mirror / reuse | Key Framework APIs |
|---|---|---|
| Minimal app shell (clock-level) | Nova `packages/tablet/clock/` | LVGL + registry only |
| Preview + photo/short record | wearable/tablet `camera/`; shared `packages/common/gui_camera.c` | `multimedia->camera` / player |
| Loop record + sub-stream RTSP | wearable `dashcam/` (private `app_dashcam_config.json`) | recorder + `stream_reg_client` + RTSP |
| Settings: WiFi / storage / video | Nova `setting/` | network + storage + camera ctrl |
| QR provisioning / QR business | Nova `setting` → `tablet_setting_qr_scan.c` | stream role `qr` + `app_stream_usage` |
| AI toggle / scene | Nova `setting` → `tablet_setting_ai.c` | `set_ai_enabled` / `fw_ai_config` |
| Audio algo UI (VAD/AEC/NS) | Nova `audio_algo/` | `multimedia->audio` |
| IPC / call UI | Nova `ipc_camera/` | network + notification |
| File browser | Nova `file_manager/` | storage |
| Volc cloud UI + push/talk | Nova `volc_engine/` + `cloud/volc_engine/` | cloud SO + multimedia/network |
| Alibaba Bailian-style | `cloud/aliyun/` (static lib) | `aliyun_app_init/deinit` |
| IOTCamera single SO (custom) | New `packages/ipcamera/<id>/` (`module_*`); business vs `dashcam` | keep-alive + gateway |
| UVC stream | `services/uvc/` (**static**) | UVC gadget |
| Weather card | `services/weather/` (**static**) | network callback |

Stream principle: do not change capability `channels/streams`; apps only change `app_stream_usage` ([framework §2.5.1](../guide/framework.en.md)). Private resolution via `load_config(private.json)`, restore default on exit.

**Needs → skill (quick ref)**

| Need | Primary skill |
|---|---|
| New tablet / wearable app | `/celestia-application` |
| New cloud adapter | `/celestia-application` |
| Stream usage / private video config | `/celestia-application` + `/celestia-framework` |
| RTSP / nebula / AI / audio | `/celestia-framework` (app-side calls) |
| Boot / desktop / headless | `/celestia-application` (+ `/celestia-project`) |
| Build / flash | `/celestia-scripts` · `/celestia-debug` |

### Prompt scenarios (copy-paste)

| Scenario | Ask AI |
|---|---|
| Mirror clock minimal shell | `Per /celestia-application mirror tablet/clock for wearable hello app, desktop register only, no camera` |
| Mirror camera preview | `From luna/camera add burst album page, reuse packages/common, application/ only` |
| Mirror dashcam loop+RTSP | `Mirror luna/dashcam as trip_record: 60s segments + sub-stream RTSP, use worksheet` |
| Private video config | `On app enter load_config(app_xxx_config.json), exit restore fw_video_config.json, see dashcam` |
| Add qr role for scan | `Add qr to camera_app, only app_stream_usage, mirror tablet_setting_qr_scan` |
| AI page in settings | `Mirror tablet_setting_ai, wearable settings AI toggle/scene, call set_ai_enabled` |
| Audio algo app | `Mirror nova/audio_algo simple AEC toggle, multimedia->audio only` |
| Extend file manager | `Filter TF recordings by date in file_manager` |
| Volc push + talk | `Per cloud checklist wire volc_engine, ref cloud/volc_engine + nova/volc_engine UI` |
| Alibaba cloud | `Mirror cloud/aliyun static lib for Bailian, link in main, no module_exit` |
| IOTCamera stream demo | `IOTCamera contract: new services SO, no GUI, single SO, RTSP+nebula+AI, module_start keep-alive` |
| UVC service | `Enable services/uvc, WiFi mutex note, config and boot wiring` |
| Weather on desktop | `launcher calls app_weather_update_request for weather card` |
| Launcher autostart | `wearable launcher boots dashcam, change autostart / app_runtime` |
| IOTCamera strip GUI | `Disable CONFIG_APP_PACKAGES_*, luna_dev_main + single SO keep-alive` |
| App not on desktop / dlopen | `Per application FAQ: installed_apps and entry keep-alive` |

### Dialogue samples

**Sample 1 — mirror dashcam new record app (recommended start)**

> **You**: Per /celestia-application, on anypi_luna **mirror `packages/wearable/dashcam/`** as `trip_record`: 60s loop + sub-stream RTSP. Reuse private json `load_config` pattern. Only `application/` and needed `app_stream_usage`; fill worksheet; `-a --product-name anypi_luna`; /celestia-debug verify file growth and RTSP≥20s.
>
> **AI should**: match dashcam layout → register → stream usage → build and flash.

**Sample 2 — QR (usage only, not capability set)**

> **You**: camera_app needs scan. **Do not** change `channels/streams`. Mirror `tablet_setting_qr_scan`, only add `qr` in `app_stream_usage` for this app in `fw_video_config.json`; `-a` flash, QR path PASS.

**Sample 3 — settings mash-up (WiFi + AI)**

> **You**: Wearable simple settings: WiFi list + AI toggle. Mirror tablet `setting` WiFi and `tablet_setting_ai.c`; via `app_main_get_network/multimedia`, no HAL includes.

**Sample 4 — IOTCamera stream (custom SO)**

> **You**: New product `IOTCamera`. Per contract new `packages/ipcamera` SO: single SO, blocking `module_start`, main H.265 + AI + RTSP + nebula. `PACKAGES_IPCAMERA=y`, wearable/tablet=`n`; verify stream and AI log.

**Sample 5 — cloud (Volc)**

> **You**: anypi_nova Volc push + voice talk. Read cloud adapter checklist, mirror `cloud/volc_engine/`; UI ref `packages/tablet/volc_engine/`; only application/cloud and needed runtime register; `-a` verify push.

**Sample 6 — minimal shell then extend**

> **You**: First mirror `nova/clock` on Luna as `hello` (desktop icon, blank page), then add `gui_camera` preview; two commits, each `-a` flash.

**Sample 7 — audio algo page**

> **You**: Mirror `nova/audio_algo`, “AEC on/off” page, only `multimedia->audio` ops; no HAL. Verify on earpiece loop.

**Sample 8 — API check (no drill-down)**

> **You**: Via /celestia-framework, document `stream_reg_client` / `load_config` / `set_ai_enabled` return values and thread rules; I only call from app side.

### Worksheet — new wearable app (dashcam baseline)

| # | Step | Output path (fill) | Key params / values (fill) | Primary skill | Status | Verify |
|---|---|---|---|---|---|---|
| 1 | Pick baseline | `packages/wearable/dashcam/` or `camera/` / `clock` pattern | baseline app name=… | `/celestia-application` | ☐ | |
| 2 | App dir + `app.json` + entry | `application/packages/wearable/<app>/` | `app_id`/`entry_so`/`entry_symbol`; visibility+keep-alive | `/celestia-application` | ☐ | |
| 3 | Makefile + feature macros + registry | `packages/wearable/Makefile`、`config.mk`、`installed_apps.json` | `ENABLED_APP_*`、`CONFIG_APP_WEARABLE_<ID>`、`install_path` | `/celestia-application` · `/celestia-project` | ☐ | |
| 4 | Stream usage (+ optional private json) | `app_stream_usage`; optional `app_<id>_config.json` | `roles_used`; `load_config` restore on exit | `/celestia-application` · `/celestia-framework` | ☐ | |
| 5 | Business logic | `wearable_<id>_*.c` | record/RTSP/AI/scan etc. (see example library) | same | ☐ | |
| 6 | Build + flash + commit | `out/<p>/` | `-a`; feature PASS; one repo one squash | `/celestia-scripts` · `/celestia-debug` · `/git-gitee` | ☐ | |

### Worksheet — new tablet app / cloud / IOTCamera (optional)

| # | Type | Mirror | Key outputs (fill) | Verify |
|---|---|---|---|---|
| N1 | Nova desktop app | `clock`→shell / `camera`→preview / `setting`→QR+AI | `packages/tablet/<id>/` + `CONFIG_APP_TABLET_*` + installed_apps | icon opens, feature PASS |
| N2 | Cloud adapter | `volc_engine` or `aliyun` | `cloud/<id>/` + runtime/link style | push/talk or init OK |
| N3 | IOTCamera single SO | New `packages/ipcamera/<id>/` | See [§2 worksheet C](#worksheet-c--ipcamera-single-app-anypi_luna_dev-typical--prefilled); not `packages/tablet` / `packages/wearable` | main stays up, stream≥20s |

Troubleshooting: [celestia-application FAQ](../skills/celestia-application.en.md#common-issues).

---

## 4. New SoC port developer

> **Input**: vendor **chip SDK** (toolchain + kernel/drivers/headers, often under `system/<NewSoC>/<SDK_Ver>/`).
> **Core work**: per product capability list, add SoC implementation for **each required HAL module** under `hardware/.../hal/<NewSoC>/`; board via Project/BSP; `system/core` only in rare cases (module IDs etc.).
> Detail: [celestia-hardware](../skills/celestia-hardware.en.md) (“new SoC”), [celestia-system-core](../skills/celestia-system-core.en.md), [bsp-and-products §9](bsp-and-products.en.md), [download place SDK](download.en.md#install-the-chip-sdk).

**Declare up front for new chip** (all required):

| Input | Example | Notes |
|---|---|---|
| Chip SDK path / version | `system/<NewSoC>/V1.00/` | Toolchain, `include/`, `.ko`, kernel |
| `CONFIG_SOC_PLATFORM` | `AnyCloud39XXXX` | = `hal/<SoC>/` directory name |
| `CONFIG_SOC_SDK_VERSION` | `V1.00` | Matches SDK directory |
| Capability list | camera / audio / display / wifi / ir… | drives `CONFIG_HW_*` and which HAL to write |
| Reference SoC | `AnyCloud39AV200` or `AnyCloud39AV100_V2` | Copy `hal_<module>.c` skeleton from ref, adapt SDK calls |
| New product / BOARD | `anypi_xxx` / `PDB_…` | Assembly: [§2](#2-new-product-port-developer) |

### Layers: what changes, what does not

| Layer | New chip work | Primary skill |
|---|---|---|
| **Chip SDK / BSP** | Place under `system/<NewSoC>/<SDK>/`; board DTS, insmod, rootfs overlay | `/celestia-project` |
| **Hardware HAL** | **Each enabled module** add `hal/<NewSoC>/` (or confirm flat module OK) + `soc_platform` | `/celestia-hardware` |
| **system/core** | Usually **not** forked by SoC; only new HAL module → `HARDWARE_MODULE_*` / error codes | `/celestia-system-core` |
| Framework / Application / `hardware/include` | **Do not** change public contract for one SoC; extend via private headers | — |

### Per-module HAL checklist (must create dir when `hal/<SoC>/` exists)

See [celestia-hardware directory map](../skills/celestia-hardware.en.md#directory-map). After capability check, **implement row by row** (ref `hal/<RefSoC>/`, wire vendor SDK API):

| Module | Path | `CONFIG_HW_*` (example) | Status |
|---|---|---|---|
| soc_platform | `hardware/soc_platform/hal/<NewSoC>/` | `CONFIG_HW_SOC_PLATFORM` | ☐ |
| camera | `hardware/camera/hal/<NewSoC>/` | `CONFIG_HW_CAMERA` | ☐ |
| audio | `hardware/audio/hal/<NewSoC>/` | `CONFIG_HW_AUDIO` | ☐ |
| osd | `hardware/osd/hal/<NewSoC>/` | `CONFIG_HW_OSD` | ☐ |
| gpu | `hardware/gpu/hal/<NewSoC>/` | `CONFIG_HW_GPU` | ☐ |
| vision/isp | `hardware/vision/isp/hal/<NewSoC>/` | `CONFIG_HW_VISION_ISP` | ☐ |
| vision/motion_detect | `hardware/vision/motion_detect/hal/<NewSoC>/` | `CONFIG_HW_VISION_MOTION_DETECT` | ☐ |
| driver/ir | `hardware/driver/ir/hal/<NewSoC>/` | `CONFIG_HW_DRIVER_IR` | ☐ |
| driver/motor | `hardware/driver/motor/hal/<NewSoC>/` | `CONFIG_HW_DRIVER_MOTOR` | ☐ |

**Flat modules** (display, network/*, input, most drivers): usually **no** `hal/<NewSoC>/`; verify with SDK drivers + board config. Add SoC subdir only if capability missing (see hardware skill).

Per-module deliverables: `HAL_MODULE_INFO_SYM` + ops + `hw_<m>_module_init()` → `hardware.c` calls under matching `CONFIG_HW_*`; Makefile collects `hal/$(CONFIG_SOC_PLATFORM)`.

### Needs → skill (quick ref)

| Need | Primary skill |
|---|---|
| Place / verify chip SDK, toolchain | `/celestia-project` + [download](download.en.md) |
| **Per-module** HAL + soc_platform + IPU weak stubs | `/celestia-hardware` |
| New `HARDWARE_MODULE_*` / error codes | `/celestia-system-core` |
| config.mk / board / DTS / insmod / new product | `/celestia-project` |
| **AOS/RTOS chip** (SDK only `.a`, port Linux to Aos, e.g. mira/Sky39AV130) | `/celestia-aos-port` ([celestia-aos-port](../skills/celestia-aos-port.en.md)) |
| Build / bring-up L1→L5 | `/celestia-scripts` · `/celestia-debug` |
| Sync SoC matrix docs | `/celestia-contributing` |

### Prompt scenarios (copy-paste)

| Scenario | Ask AI |
|---|---|
| **SDK new chip end-to-end** | `SDK at system/<NewSoC>/<Ver>/. Per roles §4: capability list → per-module hal/<NewSoC>/ → system/core only if new module IDs → project/board → L1→L5` |
| Batch HAL from checklist | `For <NewSoC> implement hardware modules with hal/ per table, ref <RefSoC>, no public header changes, fill worksheet` |
| Camera-only first | `Enable only CONFIG_HW_CAMERA + soc_platform, implement camera HAL, rest HW=n, extend later` |
| Add audio/osd/ir | `For <NewSoC> add audio/osd/ir HAL, dirs only where hal/<SoC>/ exists` |
| soc_platform / IPU | `soc_platform clk/pinmux/IRQ; weak stub if no IPU` |
| system/core changes? | `Per /celestia-system-core: skip if no new module; else append HARDWARE_MODULE_*` |
| Public headers frozen | `Extend via private/SDK headers only, no hardware/include/*.h` |
| New board DTS / insmod | `Per bsp §9 board + ko order, vendor tree read-only first` |
| Missing `/dev/video0` | `Class D first: insmod/dmesg, then HAL if needed` |
| Bring-up self-check | `L1 /dev → L2 hardware/tests → L3 fw preview → L4 App → L5 cloud, PASS each level` |

### Dialogue samples

**Sample 1 — SDK + per-module HAL (recommended start)**

> **You**: Chip SDK at `system/<NewSoC>/V1.00/`. Capabilities: camera + audio + display + wifi. Per /celestia-hardware: **for each module with `hal/<SoC>/` and capability on** add `hal/<NewSoC>/` (camera+audio+soc_platform first); flat display/wifi no dir, board verify; public headers unchanged; `system/core` unchanged if no new module IDs. config.mk sets `CONFIG_SOC_PLATFORM`/`CONFIG_SOC_SDK_VERSION`/`CONFIG_HW_*`. Fill §4 worksheet; `-s -h -f -a -k -p` as needed; /celestia-debug L1→L5.
>
> **AI should**: read hardware (new SoC) + system-core (boundary) + project → per-module implement → build → bring-up → suggest SoC matrix doc sync.

**Sample 2 — board only, HAL contract unchanged**

> **You**: Same SoC new board new sensor. Only `project/<p>/.../board/<Board>/` DTS and rootfs; vendor tree read-only; `-k -p` flash, check `/dev/video0` and frames.

**Sample 3 — HAL debug**

> **You**: anypi_luna preview CHN0 select timeout. /celestia layer first, then hardware FAQ `num_slices` / frame_depth; no Framework change without evidence.

### Worksheet — new chip (SDK → per-module HAL → flash)

| # | Step | Output path (fill) | Key params / values (fill) | Primary skill | Status | Verify |
|---|---|---|---|---|---|---|
| 0 | Place chip SDK | `system/<NewSoC>/<Ver>/` | Toolchain, `include/`, `.ko`, kernel | `/celestia-project` | ☐ | SDK headers found by `-h` |
| 1 | config.mk SoC + modules | `project/<p>/config.mk` | `CONFIG_SOC_PLATFORM`, `CONFIG_SOC_SDK_VERSION`, needed `CONFIG_HW_*=y` | `/celestia-project` | ☐ | `config.h` correct |
| 2 | **Per-module** `hal/<NewSoC>/` | paths in table above | Each: `HAL_MODULE_INFO_SYM` + `hw_<m>_module_init`; vendor SDK | `/celestia-hardware` | ☐ | module `-h` passes |
| 3 | soc_platform (+ IPU weak stubs) | `hardware/soc_platform/hal/<NewSoC>/` | clk/pinmux/IRQ | `/celestia-hardware` | ☐ | no platform bring-up fail |
| 4 | system/core (if needed) | `system/core/.../system_error.h` | new `HARDWARE_MODULE_<X>` / errors; skip if no new module | `/celestia-system-core` | ☐ | `-s`; upper layers OK |
| 5 | board DTS + insmod + rootfs | `project/.../board/<Board>/` | sensor/DMA; ko order; boot script | `/celestia-project` | ☐ | `/dev/*` present |
| 6 | Build + L1→L5 + docs + commit | `out/<p>/` | `-h -f -a -k -p`; `/dev`→tests→preview→app→cloud | `/celestia-scripts` · `/celestia-debug` · `/celestia-contributing` · `/git-gitee` | ☐ | all levels PASS |

Troubleshooting: [celestia-hardware FAQ](../skills/celestia-hardware.en.md#common-issues); bug classes [§5](#5-debug-and-maintenance).

---

## 5. Debug and maintenance

> On-board debug, crashes, memory/DMA, stale builds, multi-repo commits.
> **Bug taxonomy**: [celestia-debug](../skills/celestia-debug.en.md) (classes A–G); routing [celestia](../skills/celestia.en.md); commits [git-gitee](../skills/git-gitee.en.md).

**Ask with**: product name + repro steps + **raw key log** (with context) → classify A–G → fix → retest loop.

### Symptom / log → skill (quick ref)

| Log / symptom | Bug class | Read first |
|---|---|---|
| `signal=11` / `fault_addr=` / backtrace | **A crash** | `/celestia-debug` → stack → `/celestia` layer |
| `invalid inbuf` / `QBUF fail` / `select timeout` / no frames | **B preview/encode** | `/celestia-debug` + `/celestia-hardware` or `/celestia-framework` |
| ion/dma alloc fail, OOM, corrupt frames, MemFree very low | **C memory DMA** | `/celestia-debug` (decision tree); DTS → `/celestia-project` |
| missing `/dev/*`, insmod fail, wrong main, overlay ignored | **D boot/drivers** | `/celestia-project` (+ hardware) |
| `dlopen` fail, no desktop, blank icon, IOTCamera instant exit | **E App/SO** | `/celestia-application` |
| changes ignored, `module: -1`, wrong config.h, link errors | **F stale build** | `/celestia-scripts` (+ hardware/project) |
| RTSP/nebula no stream, cloud down, `stream_reg_client` fail | **G stream/network** | `/celestia-framework` (+ application) |
| unknown layer | — | `/celestia` first, then `/celestia-debug` master table |

Per-layer FAQ: `application` / `hardware` / `framework` / `project` / `scripts` “常见问题” sections.

### Prompt scenarios (copy-paste)

| Scenario | Ask AI |
|---|---|
| Paste log classify | `Product anypi_luna, log: <paste>. Per /celestia-debug classify A–G, fill debug worksheet, fix then retest` |
| Preview drop frames | `Repeated invalid inbuf cap=524288, class B known pattern, ≥20s frames after fix` |
| signal=11 | `After AI enable signal=11 fault_addr=0x90, class A race fix, stack and layer` |
| Preview dead after photo | `After photo preview stuck+QBUF fail, class B teardown/live reload` |
| OOM / corrupt video | `luna record OOM or corrupt, class C DMA vs MemTotal decision tree` |
| No /dev/video0 | `Class D: insmod/dmesg + boot script, then HAL if needed` |
| App dlopen fail | `Class E: visibility + keep-alive + installed_apps path` |
| Changes ignored | `Class F: stale out/, -c rebuild, check product-name` |
| RTSP no stream | `Class G: CONFIG_APP_RTSP + stream_reg_client, ensure class B PASS first` |
| Multi-repo commit | `Per /git-gitee one repo one squash to main, draft commits for confirm` |

### Dialogue samples

**Sample 1 — paste log classify (recommended)**

> **You**: Product `anypi_luna`. Serial:
> `invalid inbuf cap=524288` (repeat). Preview drops frames.
> Per /celestia-debug: **class B** → fill worksheet → known fix (payload after codec_mode) → `-f/-h` rebuild flash, ≥20s frames = PASS.
>
> **AI should**: hit class B table → minimal fix → close loop → fill worksheet.

**Sample 2 — crash with stack**

> **You**: After enabling AI in `ipc_camera`, `signal=11 fault_addr=0x90`, backtrace has `camera_manager`. Class A race pattern; no BSP first. 10 repros no crash = PASS.

**Sample 3 — memory triage**

> **You**: Corrupt video after record, dmesg ion alloc fail, `MemTotal` ~25MB. Class C **DMA pool low**, 20→34MB board+patch plan and `-k -p` steps; consider lowering `max_buffers` first.

**Sample 4 — boot no devices**

> **You**: No `/dev/video0` on flash. Class D: `app.sh`/`insmod`/`dmesg`, confirm ko before `/celestia-hardware`; do not jump to Framework.

**Sample 5 — app not on desktop**

> **You**: New wearable app builds but no desktop icon. Class E: `installed_apps.json`, `install_path`, entry gc; application FAQ.

**Sample 6 — stale build**

> **You**: Changed `hal_camera.c` still `Failed to get video hardware module: -1`. Class F: `-c -h`, check `hardware.o` depends on config.h.

**Sample 7 — stream integration**

> **You**: RTSP client connects, no video. Ensure class B preview/encode PASS, then class G `CONFIG_APP_RTSP` and `stream_reg_client(main)`; paste gateway logs.

**Sample 8 — commit**

> **You**: Fixed framework + hardware. Per /git-gitee: **main**, one repo one squash, executable bits; two repo message drafts, confirm before commit.

### Worksheet — debug

| # | Symptom (fill) | Class (A–G) | Evidence / log (fill) | Root cause / check (fill) | Fix (fill) | Primary skill | Verify (fill) |
|---|---|---|---|---|---|---|---|
| 1 | Preview drop frames | B | `invalid inbuf cap=524288` | payload before codec_mode | parse codec_mode first | `/celestia-debug`+fw/hw | PASS: ≥20s frames |
| 2 | | | | | | | |
| 3 | | | | | | | |

> Full taxonomy: [celestia-debug bug classes](../skills/celestia-debug.en.md#bug-classification-master-table-check-log-first). Do not build without verify.