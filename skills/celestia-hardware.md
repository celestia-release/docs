---
name: celestia-hardware
description: >-
  Celestia Hardware HAL 层（hardware/）开发、排障与新增 SoC：以 hw_module_t / hw_device_t 契约屏蔽芯片差异，
  唯一对外公开头是 hardware/include/*.h（audio/camera/display/osd/gpu/wifi/ethernet/input/gpio/...），
  各 SoC 私有实现在 hardware/<module>/hal/<SoC>/ 经 hw_register_module() 注册（AnyCloud39AV200 /
  AnyCloud39AV100_V2 / Sky39AV130 三平台，gpu 另有 Sky37Ev2 部分实现）。对下经 /dev/* ioctl 调 Kernel/BSP，对上只暴露公开头。
  SoC 选择走 project/<proj>/config.mk 的 CONFIG_SOC_PLATFORM（字符串目录名）；模块开关走 CONFIG_HW_*。
  禁止改 hardware/include/*.h 迁就单一 SoC、禁止反向 include framework/application。改动后用 build.sh -h 编译。
  排查驱动/传感器/显示/音频/编码行为、或新增 SoC（芯片 SDK → 逐模块 hal/<NewSoC>/）时使用。
  新 SoC 时 system/core 通常不动，仅新增 HAL 模块号时改 system_error.h（见 celestia-system-core）。
---

<p align="right"><a href="celestia-hardware.en.md">English</a> | <b>中文</b></p>

# Hardware HAL 层 skill

> 概念：`docs/guide/hardware-hal.md`；对外 API：`docs/guide/hardware-api.md`；全局索引：`docs/guide/api-index.md`。
> 总览见 [celestia](celestia.md)。

## 目录地图

```
hardware/
├── include/           # ❗唯一对外公开头（跨 SoC 签名一致，不得为单一 SoC 修改）
├── hardware.c         # 注册表 + hw_modules_init()，按 #if CONFIG_HW_* 调各模块 init
├── audio/  camera/  osd/  gpu/         # 有 hal/<SoC>/
├── display/                            # 扁平：display_hw.c / fb_hw.c（无 hal/，无 api 头）
├── vision/isp/hal/<SoC>/  vision/motion_detect/hal/<SoC>/   # hal 在子模块下
├── network/           # wifi/（含 aic8800/ wpa/）ethernet/ net/  —— 均扁平，无 hal/<SoC>/
├── input/             # input_module.c + evdev/（evdev.c/keyboard.c/touch.c）扁平
├── driver/            # 每外设各占顶层目录：adc blkdev efuse gpio ir led motor pwm wdt
│                      #   ir/、motor/ 有 hal/<SoC>/；其余扁平；无 driver/hal/、无 driver/mmc/
│                      #   MMC 在 driver/blkdev/mmc.c
└── soc_platform/hal/<SoC>/             # 时钟/pinmux/IRQ bring-up；IPU：hal_ipu_weak_stub.c + 各 SoC hal_ipu.c
```

**有 `hal/<SoC>/` 的模块（新增 SoC 只在这些下建目录）**：audio、camera、osd、gpu、vision/isp、vision/motion_detect、soc_platform、driver/ir、driver/motor。
**扁平（不建 hal/<SoC>/）**：display、network/{wifi,ethernet,net}、input（含 evdev）、driver/{adc,blkdev,efuse,gpio,led,pwm,wdt}。
三 SoC = `AnyCloud39AV200`（nova）/ `AnyCloud39AV100_V2`（luna 系）/ `Sky39AV130`（mira：audio、camera、vision/isp、vision/motion_detect、soc_platform、driver/ir）；`gpu/hal/` 另有 `Sky37Ev2/`（部分）。

## 公开头（`hardware/include/*.h`，共 24 个）

`adc.h audio.h camera.h display.h driver_error.h efuse.h ethernet.h gpio.h gpu.h hardware.h header.h input.h input_evdev.h ir.h isp.h led.h mmc.h motion_detect.h motor.h osd.h pwm.h soc_platform.h wdt.h wifi.h`

`header.h` 是伞形 `#include`；核心结构 `HAL_MODULE_INFO_SYM / hw_module_t / hw_device_t / hw_module_methods_t / HARDWARE_MODULE_TAG / HARDWARE_DEVICE_TAG / HARDWARE_HAL_API_VERSION` 都在 `hardware.h`。

## HAL 契约模式

每个 SoC 模块在 `hardware/<module>/hal/<SoC>/hal_<module>.c` 提供 `HAL_MODULE_INFO_SYM` + ops，并实现 `hw_<module>_module_init()` → `hw_register_module()`。`hardware.c` 暴露 `hw_register_module / hw_get_module / hw_get_module_by_class / hw_modules_init`；`hw_modules_init()` 按 `#if CONFIG_HW_*` 调各 `hw_<x>_module_init()`。**SoC 选择不在 hardware.c 写 `#ifdef`**——走 Makefile 的 `CONFIG_SOC_PLATFORM`（见下）。错误码域 `HARDWARE_DOMAIN=0x02`。

最小骨架（Audio 为例，其余同模式）：

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

int hw_audio_module_init(void) {           /* 由 hardware.c:hw_modules_init() 调用 */
    return hw_register_module(&HAL_MODULE_INFO_SYM.common);
}
```

## 头文件边界（严格，唯一表）

| 类别 | 位置 | 谁可 include |
|------|------|-------------|
| 公开 HAL 头 | `hardware/include/*.h` | Framework、HAL 自身 |
| HAL 私有 API | `hardware/<m>/hal_<m>_api.h` 或 `hardware/<m>/<m>_hw.h` | **仅** `hardware/<m>/*.c` 与 `hardware/<m>/hal/<SoC>/*.c` |
| 厂商 SDK 头 | `hardware/<m>/hal/<SoC>/<vendor>/` | 仅当前 SoC 的 `.c` |

`hal_<m>_api.h` **只存在于**：audio、camera、gpu、osd、vision/isp、vision/motion_detect、soc_platform（另有 `hal_ipu_api.h`）。其余模块只有 `<m>_hw.h` 或无 api 头（display、network/wifi、network/ethernet、input/evdev、driver/*）——不要假设 api/_hw 成对出现。

## 禁止事项（必拒）

- ❌ 修改 `hardware/include/*.h` 以迁就某一 SoC（应通过私有头扩展）；
- ❌ 在 HAL 实现里 `#include "framework/**"` 或 `"application/**"`；
- ❌ 把 SoC 厂商 SDK 符号或 SoC 名宏泄漏进 `hardware/include/*.h`（公开层必须 SoC 无关）；
- ❌ 让 Framework/App 看到 `hal_*_api.h` / `*_hw.h`。

## 排障：以 HAL 为主

1. 先在 `hardware/<module>/` 与对应 `hal/<SoC>/`（扁平模块直接看 `*_hw.c`）内分析：ops 实现、寄存器/ioctl 序列、buffer 管理、sensor/编码配置。
2. 怀疑 Kernel/BSP 时**只确认 `/dev/*` 设备与 ioctl 契约**：节点是否存在、`.ko` 是否加载（`lsmod`/`dmesg`）、ioctl 返回值。
3. 仅当驱动层证据明确（设备缺失、ioctl 错误、dmesg 报错）才下钻 [celestia-project](celestia-project.md)（BSP/DTS/insmod）——vendor BSP 默认只读，多数问题应在 HAL 层 workaround。
4. 改公开头或共性逻辑时，检查各 SoC `hal/` 是否需对称修改。

## 新增 SoC（芯片 SDK → 逐模块 HAL）

> 工作表：[roles.md §4](../quick-start/roles.md#4-新芯片移植开发者)。全流程参考：[bsp-and-products §9](../quick-start/bsp-and-products.md)。

**前置输入**：原厂芯片 SDK（工具链 + kernel/驱动 + 公开头），解压到 `system/<NewSoC>/<SDK_Ver>/`（见 [download](../quick-start/download.md#放置芯片-sdk)）。HAL 通过 Makefile `-I$(SYSTEM_ROOT)/$(CONFIG_SOC_PLATFORM)/include` 等路径消费 SDK 头；**禁止**把厂商符号泄漏进 `hardware/include/*.h`。

**层边界**（勿混）：

| 层 | 新 SoC 动作 |
|---|---|
| `system/<NewSoC>/<SDK>/` + project board | 放 SDK、DTS、insmod、overlay → [celestia-project](celestia-project.md) |
| `hardware/**/hal/<NewSoC>/` | **每个启用且有 `hal/<SoC>/` 的模块**各建一份实现（本节） |
| `system/core` | 通常不动；仅新增 HAL 模块时追加 `HARDWARE_MODULE_*` → [celestia-system-core](celestia-system-core.md) |
| Framework / App / `hardware/include` | 不为单一 SoC 改公开契约 |

**config.mk**（范例 `project/anypi_nova_dev/config.mk`）：SoC 用字符串选择，模块用 `=` 开关：

```make
CONFIG_SOC_PLATFORM   = AnyCloud39AV200     # 目录名字符串，决定 hal/<SoC>/ 路径
CONFIG_SOC_SDK_VERSION = V1.04

CONFIG_HW_AUDIO = y      CONFIG_HW_CAMERA = y     CONFIG_HW_DISPLAY = y
CONFIG_HW_GPU = y        CONFIG_HW_OSD = y
CONFIG_HW_VISION_ISP = y CONFIG_HW_VISION_MOTION_DETECT = y
CONFIG_HW_WIFI = y       CONFIG_HW_ETHERNET = y
CONFIG_HW_DRIVER_GPIO = y  ... ADC/PWM/MOTOR/LED/IR/WDT/EFUSE/BLKDEV/MMC = y
CONFIG_HW_INPUT_KEYBOARD = y  CONFIG_HW_INPUT_TOUCH = y
CONFIG_HW_SOC_PLATFORM = y   # 启用 soc_platform *模块*，≠ CONFIG_SOC_PLATFORM（勿混淆）
```

**Makefile**（范例 `hardware/audio/Makefile`、isp/ir 等）用 `CONFIG_SOC_PLATFORM` 变量化路径，无 per-module `ifeq` 门控：

```make
include $(COMPILATION_RULES)
SRCS    := $(wildcard $(SRC_DIR)/*.c) $(wildcard $(SRC_DIR)/hal/$(CONFIG_SOC_PLATFORM)/*.c)
INCLUDE += -I$(MODULE_ROOT)/hal/$(CONFIG_SOC_PLATFORM)
# SDK 头：-I$(SYSTEM_ROOT)/$(CONFIG_SOC_PLATFORM)/include
# 头拷贝步骤：$(shell cp -f $(MODULE_ROOT)/../include/<m>.h $(MODULES_INCLUDE))
```

变量用 `MODULE_ROOT / SRCS / INCLUDE`（或 `HARDWARE_INCLUDE`），**不是** `CELESTIA_ROOT / MODULE_SRCS / INC_FLAGS`；`HW_SOC` 不存在。

### 逐模块步骤（必做）

1. 对照能力清单，只打开需要的 `CONFIG_HW_*`；未用模块保持 `n`。
2. **对每个「有 `hal/<SoC>/`」且已启用的模块**：新建 `hardware/<module>/hal/<NewSoC>/`（vision 在子路径下），参考基线 SoC 同名文件，实现 `HAL_MODULE_INFO_SYM` + ops + `hw_<m>_module_init()`，内部只调本芯片 SDK / `/dev` ioctl。
3. 必做 `soc_platform/hal/<NewSoC>/`（时钟/pinmux/IRQ）；缺 IPU 时按 `hal_ipu_weak_stub.c` + 各 SoC `hal_ipu.c` 弱桩补齐。
4. **扁平模块**（display、network/*、input、多数 driver）**不建** `hal/<NewSoC>/`；用板级驱动 + 现有 `*_hw.c` 验证。
5. 若引入**全新** HAL 模块（新 ID）：在 `system/core` 的 `system_error.h`（`HARDWARE_DOMAIN=0x02`）追加 `HARDWARE_MODULE_*`，并改 `hardware.c` 注册；独立提交 system/core 仓。仅移植已有模块到新 SoC → **不动** system/core。
6. `config.mk` 设 `CONFIG_SOC_PLATFORM=<NewSoC>` + `CONFIG_SOC_SDK_VERSION`；project/board 见 bsp §9。

**模块勾选表**（有 `hal/<SoC>/`）：`soc_platform` · `camera` · `audio` · `osd` · `gpu` · `vision/isp` · `vision/motion_detect` · `driver/ir` · `driver/motor`。

**bring-up 自检**：L1 `/dev/*` → L2 `hardware/tests/` → L3 Framework preview/recorder → L4 App → L5 云。完成后同步 `docs/guide/hardware-hal.md` SoC 矩阵、`docs/architecture/overview.md`、`docs/guide/api-index.md`。

## 常见问题

| 现象 | 根因 | 修复 |
|---|---|---|
| `/dev/video0` 缺失 | insmod 顺序 / ko 路径错 | 核对启动脚本 + `lsmod`/`dmesg` |
| `hw_get_module` 失败 | `hw_<m>_module_init` 未被调用 | 开 `CONFIG_HW_<M>` + `hardware.c` 调用 + 重编 |
| `Failed to get video hardware module: -1` | stale `hardware.o`（config.h 非 make 依赖）| Makefile 给 `hardware.o` 加 config.h 依赖；`-c -h` |
| sensor 不出帧 / CHN0 select timeout | `num_slices≥2` 但 VENC 未 bind 到 VI（slice 通道靠内核编码，不出 YUV 帧）| AV100_V2 已支持：`ak_app_video_bind_chn(KERNEL_ENCODE_MODE)` + `ak_app_video_venc_get_stream`；其它 SoC 未实现则保持 `num_slices=1` |
| `vi 3dnr memsize ... 5040 KB` 异常 | 3DNR memsize 常量未 clamp | clamp + `HAL_CAMERA_VI_3DNR_KB` env |
| 预览 `invalid inbuf cap=524288` | `max_frame_payload` 在 `codec_mode` 前算 | 先解析 `codec_mode` 再算 payload |
| 改公开头后另一 SoC 编不过 | 公开头被破 SoC 无关 | 私有头扩展，两 SoC 都编 |
| Makefile 路径变量报错 | 用了 `CELESTIA_ROOT`/`MODULE_SRCS` | 改 `MODULE_ROOT`/`SRCS`/`INCLUDE` |

AI 工作表见 [roles.md §新芯片移植](../quick-start/roles.md#4-新芯片移植开发者)。

## 编码完成 → 编译（必做）

```bash
cd ~/celestia
./build.sh -h --product-name anypi_nova_dev    # AnyCloud39AV200
./build.sh -h --product-name anypi_luna    # AnyCloud39AV100_V2
# HAL 改动常影响 framework，按依赖顺序连编：
./build.sh -h -f -a --product-name anypi_luna
```

解决全部编译错误再交付。`hardware` 依赖 `system/core` + SoC SDK；缺 SDK 头/工具链时核对 `project/<proj>/config.mk` 的 `CONFIG_SOC_PLATFORM` / `CONFIG_SOC_SDK_VERSION`。改公开头后两 SoC 都要能编过。
