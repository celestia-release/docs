---
name: celestia-system-core
description: >-
  Celestia System Core layer (system/core/) development and troubleshooting: independent
  foundation library horizontally used by Application, Framework, and HAL — log, config (INI), thread/locks,
  time/timer, watchdog, memory, crypto (md5/crc32/base64), str/geometry/image_convert utils,
  list/ring_buffer/circular_buffer containers, unified error codes in system_error.h.
  Public headers under system/core/include/*.h, umbrella system_core.h. Must not depend upward
  on framework/hardware. Build with build.sh -s and assess three-layer compatibility. Use for
  logs/threads/config/timers/error codes; new SoC ports usually skip this layer except new HAL
  module IDs in system_error.h (see "New SoC port").
---

<p align="right"><b>English</b> | <a href="celestia-system-core.md">中文</a></p>

# System Core layer skill

> Concepts: `docs/guide/system-core.md`; API cheat sheet: `docs/guide/system-core-api.md`.
> Overview and cross-layer rules: [celestia](celestia.en.md).

## Role

Horizontal shared layer (version 0.12.0), **included by App / Framework / HAL**. It is a **leaf layer — never depends upward on framework/hardware**; changes here affect everyone; keep public headers and behavior **backward compatible**.

## Directory map

`system/core/` has 11 module subdirs: `config crypto file geometry image_convert log memory str thread time watchdog`, plus `include/` (27 public headers) and Makefile. Outputs: `libsystem_core.so` and static `libsystem_core.a`.

Public headers (`system/core/include/*.h`, 27 total — see `include/`):

- Umbrella/base: `system_core.h` (aggregate entry, `SYSTEM_CORE_IMPORT_*` macros), `system_error.h system_types.h system_macros.h system_version.h system_assert.h system_debug.h system_reentry.h`
- Modules: `log.h`, `system_config.h`, `system_thread.h gthread.h`, `system_time.h timer.h`, `watchdog.h`, `system_memory.h`, `md5.h crc32.h base64.h`, `string_utils.h geometry.h image_convert.h system_pixel_format.h system_file.h`
- Containers: `list.h ring_buffer.h circular_buffer.h`

## Key conventions

- **Error codes** `system_error.h`: `SYSTEM_ERROR_CODE(domain, module, error)`, domains `SYSTEM_DOMAIN=0x00` / `FRAMEWORK_DOMAIN=0x01` / `HARDWARE_DOMAIN=0x02`; 0 success, negative error. Per-module splits in **sub-headers** (`config/config_error.h`, `log/log_error.h`, `thread/thread_error.h`, …) with `SYSTEM_MODULE_*` table in `system_error.h`.
- **Logging** `log.h`: `log_error / log_warn / log_info / log_debug / log_verbose`. Each `.c`: `#define LOG_TAG "..."` at top.
- **Reentry guards** `system_reentry.h`: `REENTRY_INIT / REENTRY_START / REENTRY_STOP / REENTRY_GET_INIT_STATUS` standard pattern for worker init/start/stop/check.
- **Config** `system_config.h`: INI read/write via iniparser.
- **Threads** `system_thread.h`: threads + mutex + rwlock.
- **Geometry** `geometry.h`: `rect_intersection` (overlap 1/0), `rect_right`/`rect_bottom`, `rect_to_uv_rect`, `percent_to_real_value`/`percent_to_value`, `uv_rect_t` (**no IoU**; `rect_t` in `system_pixel_format.h`).
- **Image convert** `image_convert.h`: YUV420SP (NV12) ↔ rgbline, ROI scale/copy — listed helpers (**no JPEG, no RGB24→JPEG**).

## Pitfall: partial umbrella

`system_core.h` aggregation is **partial**: even with `SYSTEM_CORE_IMPORT_ALL=1`, it does **not** pull `system_macros.h / system_assert.h / system_debug.h / system_reentry.h / system_file.h / system_pixel_format.h / list.h / ring_buffer.h` — **include those headers directly** when needed.

## Forbidden (must refuse)

- ❌ `#include` any `framework/**` or `hardware/**` (including public headers) — upward dependency;
- ❌ Change existing public signatures/struct layout/macro semantics (breaks three layers);
- ✅ Prefer **appending** new functions/headers, not changing existing signatures.

## New SoC port (boundary with HAL / BSP)

`system/core` is **shared across SoCs**, no `hal/<SoC>/` here. Follow [celestia-hardware new SoC](celestia-hardware.en.md#新增-soc芯片-sdk--逐模块-hal) + [roles §4](../quick-start/roles.en.md#4-新芯片移植开发者).

| Scenario | Change system/core? |
|---|---|
| Add HAL for existing modules (camera/audio/…) on new SoC | **No** |
| Brand-new HAL module (new module id) | **Yes**: add `HARDWARE_MODULE_*` in `system_error.h` (`HARDWARE_DOMAIN=0x02`), error sub-headers as needed; separate commit |
| HAL needs new horizontal utils (log/thread/pixel format, etc.) | Reuse existing headers first; if new API required, **append** only, stay compatible, verify with `-s -h -f -a` |

HAL code should `#include "system_core.h"` (and needed sub-headers); **do not** fork by chip inside system/core.

## Troubleshooting

1. Analyze in `system/core/<module>/` — leaf layer; issues are here or caller misuse.
2. If upper layer fails: distinguish **misused API** (params, lifetime, threading) vs this-layer bug — use `tests/` or minimal repro.
3. After changes **grep all three layers** for touched symbols; confirm compatibility.

## Build after coding (required)

```bash
cd ~/celestia
./build.sh -s --product-name anypi_luna      # system/core only
```

- After **public header** changes, rebuild all dependents: `./build.sh -s -h -f -a --product-name anypi_luna`; any hardware/framework/application failure = incompatible change — revert or add compat layer.
- `system/core` needs third_party cjson headers; if missing, `./build.sh -t cjson` first.