---
name: celestia-application
description: >-
  Celestia Application layer (application/) development and troubleshooting: app_main processes
  (ipc_main/nova_main/luna_main/luna_dev_main/mira_main), startup flow, packages/ (ipcamera IOTCamera,
  tablet TABLET, wearable WEARABLE, packages/common shared), IOTCamera business (ipc_cam), tablet/wearable
  apps, cloud adapters (cloud/aliyun, cloud/volc_engine), system services (services/), settings.json
  (embedded installed_apps). Call Framework only via app_main_get_*() and *_open(); no direct HAL/BSP.
  Checklists for new tablet App and cloud adapter. Build with build.sh -a.
---

<p align="right"><b>English</b> | <a href="celestia-application.md">中文</a></p>

# Application layer skill

> Concepts and API: `docs/guide/application.md`; API index: `docs/guide/api-index.md`;
> overview and cross-layer rules: [celestia](celestia.en.md).

## Directory map

```
application/
├── app_main/          # main process: main.c / app_main.c / app_config.c
│                      #   output = CONFIG_APP_MAIN_TARGET:
│                      #   ipc_main(anypi_nova) / nova_main(anypi_nova_dev)
│                      #   / luna_main(anypi_luna) / luna_dev_main(anypi_luna_dev)
│                      #   / mira_main(anypi_mira_dev)
├── packages/
│   ├── common/        # cross-product GUI (e.g. gui_camera.c preview)
│   ├── tablet/        # tablet TABLET desktop (800x480 / 1024x600); each app tablet_<id>.c/.h
│   ├── wearable/      # wearable WEARABLE launcher (240x240, ActionCam); wearable_<id>_*.c/.h
│   └── ipcamera/      # IOTCamera business SO (module_init / module_start): ipc_cam/
├── cloud/             # cloud: aliyun / volc_engine / mike
├── services/          # system services: uvc / weather ...
└── tests/             # app demos / unit tests
```

## Upward/downward interface (only these paths)

Application has **no outward API**; two paths to lower layers:

| Call | Header | Purpose |
|------|--------|------|
| `app_main_get_multimedia/network/notification/nebula/runtime()` | `application/app_main/app_main.h` | Framework service handles (`void*` cast) |
| `*_open()` + ops (`multimedia_t`/…) | `framework/services/*/include/*.h` | Framework capabilities |
| `#include "system_core.h"` and sub-headers | `system/core/include/*.h` | log / thread / config / errors |

Startup (`application/app_main/main.c`):
`app_main_platform_startup()` → `runtime_bootstrap_open()` (+ `runtime->load_config`, `app_main_register_runtime`)
→ `app_main_init()` → `app_main_load_config()` → `app_main_run()`
→ `app_main_start_framework_modules(headless, gui_config)`
→ `tablet_run()` (`CONFIG_APP_PACKAGES_TABLET`) or `app_gui_run()` (`CONFIG_APP_PACKAGES_WEARABLE`).

Config:
- `settings.json`: `{ "program", "launch_mode", "modules": ... }`, SO hot-plug; `so_path` runtime absolute (e.g. `/usr/lib/libipc_cam_app.so`, cloud `/usr/lib/lib*_app.so`).
- Desktop app list embedded in `settings.json` `installed_apps`: `{"schema_version":1,"apps":[...]}`; `install_path` like `/usr/apps/<id>` (luna) or `/mnt/install/apps/<id>` (nova). Standalone `/etc/config/installed_apps.json` legacy fallback only.

## Forbidden (must refuse)

- ❌ `#include "audio.h"/"camera.h"` or any `hardware/include/*.h`, SoC-private (`hal_*_api.h`/`*_hw.h`);
- ❌ direct `ioctl` / `/dev/*` / BSP;
- ✅ hardware needs → Framework ops; gaps → [celestia-framework](celestia-framework.en.md).

## Troubleshooting: application-first

1. Analyze `application/`: startup, GUI events, app registration, `settings.json`, SO `dlopen/dlsym`.
2. If Framework suspected, **only confirm interface contract** in `framework/services/*/include/*.h`; check `*_open()`/ops params, order, NULL handles.
3. Only when Framework returns errors/logs proving failure → [celestia-framework](celestia-framework.en.md).
4. Shared code in `packages/common`, `app_main`: assess tablet / wearable / ipcamera.

## Add tablet App (checklist)

Reference: `application/packages/tablet/clock/`, `.../camera/`. SO mode — **no main process rebuild**.

1. **Dir** `application/packages/tablet/<id>/`: `app.json`, `app_entry_<id>.c`, `tablet_<id>.c`, `tablet_<id>.h`, `res/icon.png`. Shared macros in `application/packages/tablet/launcher/tablet_common.h` (no `lv_demo_*`, no `lv_demo_common.h`).
2. **app.json** (match real files): `app_id / name / action / icon / entry_so / entry_symbol / version`, e.g.:
   ```json
   { "app_id": "<id>", "name": "<Name>", "action": "<id>",
     "icon": "res/icon.png", "entry_so": "lib/lib<id>_app.so",
     "entry_symbol": "app_entry_start_<id>", "version": "1.0.0" }
   ```
   SO name = basename of `entry_so` (per-app, not a single pattern).
3. **Entry** `app_entry_<id>.c` (SPDX first line):
   ```c
   // SPDX-License-Identifier: Apache-2.0
   #define LOG_TAG "TABLET_APP_ENTRY_<ID_UPPER>"
   #include "tablet_<id>.h"
   #define APP_ENTRY_EXPORT __attribute__((visibility("default"))) __attribute__((used))
   APP_ENTRY_EXPORT int app_entry_start_<id>(lv_obj_t *main) {
       if (!main) return -1;
       lv_obj_add_flag(main, LV_OBJ_FLAG_HIDDEN);
       (void)tablet_<id>_create(main);
       return 0;
   }
   /* keep-alive self-ref against --gc-sections (required) */
   static int (*const s_keep_app_entry_<id>)(lv_obj_t *) = app_entry_start_<id>;
   ```
   `tablet_<id>.h` guard uses `CONFIG_APP_TABLET_<ID_UPPER>`.
4. **Feature macro**: `project/<product>/config.mk` → `CONFIG_APP_TABLET_<ID_UPPER> = y`.
5. **No Makefile edit** — `application/packages/tablet/Makefile` scans `app.json` dirs, enables via `CONFIG_APP_TABLET_<ID>`; `.so` from `entry_so`; per-app `app.mk` (`APP_CFLAGS_EXTRA`, `<ID>_SHLIB_LIBS`, etc.). Makefile `-include .../config.h` for `CONFIG_APP_TABLET_*`; INI API header is `system_config.h`.
6. **Register** in board `settings.json` `installed_apps.apps[]`, `install_path` matches rootfs (nova often `/mnt/install/apps/<id>`, fixed `/usr` apps `/usr/apps/<id>`). Nova registry load — usually **no** `modules` change.
7. **Self-check**:
   ```bash
   grep -rE '#include\s+[<"](audio|camera|display|osd|gpu|isp|wifi|gpio|hardware)\.h[>"]' application/packages/tablet/<id>/
   ${CROSS}nm -D out/<product>/install/application/lib/lib<id>_app.so | grep app_entry_start_<id>
   python3 -c "import json;json.load(open('application/packages/tablet/<id>/app.json'))"
   ```

> Wearable checklist below. Studio workflow: [celestia-studio](celestia-studio.en.md). Edits stay in same dir; push still overwrites `/mnt/install/apps/<id>`.

## Add cloud adapter (checklist)

Two existing styles — pick one:
- `application/cloud/aliyun/`: static `libaliyun_app.a`, `PROJ = aliyun_app`, exports `aliyun_app_init()/aliyun_app_deinit()`.
- `application/cloud/volc_engine/`: shared `libvolc_engine_svc.so`, `PROJ = volc_engine_svc`, exports `volc_engine_init()/volc_engine_deinit()`.
- `tuya` config placeholder only.

1. **Dir** `application/cloud/<id>/`: `Makefile`, `include/<id>_app.h`, `<id>_app.c` (optional `src/`, vendor `lib/`). Adapter exports `<id>_app_init()/<id>_app_deinit()`.
2. **Generic SO contract**: `application/app_main/app_config.c::load_so_module()` only `dlsym` `module_init(const char*)` and `module_start(void)` (**no** `module_exit`). Only needed on generic SO path; aliyun/volc link into main.
3. **Makefile**:
   ```make
   include $(COMPILATION_RULES)
   include $(PROJECT_CONFIG_FILE)
   PROJ = <name>
   # install to $(APPLICATION_INSTALL_DIR)/lib
   ```
4. **Top wiring** `application/Makefile`: `obj-$(CONFIG_APP_CLOUD_<ID>) += cloud/<id>` (token may differ from dir name, e.g. `volc_engine` → `CONFIG_APP_CLOUD_VOLC`).
5. Same forbidden rules as above; Framework `*_open()`, system/core, vendor SDK inside adapter OK; no HAL headers or SO UI.

## Examples (copy existing)

| Goal | Reference path |
|---|---|
| Minimal shell | `application/packages/tablet/clock/` |
| Preview/capture | `packages/wearable/camera/`, `packages/tablet/camera/`, `packages/common/` |
| Loop record + RTSP | `packages/wearable/dashcam/` |
| QR / AI settings | `packages/tablet/setting/tablet_setting_qr_scan.c`, `tablet_setting_ai.c` |
| Audio algo UI | `packages/tablet/audio_algo/` |
| New IOTCamera SO | `packages/ipcamera/<id>/` with `module_init`/`module_start`; see `dashcam` |
| Volc / Ali cloud | `cloud/volc_engine/`, `cloud/aliyun/` |
| UVC / weather | `services/uvc/`, `services/weather/` (`type:static` in app_runtime) |

Sample forms: [roles §application dev](../quick-start/roles.en.md#3-应用开发者).

## IOTCamera single app (recommended)

**When**: new `IOTCamera` + single business (e.g. `anypi_luna_dev`). See [roles §2 table C](../quick-start/roles.en.md#填写表-c--iotcamera-单应用anypi_luna_dev-典型-已回填).

Load contract (`app_config.c` + `main.c`):

- `launch_mode` ≠ `ui` ⇒ headless; both GUI macros off without blocking `module_start` ⇒ `main` returns after framework start.
- `type:so` modules: `dlsym("module_init")` / `dlsym("module_start")`.
- `services/uvc`, `services/weather` are **static** modules.

| Item | Action |
|---|---|
| Main | `CONFIG_APP_MAIN_TARGET=luna_dev_main`; `CONFIG_APP_PACKAGES_TABLET=n`; `CONFIG_APP_PACKAGES_WEARABLE=n` |
| Dir | `application/packages/ipcamera/<id>/` (Makefile → `.so`) |
| Exports | `int module_init(const char *config_path)`; `int module_start(void)` |
| Keep-alive | **Block inside `module_start`** (business loop) |
| Reference | `app_main_get_multimedia/network/nebula`; record/RTSP like `packages/wearable/dashcam/` (no LVGL) |
| Register | `settings.json`: `"launch_mode":"headless"`, one `type:so` + runtime `so_path` |
| Desktop | No `installed_apps.json` |
| Launch | script starts `luna_dev_main` |

Display single-app still uses `packages/wearable|tablet/<id>/` + `installed_apps`.

## Add wearable App (key points)

Multi-file: `wearable_<id>_*.c/.h` + `app_entry_<id>.c` + `app.json` + `res/icon.png`.

1. Entry: `visibility("default")` + keep-alive (anti `--gc-sections`)
2. Makefile: `ENABLED_APP_DIRS/IDS`, `APP_SO_FILE_<id>`, install case
3. Macro: `project/<p>/config.mk` → `CONFIG_APP_WEARABLE_<ID>=y`
4. Register: `installed_apps.json` (`install_path=/usr/apps/<id>`); stream usage only in `fw_video_config.json` `app_stream_usage` (see `docs/guide/framework.md` §2.5.1)
5. Registry: `wearable_app_registry.c`; entry via `app_gui_run()`

Worksheet: [roles §application dev](../quick-start/roles.en.md#3-应用开发者).

## Common issues

| Symptom | Cause | Fix |
|---|---|---|
| SO `dlopen` / missing entry | `--gc-sections` stripped export | `visibility("default")` + keep-alive |
| App not on desktop | not in `installed_apps` / wrong `install_path` | register `apps[]`, fix path |
| `app_runtime` SO not loaded | `so_path` is source path | runtime `/usr/lib/...` per rootfs |
| `module_exit` never runs | loader only `module_init`/`module_start` | do not rely on `module_exit` |
| Stream config ignored | changed `channels/streams` capability | only `app_stream_usage` §2.5.1 |
| Cross-layer include errors | `#include hardware/include` | Framework `*_open()` |
| `CONFIG_APP_TABLET_*` dead | Makefile missing config.h | `-include $(PROJECT_GEN_INCLUDE_DIR)/config.h` |
| IOTCamera main exits immediately | no GUI path returns from `main` | SO `module_start` blocks |

## Build after coding (required)

```bash
cd ~/celestia
./build.sh -a --product-name anypi_nova
./build.sh -a --product-name anypi_nova_dev
./build.sh -a --product-name anypi_luna
./build.sh -a --product-name anypi_luna_dev
./build.sh -a --product-name anypi_mira_dev
```

Fix all errors; if shared code changed, `-a` on another product. Framework errors: caller misuse vs drill-down. Details: `docs/guide/application.md`, signatures: `docs/guide/api-index.md`.