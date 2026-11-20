---
name: celestia-application
description: >-
  Celestia 应用层（application/）开发与排障：主进程 app_main（ipc_main/nova_main/luna_main/luna_dev_main/mira_main）、
  启动流程、`packages/` 三类包（ipcamera（`IOTCamera`）、tablet（`TABLET`）、wearable（`WEARABLE`））与 packages/common 共用组件、
  IOTCamera 业务（packages/ipcamera：ipc_cam 通用）、tablet/wearable App、云平台适配（cloud/aliyun、cloud/volc_engine）、
  系统服务（services/）、settings.json（含内嵌 installed_apps）。仅经 app_main_get_*() 与 *_open() 调 Framework，
  禁止直连 HAL/BSP。含"新增 tablet App""新增 cloud 适配"清单。改动后用 build.sh -a 编译。
---

<p align="right"><a href="celestia-application.en.md">English</a> | <b>中文</b></p>

# Application 层 skill

> 概念与 API：`docs/guide/application.md`；API 索引：`docs/guide/api-index.md`；
> 总览与跨层规则见 [celestia](celestia.md)。

## 目录地图

```
application/
├── app_main/          # 主进程：main.c / app_main.c / app_config.c
│                      #   产出 = CONFIG_APP_MAIN_TARGET：
│                      #   ipc_main(anypi_nova) / nova_main(anypi_nova_dev)
│                      #   / luna_main(anypi_luna) / luna_dev_main(anypi_luna_dev)
│                      #   / mira_main(anypi_mira_dev)
├── packages/
│   ├── common/        # 跨产品共用 GUI（如 gui_camera.c 预览）
│   ├── tablet/        # 平板级（`TABLET`）桌面（800x480 / 1024x600）；每 App = tablet_<id>.c/.h
│   ├── wearable/      # 穿戴级（`WEARABLE`） Launcher（240x240，ActionCam）；每 App = wearable_<id>_*.c/.h
│   └── ipcamera/      # IOTCamera 业务 SO（module_init / module_start）：ipc_cam/ 通用
├── cloud/             # 云平台适配：aliyun / volc_engine / mike
├── services/          # 系统服务：uvc / weather ...
└── tests/             # 应用层 demo / 单测
```

## 对外/对下接口（只能这样调）

应用层**没有自身对外 API**，通过两条路与下层交互：

| 调用 | 头文件 | 用途 |
|------|--------|------|
| `app_main_get_multimedia/network/notification/nebula/runtime()` | `application/app_main/app_main.h` | 取 Framework 服务句柄（`void*` 需强转）|
| `*_open()` + ops（`multimedia_t`/`network_t`/...） | `framework/services/*/include/*.h` | 调用 Framework 能力 |
| `#include "system_core.h"` 及子头 | `system/core/include/*.h` | 日志 / 线程 / 配置 / 错误码 |

启动序列（`application/app_main/main.c`）：
`app_main_platform_startup()` → `runtime_bootstrap_open()`（+`runtime->load_config`、`app_main_register_runtime`）
→ `app_main_init()` → `app_main_load_config()` → `app_main_run()`
→ `app_main_start_framework_modules(headless, gui_config)`
→ `tablet_run()`（`CONFIG_APP_PACKAGES_TABLET`）或 `app_gui_run()`（`CONFIG_APP_PACKAGES_WEARABLE`）。

配置文件：
- `settings.json`：`{ "program", "launch_mode", "modules": ... }`，SO 热插拔；`so_path` 为运行期绝对路径（如 `/usr/lib/libipc_cam_app.so`，云 SO `/usr/lib/lib*_app.so`）。
- 桌面 App 清单内嵌在 `settings.json` 的 `installed_apps` 对象：`{"schema_version":1,"apps":[...]}`，每项 `install_path` 形如 `/usr/apps/<id>`（luna）或 `/mnt/install/apps/<id>`（nova）；独立 `/etc/config/installed_apps.json` 仅作 legacy 回退。

## 禁止事项（必拒）

- ❌ `#include "audio.h"/"camera.h"` 或任何 `hardware/include/*.h`、SoC 私有头（`hal_*_api.h`/`*_hw.h`）；
- ❌ 直接 `ioctl` / 访问 `/dev/*` / 调 BSP；
- ✅ 需硬件能力时改走 Framework 服务 ops；缺能力时下钻 [celestia-framework](celestia-framework.md)。

## 排障：以应用层为主

1. 先在 `application/` 内分析：启动流程、GUI 事件、App 注册、`settings.json` 解析、SO `dlopen/dlsym`。
2. 怀疑下层时**只确认 Framework 接口契约**：核对 `framework/services/*/include/*.h` 签名/返回值，看本层 `*_open()`/ops 的参数、时序、句柄是否为 NULL。
3. 仅当 Framework 接口返回错误码或日志证明其失败，才切到 [celestia-framework](celestia-framework.md) 下钻。
4. 双产品：改 `packages/common`、`app_main` 时评估 tablet / wearable / ipcamera 三种形态。

## 新增 tablet App（清单）

参考现有：`application/packages/tablet/clock/`、`.../camera/`。SO 模式，**不需重编主进程**。

1. **目录** `application/packages/tablet/<id>/`：`app.json`、`app_entry_<id>.c`、`tablet_<id>.c`、`tablet_<id>.h`、`res/icon.png`。
   共用宏在 `application/packages/tablet/launcher/tablet_common.h`（无 `lv_demo_*`、无 `lv_demo_common.h`）。
2. **app.json** 字段（对齐真实文件）：`app_id / name / action / icon / entry_so / entry_symbol / version`，例：
   ```json
   { "app_id": "<id>", "name": "<Name>", "action": "<id>",
     "icon": "res/icon.png", "entry_so": "lib/lib<id>_app.so",
     "entry_symbol": "app_entry_start_<id>", "version": "1.0.0" }
   ```
   SO 名 = `entry_so` 的 basename（各 App **不同**，非统一 lib<id>_app.so）。
3. **入口** `app_entry_<id>.c`（首行 SPDX）：
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
   /* 防 --gc-sections 剪除的 keep-alive 自引用（必写）*/
   static int (*const s_keep_app_entry_<id>)(lv_obj_t *) = app_entry_start_<id>;
   ```
   `tablet_<id>.h` 头文件 guard 复用同一特性宏 `CONFIG_APP_TABLET_<ID_UPPER>`。
4. **特性宏**：在 `project/<product>/config.mk` 加 `CONFIG_APP_TABLET_<ID_UPPER> = y`（无独立 `LV_DEMO_*`）。
5. **Makefile 无需修改**——`application/packages/tablet/Makefile` 不含任何应用信息：它扫描带 `app.json` 的目录发现应用、按 `CONFIG_APP_TABLET_<ID>` 开关启用；`.so` 名取自该应用 `app.json` 的 `entry_so` basename；应用专属编译/链接选项（额外 include、链接库、额外对象、过滤源码）写在应用目录的 `app.mk`（变量 `APP_CFLAGS_EXTRA` 目标规则、`<ID>_SHLIB_LIBS`、`<ID>_APP_EXTRA_OBJS`、`<ID>_SRCS_FILTER_OUT`），参照 `ipc_camera/app.mk`、`setting/app.mk`。
   > 该 Makefile 用 `-include .../config.h` 强制包含生成的项目 config.h，让每个 nova 源文件都能看到 `CONFIG_APP_TABLET_*` 宏；system/core 的 INI API 头是 `system_config.h`（已改名），与项目 config.h 不冲突。
6. **注册**：在 `settings.json`（板级 overlay `project/<p>/system/<SoC>/<SDK>/board/<Board>/rootfs/utils/etc/config/settings.json`）的 `installed_apps.apps[]` 追加，`install_path` 与 rootfs 内实际安装路径一致（nova 多为 `/mnt/install/apps/<id>`，固进 `/usr` 的 App 为 `/usr/apps/<id>`）。
   Nova 桌面经 Registry 加载 App，一般**无需**改 `modules`。
7. **自检**：
   ```bash
   grep -rE '#include\s+[<"](audio|camera|display|osd|gpu|isp|wifi|gpio|hardware)\.h[>"]' application/packages/tablet/<id>/   # 应空
   ${CROSS}nm -D out/<product>/install/application/lib/lib<id>_app.so | grep app_entry_start_<id>   # CROSS: AV200=riscv64-anyka-linux-musl- / luna=arm-anycloud-linux-uclibcgnueabi-
   python3 -c "import json;json.load(open('application/packages/tablet/<id>/app.json'))"
   ```

> wearable App 清单见下节「新增 wearable App」。
>
> 用同步工具新增、修改或删除 tablet APP 的步骤见 [celestia-studio](celestia-studio.md)。修改 APP 只动原目录，推送仍覆盖原来的 `/mnt/install/apps/<id>`。

## 新增 cloud 适配（清单）

已实现两种，形态不同——按其一仿制：
- `application/cloud/aliyun/`：静态库 `libaliyun_app.a`，`PROJ = aliyun_app`，导出 `aliyun_app_init()/aliyun_app_deinit()`。
- `application/cloud/volc_engine/`：动态库 `libvolc_engine_svc.so`，`PROJ = volc_engine_svc`，导出 `volc_engine_init()/volc_engine_deinit()`。
- `tuya` 仅配置占位（无目录）。

1. **目录** `application/cloud/<id>/`：`Makefile`、`include/<id>_app.h`、`<id>_app.c`（可选 `src/`、`lib/` 厂商 SDK）。
   适配器真正导出的是 `<id>_app_init()/<id>_app_deinit()`（业务契约）。
2. **通用 SO 加载契约**：`application/app_main/app_config.c::load_so_module()` 仅 `dlsym`
   `module_init(const char*)` 与 `module_start(void)`（**无** `module_exit`，无人调用它）。
   若走该通用装载路径才需导出 `module_init/module_start`；已发布的 aliyun/volc 是链接进主进程，非此路径。
3. **Makefile**（对齐真实写法）：
   ```make
   include $(COMPILATION_RULES)
   include $(PROJECT_CONFIG_FILE)
   PROJ = <name>
   # ... 源码/编译规则 ...
   # 安装到 $(APPLICATION_INSTALL_DIR)/lib
   ```
   （非 `CELESTIA_ROOT/scripts/rules.mk`、非 `MODULE_NAME/SO_NAME/INSTALL_DIR`。）
4. **顶层接线** `application/Makefile` 用 kbuild 风格：
   ```make
   obj-$(CONFIG_APP_CLOUD_<ID>) += cloud/<id>
   ```
   宏后缀是选定短 token，可与目录名不同（目录 `volc_engine` → `CONFIG_APP_CLOUD_VOLC`）。
5. **允许/禁止** 同"禁止事项"：可用 Framework `*_open()`（network/multimedia/notification/storage 等）、
   `system/core` 横向能力、适配器内部厂商 SDK 头；禁 HAL 公共/私有头、SO 内开 UI。

## 应用例子（仿现有，勿从零）

| 目标 | 仿照路径 |
|---|---|
| 最小壳 | `application/packages/tablet/clock/` |
| 预览/拍照 | `packages/wearable/camera/`、`packages/tablet/camera/`、`packages/common/` |
| 循环录+RTSP+私有配置 | `packages/wearable/dashcam/` |
| 扫码 / AI 设置 | `packages/tablet/setting/tablet_setting_qr_scan.c`、`tablet_setting_ai.c` |
| 音频算法 UI | `packages/tablet/audio_algo/` |
| IOTCamera 单 SO（新建）| 新建 `packages/ipcamera/<id>/`：导出 `module_init`/`module_start`；业务对照 `dashcam` / Framework API |
| 火山云 / 阿里云 | `cloud/volc_engine/`、`cloud/aliyun/`（业务入口 `*_init`；通用装载另认 `module_*`）|
| UVC / 天气 | `services/uvc/`、`services/weather/`（`app_runtime` 中为 `type:static`）|

完整提问样例与填写表见 [roles.md §应用开发者](../quick-start/roles.md#3-应用开发者)。

## IOTCamera 单应用（推荐方案）

**适用**：新产品 `IOTCamera` 形态 + 单业务（如 `anypi_luna_dev`）。见 [roles.md §2 表 C](../quick-start/roles.md#填写表-c--iotcamera-单应用anypi_luna_dev-典型-已回填)。

装载契约（`application/app_main/app_config.c` + `main.c`）：

- `launch_mode` ≠ `ui` 为 headless；两 GUI 宏皆关时，若无阻塞的 `module_start`，`main` 会在 framework 启动后返回。
- `type:so` 模块经 `dlsym("module_init")` / `dlsym("module_start")` 调用。
- `services/uvc`、`services/weather` 为 **static** 模块，用途不同；云 SO 业务入口多为 `*_init`。

| 项 | 做法 |
|---|---|
| 主程序 | `CONFIG_APP_MAIN_TARGET=luna_dev_main`；`CONFIG_APP_PACKAGES_TABLET=n`；`CONFIG_APP_PACKAGES_WEARABLE=n` |
| 目录 | 新建 `application/packages/ipcamera/<id>/`（Makefile 产出 `.so`）|
| 导出 | `int module_init(const char *config_path)`；`int module_start(void)` |
| 保活 | 在 **`module_start` 内阻塞**（业务循环）|
| 业务参考 | `app_main_get_multimedia/network/nebula`；录像/RTSP 可对照 `packages/wearable/dashcam/`（去掉 LVGL）|
| 注册 | `settings.json`：`"launch_mode":"headless"`，唯一 `type:so` + 运行期 `so_path` |
| 桌面表 | 不必 `installed_apps.json` |
| 启动 | 脚本拉 `luna_dev_main` |

带显示单应用仍走 `packages/wearable|tablet/<id>/` + `installed_apps`。

## 新增 wearable App（要点）

真实多文件：`application/packages/wearable/<id>/wearable_<id>_*.c/.h` + `app_entry_<id>.c` + `app.json` + `res/icon.png`。

1. **入口**：`visibility("default")` + keep-alive 自引用（防 `--gc-sections`）
2. **Makefile**：`ENABLED_APP_DIRS/IDS`、`APP_SO_FILE_<id>`、`APP_SO_TARGETS`、`install` case
3. **特性宏**：`project/<p>/config.mk` → `CONFIG_APP_WEARABLE_<ID>=y`
4. **注册**：`installed_apps.json`（`install_path=/usr/apps/<id>`）；码流用量只动 `fw_video_config.json` 的 `app_stream_usage`（见 `docs/guide/framework.md` §2.5.1）
5. **注册表**：`wearable_app_registry.c`；入口经 `app_gui_run()`

AI 工作表见 [roles.md §应用开发者](../quick-start/roles.md#3-应用开发者)。

## 常见问题

| 现象 | 根因 | 修复 |
|---|---|---|
| App SO `dlopen` 失败 / 入口找不到 | `--gc-sections` 剪除 entry 符号 | `visibility("default")` + keep-alive 自引用 |
| App 不出现在桌面 | `installed_apps.json` 未注册 / `install_path` 错 | 注册 `apps[]`、路径对齐运行期 |
| `app_runtime` SO 没加载 | `so_path` 用源码路径而非运行期 | 运行期用 `/usr/lib/...`（按 rootfs 实际安装路径）|
| cloud `module_exit` 不执行 | 通用装载只 `dlsym` `module_init/module_start` | 不依赖 `module_exit`；主进程链接或自管生命周期 |
| 码流改了不生效 | 改了 `channels/streams` 能力集 | 按 §2.5.1 只动 `app_stream_usage` |
| 越层 include 编译报未知类型 | `#include "hardware/include/*.h"` | 改走 Framework `*_open()` ops |
| `CONFIG_APP_TABLET_*` 不生效 | `packages/tablet/Makefile` 未 `-include` config.h | `-include $(PROJECT_GEN_INCLUDE_DIR)/config.h` |
| IOTCamera 形态主进程启动后即退出 | `main.c` 无 GUI 时 return | app SO `module_start` 阻塞保活 |

## 编码完成 → 编译（必做）

```bash
cd ~/celestia
./build.sh -a --product-name anypi_nova          # 量产 / ipc_main
./build.sh -a --product-name anypi_nova_dev      # 平板级（`TABLET`） EVB / nova_main
./build.sh -a --product-name anypi_luna          # 穿戴级（`WEARABLE`） / luna_main
./build.sh -a --product-name anypi_luna_dev      # IOTCamera / luna_dev_main
./build.sh -a --product-name anypi_mira_dev          # IOTCamera aos / mira_main
```

解决所有编译错误后再交付；改了共用代码时对**另一产品也跑一次 `-a`**。
错误若来自 Framework 头/符号，先按排障原则确认是调用误用还是需下钻 framework。
概念细节见 `docs/guide/application.md`，接口签名见 `docs/guide/api-index.md`。
