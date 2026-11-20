<p align="right"><a href="roles.en.md">English</a> | <b>中文</b></p>

# 按角色导读（AI 协作入口）

> 本文是 **AI 编程对讲工作表**：按角色给出「代码生成填写表」与提问模板。
> 目录约定 / 接口边界 / 编译命令 / 排障细节 → 对应 [skills](../skills/README.md)；概念与 API → [guide](../guide/api-index.md)；以代码为准。

## 1. 如何与 AI 协作

**快速开发方案**：选角色 → 从该节「可提问场景」复制一句 → 需要端到端时再贴「填写表」→ AI 读 skill 落地并回填 → 编译上板。

1. **选角色** → 贴对应填写表 → 要求 AI **逐项推进并回填路径/参数/状态/验证**
2. AI **先读该行主 skill**（`docs/skills/<name>.md`），再按需下钻 guide / 代码
3. 改完 **编译 + 上板**：对应 `build.sh` flag → 烧录 → 串口 PASS/FAIL（[celestia-debug](../skills/celestia-debug.md)）
4. 提交走 [git-gitee](../skills/git-gitee.md)（一仓一 squash、默认 `release`）

**提示词模板**：`按下表逐项走对应 skill，改完 -k/-f/-a/-p 编译打包并上板验证：<贴表>`

触发 skill：斜杠 `/celestia-application` 等（链接见 [skills/README](../skills/README.md)），或让 AI 直接读 `docs/skills/<name>.md`。不确定归层时先 `/celestia`。

| 角色 | 一句话上手 | 场景与样例 |
|---|---|---|
| 新产品 | 复制基线产品 + 改 config/board | [§2](#2-新产品移植开发者) |
| 应用 | 新 App / 改码流 / 接云 | [§3](#3-应用开发者) |
| 新芯片 | 芯片 SDK → 逐模块 HAL + BSP | [§4](#4-新芯片移植开发者) |
| 调试 | 贴 log → 归 A–G 类 → 闭环 | [§5](#5-调试与维护) |

| skill | 源 | 配套 guide / 文档 |
|---|---|---|
| `/celestia` | [celestia.md](../skills/celestia.md) | [architecture/overview](../architecture/overview.md) |
| `/celestia-application` | [celestia-application.md](../skills/celestia-application.md) | [application](../guide/application.md)、[framework-api](../guide/framework-api.md) |
| `/celestia-framework` | [celestia-framework.md](../skills/celestia-framework.md) | [framework](../guide/framework.md)、[framework-api](../guide/framework-api.md) |
| `/celestia-hardware` | [celestia-hardware.md](../skills/celestia-hardware.md) | [hardware-hal](../guide/hardware-hal.md)、[hardware-api](../guide/hardware-api.md) |
| `/celestia-system-core` | [celestia-system-core.md](../skills/celestia-system-core.md) | [system-core](../guide/system-core.md)、[system-core-api](../guide/system-core-api.md) |
| `/celestia-project` | [celestia-project.md](../skills/celestia-project.md) | [bsp-and-products](bsp-and-products.md) |
| `/celestia-scripts` | [celestia-scripts.md](../skills/celestia-scripts.md) | [build](build.md) |
| `/celestia-third-party` | [celestia-third-party.md](../skills/celestia-third-party.md) | — |
| `/celestia-debug` | [celestia-debug.md](../skills/celestia-debug.md) | [deploy](deploy.md) |
| `/celestia-contributing` | [celestia-contributing.md](../skills/celestia-contributing.md) | — |
| `/git-gitee` | [git-gitee.md](../skills/git-gitee.md) | — |

上板前序：[download](download.md) → [build](build.md) → [deploy](deploy.md)；架构总览 [overview](../architecture/overview.md)。

| 你要做 | 主 skill | 本节 |
|---|---|---|
| 新产品 / 新板型组装 | `/celestia-project` | [§2](#2-新产品移植开发者) |
| 加 App / 云适配 / GUI | `/celestia-application` | [§3](#3-应用开发者) |
| 新 SoC：SDK + 逐模块 HAL + BSP | `/celestia-hardware`（+ `/celestia-system-core` / `/celestia-project`）| [§4](#4-新芯片移植开发者) |
| 上板排障 / 崩溃 | `/celestia-debug` | [§5](#5-调试与维护) |

状态：☐ 待办 / ⏳ 进行中 / ✅ 完成；验证填 `PASS` 或 `FAIL + 现象`。全部 ✅ + 上板 PASS 才算完成。

---

## 2. 新产品移植开发者

> 已有 SoC 上组装新产品 / 新板。细节：[celestia-project](../skills/celestia-project.md)（含 **指定 BOARD**、**UI 形态**、**Board 处理**、**config.mk 精简**）、[bsp-and-products](bsp-and-products.md)。

**开新产品时先声明**（缺一不可）：

| 输入 | 示例 | 说明 |
|---|---|---|
| 基线产品 | `anypi_luna` | `cp -r` 来源 |
| 新产品名 | `anypi_luna_dev` | `project/<name>/` |
| **BOARD 名** | `EVB_MBD_KM01A` | = `CONFIG_BOARD` = overlay 目录名；优先用 `system/AnyCloud39AV100_V2` 原生内容 |
| **UI 形态** | `IOTCamera`（无显示屏） / `AI_GLASSES`（移植中） / `TABLET`（平板级） / `WEARABLE`（穿戴级） | 见下「UI 与单应用选型」|
| 能力清单 | RTSP、nebula（含对讲）、AI、OSD、**单 App=`ipc_cam`** | 单 App 放哪由形态决定 |

### UI 与单应用选型（先定形态再写 App）

| 产品目录 | `CONFIG_APP_MAIN_TARGET`（以各产品 `config.mk` 为准）| packages 开关 | 单应用放哪（推荐）| 如何挂上 |
|---|---|---|---|---|
| `anypi_luna_dev`（IOTCamera） | `luna_dev_main` | `PACKAGES_IPCAMERA=y`；wearable/tablet=`n` | **`application/packages/ipcamera/<id>/` SO**| `settings.json`：`launch_mode=ipcamera`（或 `headless`）+ **唯一** `type:so`；`module_start` **阻塞保活** |
| `anypi_luna`（`WEARABLE` 形态） | `luna_main` | `PACKAGES_WEARABLE=y` | `application/packages/wearable/<id>/` | `installed_apps`（内嵌于 settings.json）+ launcher |
| `anypi_nova`（`TABLET` 形态） | `ipc_main` | `PACKAGES_TABLET=y` | `application/packages/tablet/<id>/` | 同上 |
| `anypi_nova_dev` | `nova_main` | `PACKAGES_TABLET=y` | 同上 | 同上 |

主程序名由 **`project/<产品>/config.mk` 的 `CONFIG_APP_MAIN_TARGET`** 传入；未设置时 `application/app_main/Makefile` 按目录名推导。

**IOTCamera 形态不要进 `packages/wearable`**：桌面 App 依赖 LVGL；无 GUI 时须靠 SO 的 `module_start` 保活。契约见 `app_config.c`；业务可对照 `packages/wearable/dashcam/`（去掉 LVGL）。

**`anypi_luna_dev`（无 UI + 单应用）推荐步骤**：① 基线 luna → 产品 + BOARD=`EVB_MBD_KM01A` + **ipcamera**；② `luna_dev_main`、关 wearable/tablet；③ **新建** `application/packages/ipcamera/ipc_cam/` SO（`module_init`/`module_start` 保活；业务对照 `dashcam` 去 GUI）；④ `app_runtime` 只挂这一 SO；⑤ **不必** `installed_apps`；⑥ 启动脚本拉 `luna_dev_main` → 精简 config → 上板。详见 **填写表 C**。

**需求 → skill（速查）**

| 需求 | 主 skill |
|---|---|
| 基线 / SoC / **BOARD** / overlay | `/celestia-project` |
| **指定 UI** / 主程序 / 单应用落点 | `/celestia-project` + `/celestia-application` |
| IOTCamera 单 SO（RTSP/nebula/AI）| `/celestia-application`（IOTCamera 契约）|
| Board 处理 / config 精简 | `/celestia-project` |
| HAL / fw / 网关 | `/celestia-hardware` · `/celestia-framework` |
| 编译 / 上板 / 提交 | `/celestia-scripts` · `/celestia-debug` · `/git-gitee` |

### 可提问场景（复制即用）

| 场景 | 怎么问 AI |
|---|---|
| **IOTCamera + 单应用** | `anypi_luna_dev：无 UI、单应用 ipc_cam。luna_dev_main + PACKAGES_IPCAMERA；App 放 packages/ipcamera/ipc_cam；app_runtime 只挂一 SO；按表 C` |
| 带显示单应用（wearable）| `WEARABLE` 形态 + 只装 dashcam：luna_main+PACKAGES_WEARABLE；仿 packages/wearable/dashcam；installed_apps 只一项` |
| 指定 BOARD 开新产品 | `基于 anypi_luna 建 anypi_luna_dev，CONFIG_BOARD=…，UI=ipcamera，按表 A+C` |
| 复用旧板 | `基于 anypi_nova 建新产品，BOARD 不变，只改形态与 App` |
| 新 BOARD 全套 | `按 Board 处理清单改 DTS(board+patch)/rootfs/启动脚本/板级 config/burntool` |
| 精简 config.mk | `对照 app_runtime，未用 CONFIG_*→n，重编上板` |
| 开 RTSP+nebula | `只开所需协议开关，fw_video 对齐出流` |
| 启动拉错主进程 | `启动脚本必须拉 CONFIG_APP_MAIN_TARGET（如 luna_dev_main）` |

### 对话样例

**样例 1 — anypi_luna_dev 无 UI 单应用（推荐直接用）**

> **你**：`anypi_luna_dev`：**没有 UI**，只要 **一个应用**做采集+RTSP+nebula+AI。
> 选型确认：ipcamera；App 放 `application/packages/ipcamera/ipc_cam`（导出 module_init/module_start；业务仿 dashcam 去 GUI）。
> ① 新建目录+Makefile 产出 .so；② `module_init`/`module_start`，start 内阻塞保活；③ `settings.json` **只**挂这一 SO（运行期 `so_path`）；④ config.mk 关 GUI、开 RTSP/NEBULA/CAMERA/AI/OSD；⑤ 启动脚本拉 `luna_dev_main`；⑥ `-a -p` 上板：主进程不退出、出流 PASS。
> 不要创建桌面 `installed_apps`。按表 C 回填。
>
> **AI 应做**：表 A（形态）+ 表 C；禁止做成 wearable GUI App。

**样例 2 — 指定 BOARD + 声明 UI**

> **你**：基线 `anypi_luna` → `anypi_luna_dev`，BOARD=`PDB_MBD_KM02_ACTCAM`，**UI=headless**，单应用。先 Board 清单，再表 A+C，精简 config，`-k -f -a -p` 验出流。

**样例 3 — 带显示但只要一个桌面 App**

> **你**：改为 **`WEARABLE` 形态**、只留 dashcam：`luna_main`+`PACKAGES_WEARABLE=y`；仿 `packages/wearable/dashcam`；`installed_apps` 一项；`launch_mode=ui`。不要用 services/ headless。

**样例 4 — 复用 BOARD 只改形态**

> **你**：基于 `anypi_nova` 建 `anypi_ipc_lite2`，BOARD 不变，保留 `ipc_main`；关云与 UVC，保留 RTSP；按需裁 GUI/模块；输出 y/n 清单后全量编译上板。

**样例 5 — 精简 config**

> **你**：出流已通。对照 `app_runtime` 把未用 `CONFIG_APP_*/FW_*/HW_*` 设 `n`，diff 后重编上板，勿改其他产品。

### 填写表 A — 新产品（须指定 BOARD + UI）— anypi_luna_dev 已回填

| # | 需求项 | 产物路径（填写）| 关键参数 / 取值（填写）| 主 skill | 状态 | 验证 |
|---|---|---|---|---|---|---|
| 1 | 声明输入 | — | 基线=`anypi_luna` / 产品=`anypi_luna_dev` / **BOARD=`EVB_MBD_KM01A`** / **UI=ipcamera** / 单App=`ipc_cam` / RTSP+nebula+AI+OSD | `/celestia-project` | ☑ | |
| 2 | 项目骨架 + Board | `project/anypi_luna_dev/`、`…/board/EVB_MBD_KM01A/` | `cp -r` 自 luna；board 自 KM01W 改名并去 LCD 启动 | `/celestia-project` | ☑ | |
| 3 | 形态选型 | `project/anypi_luna_dev/config.mk` | `MAIN_TARGET=luna_dev_main`；`PACKAGES_IPCAMERA=y`；wearable/tablet=`n` | `/celestia-project` | ☑ | |
| 4 | **单应用落地** | `IOTCamera` → **表 C** | `packages/ipcamera/ipc_cam` | `/celestia-application` | ☑ | |
| 5 | fw / HAL / 网关 | board `fw_*`/`hal_*` + gateway | RTSP/nebula/AI/OSD=`y`；GUI/LVGL=`n` | hw / fw | ☑ | |
| 6 | config 精简 + 编译上板 | `out/anypi_luna_dev/` | `-a -p --product-name anypi_luna_dev`；主进程不退出、出流 PASS | project / scripts / debug | ☐ | 待上板 |

### 填写表 B — 新增 Board（产品名不变）

| # | 步骤 | 产物路径（填写）| 关键参数 / 取值（填写）| 主 skill | 状态 | 验证 |
|---|---|---|---|---|---|---|
| 1 | 声明并复制 Board | `…/board/<NewBoard>/` | `CONFIG_BOARD=<NewBoard>` | `/celestia-project` | ☐ | |
| 2 | Board 处理清单 | 同上 | DTS(board+patch)、rootfs、启动脚本、板级 config、burntool | `/celestia-project` | ☐ | |
| 3 | 选板 + 编译上板 | `config.mk`、`out/<p>/` | 主进程与脚本一致；`-k -p` | project / scripts / debug | ☐ | |

### 填写表 C — IOTCamera 单应用（anypi_luna_dev 典型）— 已回填

> 前提：表 A 已选 **UI=ipcamera**。App **不**放 `packages/wearable|tablet`。

| # | 步骤 | 产物路径（填写）| 关键参数 / 取值（填写）| 主 skill | 状态 | 验证 |
|---|---|---|---|---|---|---|
| 1 | 确认选型 | — | `luna_dev_main`；`PACKAGES_IPCAMERA=y`；wearable/tablet=`n` | `/celestia-application` | ☑ | |
| 2 | 新建服务 SO | `application/packages/ipcamera/ipc_cam/` | Makefile → `libipc_cam_app.so` | `/celestia-application` | ☑ | |
| 3 | 导出 + 保活 | `ipc_cam_app.c` | `module_init`/`module_start`；start 内 `sleep` 阻塞 | `/celestia-application` | ☑ | |
| 4 | 编进产品 | `packages/ipcamera/Makefile`、`config.mk` | `CONFIG_APP_PACKAGES_IPCAMERA=y` | application · project | ☑ | |
| 5 | `settings.json` | 板级 overlay `board/EVB_MBD_KM01A/rootfs/utils/etc/config/` | `launch_mode=ipcamera`；仅 `ipc_cam`；`so_path=/usr/lib/libipc_cam_app.so` | `/celestia-application` | ☑ | |
| 6 | 业务接线 | 复用 dashcam record/rtsp/storage + nebula/AI | RTSP+nebula+AI+OSD；经 Framework | fw + application | ☑ | |
| 7 | 启动脚本 | `board/EVB_MBD_KM01A/.../app.sh` | 拉起 **`luna_dev_main`**；无 LCD/`installed_apps` | `/celestia-project` | ☑ | |
| 8 | 精简 + 上板 | `out/anypi_luna_dev/` | 主进程不退出；出流≥20s | scripts / debug | ☐ | 待 `-a -p` 上板 |

排障见 [celestia-project](../skills/celestia-project.md)；IOTCamera SO 见 [celestia-application](../skills/celestia-application.md#iotcamera-单应用推荐方案)。

---
## 3. 应用开发者

> 在 `application/` 加 App / GUI / 云适配，不碰 HAL/BSP。细节：[celestia-application](../skills/celestia-application.md)、[application guide](../guide/application.md)。
> **优先仿现有 App**，不要从零猜接口。

### 现有能力 → 仿谁（例子库）

| 你要做的应用 | 仿照 / 复用 | 关键能力（Framework）|
|---|---|---|
| 最小 App 壳（时钟级）| Nova `packages/tablet/clock/` | 仅 LVGL + 注册表 |
| 预览 + 拍照/短录 | wearable/tablet `camera/`；共用 `packages/common/gui_camera.c` | `multimedia->camera` / player |
| 循环录像 + 子码流 RTSP | wearable `dashcam/`（含私有 `app_dashcam_config.json`）| recorder + `stream_reg_client` + RTSP |
| 设置页：WiFi / 存储 / 视频 | Nova `setting/` | network + storage + camera ctrl |
| 扫码配网 / 扫码业务 | Nova `setting` → `tablet_setting_qr_scan.c` | 码流角色 `qr` + `app_stream_usage` |
| AI 开关 / 场景 | Nova `setting` → `tablet_setting_ai.c` | `set_ai_enabled` / `fw_ai_config` |
| 音频算法 UI（VAD/AEC/NS）| Nova `audio_algo/` | `multimedia->audio` |
| IPC / 通话类 UI | Nova `ipc_camera/` | network + notification |
| 文件浏览 | Nova `file_manager/` | storage |
| 火山云 UI + 推流/对讲 | Nova `volc_engine/` + `cloud/volc_engine/` | cloud SO + multimedia/network |
| 阿里云 Bailian 类 | `cloud/aliyun/`（静态库）| `aliyun_app_init/deinit` |
| IOTCamera 单 SO（自建）| 新建 `packages/ipcamera/<id>/`（`module_*`）；业务对照 `dashcam` | 保活 + gateway |
| UVC 出流 | `services/uvc/`（**static**）| UVC gadget |
| 天气卡片 | `services/weather/`（**static**）| network 回调 |

码流原则：能力集 `channels/streams` 不动；应用只改 `app_stream_usage`（[framework §2.5.1](../guide/framework.md)）。私有分辨率用 `load_config(私有json)`，退出还原默认。

**需求 → skill（速查）**

| 需求 | 主 skill |
|---|---|
| 新增 tablet / wearable App | `/celestia-application` |
| 新增 cloud 适配 | `/celestia-application` |
| 改码流用量 / 私有 video 配置 | `/celestia-application` + `/celestia-framework` |
| 接 RTSP / nebula / AI / 音频 | `/celestia-framework`（App 侧调用）|
| 启动 / 桌面 / headless | `/celestia-application`（+ `/celestia-project`）|
| 编译 / 上板 | `/celestia-scripts` · `/celestia-debug` |

### 可提问场景（复制即用）

| 场景 | 怎么问 AI |
|---|---|
| 仿 clock 最小壳 | `按 /celestia-application 仿 tablet/clock 新建 wearable hello App，只注册桌面，不上相机` |
| 仿 camera 预览拍照 | `基于 luna/camera 加连拍相册页，复用 packages/common，只动 application/` |
| 仿 dashcam 循环录+RTSP | `仿 luna/dashcam 新建 trip_record：60s 分段 + 子码流 RTSP，走填写表` |
| 私有 video 配置 | `进入 App 时 load_config(app_xxx_config.json)，退出还原 fw_video_config.json，参考 dashcam` |
| 加 qr 角色扫码 | `给 camera_app 加 qr，只改 app_stream_usage，仿 tablet_setting_qr_scan` |
| 设置里加 AI 页 | `仿 tablet_setting_ai，在 wearable 设置加 AI 开关/场景，调 set_ai_enabled` |
| 音频算法 App | `仿 nova/audio_algo 做简单 AEC 开关页，只走 multimedia->audio` |
| 文件管理扩展 | `在 file_manager 增加按日期筛 TF 卡录像文件` |
| 接火山推流+对讲 | `按 cloud 清单接 volc_engine，参考 cloud/volc_engine + nova/volc_engine UI` |
| 接阿里云 | `仿 cloud/aliyun 静态库模式接 Bailian，主进程链接，不走 module_exit` |
| IOTCamera 出流 demo | `按 IOTCamera 契约新建 services SO：无 GUI、单 SO、RTSP+nebula+AI、module_start 保活` |
| UVC 服务 | `启用 services/uvc，说明与 WiFi 互斥，改 config 与启动接线` |
| 天气服务接入桌面 | `launcher 调 app_weather_update_request 显示天气卡片` |
| Launcher 默认启动 | `wearable launcher 开机自启 dashcam，改 autostart / app_runtime` |
| IOTCamera 裁 GUI | `关 CONFIG_APP_PACKAGES_*，luna_dev_main + 单 SO 保活` |
| App 不上桌面 / dlopen | `按 application 常见问题查 installed_apps 与 entry keep-alive` |

### 对话样例

**样例 1 — 仿 dashcam 新录像 App（推荐起步）**

> **你**：按 /celestia-application，在 anypi_luna **仿 `packages/wearable/dashcam/`** 新增 `trip_record`：循环录像 60s 分段 + 子码流 RTSP。可复用其 `load_config` 私有 json 模式。只动 `application/` 与必要的 `app_stream_usage`；按填写表回填；`-a --product-name anypi_luna`；/celestia-debug 验分段文件增长与 RTSP≥20s。
>
> **AI 应做**：对照 dashcam 目录结构建 App → 注册 → 码流用量 → 编译上板。

**样例 2 — 扫码（只改用量，不改能力集）**

> **你**：camera_app 要扫码。**禁止**改 `channels/streams`。仿 `tablet_setting_qr_scan`，只在 `fw_video_config.json` 的 `app_stream_usage` 为该 App 加 `qr`；`-a` 上板确认扫码通路 PASS。

**样例 3 — 设置页拼装（WiFi + AI）**

> **你**：wearable 要一个简易设置 App：WiFi 列表 + AI 开关。分别仿 tablet `setting` 的 WiFi 与 `tablet_setting_ai.c`；经 `app_main_get_network/multimedia`，禁止 include HAL。

**样例 4 — IOTCamera 出流（自建 SO）**

> **你**：新产品 `IOTCamera` 形态。按 IOTCamera 契约新建 `packages/ipcamera` SO：单 SO、`module_start` 阻塞保活、main H.265 + AI + RTSP + nebula。`PACKAGES_IPCAMERA=y`、wearable/tablet=`n`；上板验出流与 AI log。

**样例 5 — 云平台（火山）**

> **你**：给 anypi_nova 接火山推流 + 语音对讲。读 cloud 适配清单，仿 `cloud/volc_engine/`；UI 可参考 `packages/tablet/volc_engine/`；只动 application/cloud 与必要 runtime 注册；`-a` 上板看推流连通。

**样例 6 — 最小壳再扩展**

> **你**：先仿 `nova/clock` 在 Luna 建 `hello` App（仅桌面图标可进空白页），注册通过后再加 `gui_camera` 预览；分两步提交，每步 `-a` 上板。

**样例 7 — 音频算法页**

> **你**：仿 `nova/audio_algo`，做「AEC 开/关」小页，只调 `multimedia->audio` ops；不要改 HAL。上板听筒回路验证。

**样例 8 — 接口核对（不下钻）**

> **你**：走 /celestia-framework，说明 `stream_reg_client` / `load_config` / `set_ai_enabled` 的返回值与线程约束；我只在 App 侧调用。

### 填写表 — 新增 wearable App（以 dashcam 为基线）

| # | 步骤 | 产物路径（填写）| 关键参数 / 取值（填写）| 主 skill | 状态 | 验证 |
|---|---|---|---|---|---|---|
| 1 | 选定仿照基线 | `packages/wearable/dashcam/` 或 `camera/` / `clock` 模式 | 基线 App 名=… | `/celestia-application` | ☐ | |
| 2 | App 目录 + `app.json` + 入口 | `application/packages/wearable/<app>/` | `app_id`/`entry_so`/`entry_symbol`；visibility+keep-alive | `/celestia-application` | ☐ | |
| 3 | Makefile + 特性宏 + 注册表 | `packages/wearable/Makefile`、`config.mk`、`installed_apps.json` | `ENABLED_APP_*`、`CONFIG_APP_WEARABLE_<ID>`、`install_path` | `/celestia-application` · `/celestia-project` | ☐ | |
| 4 | 码流用量（及可选私有 json）| `app_stream_usage`；可选 `app_<id>_config.json` | `roles_used`；`load_config` 进入/退出还原 | `/celestia-application` · `/celestia-framework` | ☐ | |
| 5 | 业务逻辑 | `wearable_<id>_*.c` | 录像/RTSP/AI/扫码等（对照例子库）| 同上 | ☐ | |
| 6 | 编译 + 上板 + 提交 | `out/<p>/` | `-a`；功能 PASS；一仓一 squash | `/celestia-scripts` · `/celestia-debug` · `/git-gitee` | ☐ | |

### 填写表 — 新增 tablet App / 云 / IOTCamera（可选）

| # | 类型 | 仿照 | 关键产物（填写）| 验证 |
|---|---|---|---|---|
| N1 | Nova 桌面 App | `clock`→壳 / `camera`→预览 / `setting`→扫码AI | `packages/tablet/<id>/` + `CONFIG_APP_TABLET_*` + installed_apps | 图标可进、功能 PASS |
| N2 | Cloud 适配 | `volc_engine` 或 `aliyun` | `cloud/<id>/` + runtime/链接方式 | 推流/对讲或 init 成功 |
| N3 | IOTCamera 单 SO | 新建 `packages/ipcamera/<id>/` | 见 [§2 表 C](#填写表-c--ipcamera-单应用anypi_luna_dev-典型)；勿放 `packages/tablet` / `packages/wearable` | 主进程不退出、出流≥20s |

排障见 [celestia-application 常见问题](../skills/celestia-application.md#常见问题)。

---

## 4. 新芯片移植开发者

> **输入**：原厂提供的**芯片 SDK**（工具链 + kernel/驱动/头文件，常解压到 `system/<NewSoC>/<SDK_Ver>/`）。
> **核心工作**：按产品能力清单，对**每个需要的 HAL 模块**在 `hardware/.../hal/<NewSoC>/` 增加 SoC 实现；板级走 Project/BSP；`system/core` 仅在扩模块号等少数场景改。
> 细节：[celestia-hardware](../skills/celestia-hardware.md)（含「新增 SoC」）、[celestia-system-core](../skills/celestia-system-core.md)、[bsp-and-products §9](bsp-and-products.md)、[download 放置 SDK](download.md#放置芯片-sdk)。

**开新芯片时先声明**（缺一不可）：

| 输入 | 示例 | 说明 |
|---|---|---|
| 芯片 SDK 路径 / 版本 | `system/<NewSoC>/V1.00/` | 工具链、`include/`、`.ko`、内核 |
| `CONFIG_SOC_PLATFORM` | `AnyCloud39XXXX` | = `hal/<SoC>/` 目录名字符串 |
| `CONFIG_SOC_SDK_VERSION` | `V1.00` | 与 SDK 目录一致 |
| 能力清单 | camera / audio / display / wifi / ir… | 决定开哪些 `CONFIG_HW_*`、要写哪些 HAL |
| 基线参考 SoC | `AnyCloud39AV200` 或 `AnyCloud39AV100_V2` | 拷贝同名 `hal_<module>.c` 骨架再改 SDK 调用 |
| 新产品 / BOARD | `anypi_xxx` / `PDB_…` | 组装见 [§2](#2-新产品移植开发者) |

### 分层：谁改、谁不改

| 层 | 新芯片要做什么 | 主 skill |
|---|---|---|
| **芯片 SDK / BSP** | 放入 `system/<NewSoC>/<SDK>/`；板级 DTS、insmod、rootfs overlay | `/celestia-project` |
| **Hardware HAL** | **每个启用模块**补 `hal/<NewSoC>/`（或确认扁平模块可用）+ `soc_platform` | `/celestia-hardware` |
| **system/core** | 通常**不按 SoC 分叉**；仅当新增 HAL 模块时追加 `HARDWARE_MODULE_*` / 错误码 | `/celestia-system-core` |
| Framework / Application / `hardware/include` | **禁止**为单一 SoC 改公开契约；能力扩展走私有头 | — |

### 逐模块 HAL 清单（有 `hal/<SoC>/` 的必须新建目录）

对照 [celestia-hardware 目录地图](../skills/celestia-hardware.md#目录地图)。能力清单勾选后，**逐行实现**（参考基线 `hal/<RefSoC>/`，对接本芯片 SDK API）：

| 模块 | 路径 | `CONFIG_HW_*`（示例）| 状态 |
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

**扁平模块**（display、network/*、input、多数 driver）：一般**不建** `hal/<NewSoC>/`；用 SDK 驱动 + 板级配置验证即可。缺能力再评估是否升格为 SoC 分目录（见 hardware skill）。

每模块交付物：`HAL_MODULE_INFO_SYM` + ops + `hw_<m>_module_init()` → 由 `hardware.c` 在对应 `CONFIG_HW_*` 下调用；Makefile 用 `hal/$(CONFIG_SOC_PLATFORM)` 收源。

### 需求 → skill（速查）

| 需求 | 主 skill |
|---|---|
| 放置 / 核对芯片 SDK、工具链 | `/celestia-project` + [download](download.md) |
| **逐模块** HAL + soc_platform + IPU 弱桩 | `/celestia-hardware` |
| 新 `HARDWARE_MODULE_*` / 错误码 | `/celestia-system-core` |
| config.mk / board / DTS / insmod / 新产品 | `/celestia-project` |
| **AOS/RTOS 芯片**（SDK 只产 `.a`，Linux 产品移植 Aos，如 mira/Sky39AV130） | `/celestia-aos-port`（[celestia-aos-port](../skills/celestia-aos-port.md)） |
| 编译 / bring-up L1→L5 | `/celestia-scripts` · `/celestia-debug` |
| 同步 SoC 矩阵文档 | `/celestia-contributing` |

### 可提问场景（复制即用）

| 场景 | 怎么问 AI |
|---|---|
| **SDK 起新芯片全流程** | `已有 SDK 在 system/<NewSoC>/<Ver>/。按 roles §4：声明能力清单 → 逐模块补 hal/<NewSoC>/ → system/core 仅必要时加模块号 → project/board → L1→L5` |
| 按清单批量补 HAL | `为 <NewSoC> 按 hardware 有 hal/ 的模块表逐项实现，参考 <RefSoC>，公开头不改，填填写表` |
| 只做 camera 先行 | `先只开 CONFIG_HW_CAMERA + soc_platform，实现 camera HAL，其余 HW=n，再扩展` |
| 补 audio/osd/ir | `为 <NewSoC> 补 audio/osd/ir HAL，只在有 hal/<SoC>/ 的路径建目录` |
| soc_platform / IPU | `实现 soc_platform 时钟/pinmux/IRQ；缺 IPU 用弱桩` |
| system/core 要不要改 | `对照 /celestia-system-core：无新模块则不动；有新 HAL 模块再追加 HARDWARE_MODULE_*` |
| 公开头不动 | `扩展能力只走私有头 / SDK 头，禁止改 hardware/include/*.h` |
| 新板 DTS / insmod | `按 bsp §9 加板型与 ko 顺序，vendor 树只读优先` |
| `/dev/video0` 缺失 | `先 D 类查 insmod/dmesg，再判 HAL` |
| bring-up 自检 | `L1 /dev → L2 hardware/tests → L3 fw preview → L4 App → L5 云，逐级 PASS` |

### 对话样例

**样例 1 — SDK + 逐模块 HAL（推荐起步）**

> **你**：芯片 SDK 已放到 `system/<NewSoC>/V1.00/`。能力：camera + audio + display + wifi。按 /celestia-hardware：**对每个有 `hal/<SoC>/` 且能力启用的模块**建 `hal/<NewSoC>/`（先 camera+audio+soc_platform）；扁平 display/wifi 不建目录、板级验证；公开头不改；`system/core` 无新模块号则不动。config.mk 设 `CONFIG_SOC_PLATFORM`/`CONFIG_SOC_SDK_VERSION`/对应 `CONFIG_HW_*`。填 §4 填写表；`-s -h -f -a -k -p` 按需；/celestia-debug 跑 L1→L5。
>
> **AI 应做**：读 hardware（新增 SoC）+ system-core（边界）+ project → 按模块清单实现 → 编译 → bring-up → 提示同步 SoC 矩阵。

**样例 2 — 只改板级、不动 HAL 契约**

> **你**：同 SoC 新板换 sensor。只改 `project/<p>/.../board/<Board>/` 的 DTS 与 rootfs；vendor 树只读；`-k -p` 上板看 `/dev/video0` 与出帧。

**样例 3 — HAL 排障**

> **你**：anypi_luna 预览 CHN0 select timeout。先 /celestia 归层，再按 hardware 常见问题查 `num_slices` / frame_depth；证据不足不要改 Framework。

### 填写表 — 新芯片（SDK → 逐模块 HAL → 上板）

| # | 步骤 | 产物路径（填写）| 关键参数 / 取值（填写）| 主 skill | 状态 | 验证 |
|---|---|---|---|---|---|---|
| 0 | 放置芯片 SDK | `system/<NewSoC>/<Ver>/` | 工具链、`include/`、`.ko`、内核 | `/celestia-project` | ☐ | SDK 头可被 `-h` 找到 |
| 1 | config.mk 选 SoC / 开模块 | `project/<p>/config.mk` | `CONFIG_SOC_PLATFORM`、`CONFIG_SOC_SDK_VERSION`、所需 `CONFIG_HW_*=y` | `/celestia-project` | ☐ | `config.h` 生成正确 |
| 2 | **逐模块**补 `hal/<NewSoC>/` | 见上表各路径 | 每模块：`HAL_MODULE_INFO_SYM` + `hw_<m>_module_init`；对接 SDK | `/celestia-hardware` | ☐ | 模块 `-h` 通过 |
| 3 | soc_platform（+ IPU 弱桩）| `hardware/soc_platform/hal/<NewSoC>/` | 时钟/pinmux/IRQ | `/celestia-hardware` | ☐ | bring-up 无 platform 失败 |
| 4 | system/core（仅必要时）| `system/core/.../system_error.h` | 新 `HARDWARE_MODULE_<X>` / 错误码；无新模块则跳过 | `/celestia-system-core` | ☐ | `-s`；上层仍兼容 |
| 5 | board DTS + insmod + rootfs | `project/.../board/<Board>/` | sensor/DMA；ko 顺序；启动脚本 | `/celestia-project` | ☐ | `/dev/*` 存在 |
| 6 | 编译 + L1→L5 + 文档 + 提交 | `out/<p>/` | `-h -f -a -k -p`；`/dev`→tests→preview→app→cloud | `/celestia-scripts` · `/celestia-debug` · `/celestia-contributing` · `/git-gitee` | ☐ | 各级 PASS |

排障见 [celestia-hardware 常见问题](../skills/celestia-hardware.md#常见问题)；Bug 归类见 [§5](#5-调试与维护)。

---

## 5. 调试与维护

> 上板排障、崩溃、内存/DMA、构建 stale、多仓提交。
> **Bug 分类与解法总册**：[celestia-debug](../skills/celestia-debug.md)（A–G 类）；路由 [celestia](../skills/celestia.md)；提交 [git-gitee](../skills/git-gitee.md)。

**提问要点**：产品名 + 复现步骤 + **关键 log 原文**（含前后几行）→ 先归 A–G 类 → 再改 → 闭环复测。

### 问题 / log → skill（速查）

| 你看到的 log / 现象 | Bug 类 | 先读 skill |
|---|---|---|
| `signal=11` / `fault_addr=` / backtrace | **A 崩溃** | `/celestia-debug` → 按栈 `/celestia` 归层 |
| `invalid inbuf` / `QBUF fail` / `select timeout` / 不出帧 | **B 预览编码** | `/celestia-debug` + `/celestia-hardware` 或 `/celestia-framework` |
| ion/dma alloc fail、OOM、花屏、MemFree 极低 | **C 内存 DMA** | `/celestia-debug`（决策树）；DTS → `/celestia-project` |
| `/dev/*` 无、insmod 失败、拉错主进程、overlay 无效 | **D 启动驱动** | `/celestia-project`（+ hardware）|
| `dlopen` 失败、不上桌面、图标空白、IOTCamera 形态秒退 | **E App/SO** | `/celestia-application` |
| 改源不生效、`module: -1`、config.h 不对、链接缺符 | **F 构建 stale** | `/celestia-scripts`（+ hardware/project）|
| RTSP/nebula 无流、云连不上、`stream_reg_client` 失败 | **G 出流网络** | `/celestia-framework`（+ application）|
| 不知哪一层 | — | 先 `/celestia`，再回 `/celestia-debug` 总表 |

层内补充：`application` / `hardware` / `framework` / `project` / `scripts` 各自「常见问题」节。

### 可提问场景（复制即用）

| 场景 | 怎么问 AI |
|---|---|
| 贴 log 归类 | `产品 anypi_luna，log 如下：<粘贴>。按 /celestia-debug 归 A–G 类，填排障表，给解法再改` |
| 预览丢帧 | `反复 invalid inbuf cap=524288，走 B 类已知模式，改完出帧≥20s` |
| signal=11 | `开 AI 后 signal=11 fault_addr=0x90，对照 A 类竞态修法，抓栈归层` |
| 拍照后预览死 | `拍照后预览卡死+QBUF fail，B 类 teardown/live reload` |
| OOM / 花屏 | `luna 录像 OOM 或花屏，走 C 类 DMA vs MemTotal 决策树给调档` |
| 无 /dev/video0 | `D 类：insmod/dmesg + 启动脚本，再决定是否下钻 HAL` |
| App dlopen 失败 | `E 类：visibility + keep-alive + installed_apps path` |
| 改了不生效 | `F 类：怀疑 stale out/，-c 再编并核 product-name` |
| RTSP 无流 | `G 类：CONFIG_APP_RTSP + stream_reg_client + 先保证编码 B 类 PASS` |
| 多仓提交 | `按 /git-gitee 一仓一 squash 到 release，列 commit 草案等确认` |

### 对话样例

**样例 1 — 贴 log 归类（推荐起步）**

> **你**：产品 `anypi_luna`。串口：
> `invalid inbuf cap=524288`（反复）。预览丢帧。
> 请按 /celestia-debug：**归 B 类** → 填排障表 → 套已知解法（codec_mode 后再算 payload）→ `-f/-h` 重编上板，连续出帧≥20s 为 PASS。
>
> **AI 应做**：命中 B 类表 → 最小修复 → 闭环 → 回填表。

**样例 2 — 崩溃带栈**

> **你**：`ipc_camera` 开 AI 后 `signal=11 fault_addr=0x90`，backtrace 含 `camera_manager`。按 A 类竞态模式修；不要先改 BSP。上板复现 10 次无崩为 PASS。

**样例 3 — 内存分流**

> **你**：录像一段时间后花屏，dmesg 有 ion alloc fail，`MemTotal` 仍约 25MB。按 C 类判 **DMA 池不足**，给 20→34MB 的 board+patch 双改方案与 `-k -p` 验证步骤；同时评估是否先降 `max_buffers`。

**样例 4 — 启动无设备**

> **你**：上板无 `/dev/video0`。先 D 类查 `app.sh`/`insmod`/`dmesg`，确认 ko 后再下钻 `/celestia-hardware`；禁止一上来改 Framework。

**样例 5 — App 不上桌面**

> **你**：新 wearable App 编译过但不出现在桌面。按 E 类查 `installed_apps.json`、`install_path`、entry 是否被 gc；对照 application 常见问题。

**样例 6 — stale 构建**

> **你**：改了 `hal_camera.c` 板上仍报 `Failed to get video hardware module: -1`。按 F 类：`-c -h`，并检查 `hardware.o` 对 config.h 依赖。

**样例 7 — 出流联调**

> **你**：RTSP 客户端连上无画面。先确认 B 类预览/编码 PASS，再 G 类查 `CONFIG_APP_RTSP` 与 `stream_reg_client(main)`；贴 gateway 相关 log。

**样例 8 — 提交**

> **你**：修了 framework + hardware。按 /git-gitee：release、一仓一 squash、可执行位；给出两仓 message 草案，确认后再提交。

### 填写表 — 排障工作表

| # | 现象（填写）| Bug 类 (A–G) | 证据 / log（填写）| 根因 / 核对（填写）| 修复（填写）| 主 skill | 验证（填写）|
|---|---|---|---|---|---|---|---|
| 1 | 预览丢帧 | B | `invalid inbuf cap=524288` | payload 在 codec_mode 前算 | 先解析 codec_mode | `/celestia-debug`+fw/hw | PASS：≥20s 出帧 |
| 2 | | | | | | | |
| 3 | | | | | | | |

> 分类细则与解法全文：[celestia-debug Bug 分类](../skills/celestia-debug.md#bug-分类总表贴-log-先查这里)。禁止只编不验。