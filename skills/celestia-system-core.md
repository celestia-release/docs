---
name: celestia-system-core
description: >-
  Celestia System Core 层（system/core/）开发与排障：被 Application/Framework/HAL 三层横向依赖的
  独立基础库——log、config(INI)、thread/锁、time/timer、watchdog、memory、crypto(md5/crc32/base64)、
  str/geometry/image_convert 工具、list/ring_buffer/circular_buffer 容器、统一错误码 system_error.h。
  对外头 system/core/include/*.h，统一入口 system_core.h。绝不上向依赖 framework/hardware。
  改动后用 build.sh -s 编译并评估三层兼容。排查日志/线程/配置/定时/错误码时使用；
  新芯片移植通常不动本层，仅新增 HAL 模块号时改 system_error.h（见「新芯片移植时」）。
---

<p align="right"><a href="celestia-system-core.en.md">English</a> | <b>中文</b></p>

# System Core 层 skill

> 概念：`docs/guide/system-core.md`；API 速查：`docs/guide/system-core-api.md`。
> 总览与跨层规则见 [celestia](celestia.md)。

## 定位

横向公共能力层（version 0.12.0），**被 App / Framework / HAL 三层共同 `#include` 依赖**。它是**叶子层，绝不上向依赖 framework/hardware**；改这里牵一发动全身，必须保持对外头与行为**向后兼容**。

## 目录地图

`system/core/` 下 11 个模块子目录：`config crypto file geometry image_convert log memory str thread time watchdog`，加 `include/`（27 个对外公开头）与 Makefile。产物：`libsystem_core.so` 及静态库 `libsystem_core.a`。

公开头分组（`system/core/include/*.h`，共 27 个，逐一见 `include/`）：

- 入口/基础：`system_core.h`（聚合入口，含 `SYSTEM_CORE_IMPORT_*` 宏）、`system_error.h system_types.h system_macros.h system_version.h system_assert.h system_debug.h system_reentry.h`
- 各模块：`log.h`、`system_config.h`、`system_thread.h gthread.h`、`system_time.h timer.h`、`watchdog.h`、`system_memory.h`、`md5.h crc32.h base64.h`、`string_utils.h geometry.h image_convert.h system_pixel_format.h system_file.h`
- 容器：`list.h ring_buffer.h circular_buffer.h`

## 关键约定

- **错误码** `system_error.h`：`SYSTEM_ERROR_CODE(domain, module, error)`，Domain = `SYSTEM_DOMAIN=0x00`/`FRAMEWORK_DOMAIN=0x01`/`HARDWARE_DOMAIN=0x02`；0=成功，负数=错误。各模块错误码拆分在**分模块子头**（`config/config_error.h`、`log/log_error.h`、`thread/thread_error.h` …），配合 `system_error.h` 里的 `SYSTEM_MODULE_*` 表查模块号。
- **日志** `log.h`：`log_error / log_warn / log_info / log_debug / log_verbose`。每个 `.c` 顶部 `#define LOG_TAG "..."`。
- **重入守卫** `system_reentry.h`：`REENTRY_INIT / REENTRY_START / REENTRY_STOP / REENTRY_GET_INIT_STATUS` 宏，worker 模块 init/start/stop/check 的标准重入保护范式。
- **配置** `system_config.h`：封装 iniparser 的 INI 读写。
- **线程** `system_thread.h`：线程 + mutex + rwlock。
- **几何** `geometry.h`：`rect_intersection`（求交集矩形，返回 1/0 是否重叠）、`rect_right`/`rect_bottom`、`rect_to_uv_rect`、`percent_to_real_value`/`percent_to_value`、`uv_rect_t`（**无 IoU**；`rect_t` 定义在 `system_pixel_format.h`，不在此头）。
- **图像转换** `image_convert.h`：YUV420SP(NV12 系)↔rgbline(packed RGB 扫描线) 与 ROI 缩放/拷贝——`yuv420sp_roi_resize`、`yuv420sp_roi_copy`、`yuv420sp_roi_resize_rgbline_version1/2`、`rgbline_to_yuv_420sp`、`rgbline_roi_resize_version1/2/3`、`save_rgbline_and_yuv`（**无 JPEG，无 RGB24→JPEG**）。

## 陷阱：聚合入口不全

`system_core.h` 的聚合是**部分的**：即便 `SYSTEM_CORE_IMPORT_ALL=1`，它也**不会**引入 `system_macros.h / system_assert.h / system_debug.h / system_reentry.h / system_file.h / system_pixel_format.h / list.h / ring_buffer.h`——用到这些必须**直接 `#include` 对应头**。

## 禁止事项（必拒）

- ❌ `#include` 任何 `framework/**` 或 `hardware/**`（含公开头）——会形成上向依赖；
- ❌ 破坏现有公开头的函数签名/结构体布局/宏语义（三层调用方会编译失败或 ABI 不兼容）；
- ✅ 新增能力优先**追加**新函数/新头，不改动既有签名。

## 新芯片移植时（与 HAL / BSP 的边界）

`system/core` **跨 SoC 共用**，不在本层实现 `hal/<SoC>/`。新芯片流程以 [celestia-hardware 新增 SoC](celestia-hardware.md#新增-soc芯片-sdk--逐模块-hal) + [roles §4](../quick-start/roles.md#4-新芯片移植开发者) 为准。

| 场景 | 是否改 system/core |
|---|---|
| 仅为已有模块（camera/audio/…）补新 SoC HAL | **否** |
| 新增**全新** HAL 模块（新 module id）| **是**：在 `system_error.h`（`HARDWARE_DOMAIN=0x02`）追加 `HARDWARE_MODULE_*`，错误码子头按需；独立提交本仓 |
| HAL 需要新的横向工具（log/线程/像素格式等）| 优先复用现有公开头；确需新 API 则**追加**，保持兼容，并用 `-s -h -f -a` 验证三层 |

HAL 实现侧应 `#include "system_core.h"`（及所需子头）使用 log/错误码/线程等，**不要**在 system/core 内按芯片分叉。

## 排障要点

1. 在 `system/core/<module>/` 内分析实现——叶子层没有"下层可甩锅"，问题基本就在本层或调用方误用。
2. 若上层报错：先区分是上层**用错接口**（参数、生命周期、线程安全）还是本层实现 bug——写/看 `tests/` 或最小复现来判定。
3. 改动后**必须评估三层调用方**：grep `framework/`、`application/`、`hardware/` 中对改动符号的引用，确认兼容。

## 编码完成 → 编译（必做）

```bash
cd ~/celestia
./build.sh -s --product-name anypi_luna      # 单编 system/core
```

- 改了**公开头**后，三层依赖方都要重编验证兼容：`./build.sh -s -h -f -a --product-name anypi_luna`；hardware/framework/application 任一编不过即视为不兼容改动，需回退或加兼容层。
- `system/core` 依赖 third_party 的 cjson 头；若报缺头先 `./build.sh -t cjson`。
