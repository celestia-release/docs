---
name: celestia-framework
description: >-
  Celestia Framework 层（framework/）开发、排障与新增服务：平台无关服务句柄 multimedia /
  network / notification / device / storage（services/，`<svc>_open()` 返回带命名 sub-ops
  的聚合句柄）、网关协议 nebula/rtsp（gateway/）、runtime bootstrap / console / OTA（runtime/）、
  GUI 端口（ui/）。对上经 `*_open()` 供 App，对下只 include `hardware/include/*.h` 公开 HAL 头，
  禁止碰 `hardware/*/hal/<SoC>/` 芯片私有实现。含"新增服务"清单；multimedia 含
  `FFMPEG_LINK_MODE`（none/decode/normal/whole，Nova 16MB 用 decode）。改动后用 `./build.sh -f` 编译。
  当排查或扩展多媒体/网络/存储/通知/协议/runtime 时使用。
---

<p align="right"><a href="celestia-framework.en.md">English</a> | <b>中文</b></p>

# Framework 层 skill

> 概念：`docs/guide/framework.md`；对外 API 速查：`docs/guide/framework-api.md`。
> 总览与跨层规则见 [celestia](celestia.md)。

## 目录地图

服务的真实布局：主实现文件在服务根目录（或 `core/`），每个功能是带自己 `Makefile` 的**子目录**，`include/` 放唯一对外头。**没有 `src/` 目录，也没有 `<svc>_ops.c`/`<svc>_hal_glue.c` 约定**——ops getter 就在各子模块文件里（如 `led/led_manager.c` 的 `led_get_ops()`）。

```
framework/
├── services/          # 5 个服务：multimedia network notification device storage
│   ├── multimedia/    # libmultimedia.so   core/(multimedia.c) av_codec/ audio/ camera/ media/ include/
│   ├── network/       # libnetwork.so      network.c include/ wifi/ ethernet/ ntp/ ping/ rtsp/
│   ├── notification/  # libnotification.so core/(notification.c) message/ alarm/ internal/ include/
│   ├── device/        # libdevice.so       device.c include/ ptz/ led/ power/ backlight/ night_vision/
│   └── storage/       # libstorage.so      storage.c include/ tfcard/ flash/
├── gateway/
│   ├── nebula_protocol/   # libnebula_protocol.so  (PROJ=nebula_protocol)
│   └── rtsp_protocol/     # librtsp_protocol.so    (PROJ=rtsp_protocol；注意上游 librtsp_server.so 另在 network/rtsp/rtsp_server/)
├── runtime/           # bootstrap/(runtime_init.c) console/ ota/ power_button/ watchdog/ include/
└── ui/                # common/include/gui_display.h、lvgl/include/lv_port.h、graphics/ awtk/
```

公开头（对上）：`framework/services/*/include/*.h`、`framework/runtime/include/*.h`、`framework/gateway/*/include/*.h`、`framework/ui/**/include/*.h`（如 `multimedia.h` `network.h` `notification.h` `device.h` `storage.h` `runtime_bootstrap.h` `console.h` `ota.h` `nebula_protocol.h` `rtsp_protocol.h` `gui_display.h`）。
**每个模块只允许一个对外头**（如 `multimedia/include/multimedia.h`）；禁止再新增同模块公开 `.h`。新能力优先挂到已有 sub-ops / `reserved`，能改内部接口就不要扩对外符号。

## 服务契约模式

每个服务 `<svc>_open(<svc>_t **)` / `<svc>_close(<svc>_t *)`。句柄不是单一 ops 表，而是**聚合句柄**：一个静态实例携带多个**命名 sub-ops 指针**，惰性初始化并由 `#ifdef CONFIG_FW_*` 守卫。以 `device.c` 为准：

```c
/* device.h：命名 sub-ops + reserved 保二进制兼容 */
typedef struct {
    led_ops_t       *led;
    ptz_ops_t       *ptz;
    power_ops_t     *power;
    nv_ops_t        *nv;
    backlight_ops_t *backlight;
    void            *reserved[10];
} device_t;

/* device.c：static 实例，open 时按开关惰性填充 sub-ops，close 对称释放 */
static device_t g_device = { .led = NULL, /* ... */ };
int device_open(device_t **device) {
    if (!device) return DEVICE_ERROR_INVALID_PARAM;
#ifdef CONFIG_FW_DEVICE_LED
    if (!g_device.led) g_device.led = led_get_ops();   /* getter 在 led/led_manager.c */
#endif
    /* ptz/power/nv/backlight 同理 */
    *device = &g_device;
    return DEVICE_OK;
}
```

调用方：`device_t *d; device_open(&d); d->led->set_scene(&cfg);`。

要点：
- 顶层 `*_open` **不使用 REENTRY 宏，也没有 `<svc>_service_is_open()` 符号**。就绪状态各服务自定：`multimedia` 用 ops 成员 `is_open` + `g_multimedia_initialized` 标志。
- `system/core` 的 `REENTRY_*` 宏只用于**内部 worker 叶子模块**（`media_record.c`、`rtsp_server.c`、`tfcard_utils.c`），不用于顶层 open。
- 错误码走 `SYSTEM_ERROR_CODE(FRAMEWORK_DOMAIN, module, error)`，0=成功、负数=错误；禁止裸 `int` 返回码。

### `*_open()` 入口表

| 服务 | 入口 / 出参 | 库 | 配置开关 |
|------|-------------|----|---------|
| multimedia | `multimedia_open(multimedia_t **)` | libmultimedia.so | `CONFIG_FW_MULTIMEDIA` |
| network | `network_open(network_t **)` | libnetwork.so | `CONFIG_FW_NETWORK` |
| notification | `notification_open(notification_t **)` | libnotification.so | `CONFIG_FW_NOTIFICATION` |
| device | `device_open(device_t **)` | libdevice.so | `CONFIG_FW_DRIVER`（注意不是 FW_DEVICE） |
| storage | `storage_open(storage_t **)` | libstorage.so | `CONFIG_FW_STORAGE` |
| runtime | `runtime_bootstrap_open(...)` | — | `CONFIG_FW_SYSTEM` |
| gateway | nebula/rtsp 协议 open | libnebula_protocol.so / librtsp_protocol.so | `CONFIG_APP_NEBULA` / `CONFIG_APP_RTSP` |
| ui | lv_port / gui_display | — | `CONFIG_FW_GUI_LVGL` |

服务由**应用层（app_main）打开**，不是 `runtime/bootstrap/runtime_init.c` 自动拉起（后者只做 runtime bootstrap init/open/close + load_config）。

## 对下接口（只能这样调）

| 调用 | 头文件 | 说明 |
|------|--------|------|
| HAL 公开能力 | **整套** `hardware/include/*.h`（24 个：adc audio camera display driver_error efuse ethernet gpio gpu hardware header input input_evdev ir isp led mmc motion_detect motor osd pwm soc_platform wdt wifi） | 经 `hw_get_module()` 拿 `hw_device_t` 后调 ops |
| 横向能力 | `system/core/include/*.h` | log/thread/config/timer/错误码 |

## 禁止事项（必拒）

- ❌ `#include "hal_audio_api.h"/"audio_hw.h"/"hal_camera_api.h"/"camera_hw.h"` 等 HAL 私有头；
- ❌ 引用 `hardware/*/hal/<SoC>/` 下任何芯片私有路径、出现 `AnyCloud39AV200`/`AnyCloud39AV100_V2` 字样；
- ❌ 反向依赖 application（`application/**`）；
- ❌ 把配置路径硬编码为绝对路径 `/mnt/...`（应经 `config.h::config_build_path()` 生成）；
- ✅ 跨 SoC 差异封装在 HAL 实现里，Framework 只见统一的 `hardware/include/*.h` 契约。

## 新增服务 / 子能力（清单）

1. **目录**：`framework/services/<svc>/<svc>.c`（提供 `<svc>_open/close`）+ `include/<svc>.h`（唯一对外头）+ 每个功能一个带 `Makefile` 的子目录，其中含 `<feat>_get_ops()`。对齐 `framework/services/device/`。
2. **头文件**：结构从 `framework/services/device/include/device.h` 复制——版本宏 + 错误码宏 + 命名 sub-ops 的聚合结构 + `void *reserved[N]` + `<svc>_open/close` 声明。约 10 行骨架：
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
3. **错误码**：`system/core/include/system_error.h` 已有模块 ID 映射，不是简单顺延——`FRAMEWORK_MODULE_NETWORK=0x00 NOTIFICATION=0x01 MULTIMEDIA=0x02 RUNTIME=0x03 DRIVER==DEVICE=0x04 GUI==UI=0x05 AI=0x06(保留) STORAGE=0x07 GATEWAY=0x08 NEBULA_PROTOCOL=0x09`。注意 DRIVER/DEVICE、GUI/UI 是**别名**（同值）。新增时占空位并同步 `docs/guide/system-core-api.md`。
4. **Makefile**（照抄 `framework/services/device/Makefile`）：
   ```make
   MODULE_ROOT ?= $(shell pwd)
   include $(COMPILATION_RULES)      # 由 build.sh 传入，无 config.mk/rules.mk 文件
   include $(PROJECT_CONFIG_FILE)
   PROJ = <svc>                      # 产出 lib$(PROJ).a / lib$(PROJ).so.$(VERSION)
   obj-$(CONFIG_FW_<SVC>_<FEAT>) += <feat>/     # 子目录聚合
   SRCS := $(MODULE_ROOT)/<svc>.c               # 顶层主实现
   # install 把 include/<svc>.h 拷到 $(FRAMEWORK_INSTALL_DIR)/include/
   ```
   顶层 `framework/Makefile` 注册（plain obj-y，非 FW_SUBDIRS）：`obj-$(CONFIG_FW_<X>) += services/<svc>/`。
5. **接入点**：在**应用层打开服务处**（app_main）wire `<svc>_open()`/`<svc>_close()`，勿假设 runtime 自动拉起。
6. **配置**：配置文件是 JSON（cjson），非 INI——如 `fw_<svc>.json`（或视项目而定）。
7. **文档**：同步 `docs/guide/framework-api.md`、`docs/guide/framework.md` 模块清单，新错误码同步 `system_error.h` + `docs/guide/system-core-api.md`。

## 排障：以 Framework 为主

1. 先分析本服务实现：ops 流程、聚合句柄状态、线程/缓冲、JSON 配置解析。
2. 怀疑 HAL 时**只确认 `hardware/include/*.h` 契约**（签名、版本宏、返回值、`hw_get_module` 是否拿到 device），核对本层调用是否正确。
3. 仅当 HAL 接口返回错误/日志证明其失败，才切到 [celestia-hardware](celestia-hardware.md) 下钻。
4. 怀疑通用工具/线程/配置问题时下钻 [celestia-system-core](celestia-system-core.md)。
5. 双产品：HAL 行为差异由 AV200 / AV100_V2 各自实现，Framework 代码本身应 SoC 无关。

## 常见问题（Framework）

| 现象 / log | 根因 | 修复 |
|---|---|---|
| `invalid inbuf cap=524288` | payload 在 `codec_mode` 前计算 | 先解析 `codec_mode` 再算 |
| live 预览被拍照/录像打挂 | teardown 或 `load_config` 清 live | 对称 teardown；live 期不 reload |
| `stream_reg_client` 失败 | 通道/角色未在能力集或未 open camera | 核 `fw_video_config` 与 open 时序 |
| 开 AI 后崩溃 | 与 camera_manager 生命周期竞态 | mutex + 本地持有 device 指针 |
| JSON 配了不生效 | 路径不在 board overlay 或被 patch 盖 | 改 `board/<BOARD>/rootfs/.../etc/config/` |
| 通话无画面 / `VIDEO_DECODER_SW_STUB` | `CONFIG_FRAMEWORK_MULTIMEDIA_FFMPEG_LINK_MODE=none` | Nova：`decode`（链 SW H.264，无 file demux）+ third_party `DECODE_ONLY`；见 [celestia-project](celestia-project.md) §APP/`libmultimedia` |
| `usr.sqsh4` 顶破 APP（`0x82E000`） | FFmpeg 链进 `libmultimedia` 过大 | `LINK_MODE=decode` + `DECODE_ONLY`；或把内容挪 SD `install/` |

运行期归类与更多解法见 [celestia-debug](celestia-debug.md) **B/D/E/G 类**。

## 编码完成 → 编译（必做）

```bash
cd ~/celestia
./build.sh -f --product-name anypi_nova_dev
./build.sh -f --product-name anypi_luna
# 同时改了 HAL 契约用法时，先编 hardware 再编 framework：
./build.sh -h -f --product-name anypi_luna
# luna 闭环（如改 multimedia 预览）：编译 + 烧录上板，串口看 log
./build.sh -f --product-name anypi_luna
```

解决全部编译错误再交付。链接缺符号→确认是 third_party（lvgl/cjson）未先构建，还是 HAL 公开头签名变化。改了服务开关（`CONFIG_FW_*`）时核对两产品 `project/<proj>/config.mk`。
