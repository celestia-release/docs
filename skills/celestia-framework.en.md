---
name: celestia-framework
description: >-
  Celestia Framework layer (framework/) development, troubleshooting, and new services: OS-neutral
  service handles multimedia / network / notification / device / storage (services/, `<svc>_open()`
  returns aggregate handles with named sub-ops), gateway protocols nebula/rtsp (gateway/), runtime
  bootstrap / console / OTA (runtime/), GUI ports (ui/). Exposes `*_open()` to App; may only include
  public `hardware/include/*.h`, never chip-private `hardware/*/hal/<SoC>/`. Includes "add service"
  checklist; multimedia covers `FFMPEG_LINK_MODE` (none/decode/normal/whole; Nova 16MB uses decode).
  Build with `./build.sh -f`. Use when extending or debugging multimedia/network/storage/notification/protocols/runtime.
---

<p align="right"><b>English</b> | <a href="celestia-framework.md">中文</a></p>

# Framework layer skill

> Concepts: `docs/guide/framework.md`; API index: `docs/guide/framework-api.md`.
> Overview and cross-layer rules: [celestia](celestia.en.md).

## Directory map

Service layout: main implementation at service root (or `core/`); each feature is a **subdirectory** with its own `Makefile`; `include/` holds the single public header. **No `src/` dir, no `<svc>_ops.c`/`<svc>_hal_glue.c` convention** — ops getters live in submodule files (e.g. `led/led_manager.c` → `led_get_ops()`).

```
framework/
├── services/          # 5 services: multimedia network notification device storage
│   ├── multimedia/    # libmultimedia.so   core/(multimedia.c) av_codec/ audio/ camera/ media/ include/
│   ├── network/       # libnetwork.so      network.c include/ wifi/ ethernet/ ntp/ ping/ rtsp/
│   ├── notification/  # libnotification.so core/(notification.c) message/ alarm/ internal/ include/
│   ├── device/        # libdevice.so       device.c include/ ptz/ led/ power/ backlight/ night_vision/
│   └── storage/       # libstorage.so      storage.c include/ tfcard/ flash/
├── gateway/
│   ├── nebula_protocol/   # libnebula_protocol.so  (PROJ=nebula_protocol)
│   └── rtsp_protocol/     # librtsp_protocol.so    (PROJ=rtsp_protocol; upstream librtsp_server.so is under network/rtsp/rtsp_server/)
├── runtime/           # bootstrap/(runtime_init.c) console/ ota/ power_button/ watchdog/ include/
└── ui/                # common/include/gui_display.h, lvgl/include/lv_port.h, graphics/ awtk/
```

Public headers (upward): `framework/services/*/include/*.h`, `framework/runtime/include/*.h`, `framework/gateway/*/include/*.h`, `framework/ui/**/include/*.h` (e.g. `multimedia.h` `network.h` `notification.h` `device.h` `storage.h` `runtime_bootstrap.h` `console.h` `ota.h` `nebula_protocol.h` `rtsp_protocol.h` `gui_display.h`).
**One public header per module** (e.g. `multimedia/include/multimedia.h`); do not add more public `.h` for the same module. Prefer hanging capabilities on existing sub-ops / `reserved`; extend internals before expanding public symbols.

## Service contract pattern

Each service: `<svc>_open(<svc>_t **)` / `<svc>_close(<svc>_t *)`. The handle is an **aggregate**, not a single ops table: one static instance holds multiple **named sub-ops pointers**, lazy init, guarded by `#ifdef CONFIG_FW_*`. See `device.c`:

```c
/* device.h: named sub-ops + reserved for binary compat */
typedef struct {
    led_ops_t       *led;
    ptz_ops_t       *ptz;
    power_ops_t     *power;
    nv_ops_t        *nv;
    backlight_ops_t *backlight;
    void            *reserved[10];
} device_t;

/* device.c: static instance; open fills sub-ops per switches; close tears down symmetrically */
static device_t g_device = { .led = NULL, /* ... */ };
int device_open(device_t **device) {
    if (!device) return DEVICE_ERROR_INVALID_PARAM;
#ifdef CONFIG_FW_DEVICE_LED
    if (!g_device.led) g_device.led = led_get_ops();   /* getter in led/led_manager.c */
#endif
    /* ptz/power/nv/backlight similarly */
    *device = &g_device;
    return DEVICE_OK;
}
```

Caller: `device_t *d; device_open(&d); d->led->set_scene(&cfg);`.

Notes:
- Top-level `*_open` **does not use REENTRY macros** and has **no `<svc>_service_is_open()`**. Readiness is per service: `multimedia` uses ops member `is_open` + `g_multimedia_initialized`.
- `system/core` `REENTRY_*` macros are for **internal worker leaf modules** (`media_record.c`, `rtsp_server.c`, `tfcard_utils.c`), not top-level open.
- Error codes: `SYSTEM_ERROR_CODE(FRAMEWORK_DOMAIN, module, error)`; 0 = success, negative = error; no raw `int` return codes.

### `*_open()` entry table

| Service | Entry / out param | Library | Config switch |
|------|-------------|----|---------|
| multimedia | `multimedia_open(multimedia_t **)` | libmultimedia.so | `CONFIG_FW_MULTIMEDIA` |
| network | `network_open(network_t **)` | libnetwork.so | `CONFIG_FW_NETWORK` |
| notification | `notification_open(notification_t **)` | libnotification.so | `CONFIG_FW_NOTIFICATION` |
| device | `device_open(device_t **)` | libdevice.so | `CONFIG_FW_DRIVER` (not FW_DEVICE) |
| storage | `storage_open(storage_t **)` | libstorage.so | `CONFIG_FW_STORAGE` |
| runtime | `runtime_bootstrap_open(...)` | — | `CONFIG_FW_SYSTEM` |
| gateway | nebula/rtsp protocol open | libnebula_protocol.so / librtsp_protocol.so | `CONFIG_APP_NEBULA` / `CONFIG_APP_RTSP` |
| ui | lv_port / gui_display | — | `CONFIG_FW_GUI_LVGL` |

Services are opened by **application (app_main)**, not auto-started in `runtime/bootstrap/runtime_init.c` (that file only runtime bootstrap init/open/close + load_config).

## Downward interface (only this way)

| Call | Header | Notes |
|------|--------|------|
| HAL public APIs | Full set `hardware/include/*.h` (24: adc audio camera display driver_error efuse ethernet gpio gpu hardware header input input_evdev ir isp led mmc motion_detect motor osd pwm soc_platform wdt wifi) | `hw_get_module()` → `hw_device_t` → ops |
| Horizontal | `system/core/include/*.h` | log/thread/config/timer/errors |

## Forbidden (must refuse)

- ❌ `#include "hal_audio_api.h"/"audio_hw.h"/"hal_camera_api.h"/"camera_hw.h"` or other HAL-private headers;
- ❌ Any chip-private path under `hardware/*/hal/<SoC>/`, or strings like `AnyCloud39AV200`/`AnyCloud39AV100_V2`;
- ❌ Reverse dependency on application (`application/**`);
- ❌ Hard-coded absolute paths `/mnt/...` for config (use `config.h::config_build_path()`);
- ✅ SoC differences stay in HAL; Framework sees unified `hardware/include/*.h` only.

## Add service / sub-capability (checklist)

1. **Dirs**: `framework/services/<svc>/<svc>.c` (`<svc>_open/close`) + `include/<svc>.h` (single public header) + one `Makefile` subdir per feature with `<feat>_get_ops()`. Mirror `framework/services/device/`.
2. **Header**: Copy structure from `framework/services/device/include/device.h` — version macros + error macros + aggregate with named sub-ops + `void *reserved[N]` + `<svc>_open/close`. ~10-line skeleton:
   ```c
   #ifndef FRAMEWORK_<SVC_UPPER>_H
   #define FRAMEWORK_<SVC_UPPER>_H
   #include "system_error.h"
   #define <SVC_UPPER>_OK  SYSTEM_SUCCESS
   #define <SVC_UPPER>_ERROR_INVALID_PARAM \
       SYSTEM_ERROR_CODE(FRAMEWORK_DOMAIN, FRAMEWORK_MODULE_<X>, ERROR_INVALID_PARAM)
   typedef struct { feat_ops_t *feat; void *reserved[8]; } <svc>_t;
   int <svc>_open(<svc>_t **out);
   int <svc>_close(<svc>_t *self);
   #endif
   ```
3. **Error codes**: `system/core/include/system_error.h` has module ID map — `FRAMEWORK_MODULE_NETWORK=0x00 NOTIFICATION=0x01 MULTIMEDIA=0x02 RUNTIME=0x03 DRIVER==DEVICE=0x04 GUI==UI=0x05 AI=0x06(reserved) STORAGE=0x07 GATEWAY=0x08 NEBULA_PROTOCOL=0x09`. DRIVER/DEVICE and GUI/UI are **aliases** (same value). Claim free slots; sync `docs/guide/system-core-api.md`.
4. **Makefile** (copy `framework/services/device/Makefile`):
   ```make
   MODULE_ROOT ?= $(shell pwd)
   include $(COMPILATION_RULES)      # from build.sh; no config.mk/rules.mk here
   include $(PROJECT_CONFIG_FILE)
   PROJ = <svc>                      # lib$(PROJ).a / lib$(PROJ).so.$(VERSION)
   obj-$(CONFIG_FW_<SVC>_<FEAT>) += <feat>/
   SRCS := $(MODULE_ROOT)/<svc>.c
   # install copies include/<svc>.h to $(FRAMEWORK_INSTALL_DIR)/include/
   ```
   Register in top `framework/Makefile`: `obj-$(CONFIG_FW_<X>) += services/<svc>/`.
5. **Wire-up**: Call `<svc>_open()`/`<svc>_close()` in **app_main**; do not assume runtime auto-start.
6. **Config**: JSON (cjson), not INI — e.g. `fw_<svc>.json`.
7. **Docs**: Sync `docs/guide/framework-api.md`, `docs/guide/framework.md` module list; new error codes → `system_error.h` + `docs/guide/system-core-api.md`.

## Troubleshooting: Framework-first

1. Analyze this service: ops flow, aggregate state, threads/buffers, JSON config.
2. If HAL suspected, **only confirm `hardware/include/*.h` contract** (signatures, version macros, returns, `hw_get_module` success); verify Framework usage.
3. Only with HAL error codes/logs proving failure, switch to [celestia-hardware](celestia-hardware.en.md).
4. For log/thread/config issues, [celestia-system-core](celestia-system-core.en.md).
5. Dual products: HAL differs on AV200 / AV100_V2; Framework code should stay SoC-neutral.

## Common issues (Framework)

| Symptom / log | Root cause | Fix |
|---|---|---|
| `invalid inbuf cap=524288` | payload computed before `codec_mode` | Parse `codec_mode` first |
| Live preview broken by capture/record | teardown or `load_config` clears live | Symmetric teardown; no reload during live |
| `stream_reg_client` fails | Channel/role not in capability set or camera not open | Check `fw_video_config` and open order |
| Crash after AI enable | Race with camera_manager lifecycle | mutex + local device pointer |
| JSON config ignored | Path not in board overlay or patched over | Edit `board/<BOARD>/rootfs/.../etc/config/` |
| No video in call / `VIDEO_DECODER_SW_STUB` | `CONFIG_FRAMEWORK_MULTIMEDIA_FFMPEG_LINK_MODE=none` | Nova: `decode` + third_party `DECODE_ONLY`; see [celestia-project](celestia-project.en.md) §APP/`libmultimedia` |
| `usr.sqsh4` exceeds APP (`0x82E000`) | FFmpeg linked into `libmultimedia` too large | `LINK_MODE=decode` + `DECODE_ONLY`; or move content to SD `install/` |

Runtime taxonomy: [celestia-debug](celestia-debug.en.md) **classes B/D/E/G**.

## Build after coding (required)

```bash
cd ~/celestia
./build.sh -f --product-name anypi_nova_dev
./build.sh -f --product-name anypi_luna
# If HAL contract usage changed, hardware first:
./build.sh -h -f --product-name anypi_luna
# luna loop (e.g. multimedia preview): build + flash + serial log
./build.sh -f --product-name anypi_luna
```

Fix all compile errors before delivery. Missing symbols → third_party (lvgl/cjson) not built, or HAL public header changed. After `CONFIG_FW_*` changes, check both products’ `project/<proj>/config.mk`.