---
name: celestia-hardware
description: >-
  Celestia Hardware HAL layer (hardware/) development, troubleshooting, and new SoC: hw_module_t /
  hw_device_t contract hides chip differences; sole public headers are hardware/include/*.h
  (audio/camera/display/osd/gpu/wifi/ethernet/input/gpio/...); per-SoC private code under
  hardware/<module>/hal/<SoC>/ via hw_register_module() (AnyCloud39AV200 / AnyCloud39AV100_V2 /
  Sky39AV130; gpu also partial Sky37Ev2). Downward /dev/* ioctl to Kernel/BSP; upward only public
  headers. SoC via project/<proj>/config.mk CONFIG_SOC_PLATFORM; modules via CONFIG_HW_*. Do not
  change hardware/include/*.h for one SoC or include framework/application upward. Build with build.sh -h.
  Use for driver/sensor/display/audio/encode issues or new SoC port. system/core usually unchanged
  except new HAL module IDs in system_error.h (see celestia-system-core).
---

<p align="right"><b>English</b> | <a href="celestia-hardware.md">中文</a></p>

# Hardware HAL layer skill

> Concepts: `docs/guide/hardware-hal.md`; public API: `docs/guide/hardware-api.md`; index: `docs/guide/api-index.md`.
> Overview: [celestia](celestia.en.md).

## Directory map

```
hardware/
├── include/           # ❗ sole public headers (cross-SoC identical signatures; never per-SoC tweaks)
├── hardware.c         # registry + hw_modules_init(); #if CONFIG_HW_* per module
├── audio/  camera/  osd/  gpu/         # hal/<SoC>/
├── display/                            # flat: display_hw.c / fb_hw.c (no hal/, no api header)
├── vision/isp/hal/<SoC>/  vision/motion_detect/hal/<SoC>/
├── network/           # wifi/ (aic8800, wpa) ethernet/ net/ — all flat
├── input/             # input_module.c + evdev/ flat
├── driver/            # adc blkdev efuse gpio ir led motor pwm wdt per top-level dir
│                      #   ir/, motor/ have hal/<SoC>/; MMC in driver/blkdev/mmc.c
└── soc_platform/hal/<SoC>/             # clock/pinmux/IRQ; IPU weak stub + per-SoC hal_ipu.c
```

**Modules with `hal/<SoC>/` (new SoC only here)**: audio, camera, osd, gpu, vision/isp, vision/motion_detect, soc_platform, driver/ir, driver/motor.
**Flat (no hal/<SoC>/)**: display, network/*, input/evdev, driver/{adc,blkdev,efuse,gpio,led,pwm,wdt}.
Three SoCs: `AnyCloud39AV200` (nova) / `AnyCloud39AV100_V2` (luna) / `Sky39AV130` (mira: audio, camera, vision/*, soc_platform, driver/ir); `gpu/hal/` also `Sky37Ev2/` (partial).

## Public headers (`hardware/include/*.h`, 24)

`adc.h audio.h camera.h display.h driver_error.h efuse.h ethernet.h gpio.h gpu.h hardware.h header.h input.h input_evdev.h ir.h isp.h led.h mmc.h motion_detect.h motor.h osd.h pwm.h soc_platform.h wdt.h wifi.h`

`header.h` umbrella; core structs `HAL_MODULE_INFO_SYM / hw_module_t / hw_device_t / hw_module_methods_t / HARDWARE_MODULE_TAG / HARDWARE_DEVICE_TAG / HARDWARE_HAL_API_VERSION` in `hardware.h`.

## HAL contract pattern

Each SoC module in `hardware/<module>/hal/<SoC>/hal_<module>.c` supplies `HAL_MODULE_INFO_SYM` + ops and `hw_<module>_module_init()` → `hw_register_module()`. `hardware.c` exposes `hw_register_module / hw_get_module / hw_get_module_by_class / hw_modules_init`; the latter calls each `hw_<x>_module_init()` under `#if CONFIG_HW_*`. **No `#ifdef` SoC pick in hardware.c** — Makefile `CONFIG_SOC_PLATFORM`. Errors: `HARDWARE_DOMAIN=0x02`.

Minimal skeleton (audio; others same):

```c
static struct hw_module_methods_t s_methods = { .open = audio_hw_open };

static struct audio_module HAL_MODULE_INFO_SYM = {
    .common = {
        .tag                = HARDWARE_MODULE_TAG,
        .hal_api_version    = HARDWARE_HAL_API_VERSION,
        .id                 = AUDIO_HARDWARE_MODULE_ID,   /* "audio_hal" */
        .name               = "<SoC> Audio HAL",
        .methods            = &s_methods,
    },
};

int hw_audio_module_init(void) {
    return hw_register_module(&HAL_MODULE_INFO_SYM.common);
}
```

## Header boundaries (strict)

| Kind | Location | Who may include |
|------|------|-------------|
| Public HAL | `hardware/include/*.h` | Framework, HAL itself |
| HAL private API | `hardware/<m>/hal_<m>_api.h` or `hardware/<m>/<m>_hw.h` | **Only** `hardware/<m>/*.c` and `hardware/<m>/hal/<SoC>/*.c` |
| Vendor SDK | `hardware/<m>/hal/<SoC>/<vendor>/` | Only that SoC’s `.c` |

`hal_<m>_api.h` **only in**: audio, camera, gpu, osd, vision/isp, vision/motion_detect, soc_platform (+ `hal_ipu_api.h`). Others use `<m>_hw.h` or none (display, network, input, driver/*) — do not assume api/_hw pairs.

## Forbidden (must refuse)

- ❌ Change `hardware/include/*.h` for one SoC (extend via private headers);
- ❌ `#include "framework/**"` or `"application/**"` in HAL;
- ❌ Leak vendor SDK symbols or SoC name macros into `hardware/include/*.h`;
- ❌ Expose `hal_*_api.h` / `*_hw.h` to Framework/App.

## Troubleshooting: HAL-first

1. Analyze `hardware/<module>/` and `hal/<SoC>/` (flat: `*_hw.c`): ops, registers/ioctl, buffers, sensor/encode config.
2. If Kernel/BSP suspected, **only confirm `/dev/*` and ioctl contract**: node exists, `.ko` loaded (`lsmod`/`dmesg`), ioctl returns.
3. Only with driver evidence (missing device, ioctl errors, dmesg) drill to [celestia-project](celestia-project.en.md) (BSP/DTS/insmod) — vendor BSP read-only; prefer HAL workarounds.
4. When changing public headers or shared logic, check all SoC `hal/` implementations.

## New SoC (chip SDK → per-module HAL)

> Worksheet: [roles §4](../quick-start/roles.en.md#4-新芯片移植开发者). Full flow: [bsp-and-products §9](../quick-start/bsp-and-products.en.md).

**Prerequisites**: Vendor SDK (toolchain + kernel/drivers/headers) under `system/<NewSoC>/<SDK_Ver>/` ([download](../quick-start/download.en.md#放置芯片-sdk)). HAL consumes SDK via Makefile `-I$(SYSTEM_ROOT)/$(CONFIG_SOC_PLATFORM)/include`; **never** leak vendor symbols into `hardware/include/*.h`.

**Layer split**:

| Layer | New SoC actions |
|---|---|
| `system/<NewSoC>/<SDK>/` + project board | SDK, DTS, insmod, overlay → [celestia-project](celestia-project.en.md) |
| `hardware/**/hal/<NewSoC>/` | **Each enabled module with `hal/<SoC>/`** gets an implementation (this section) |
| `system/core` | Usually unchanged; new HAL module → `HARDWARE_MODULE_*` in [celestia-system-core](celestia-system-core.en.md) |
| Framework / App / `hardware/include` | Do not change public contract for one SoC |

**config.mk** (example `project/anypi_nova_dev/config.mk`): SoC string + module switches:

```make
CONFIG_SOC_PLATFORM   = AnyCloud39AV200
CONFIG_SOC_SDK_VERSION = V1.04

CONFIG_HW_AUDIO = y      CONFIG_HW_CAMERA = y     CONFIG_HW_DISPLAY = y
CONFIG_HW_GPU = y        CONFIG_HW_OSD = y
CONFIG_HW_VISION_ISP = y CONFIG_HW_VISION_MOTION_DETECT = y
CONFIG_HW_WIFI = y       CONFIG_HW_ETHERNET = y
CONFIG_HW_DRIVER_GPIO = y  ... ADC/PWM/MOTOR/LED/IR/WDT/EFUSE/BLKDEV/MMC = y
CONFIG_HW_INPUT_KEYBOARD = y  CONFIG_HW_INPUT_TOUCH = y
CONFIG_HW_SOC_PLATFORM = y   # soc_platform module; ≠ CONFIG_SOC_PLATFORM string
```

**Makefile** (e.g. `hardware/audio/Makefile`): path via `CONFIG_SOC_PLATFORM`, no per-module `ifeq`:

```make
include $(COMPILATION_RULES)
SRCS    := $(wildcard $(SRC_DIR)/*.c) $(wildcard $(SRC_DIR)/hal/$(CONFIG_SOC_PLATFORM)/*.c)
INCLUDE += -I$(MODULE_ROOT)/hal/$(CONFIG_SOC_PLATFORM)
```

Use `MODULE_ROOT / SRCS / INCLUDE`, not `CELESTIA_ROOT / MODULE_SRCS / INC_FLAGS`; `HW_SOC` does not exist.

### Per-module steps

1. Enable only needed `CONFIG_HW_*`.
2. **Each enabled module with `hal/<SoC>/`**: create `hardware/<module>/hal/<NewSoC>/`, copy baseline SoC files, implement `HAL_MODULE_INFO_SYM` + ops + `hw_<m>_module_init()`, call SDK / `/dev` ioctl only inside.
3. Required `soc_platform/hal/<NewSoC>/`; IPU via `hal_ipu_weak_stub.c` + `hal_ipu.c` if missing.
4. **Flat modules**: no `hal/<NewSoC>/`; validate with board drivers + existing `*_hw.c`.
5. **Brand-new HAL module**: append `HARDWARE_MODULE_*` in `system_error.h`, register in `hardware.c`; separate system/core commit. Port of existing modules → **no** system/core change.
6. Set `CONFIG_SOC_PLATFORM=<NewSoC>` + `CONFIG_SOC_SDK_VERSION`; board per bsp §9.

**Module table** (with `hal/<SoC>/`): `soc_platform` · `camera` · `audio` · `osd` · `gpu` · `vision/isp` · `vision/motion_detect` · `driver/ir` · `driver/motor`.

**Bring-up self-check**: L1 `/dev/*` → L2 `hardware/tests/` → L3 Framework preview/recorder → L4 App → L5 cloud. Sync `docs/guide/hardware-hal.md` SoC matrix, `docs/architecture/overview.md`, `docs/guide/api-index.md`.

## Common issues

| Symptom | Root cause | Fix |
|---|---|---|
| Missing `/dev/video0` | insmod order / ko path | start scripts + `lsmod`/`dmesg` |
| `hw_get_module` fails | `hw_<m>_module_init` not called | `CONFIG_HW_<M>` + `hardware.c` + rebuild |
| `Failed to get video hardware module: -1` | stale `hardware.o` (config.h not dep) | Makefile config.h dep; `-c -h` |
| No frames / CHN0 select timeout | `num_slices≥2` but VENC not bound to VI | AV100_V2: bind + `venc_get_stream`; else `num_slices=1` |
| Abnormal `vi 3dnr memsize` | 3DNR constant not clamped | clamp + `HAL_CAMERA_VI_3DNR_KB` |
| `invalid inbuf cap=524288` | payload before `codec_mode` | parse `codec_mode` first |
| Other SoC breaks after public header edit | broke SoC-neutral contract | extend via private headers; build all SoCs |
| Makefile path vars wrong | used `CELESTIA_ROOT`/`MODULE_SRCS` | use `MODULE_ROOT`/`SRCS`/`INCLUDE` |

Worksheet: [roles §new SoC](../quick-start/roles.en.md#4-新芯片移植开发者).

## Build after coding (required)

```bash
cd ~/celestia
./build.sh -h --product-name anypi_nova_dev    # AnyCloud39AV200
./build.sh -h --product-name anypi_luna    # AnyCloud39AV100_V2
./build.sh -h -f -a --product-name anypi_luna
```

Fix all errors. `hardware` needs `system/core` + SoC SDK; verify `CONFIG_SOC_PLATFORM` / `CONFIG_SOC_SDK_VERSION`. After public header changes, both SoCs must build.