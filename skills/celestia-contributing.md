---
name: celestia-contributing
description: >-
  Celestia 代码贡献规范：C/C++ 统一命名与风格（命名约定、Include Guard、SPDX/文件头、错误码、
  日志、禁止新增项）与文档编写规则（事实来源优先、各层文档与代码对齐，代码为准）。
  新增代码或头文件前必读命名规范；更新 docs/ 文档前必读文档编写规则。
  当需要检查命名规范、文件头格式、Include Guard、错误码编码、日志规范时使用。
---

<p align="right"><a href="celestia-contributing.en.md">English</a> | <b>中文</b></p>

# 贡献规范 skill

> Luna 上板环境见 [celestia-project](celestia-project.md) 的「Luna 上板」；
> 总览与跨层规则见 [celestia](celestia.md)。
>
> 适用范围：`application/`、`framework/`、`hardware/`、`system/core/` 及依赖它们的 C/C++ 代码。
> 一致性优先：所有对外符号（函数/全局变量/宏/错误码）必须带模块前缀；不混用 `AK_*`、`g*` 历史前缀。
> 存量代码在功能迭代中顺带收敛，避免为“纯迁移”做大规模机械替换。

## 命名规范

| 类别 | 规则 | 示例 |
|------|------|------|
| 基础类型 | 用 `<stdint.h>`/`<stdbool.h>` 标准类型 | `uint8_t`、`int32_t`、`bool` |
| 结构体 typedef | `snake_case_t` | `config_path_t`、`audio_config_base_t` |
| 枚举类型 | `module_xxx_e` 或 `_t`；枚举值全大写 + 模块前缀 | `system_state_e`、`SYSTEM_STATE_INIT` |
| 函数指针类型 | `MODULE_*_FUNCTION` | `SYSTEM_COMPARE_FUNCTION` |
| 函数 | `<module>_<verb>[_<object>]` | `watchdog_init`、`audio_hw_open_output_stream` |
| 回调 | `_callback` / `_cb` / `_handler` 后缀 | `module_xxx_callback_t` |
| 宏/常量 | 全大写 + 模块前缀 | `SYSTEM_MAX_PATH`、`WATCHDOG_FILE_PATH` |
| 特性开关 | `MODULE_FEATURE_ENABLE/DISABLE` 或值 0/1 | `CONFIG_NETWORK_RTSP_ENABLE` |
| 全局变量 | `g_` 进程级全局；`s_` 文件内静态 | `g_app_state`、`s_watchdog_fd` |
| 文件 | `module[_submodule]_action.c`，`.c`↔`.h` 同名 | `config_path.c`、`audio_hw.c` |
| 线程入口 | `<module>[_<role>]_thread`；生命周期 `_start/_stop/_join` | `watchdog_thread` |
| 并发标识 | 调用者加锁 `_unsafe`；内部已锁 `_locked` | — |
| 测试 | 文件 `xxx_test.c`；函数 `test_<module>_<case>` | `test_watchdog_invalid_param` |

- 模块内有 3 个及以上相关静态变量时，组织为 `module_context_t` 结构体（实例名 `s_module_ctx`）；单个独立常量/标志可保持独立。
- 层间依赖：`application` 只依赖 `framework` / 各层 `include/` / `system_core`，不依赖 hardware include；`framework` 只经 `hardware/include` 访问硬件，不 include 芯片 HAL；`system/core` 不依赖上层。
- 第三方/兼容层（glib、iniparser 等）保留原生命名，但禁止在自有模块仿其风格新增公共接口。

## Include Guard

格式 `<LAYER>_<FILENAME>_H`（文件名去 `.h`、转大写、下划线分隔），`#endif` 后必须带同名注释。总长建议 ≤50 字符；同层重名时加路径信息区分。

```c
#ifndef <LAYER>_<FILENAME>_H
#define <LAYER>_<FILENAME>_H
/* ... */
#endif /* <LAYER>_<FILENAME>_H */
```

| 目录 | 层级前缀 | 示例 |
|------|--------|------|
| `system/core/` | `SYSTEM_CORE` | `SYSTEM_CORE_CONFIG_H` |
| `framework/` | `FRAMEWORK` | `FRAMEWORK_RUNTIME_H`、`FRAMEWORK_DEVICE_H` |
| `hardware/` | `HARDWARE` | `HARDWARE_AUDIO_H`、`HARDWARE_AUDIO_HW_H` |
| `application/` | `APPLICATION` | `APPLICATION_APP_MAIN_H`、`APPLICATION_ALIYUN_APP_H` |

禁止旧格式（`CELESTIA_*_H`、`__MODULE_H__` 等）。对外头文件用 `extern "C"` 保证 C/C++ 互操作；优先前置声明减少编译依赖。

## 文件头 / SPDX

所有自有 `.c` / `.h` 首行必须是 SPDX 许可声明（`//` 单行形式），随后版权块 + 路径 + 简述；文件头不用 `@file`/`@brief` 标签。

```c
// SPDX-License-Identifier: Apache-2.0
/*
 * Copyright (C) <当前年份> Guangzhou Anyka Microelectronics Co., Ltd.
 *
 * This file is part of Celestia/<domain>/<module>.
 *
 * Short description of this file/module.
 */
```

- SPDX 行必须在 include guard、`#include`、版权块之前，是文件绝对首行；文件不得以 BOM 起始。新代码一律补齐；第三方文件按实际许可（`GPL-2.0`、`MIT` 等）填写。
- 版权年份填**当前年份**，公司名固定为 "Guangzhou Anyka Microelectronics Co., Ltd."。
- 对外函数（非 `static`）必须有 `/** @brief ... @param ... @return ... */`；类型/枚举用 `/** ... */`，成员注释用 `/**< ... */`，行内用 `//`（后跟空格）或 `/* */`。注释用英文，解释设计意图而非逐行翻译。禁止无主 TODO（须带 owner）、长期死代码。

## 错误码

- 成功 `0`（`XXX_OK`）；错误为负数（`int32_t` / `xxx_error_t`）。
- 编码：`SYSTEM_ERROR_CODE(domain, module, error)`，统一从 `system/core/include/system_error.h` 分配。
- 域：`SYSTEM_DOMAIN=0x00` / `FRAMEWORK_DOMAIN=0x01` / `HARDWARE_DOMAIN=0x02`。
- 每个对外模块有独立错误头（`config_error.h`、`audio_error.h`、`network_error.h` 等）。
- 对外不直接返回 `-errno`；如需可在日志中携带 `errno`/`strerror`，跨层映射用统一适配函数。

## 日志

- 每个 `.c` 顶部定义 `#define LOG_TAG "MODULE_NAME"`。
- 接口：`log_error / log_warn / log_info / log_debug / log_verbose`；断言用 `system_macros.h` 的 `SYSTEM_EXPECT_VERBOSE_RETURN*`。
- 禁止打印明文密码/密钥/Token 等敏感信息（必要时掩码）。

错误码日志策略：

| 级别 | 场景 | 处理 |
|------|------|------|
| 必须记录 | 内存/IO 失败、设备/硬件错误、系统调用失败、关键流程失败 | `log_error` + 错误码 |
| 建议记录 | 未初始化、无效状态、操作失败、超时 | 视情况 `log_error/warn` |
| 可跳过 | 参数校验失败、队列空/满等预期高频错误、调用链上层已记录 | 直接返回，不记日志 |

## 禁止新增

- `AK_*` 类型/宏（仅 `hardware/*/hal/` 内部可保留，不得向上层暴露）。
- `g*` glib 风格类型（`gint`、`gboolean`、`gpointer`）与重定义基本类型（`U8`、`S32`）。
- `SUCCESS/FAILED`、`TRUE/FALSE`、无模块前缀的宏与错误码。
- 无模块前缀的对外公共函数（如 `Init`、`StartService`）及 CamelCase 混合前缀（`AKWatchdogInit`）。

## 文档编写规则

事实来源优先级：`tests/` > `*.c/*.cpp` > `*.h` > `Makefile/build.sh/config.mk` > `docs/`。

各层文档与代码对齐，矛盾时**以代码为准**并更新文档。锚点：

- Framework：只列 `framework/**/include/*.h` 符号（Framework 汇总头 `framework/runtime/include/runtime.h`）。
- HAL：只列 `hardware/include/*.h`，禁止写入 `hal/<SoC>/` 私有头。
- System Core：以 `system/core/include/system_core.h` 聚合头为入口逐头核对签名。
- Application：句柄 API 以 `application/app_main/app_main.h` 为准（`app_main_get_*`，非 `launcher_get_*`）；GUI 目录 `packages/tablet` / `packages/wearable`。

产品：`anypi_nova`（`ipc_main`）/ `anypi_nova_dev`（`nova_main`）/ `anypi_luna`（`luna_main`）/ `anypi_luna_dev`（`luna_dev_main`）/ `anypi_mira_dev`（`mira_main`）。未传 `--product-name` 时交互选择并缓存。避免废弃路径 `gui/stella`、`docs/release/`。

提交前自检：函数名/路径/配置宏可在源码找到；无过时引用；内部链接均指向 `docs/` 现有文件；示例命令与 `build.sh --help` 一致；行尾无空白。
