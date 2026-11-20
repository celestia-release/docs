---
name: celestia-contributing
description: >-
  Celestia contribution conventions: unified C/C++ naming and style (naming, include guards,
  SPDX/file headers, error codes, logging, forbidden new patterns) and documentation rules
  (fact-source priority, keep layer docs aligned with code; code wins). Read naming rules
  before new code or headers; read doc rules before updating docs/. Use when checking naming,
  header format, include guards, error code encoding, or log conventions.
---

<p align="right"><b>English</b> | <a href="celestia-contributing.md">中文</a></p>

# Contributing conventions skill

> Luna on-device environment: [celestia-project](celestia-project.en.md) “Luna bring-up”;
> overview and cross-layer rules: [celestia](celestia.en.md).
>
> Scope: `application/`, `framework/`, `hardware/`, `system/core/` and C/C++ that depends on them.
> Consistency first: all public symbols (functions/globals/macros/error codes) must use a module prefix;
> do not mix in legacy `AK_*` or `g*` prefixes.
> Converge legacy code during feature work; avoid large mechanical-only migrations.

## Naming conventions

| Category | Rule | Example |
|------|------|------|
| Basic types | Use `<stdint.h>` / `<stdbool.h>` | `uint8_t`, `int32_t`, `bool` |
| Struct typedef | `snake_case_t` | `config_path_t`, `audio_config_base_t` |
| Enum types | `module_xxx_e` or `_t`; values ALL_CAPS + module prefix | `system_state_e`, `SYSTEM_STATE_INIT` |
| Function pointer types | `MODULE_*_FUNCTION` | `SYSTEM_COMPARE_FUNCTION` |
| Functions | `<module>_<verb>[_<object>]` | `watchdog_init`, `audio_hw_open_output_stream` |
| Callbacks | Suffix `_callback` / `_cb` / `_handler` | `module_xxx_callback_t` |
| Macros/constants | ALL_CAPS + module prefix | `SYSTEM_MAX_PATH`, `WATCHDOG_FILE_PATH` |
| Feature switches | `MODULE_FEATURE_ENABLE/DISABLE` or 0/1 values | `CONFIG_NETWORK_RTSP_ENABLE` |
| Globals | `g_` process-wide; `s_` file static | `g_app_state`, `s_watchdog_fd` |
| Files | `module[_submodule]_action.c`, `.c` ↔ `.h` same base name | `config_path.c`, `audio_hw.c` |
| Thread entry | `<module>[_<role>]_thread`; lifecycle `_start/_stop/_join` | `watchdog_thread` |
| Concurrency | Caller locks: `_unsafe`; already locked: `_locked` | — |
| Tests | File `xxx_test.c`; function `test_<module>_<case>` | `test_watchdog_invalid_param` |

- With 3+ related statics in a module, prefer a `module_context_t` (`s_module_ctx`); lone flags/constants may stay separate.
- Layer deps: `application` depends only on `framework` / each layer `include/` / `system_core`, not hardware include; `framework` reaches hardware only via `hardware/include`, never chip HAL; `system/core` does not depend upward.
- Third-party/compat (glib, iniparser, etc.) keep native names; do not imitate their style for new public APIs.

## Include guards

Format `<LAYER>_<FILENAME>_H` (strip `.h`, uppercase, underscores); `#endif` must repeat the same name in a comment. Prefer ≤50 chars; add path bits if same-layer collision.

```c
#ifndef <LAYER>_<FILENAME>_H
#define <LAYER>_<FILENAME>_H
/* ... */
#endif /* <LAYER>_<FILENAME>_H */
```

| Directory | Layer prefix | Example |
|------|--------|------|
| `system/core/` | `SYSTEM_CORE` | `SYSTEM_CORE_CONFIG_H` |
| `framework/` | `FRAMEWORK` | `FRAMEWORK_RUNTIME_H`, `FRAMEWORK_DEVICE_H` |
| `hardware/` | `HARDWARE` | `HARDWARE_AUDIO_H`, `HARDWARE_AUDIO_HW_H` |
| `application/` | `APPLICATION` | `APPLICATION_APP_MAIN_H`, `APPLICATION_ALIYUN_APP_H` |

Disallow legacy (`CELESTIA_*_H`, `__MODULE_H__`, etc.). Public headers use `extern "C"` for C/C++ interop; prefer forward declarations to reduce compile deps.

## File headers / SPDX

All owned `.c` / `.h` start with SPDX (`//` single line), then copyright block + path + short description; no `@file`/`@brief` in the header block.

```c
// SPDX-License-Identifier: Apache-2.0
/*
 * Copyright (C) <current year> Guangzhou Anyka Microelectronics Co., Ltd.
 *
 * This file is part of Celestia/<domain>/<module>.
 *
 * Short description of this file/module.
 */
```

- SPDX must be the absolute first line (before guard, `#include`, copyright); no BOM. New code must have it; third-party files use actual license (`GPL-2.0`, `MIT`, etc.).
- Copyright year = **current year**; company name fixed: "Guangzhou Anyka Microelectronics Co., Ltd."
- Public (non-`static`) functions need `/** @brief ... @param ... @return ... */`; types/enums use `/** ... */`, members `/**< ... */`, inline `//` (space after) or `/* */`. Comments in English, explain intent not line-by-line translation. No orphan TODOs (need owner), no long-term dead code.

## Error codes

- Success `0` (`XXX_OK`); errors negative (`int32_t` / `xxx_error_t`).
- Encoding: `SYSTEM_ERROR_CODE(domain, module, error)`, allocated from `system/core/include/system_error.h`.
- Domains: `SYSTEM_DOMAIN=0x00` / `FRAMEWORK_DOMAIN=0x01` / `HARDWARE_DOMAIN=0x02`.
- Each public module has its own error header (`config_error.h`, `audio_error.h`, `network_error.h`, etc.).
- Do not return raw `-errno` publicly; may log `errno`/`strerror`; cross-layer mapping uses unified adapters.

## Logging

- Each `.c`: `#define LOG_TAG "MODULE_NAME"` at top.
- APIs: `log_error / log_warn / log_info / log_debug / log_verbose`; assertions via `system_macros.h` `SYSTEM_EXPECT_VERBOSE_RETURN*`.
- No plaintext passwords/keys/tokens (mask if needed).

Error-code logging policy:

| Level | Scenario | Handling |
|------|------|------|
| Must log | Memory/IO failure, device/hardware errors, syscall failure, critical flow failure | `log_error` + error code |
| Should log | Not initialized, invalid state, op failure, timeout | `log_error/warn` as appropriate |
| May skip | Param validation, expected high-frequency errors (empty/full queue), already logged upstream | Return without log |

## Do not add

- `AK_*` types/macros (only inside `hardware/*/hal/`, not exposed upward).
- glib-style `g*` types (`gint`, `gboolean`, `gpointer`) or redefined basics (`U8`, `S32`).
- `SUCCESS/FAILED`, `TRUE/FALSE`, macros/error codes without module prefix.
- Unprefixed public APIs (`Init`, `StartService`) or CamelCase mixed prefixes (`AKWatchdogInit`).

## Documentation rules

Fact priority: `tests/` > `*.c/*.cpp` > `*.h` > `Makefile/build.sh/config.mk` > `docs/`.

Keep layer docs aligned with code; on conflict **code wins** and update docs. Anchors:

- Framework: only symbols in `framework/**/include/*.h` (umbrella `framework/runtime/include/runtime.h`).
- HAL: only `hardware/include/*.h`, never `hal/<SoC>/` private headers in docs.
- System Core: verify signatures starting from `system/core/include/system_core.h`.
- Application: handle APIs per `application/app_main/app_main.h` (`app_main_get_*`, not `launcher_get_*`); GUI under `packages/tablet` / `packages/wearable`.

Products: `anypi_nova` (`ipc_main`) / `anypi_nova_dev` (`nova_main`) / `anypi_luna` (`luna_main`) / `anypi_luna_dev` (`luna_dev_main`) / `anypi_mira_dev` (`mira_main`). If `--product-name` omitted, interactive pick and cache. Avoid dead paths `gui/stella`, `docs/release/`.

Pre-submit self-check: names/paths/CONFIG macros exist in source; no stale references; internal links point to existing `docs/` files; example commands match `build.sh --help`; no trailing whitespace.