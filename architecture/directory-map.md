<p align="right"><a href="directory-map.en.md">English</a> | <b>中文</b></p>

# 目录地图

> 回答两个问题：**每个目录负责什么**、**新文件该放哪**。
> 术语以 [术语表](../glossary.md) 为准；分层契约与运行期模型见 [架构总览](overview.md)。

## 1. 一级目录职责

```text
celestia/                        # 源码根
├── build.sh -> scripts/build.sh # 唯一构建入口
├── application/                 # ① 应用层
├── framework/                   # ② 框架层
├── hardware/                    # ③ 硬件适配层
├── system/core/                 # ④ 系统基础库
├── project/<product>/           # ⑤ 产品工程（配置 + overlay）
├── scripts/                     # 构建脚本
├── third_party/                 # 第三方组件
└── docs/                        # 文档（本目录）
```

| 目录 | 层 | 职责 | 不该出现的东西 |
|:-----|:---|:-----|:---------------|
| `application/` | ① | 主程序装配、GUI 应用包、系统服务、云接入 | 直接操作 `/dev/*`、include HAL 私有头 |
| `framework/` | ② | 多媒体、网络、设备、通知、存储、运行时、网关协议 | 产品分支、`#ifdef CONFIG_SOC_*`、芯片私有头 |
| `hardware/` | ③ | 各 SoC 的外设与媒体能力实现，向上统一 ops | 业务逻辑、GUI、对上层符号的引用 |
| `system/core/` | ④ | 日志、配置、线程、定时器、错误码、内存、加密、容器等基础库 | 任何对 ①②③ 的依赖 |
| `project/` | ⑤ + 配置 | 每产品一份 `config.mk` 与 BSP overlay、rootfs 配置、打包脚本 | 公用业务代码 |
| `scripts/` | 构建 | `build.sh` 主体、打包、工具链注入 | 运行时逻辑 |
| `third_party/` | 依赖 | 第三方库源码与构建规则 | 业务代码、产品开关 |
| `docs/` | 文档 | 本套文档 | 代码 |

## 2. 关键子目录

### 2.1 `application/`

| 路径 | 说明 |
|:-----|:-----|
| `app_main/` | 主程序：启动、配置加载、服务注册、SO 模块加载、形态分岔 |
| `packages/tablet/` | **平板级（`TABLET`）**桌面与内置 App（`ipc_camera` `camera` `setting` `clock` `file_manager` `audio_algo` `volc_engine`） |
| `packages/wearable/` | **穿戴级（`WEARABLE`）**界面（launcher、camera、dashcam、power_keyd） |
| `packages/ipcamera/` | **`IOTCamera`** 形态业务 SO（`ipc_cam`），导出 `module_init` / `module_start` |
| `packages/common/` | tablet / wearable 共用 GUI 组件 |
| `services/` | 与界面无关的常驻服务（`weather`、`uvc`），以 static 方式链入 |
| `cloud/` | 云平台适配（`aliyun`、`volc_engine`），以 SO 热插拔 |
| `tests/` | 应用层自测 |

### 2.2 `framework/`

| 路径 | 说明 |
|:-----|:-----|
| `services/multimedia/` | 预览 / 拍照 / 录像 / 播放 / 录音 / 推流拉流；下设 `camera`（`device` `core` `capture` `af` `ai` `osd`）、`media`（`record` `play`）、`av_codec`、`audio` |
| `services/network/` | Wi-Fi、以太网、蜂窝；含凭据加解密 |
| `services/device/` | 背光、云台、夜视、LED、电源 |
| `services/notification/` | 按键、报警、输入子系统 |
| `services/storage/` | Flash / TF 卡、空间与挂载 |
| `gateway/rtsp_protocol/` | RTSP 服务端 |
| `gateway/nebula_protocol/` | Nebula 信令与对讲 |
| `runtime/` | bootstrap、console、security、OTA、watchdog、power_button |
| `ui/` | LVGL 移植层（`graphics` / `lvgl` / `awtk`） |
| `tests/` | 框架层自测 |

### 2.3 `hardware/`

| 路径 | 说明 |
|:-----|:-----|
| `include/` | **HAL 公开头（跨层唯一契约）**：`audio.h` `camera.h` `display.h` `wifi.h` `gpio.h` `hardware.h` … |
| `audio/` `camera/` `display/` `input/` `network/` `osd/` `gpu/` | 各能力模块；实现按 `hal/<SoC 全名>/` 分离（如 `camera/hal/AnyCloud39AV200/`） |
| `vision/isp/` | ISP 调优（各 SoC 实现在 `hal/`） |
| `vision/motion_detect/` | 移动侦测 |
| `driver/` | gpio / adc / pwm / pwd / led / ir / wdt / efuse / blkdev / mmc / motor |
| `soc_platform/` | 平台抽象（启动、内存、时钟等） |
| `hardware.c` | `hw_get_module()` / `hw_device_open()` 的实现 |
| `tests/` | 适配层自测 |

### 2.4 `system/core/`

`log` `config` `thread` `time` `watchdog` `memory` `str` `file` `crypto` `geometry`
`image_convert` `include/`

> 线程统一走 `thread_create(&id, fn, param, name, stack_size, priority)`，默认栈 100KB。

### 2.5 `project/<product>/`

| 路径 | 说明 |
|:-----|:-----|
| `config.mk` | **产品唯一配置入口** |
| `include/` | 产品手写头，不参与生成：`app_globle_config.h`、`app_product_config.h`、`lv_conf.h` |
| `system/<SoC>/<SDK>/board/<BOARD>/` | 板级 overlay：`config.mk`（DTB、分区）、`rootfs/`（`settings.json`、`fw_*.ini`、启动脚本） |
| `system/<SoC>/<SDK>/patch/` | SDK 源码补丁 |
| `system/<SoC>/<SDK>/patch_fastsys/` | 快启补丁与工具（仅 `CONFIG_CELESTIA_FASTSYS=y`） |
| `package.sh` / `build_soc_system.sh` | 打包与系统镜像构建 |
| `setup_toolchain.sh` | AOS 产品专用工具链注入（`anypi_mira_dev`） |

> 由 `config.mk` **生成**的 `config.h` 不在源码树内，落盘在
> `out/<product>/build/project/include/config.h`。

### 2.6 `third_party/`

`lvgl` `awtk` `ffmpeg` `mbedtls` `openssl` `curl` `libwebsockets` `cjson` `iniparser`
`minIni` `freetype` `jpeg` `jpegdec` `libpng` `zlib` `iconv` `tslib` `zxing` `jerryscript` `asr`

> 统一由 `third_party/Makefile` + `vendor_rules.mk` 驱动，按 `CONFIG_THIRD_PARTY_*` 与
> `CONFIG_THIRD_PARTY_TARGET_ARCH`（`arm` / `riscv`）编译。**不得**在库源码里塞产品逻辑。

## 3. 新文件放哪（决策表）

| 你要新增 | 放这里 | 是否重编主程序 |
|:---------|:-------|:--------------|
| 平板级（`TABLET`） App | `application/packages/tablet/<app>/` | 否（SO） |
| 穿戴级（`WEARABLE`） App | `application/packages/wearable/<app>/` | 否（SO） |
| IOTCamera 业务模块 | `application/packages/ipcamera/<name>/` | 否（SO） |
| 云平台 | `application/cloud/<vendor>/` | 否（SO） |
| 常驻服务 | `application/services/<svc>/` | 是（static 链入） |
| 框架能力 | `framework/services/<svc>/`（含 `include/*.h`） | 是 |
| 对外协议 | `framework/gateway/<proto>/` | 是 |
| 新 SoC 的 HAL 实现 | `hardware/<module>/hal/<SoC 全名>/` | 是 |
| HAL 公开接口 | `hardware/include/*.h` | 是 |
| 基础库能力 | `system/core/<module>/` | 是 |
| 新产品工程 | `project/<product>/`（`config.mk` + `system/<SoC>/<SDK>/board/<BOARD>/`） | 是 |
| 新第三方库 | `third_party/<lib>/` + `third_party/Makefile` | 是 |
| 文档 | `docs/`（结构见 [README](../README.md)） | — |

## 4. 命名约定速查

| 对象 | 约定 | 示例 |
|:-----|:-----|:-----|
| 产品目录 | `anypi_<代号>[_dev]` | `anypi_nova_dev` |
| 板型 | `<EVB\|PDB>_<MBD\|CBD>_<芯片/模组>[_<形态>]` | `EVB_CBD_AK3918AV130` |
| GUI 包目录 | 按**形态**而非产品命名 | `tablet` / `wearable` / `ipcamera` |
| 配置宏 | `CONFIG_<域>_<模块>_<开关>` | `CONFIG_FW_MULTIMEDIA_CAMERA_OSD` |
| 框架公开头 | `framework/**/include/<svc>.h` | `framework/services/network/include/network.h` |
| SO 模块入口 | `module_init()` / `module_start()` | `application/packages/ipcamera/ipc_cam/` |

完整规则（含正例 / 反例）见 [glossary §11](../glossary.md#11-标识符命名规范)。

## 5. 相关文档

- [架构总览](overview.md) —— 分层、契约、运行期、启动、构建
- [产品形态](form-factor.md) —— 判据、启动分岔、选型
- [术语与命名对照表](../glossary.md) —— `AOS` / `IOTCamera` / `AI_GLASSES` / `TABLET` / `WEARABLE` / EVB / PDK
- [下载源码](../quick-start/download.md) —— 获取源码与 SDK 放置
- [BSP 与新增产品](../quick-start/bsp-and-products.md) —— 加板 / 加 SoC
- [约定与规范](../guide/conventions.md) —— 文档与代码的统一约定
