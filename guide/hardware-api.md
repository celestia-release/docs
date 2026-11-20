<p align="right"><a href="hardware-api.en.md">English</a> | <b>中文</b></p>

# Hardware HAL API 参考手册（对外可查接口）

> **本文范围**：Hardware HAL 对外可查接口参考：逐模块给出标识、数据结构、ops 与错误码。
>
> HAL 契约与新增模块流程见 [hardware-hal.md](hardware-hal.md)；分层、契约与运行期模型见 [架构总览](../architecture/overview.md)；术语以 [术语与命名对照表](../glossary.md) 为唯一解释源。

---

## 0. 速查目录

| 分类 | 模块 | Module ID | 主要结构 | 章节 |
| --- | --- | --- | --- | --- |
| 平台 | SoC Platform | — | 直调 API | §1 |
| 基础契约 | hardware.h | — | `hw_module_t`/`hw_device_t` | §2 |
| 显示/图形 | Display | `display` | `display_device_t` | §3 |
|  | OSD | `osd` | `osd_hw_device_t` | §4 |
|  | GPU (2D) | `gpu` | `gpu_device_t` | §5 |
| 视觉 | ISP | `vision_isp` | `isp_device_t` | §6 |
|  | Motion Detect | `motion_detect` | `motion_detect_device_t` | §7 |
| 多媒体 | Audio | `audio_hal` | `audio_hw_device_t` | §8 |
|  | Camera | `camera` | `camera_hw_device_t` | §9 |
| 网络 | WiFi | `wifi` | `wifi_hw_device_t` | §10 |
|  | Ethernet | `ethernet` | `ethernet_hal_device_t` | §11 |
| 输入 | Input (base) | `input` | `input_device_t` | §12 |
|  | Input evdev | `input`/`evdev` | `evdev_device_t` | §13 |
| 驱动 | GPIO | `gpio` | `gpio_device_t` | §14 |
|  | ADC | `adc` | `adc_device_t` | §15 |
|  | PWM | `pwm` | `pwm_device_t` | §16 |
|  | LED | `led` | `led_device_t` | §17 |
|  | Motor | `motor` | `motor_device_t` | §18 |
|  | WDT | `wdt` | `wdt_device_t` | §19 |
|  | IR (IRCUT/LED) | `ir` | `ir_device_t` | §20 |
|  | eFuse | `efuse` | `efuse_device_t` | §21 |
|  | MMC/SD | `mmc` | `mmc_device_t` | §22 |

> **约定**：
> - 所有 `hw_*_module_init()` 由 HAL 模块在 BSP 层通过构造器注册，Framework 侧通常不直接调用。
> - 所有 `*_device_t->common.close(&dev->common)` 即等价于对应的 `xxx_hw_device_close()` 内联函数。
> - 错误码域：`SYSTEM_ERROR_CODE(HARDWARE_DOMAIN, HARDWARE_MODULE_<X>, <CODE>)`；详见 `system_error.h`。

---

## 0.1 HAL 通用函数调用契约（所有 ops 共用）

为避免在每个模块重复同样的说明，以下契约**默认适用于下文 §3–§22 中的所有 HAL ops**。各模块章节只在**存在差异**时另行标注。

### 0.1.1 返回值约定

| 返回类型 | 含义 |
|---|---|
| `int` | `0` = 成功；`< 0` = `SYSTEM_ERROR_CODE(HARDWARE_DOMAIN, HARDWARE_MODULE_<X>, code)`；`> 0` 为业务数据（例如 `ping_ipv4` RTT、`preset_get_count`） |
| `uint32_t` | 实际测量/读取值（如电平、亮度、采样率），无错误通道；参数错误时 ops 可能返回 `0` 表示无效，具体见各模块 |
| `bool` / `const char *` | 查询类接口，返回值即结果（读失败返回 `false`/`NULL`） |

### 0.1.2 参数约定

- **输入指针**（例如 `const <T> *cfg`）：调用方持有，ops 仅在调用期间读取，**不得**保留指针到调用返回之后。
- **输出指针**（例如 `<T> *out`）：调用方分配，ops 写入；允许 `NULL` 时明确标注。
- **内部缓冲输出**（例如 `void **data` / `char *`）：指针由模块持有，调用方**禁止** `free()`；内容生命周期到**下一次同 ops 调用**或 `device.close` 为止（每个模块章节会特别注明）。
- **名称参数** `const char *name`：指硬件实例名（如 `"gpio_led0"`、`"pwm_bl"`、`"motor0"`），由 SoC BSP 的 DTS / config 定义。
- **通用错误**：空指针或未初始化设备 → `_INVALID_PARAM` / `_NOT_INIT`。

### 0.1.3 生命周期与调用顺序

```
hw_get_module("<id>", &mod)       // 可重入，多次返回同一 module
  → mod->methods->open(mod, "<inst>", &dev)   // 每次返回独立 dev 实例
      → dev->ops(...)             // 仅在 open/close 之间调用
  → dev->common.close(&dev->common)
```

- `open` 与 `close` 必须**配对**；不配对会导致资源泄漏或底层句柄崩溃。
- 不同 `inst` 名称对应独立实例；同 `inst` 重复 `open` 由模块自行决定（多数返回 `_BUSY`）。

### 0.1.4 线程安全

| 场景 | 约束 |
|---|---|
| `hw_get_module` | 完全线程安全，内部带锁 |
| `module->methods->open` / `close` | **不**保证 open/close 间并发安全；上层（通常 Framework）自行串行化 |
| 同一 `dev` 的 ops | 默认**单线程调用**；若需多线程并发调用同 `dev`，调用方加锁 |
| 不同 `dev` 的 ops | 线程安全（相互独立） |
| HAL 回调（notify / event） | 在 HAL 内部线程触发，**禁止**在回调内做阻塞操作或反向调用同 `dev` 的阻塞 ops |

### 0.1.5 默认错误码族

每个模块的错误码均遵循 `HARDWARE_<MODULE>_OK / _INVALID_PARAM / _NOT_INIT / _MEMORY / _TIMEOUT / _NOT_FOUND / _BUSY / _IO / _PERMISSION_DENIED / _NOT_SUPPORTED / _INVALID_STATE / _OPERATION_FAILED` 通用族；各模块若有专属错误码，在对应章节（如 §8.9 / §9.8）单列。

### 0.1.6 模块 `hw_*_module_init()` 函数

- 由 HAL so 库的构造器 (`__attribute__((constructor))`) 或 BSP 链接脚本在进程早期自动调用，完成 `hw_module_t` 注册到全局表。
- Framework / 应用侧**不直接调用**；仅作为白盒说明列出函数原型。

---

## 1. SoC Platform（`hardware/include/soc_platform.h`）

平台抽象层，开机由 `framework_init()` 调用，用于隐藏芯片差异。

### 1.1 直调 API

| 函数 | 原型 | 功能 |
| --- | --- | --- |
| `soc_platform_init` | `int soc_platform_init(int mode)` | 初始化 SoC 平台；`mode` 0=normal，1=debug |
| `soc_platform_set_log_level` | `int soc_platform_set_log_level(int level)` | 设置平台日志级别 1=ERR / 2=WARN / 3=NOTICE / 4=NORMAL / 5=INFO / 6=DEBUG |

### 1.2 错误码

| 宏 | 说明 |
| --- | --- |
| `SOC_PLATFORM_OK` | 成功 |
| `SOC_PLATFORM_ERROR_INVALID_PARAM` | 参数非法 |
| `SOC_PLATFORM_ERROR_NOT_SUPPORTED` | 功能不支持 |
| `SOC_PLATFORM_ERROR_PLATFORM_NOT_SUPPORTED` | 当前芯片平台不支持 |
| `SOC_PLATFORM_ERROR_HAL_INIT_FAILED` | HAL 初始化失败 |
| `SOC_PLATFORM_ERROR_HAL_DEINIT_FAILED` | HAL 反初始化失败 |

> 实际底层实现由 `hardware/soc_platform/hal_soc_platform_api.h` 承担（按 SoC 工程目录提供，`hal/` 下分 `AnyCloud39AV200/`、`AnyCloud39AV100_V2/` 与 `Sky39AV130/`）；此外 `soc_platform/` 还含 `hal_ipu_api.h` 与弱符号桩 `hal_ipu_weak_stub.c`（未实现 SoC 的 IPU 接口回落到弱桩）。详见 [quick-start/bsp-and-products.md](../quick-start/bsp-and-products.md)。

---

## 2. HAL 基础契约（`hardware/include/hardware.h`）

所有 HAL 模块结构以此为基：

```
struct hw_module_t {
    uint32_t              tag;
    uint16_t              module_api_version;
    uint16_t              hal_api_version;
    const char           *id;
    const char           *name;
    const char           *author;
    struct hw_module_methods_t *methods;   /* .open(...) */
    void                 *dso;
    uint64_t              reserved[32-7];
};

struct hw_device_t {
    uint32_t              tag;
    uint32_t              version;
    struct hw_module_t   *module;
    uint64_t              reserved[12];
    int (*close)(struct hw_device_t *device);
};
```

### 2.1 公共 API

| 函数 | 原型 | 说明 |
| --- | --- | --- |
| `hw_get_module` | `int hw_get_module(const char *id, const struct hw_module_t **module)` | 按 ID 拿模块 |
| `hw_get_module_by_class` | `int hw_get_module_by_class(const char *class_id, const char *inst, const struct hw_module_t **module)` | 分类 + 实例名 |
| `hw_register_module` | `int hw_register_module(const struct hw_module_t *module)` | 模块向系统注册（BSP 侧调用） |
| `module->methods->open(module, name, &device)` | — | 打开设备实例 |
| `device->close(device)` | — | 关闭设备实例 |

### 2.2 常用宏

| 宏 | 作用 |
| --- | --- |
| `HARDWARE_MODULE_TAG` | `hw_module_t.tag` 固定值 |
| `HARDWARE_DEVICE_TAG` | `hw_device_t.tag` 固定值 |
| `HARDWARE_MODULE_API_VERSION(maj, min)` | 编码 module 版本 |
| `HARDWARE_DEVICE_API_VERSION(maj, min)` | 编码 device 版本 |
| `HAL_MODULE_INFO_SYM` | 模块符号名（宏名由 `module_id` 拼接） |
| `TO_HW_DEVICE_T_OPEN(p)` | 安全向 `hw_device_t**` 转型 |

---

## 3. Display HAL（`hardware/include/display.h`）

### 3.1 标识

| 项 | 值 |
| --- | --- |
| Module ID | `"display"` (`DISPLAY_HARDWARE_MODULE_ID`) |
| Instance | `"fb"` (`DISPLAY_DEVICE_INSTANCE_FB`)，设备 `/dev/fb0` |

### 3.2 数据结构

```c
typedef struct display_buffer {
    void     *vaddr;   /* 虚拟地址 */
    uint64_t  paddr;   /* 物理地址 */
    uint32_t  size;    /* 字节数 */
} display_buffer_t;
```

### 3.3 `display_device_t` ops

| ops | 原型 | 说明 |
| --- | --- | --- |
| `get_resolution` | `int (*)(display_device_t*, uint32_t *w, uint32_t *h)` | 查询分辨率 |
| `get_offline_buffer` | `int (*)(display_device_t*, display_buffer_t*)` | 获取离屏 buffer（Nova 直写） |
| `get_pixel_format` | `int (*)(display_device_t*, pixel_format_t*)` | 像素格式 |
| `get_bpp` | `int (*)(display_device_t*, uint32_t*)` | 每像素 bit 数 |
| `flush` | `int (*)(display_device_t*)` | 刷新至屏幕 |

### 3.4 模块入口

```c
int hw_display_module_init(void);
int hw_display_get_module(const struct hw_module_t **module, const char *inst);
```

### 3.5 错误码（部分）

`HARDWARE_DISPLAY_OK` / `..._INVALID_PARAM` / `..._NOT_INIT` / `..._MEMORY` / `..._IO` / `..._NOT_FOUND` / `..._INVALID_STATE` / `..._OPERATION_FAILED` / `..._NOT_SUPPORTED` / `..._DEVICE_BUSY` / `..._MAP_FAILED` / `..._FLUSH_FAILED`。

---

## 4. OSD HAL（`hardware/include/osd.h`）

### 4.1 标识与常量

| 项 | 值 |
| --- | --- |
| Module ID | `"osd"` |
| Interface | `"osd_hw_if"` |
| `OSD_OBJ_COUNT` | 8（每 channel 最大对象数） |
| `OSD_TEXT_LEN` | 100 |
| `OSD_BOX_MAX_NUM` | 50 |

### 4.2 数据结构

```c
typedef enum {
    OSD_COLOR_MODE_RGB1 = 1, OSD_COLOR_MODE_RGB2 = 2, OSD_COLOR_MODE_RGB4 = 4,
} osd_color_mode_e;

typedef struct osd_attr_s        { osd_color_mode_e color_mode; unsigned int *pu32ColorTable; } osd_attr_t;
typedef struct osd_rect_info_s   { int x, y, width, height; }                                   osd_rect_info;
typedef struct osd_bitmap_info_s { unsigned char *pData; int iDataSize, iWidth, iHeight, iBitPerPixel; } osd_bitmap_info;
typedef struct osd_font_info_s   { char cFontName[256]; int iFontSize, iColor, iBKClor; }       osd_font_info;
typedef struct osd_chn_info_s    { int iBitPerPixel; unsigned int *pu32ColorTable; }            osd_chn_info;
typedef void  pp_box_group_t;  /* 平台 HAL 内部定义 */
```

### 4.3 `osd_hw_device_t` ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `init` | `int (*)(const osd_hw_device_t*, osd_attr_t*, const char *font_file)` | 初始化 + 加载字体 |
| `deinit` | `int (*)(const osd_hw_device_t*)` | 反初始化 |
| `draw_bmp` | `int (*)(const osd_hw_device_t*, int nChn, int nObj, osd_rect_info*, const unsigned char *pData, int iDataSize)` | 绘制位图 |
| `clean_obj` | `int (*)(const osd_hw_device_t*, int nChn, int nObj)` | 清除对象，`nObj=-1` 清全部 |
| `copy_cavas_to_vi` | `int (*)(const osd_hw_device_t*, int nChn)` | 画布 → VI 硬件 OSD |
| `set_font` | `int (*)(const osd_hw_device_t*, int nChn, osd_font_info*)` | 设置字体 |
| `get_text_wh` | `int (*)(const osd_hw_device_t*, int nChn, const char*, int len, int *w, int *h)` | 计算文字宽高 |
| `draw_text` | `int (*)(const osd_hw_device_t*, int nChn, int nObj, int x, int y, const char*, int len)` | 绘制文字 |
| `set_attr` | `int (*)(const osd_hw_device_t*, int nChn, osd_chn_info*)` | 设置 channel 属性 |
| `set_enable` | `int (*)(const osd_hw_device_t*, int nChn, int nObj, int enable)` | 使能对象 |
| `draw_box` | `int (*)(const osd_hw_device_t*, int nChn, pp_box_group_t*)` | 绘制检测框组 |
| `adjust_box_coordinates` | `int (*)(const osd_hw_device_t*, pp_box_group_t*, int video_w, int video_h)` | 归一化坐标 → 像素坐标 |

### 4.4 入口/便捷

```c
int osd_hw_device_open(const hw_module_t *module, osd_hw_device_t **device);
int osd_hw_device_close(osd_hw_device_t *device);
int hw_osd_module_init(void);
```

### 4.5 错误码

`HARDWARE_OSD_OK` / `..._INVALID_PARAM` / `..._NOT_INIT` / `..._MEMORY` / `..._NOT_FOUND` / `..._NOT_SUPPORTED` / `..._OPERATION_FAILED`。

---

## 5. GPU 2D HAL（`hardware/include/gpu.h`）

### 5.1 标识

| 项 | 值 |
| --- | --- |
| Module ID | `"gpu"` |
| Device | `"gpu"` |
| `IMAGE_OPACITY_MIN/MAX` | 2 / 253 |

### 5.2 数据结构

```c
enum blend_opt { eBLEND_SOLID_COLOR, eBLEND_ALPHA_BLEND = 2, eBLEND_PURE_BACKGROUND, eBLEND_PURE_FOREGROUND };

struct image_colorkey { uint8_t u8OpacityPositive, u8OpacityInverse, u8Differ; uint32_t u32ColorKey; };

struct blend_config   { enum blend_opt option; union { uint32_t u8ColorBackGround, u8ColorForeGround; }; };

typedef struct image_desc {
    uint32_t u32Width, u32Height;
    struct rect rect;               /* 来自 system_core/geometry.h */
    uint8_t *vaddr; uint64_t paddr;
    system_pixel_format_t format;
    transform_t transform;
    uint8_t  u8GlobalOpacity;
    int32_t  addrs_cnt;
    uint8_t *vaddrs[4]; uint64_t paddrs[4];
} image_desc_t;
```

### 5.3 `gpu_device_t` ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `image_copy` | `int (*)(gpu_device_t*, image_desc* in, image_desc* out)` | 拷贝 |
| `image_scale` | `int (*)(gpu_device_t*, image_desc* in, image_desc* out)` | 缩放 |
| `image_transform` | `int (*)(gpu_device_t*, image_desc* in, image_desc* out, transform_t t)` | 翻转/旋转 |
| `image_blend` | `int (*)(gpu_device_t*, image_desc* fg, image_desc* bg, image_desc* out, blend_config*)` | 混合（`out=NULL` 原地） |
| `image_blend_colorkey` | `int (*)(gpu_device_t*, image_desc* in, image_desc* out, image_colorkey*)` | 色键混合 |
| `image_fill` | `int (*)(gpu_device_t*, image_desc* out, uint32_t color)` | 纯色填充 |
| `check_input_format` | `bool (*)(system_pixel_format_t)` | 输入像素格式支持 |
| `check_output_format` | `bool (*)(system_pixel_format_t)` | 输出像素格式支持 |
| `supported_min_width` | `int32_t (*)(void)` | 最小宽 |
| `supported_min_height` | `int32_t (*)(void)` | 最小高 |

### 5.4 入口

```c
int hw_gpu_module_init(void);
```

### 5.5 错误码

`HARDWARE_GPU_OK` / `..._INVALID_PARAM` / `..._NOT_SUPPORTED` / `..._MEMORY` / `..._OPERATION_FAILED`。

---

## 6. ISP HAL（`hardware/include/isp.h`）

### 6.1 标识

| 项 | 值 |
| --- | --- |
| Module ID | `"vision_isp"` |
| Interface | `"vision_isp_hw_if"` |
| 维度常量 | `ISP_MD_DIMENSION_H_MAX=64`，`ISP_MD_DIMENSION_V_MAX=36` |

### 6.2 数据结构

```c
typedef enum { ISP_MODE_DAY, ISP_MODE_NIGHT, ISP_MODE_USR_DEFINE_1, ..._2, ..._3 } isp_mode_t;
typedef enum { ISP_NIGHT_TYPE_IR_LED, ISP_NIGHT_TYPE_WHITE_LED, ISP_NIGHT_TYPE_BLACKLIGHT } isp_night_type_t;

typedef struct { uint16_t u16Stat[36][64]; } isp_md_info_t;

typedef struct {
    uint32_t enNightType;
    uint32_t u32DayCheckFrameNum, u32NightCheckFrameNum, u32ExpStableCheckFrameNum;
    int32_t  s32ExpStableTimeOutMs;
    struct {
        uint32_t u32DayToNightLum, u32NightToDayLum;
        uint32_t u32AwbNightDisMax, u32AwbNightDisMin;
        uint32_t u32GainR, u32GainB;
        int32_t  s32OffsetR, s32OffsetB;
    } stIrLedNightParm;
} isp_soft_ps_param_t;
```

### 6.3 `isp_device_t` ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `init` | `int (*)(isp_device_t*, void *config)` | 初始化 |
| `deinit` | `int (*)(isp_device_t*)` | 反初始化 |
| `set_param` | `int (*)(isp_device_t*, int key, long value)` | 设置参数 |
| `get_frame` | `int (*)(isp_device_t*, void *frame_out)` | 获取帧数据 |
| `switch_mode` | `int (*)(isp_device_t*, isp_mode_t)` | 日/夜模式切换 |
| `set_soft_ps_param` | `int (*)(isp_device_t*, const isp_soft_ps_param_t*)` | 软光感参数 |
| `get_soft_ps_level` | `int (*)(isp_device_t*, int pre_status, int *day_night)` | 软光感检测结果 |

### 6.4 `isp_module_t` ops

```c
int (*open)(const isp_module_t*, isp_device_t**);
int  hw_isp_module_init(void);
```

### 6.5 直调 API（供 Framework/Driver）

| 函数 | 原型 |
| --- | --- |
| `isp_switch_mode` | `int isp_switch_mode(int dev_id, isp_mode_t mode)` |
| `isp_set_soft_ps_param` | `int isp_set_soft_ps_param(int dev_id, const isp_soft_ps_param_t *param)` |
| `isp_get_soft_ps_level` | `int isp_get_soft_ps_level(int dev_id, int pre_status, int *day_night)` |

### 6.6 错误码

`HARDWARE_ISP_OK` / `..._INVALID_PARAM` / `..._NOT_INIT` / `..._MEMORY` / `..._NOT_FOUND` / `..._NOT_SUPPORTED` / `..._OPERATION_FAILED`（域复用 `HARDWARE_MODULE_VISION`）。

---

## 7. Motion Detect HAL（`hardware/include/motion_detect.h`）

### 7.1 标识

| 项 | 值 |
| --- | --- |
| Module ID | `"motion_detect"` |
| Interface | `"motion_detect_hw_if"` |

### 7.2 数据结构

```c
typedef struct { int32_t x, y, width, height; } motion_detect_rect_t;
typedef struct { int enable; int sensitivity; motion_detect_rect_t valid_area; } motion_detect_config_t;
typedef struct { motion_detect_rect_t rect; float area_ratio; }                  motion_detect_result_t;
typedef struct { int left, top, right, bottom, count; }                          motion_detect_box_t;

typedef void* motion_detect_mrd_handle_t;

typedef void (*motion_detect_callback_t)(int n, const motion_detect_result_t *results, void *user_data);
typedef int  (*motion_detect_day_night_callback_t)(void);   /* 返回 0=day, 1=night, <0=error */
```

### 7.3 `motion_detect_device_t` ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `init` / `deinit` | `int (*)(motion_detect_device_t*)` | 初始化/反初始化 |
| `set_config` | `int (*)(motion_detect_device_t*, const motion_detect_config_t*)` | 配置 |
| `get_config` | `int (*)(motion_detect_device_t*, motion_detect_config_t*)` | 读取 |
| `set_callback` | `int (*)(motion_detect_device_t*, motion_detect_callback_t, void *user)` | 设回调 |
| `set_day_night_callback` | `int (*)(motion_detect_device_t*, motion_detect_day_night_callback_t)` | 注册日夜查询 |
| `handle_isp_md` | `int (*)(motion_detect_device_t*, const void *md_info)` | 接收 ISP MD 信息 |
| `mrd_init` / `mrd_destroy` | MRD 操作入口 | 返回/销毁 handle |
| `mrd_set_filters` | `int (*)(..., unsigned short flt_big, unsigned short flt_small)` | 大/小滤波 |
| `mrd_set_motion_region_size` | `int (*)(..., int size_min, int size_max)` | 区域大小范围 |
| `mrd_set_mode` | `int (*)(..., unsigned char mode)` | 多帧/单帧 |
| `mrd_process` | `int (*)(..., unsigned short md_stat[][64], int len, motion_detect_box_t *boxes)` | 执行检测 |

### 7.4 入口

```c
int hw_motion_detect_module_init(void);
/* motion_detect_module_t->open(module, &device) 打开实例 */
```

### 7.5 错误码

`HARDWARE_MOTION_DETECT_OK` 及 `..._INVALID_PARAM/..._NOT_INIT/..._MEMORY/..._NOT_FOUND/..._NOT_SUPPORTED/..._OPERATION_FAILED`（域与 `HARDWARE_MODULE_VISION` 共用）。

---

## 8. Audio HAL（`hardware/include/audio.h`）

> 本模块唯一的对外头文件为 `hardware/include/audio.h`。
> SoC 侧 `hal_audio_api.h` / `audio_hw.h` 等为内部实现，不对外。

### 8.1 标识与版本

| 项 | 值 |
| --- | --- |
| Module ID | `AUDIO_HARDWARE_MODULE_ID = "audio_hal"` |
| Interface | `AUDIO_HARDWARE_INTERFACE = "audio_hw_if"` |
| 模块 API 版本 | `AUDIO_MODULE_API_VERSION_CURRENT = HARDWARE_MODULE_API_VERSION(0,1)` |
| 设备 API 版本 | `AUDIO_DEVICE_API_VERSION_CURRENT = HARDWARE_DEVICE_API_VERSION(0,1)` |
| 模块初始化 | `int hw_audio_module_init(void)` |

### 8.2 枚举与常量

| 枚举 / 常量 | 取值 |
| --- | --- |
| `audio_devices_t` | `NONE=0x0` / `OUT_SPEAKER=0x01` / `OUT_LINE=0x02` / `OUT_USB_DEVICE=0x04` / `OUT_BUS=0x05` / `IN_AMIC=0x100` / `IN_DMIC=0x101` / `IN_LINE=0x102` / `IN_BUS0=0x103` / `IN_BUS1=0x104` / `IN_USB_DEVICE=0x105` / `IN_LOOPBACK=0x40000` |
| `audio_mode_t` | `NONE / NORMAL / MUSIC / CALL / MAX` |
| `audio_effect_type_t` | `NONE=0` 掩码位：`SOUND_MASK=0xff` / `DETECT_MASK=0xff00` / `SSL_MASK=0xf0000`；常用位：`ASLC / EQ / AEC / NS / RESAMPLE / VAD / CPU_CRY / NPU_CRY / SSL / LSSL / CSSL` |
| `audio_channel_mask_t` | `IN_MONO=0x010000 / IN_STEREO=0x020000 / IN_TRI / IN_QUAD / OUT_MONO=0x01000000 / OUT_STEREO=0x02000000`；`AUDIO_CHANNEL_COUNT_MAX=4` |
| `audio_format_t` | PCM/MP3/AMR/AAC/WMA/ALAW/ULAW/OPUS/FLAC/… 及 `PCM_16_BIT / PCM_8_BIT` 组合；格式掩码 `MAIN_MASK=0xFF000000`, `SUB_MASK=0x00FFFFFF` |
| `audio_dual_mono_mode_t` | `OFF / LR / LL / RR` |
| `audio_timestretch_stretch_mode_t` | `DEFAULT / VOICE` |
| `audio_timestretch_fallback_mode_t` | `CUT_REPEAT / DEFAULT / MUTE / FAIL` |
| 回放速率范围 | `SPEED_MIN=0.01f / MAX=20.0f`；`PITCH_MIN=0.25f / MAX=4.0f`；`NORMAL=1.0f` |

### 8.3 数据结构

```c
struct audio_config {                        /* 8 字节对齐 */
    uint32_t              sample_rate;
    audio_channel_mask_t  channel_mask;
    audio_format_t        format;
    uint32_t              frame_count;
};

typedef struct audio_effect_result_data  { uint8_t *buffer; int bytes; } audio_effect_result_data_t;
typedef struct audio_effect_result_detect{ int32_t value; }              audio_effect_result_detect_t;
typedef struct audio_effect_result_ssl   { int32_t azimuth, elevation, radius; /* 32Q15 */ } audio_effect_result_ssl_t;

typedef struct audio_notify_msg {
    audio_effect_type_t type;
    union {
        audio_effect_result_detect_t detect;   /* NPU/CPU cry: 1=detected */
        audio_effect_result_ssl_t    ssl;      /* 声源定位结果         */
        uint8_t                      generic[32];
    };
} audio_notify_msg_t;

struct audio_playback_rate { float mSpeed, mPitch; audio_timestretch_stretch_mode_t mStretchMode; audio_timestretch_fallback_mode_t mFallbackMode; };
typedef struct audio_stream_buffer { struct audio_config *config; void *data; uint32_t data_size; int status; } audio_stream_buffer_t;
typedef struct audio_capture_result{ uint32_t frame_number; uint64_t timestamp; audio_stream_buffer_t *output_buffer, *input_buffer; } audio_capture_result_t;

typedef int (*stream_callback_t)(stream_callback_event_t event, void *param, void *cookie);
typedef int (*stream_event_callback_t)(stream_event_callback_type_t event, void *param, void *cookie);
```

### 8.4 `audio_stream_t` 通用流 ops（13 项）

| Ops | 原型 | 说明 |
| --- | --- | --- |
| `get_sample_rate` | `uint32_t (*)(const audio_stream_t *)` | 采样率 Hz |
| `set_sample_rate` | `int (*)(audio_stream_t *, uint32_t)` | 当前保留，用 `set_parameters` |
| `get_buffer_size` | `size_t (*)(const audio_stream_t *)` | 缓冲字节数，frame 对齐 |
| `get_channels` | `audio_channel_mask_t (*)(const audio_stream_t *)` | 声道配置 |
| `get_format` | `audio_format_t (*)(const audio_stream_t *)` | 格式 |
| `set_format` | `int (*)(audio_stream_t *, audio_format_t)` | 当前保留，用 `set_parameters` |
| `dump` | `int (*)(const audio_stream_t *, char *path, char *name, int enable)` | 流落盘调试 |
| `get_device` | `audio_devices_t (*)(const audio_stream_t *)` | 当前绑定设备 |
| `set_device` | `int (*)(audio_stream_t *, audio_devices_t)` | 当前保留，用 `set_parameters` |
| `set_parameters` | `int (*)(const audio_stream_t *, const char *name, void *value, unsigned len)` | 键：`Volume / AIParam / Vad / NpuCry / CpuCry / Lssl / AIReboot` 等 |
| `get_parameters` | `char* (*)(const audio_stream_t *, const char *keys)` | 堆字符串，调用方 `free` |
| `add_audio_effect` | `int (*)(const audio_stream_t *, audio_effect_type_t)` | 加入效果 |
| `remove_audio_effect` | `int (*)(const audio_stream_t *, audio_effect_type_t)` | 移除效果 |

### 8.5 `audio_stream_out_t` 输出流 ops（`common` 之外 10 项）

| Ops | 说明 |
| --- | --- |
| `set_volume(out, float)` | 硬件音量 0.0~1.0 |
| `write(out, buf, bytes)` | 写入 PCM；若绑定 callback 则非阻塞 |
| `set_callback(out, cb, cookie)` | 注册非阻塞 write/drain 完成回调 |
| `pause(out) / resume(out)` | 支持离线回放必选 |
| `set_event_callback(out, cb, cookie)` | 编解码格式变化等事件 |
| `get_audio_description_mix_level(out, *dB)` | AD 副流混合电平 |
| `set_audio_description_mix_level(out, dB)` | 同上（设置） |
| `get_playback_rate_parameters(out, *rate)` | 速度/音高/TimeStretch 参数 |
| `set_playback_rate_parameters(out, *rate)` | 同上（设置） |

### 8.6 `audio_stream_in_t` 输入流 ops

| Ops | 说明 |
| --- | --- |
| `set_volume(in, float)` | 采集增益 0.0~1.0（预留） |
| `read(in, buf, bytes)` | 读取 PCM，含效果后处理 |

### 8.7 `audio_hw_device_t` 设备级 ops（12 项）

| Ops | 说明 |
| --- | --- |
| `set_callback_ops(dev, *cb)` | 注册 `process_capture_result` + `notify` |
| `get_supported_devices(dev)` | 返回受支持 `audio_devices_t` 位掩码 |
| `get_master_volume(dev, *vol)` | 主音量（Primary HAL 提供） |
| `set_mode(dev, mode)` | `NORMAL/MUSIC/CALL`，切模式会重新加载效果链 |
| `set_mic_mute(dev, bool)` / `get_mic_mute(dev, *bool)` | MIC 静音控制 |
| `set_parameters(stream, name, value, len)` / `get_parameters(dev, keys)` | 全局键值 |
| `get_input_buffer_size(dev)` | 推荐输入缓冲大小 |
| `open_output_stream(dev, handle, devices, *cfg, **out)` | 打开输出流 |
| `close_output_stream(dev, out)` | 关闭 |
| `open_input_stream(dev, handle, devices, *cfg, **in)` | 打开输入流 |
| `close_input_stream(dev, in)` | 关闭 |

### 8.8 回调与辅助

```c
typedef struct audio_callback_ops {
    void (*process_capture_result)(const struct audio_callback_ops *, const audio_capture_result_t *);
    void (*notify)(const struct audio_callback_ops *, const audio_notify_msg_t *);  /* VAD/Cry/SSL 等 */
} audio_callback_ops_t;

static inline int  audio_hw_device_open (const hw_module_t *m, audio_hw_device_t **dev);
static inline int  audio_hw_device_close(audio_hw_device_t *dev);

static inline audio_format_t audio_get_main_format(audio_format_t);
static inline bool           audio_is_linear_pcm(audio_format_t);
static inline bool           audio_has_proportional_frames(audio_format_t);
```

### 8.9 错误码（`HARDWARE_DOMAIN / HARDWARE_MODULE_AUDIO`）

| 错误 | 说明 |
| --- | --- |
| `HARDWARE_AUDIO_OK` | 成功（= `SYSTEM_SUCCESS`） |
| `INVALID_PARAM / NOT_INIT / MEMORY / TIMEOUT / NOT_FOUND / BUSY / IO / PERMISSION / NOT_SUPPORTED / INVALID_STATE / OPERATION_FAILED` | 通用错误 |
| `STREAM_OPEN_FAILED` (0x0017) | 流打开失败 |
| `STREAM_PARAM_INVALID` (0x0018) | 流参数非法 |
| `CODEC_UNSUPPORTED` (0x0019) | 编解码不支持 |
| `DEVICE_BUSY` (0x001A) | 设备占用 |
| `DEVICE_NOT_AVAILABLE` (0x001B) | 设备不存在 |

---

## 9. Camera HAL（`hardware/include/camera.h`）

> 本模块唯一的对外头文件为 `hardware/include/camera.h`。
> SoC 侧 `hal_camera_api.h` / `camera_hw.h` 等为内部实现，不对外。

### 9.1 标识与版本

| 项 | 值 |
| --- | --- |
| Module ID | `CAMERA_HARDWARE_MODULE_ID = "camera"` |
| Interface | `CAMERA_HARDWARE_INTERFACE = "camera_hw_if"` |
| 模块 API 版本 | `CAMERA_MODULE_API_VERSION_CURRENT = HARDWARE_MODULE_API_VERSION(1,0)` |
| 设备 API 版本 | `CAMERA_DEVICE_API_VERSION_1_0 = HARDWARE_DEVICE_API_VERSION(1,0)` |
| 模块初始化 | `int hw_camera_module_init(void)` |

### 9.2 核心枚举

| 枚举 | 取值 |
| --- | --- |
| `camera_stream_rotation_t` | `0° / 90° / 180° / 270°` 逆时针 |
| `camera_codec_mode_t` | `AVC / HEVC / MJPEG / YUV / RAW` |
| `camera_codec_profile_t` | AVC：`BASE / MAIN / HIGH / C_BASE`；HEVC：`MAIN / MAIN_STILL / MAIN_INTRA / MAIN10_HDR10 / MAIN10_HDR10Plus`；JPEG：`JPEG_PROFILE` |
| `camera_codec_level_t` | AVC L1.0~6.2、HEVC L1~6.2 |
| `camera_rc_mode_t` | `CBR / VBR / AVBR / CVBR / FIXQP / QPMAP` |
| `camera_stream_format_t` | `SINGLE / SPLICE_VER / SPLICE_HOR` |
| `camera_id_t` | `CAMERA_ID_1 … CAMERA_ID_5` / `CAMERA_ID_MAX=5` |
| `camera_channel_id_t` | `MAIN / SUB / THIRD / FOURTH / MAX` |
| `camera_buffer_status_t` | `OK / ERROR` |
| `camera_request_template_t` | `NONE / PREVIEW / VIDEO_RECORD / VIDEO_CALL / QR_SCAN / PICTURE / METADATA / AI / CUSTOM` |
| `camera_ai_performance_t` | `LOW (QVGA) / MEDIUM (VGA) / HIGH (720P) / VERY_HIGH / ULTRA` |
| `torch_mode_status_t` | `NOT_AVAILABLE / AVAILABLE_OFF / AVAILABLE_ON` |

### 9.3 AI 类型位掩码（`camera_ai_type_t`）

| 类别 | 位 | 含义 |
| --- | --- | --- |
| 掩码 | `MAIN_MASK=0xFFFFFFF0 / SUB_MASK=0x0F / DETECT_MASK=0x03FFFFF0 / DETECT_CLASSIFY_MASK=0x03FC0000 / RECOGNIZE_MASK=0xFC000000` | 分组 |
| 检测 | `HUMAN_SHAPE 0x10 / HUMAN_FACE 0x20 / UPPER_BODY 0x40 / GESTURE 0x80 / PET 0x100 / PACKAGE 0x200 / FIRE 0x400 / VEHICLE 0x1000 / NON_MOTOR_VEHICLE 0x2000 / MOTION 0x4000` | 检测主类 |
| 分类 | `CLASSIFY_GESTURE 0x040000 / CLASSIFY_ANTI_SPOOFING_FACE 0x080000` | 从检测派生分类 |
| 识别 | `RECOGNIZE_HUMAN_FACE 0x04000000 / RECOGNIZE_UPPER_BODY 0x08000000` | 特征识别 |
| 子类 | 非机动车 `MOTORCYCLE/BICYCLE/TRICYCLE`；车辆 `BUS/CAR/TRUCK`；手势 `OK/SIX/STOP/THUMB_UP/TWO`；抗欺骗 `LIVING/NON_LIVING`；宠物 `CAT/DOG` | 各检测主类下的 sub type |

### 9.4 关键数据结构

```c
typedef struct { int32_t x, y, width, height; } camera_rect_t;

typedef struct camera_codec_param  { camera_codec_mode_t codec_mode; camera_codec_profile_t profile; camera_codec_level_t level; } camera_codec_config_t;
typedef struct camera_rc_cbr_paramter { uint16_t target_bitrate; } camera_rc_cbr_t;
typedef struct camera_rc_vbr_param    { uint16_t max_bitrate, max_qp, min_qp, change_pos; } camera_rc_vbr_t;
typedef struct camera_rc_avbr_paramter{ uint16_t max_bitrate, change_pos, minstillpercent; } camera_rc_avbr_t;
typedef struct camera_rc_cvbr_paramter{ uint16_t max_bitrate, max_qp, min_qp, change_pos, min_psnr, max_psnr; } camera_rc_cvbr_t;
typedef struct camera_rc_fixqp_parameter { uint16_t qp; } camera_rc_fixqp_t;
typedef struct camera_rc_qp_map_paramter { uint16_t roi_number; uint32_t *quality; camera_rect_t *rect; } camera_rc_qp_map_t;
typedef struct camera_rc_avc_paramter  { camera_rc_mode_t rc_mode; union{ cbr; vbr; avbr; cvbr; fixqp; qpmap; }; } camera_rc_avc_t;
typedef struct camera_rc_hevc_paramter { … 同上 … } camera_rc_hevc_t;
typedef struct camera_rc_mjpeg_paramter{ int quality; } camera_rc_mjpeg_t;
typedef union  camera_rc_config { camera_rc_avc_t avc; camera_rc_hevc_t hevc; camera_rc_mjpeg_t mjpeg; } camera_rc_config_t;

typedef struct camera_gop_config { uint16_t gop_ctrl_mode, gop_len, enable_lt, freq_lt, ltrc; } camera_gop_config_t;

typedef struct camera_channel_config {
    camera_id_t device_id; uint32_t width, height; bool mirror_enabled, flip_enabled;
    uint16_t rotation; uint32_t max_buffers; camera_crop_t crop; camera_scale_t scale; void *reserved[2];
} camera_channel_config_t;

typedef struct camera_stream {
    camera_stream_format_t stream_format;
    camera_channel_config_t *channel_config; int num_channel;
    uint32_t fps; int num_slices;
    pixel_format_t format;
    camera_codec_config_t codec_config;
    camera_rc_config_t    rc_config;
    camera_gop_config_t   gop_config;
    int priv_id; void *puser; void **priv; void *reserved[4];
} camera_stream_t;

typedef struct camera_stream_buffer {
    camera_stream_t *stream; void *data; uint32_t data_size; int status;
    int acquire_fence, release_fence;
} camera_stream_buffer_t;

typedef struct camera_capture_request {
    uint32_t frame_number; camera_request_template_t request_template;
    uint32_t num_output_buffers; camera_stream_buffer_t *output_buffers;
    camera_metadata_t *settings; camera_stream_buffer_t *input_buffer;
} camera_capture_request_t;

typedef struct camera_capture_result {
    uint32_t frame_number; uint64_t timestamp;
    const camera_metadata_t *result;
    camera_stream_buffer_t *output_buffers; const camera_stream_buffer_t *input_buffer;
    uint32_t num_physcam_metadata; const camera_metadata_t **physcam_metadata;
    uint32_t num_output_buffers, partial_result; const char **physcam_ids;
} camera_capture_result_t;

typedef struct camera_ai_object {
    camera_ai_type_t ai_type; camera_rect_t rect;
    union { camera_ai_object_detect_t detect; camera_ai_object_recognize_t recognize; };
} camera_ai_object_t;

typedef struct camera_notify_msg { uint64_t u64Ts; int num_total; camera_ai_object_t *ai_object; } camera_notify_msg_t;

typedef struct camera_ai_input        { camera_ai_type_t ai_type; camera_stream_t *stream; int classify_threshold, iou_threshold; } camera_ai_input_t;
typedef struct camera_ai_tile_overlap { camera_stream_t *stream; int num_cols, num_rows; float overlap_ratio; } camera_ai_tile_overlap_t;

typedef struct camera_info { int orientation; uint32_t device_version; const camera_metadata_t *static_camera_characteristics; uint32_t max_frame_rate, max_width, max_height; camera_rect_t crop; } camera_info_t;
typedef struct camera_stream_configuration { uint32_t num_streams; camera_stream_t **streams; const camera_metadata_t *session_parameters; } camera_stream_configuration_t;
```

### 9.5 `camera_hw_device_t` 设备级 ops（14 项）

| Ops | 原型 | 说明 |
| --- | --- | --- |
| `set_callback_ops` | `int (*)(const camera_hw_device_t *, const camera_callback_ops_t *)` | 注册 `process_capture_result` + `notify` |
| `configure_streams` | `int (*)(const camera_hw_device_t *, camera_stream_configuration_t *)` | 配置输入/输出流 |
| `reconfigure_streams` | `int (*)(const camera_hw_device_t *, camera_stream_configuration_t *)` | 重配（通道增删、编码参数变更） |
| `set_parameters` | `int (*)(const camera_hw_device_t *, const char *name, void *value, unsigned len)` | 键：`AiConfig`, … |
| `get_parameters` | `char* (*)(const camera_hw_device_t *, const char *keys)` | 堆字符串 |
| `add_camera_ai` | `int (*)(const camera_hw_device_t *, camera_ai_type_t)` | 启用 AI 能力 |
| `remove_camera_ai` | `int (*)(const camera_hw_device_t *, camera_ai_type_t)` | 关闭 AI 能力 |
| `enable_camera_ai_tile_overlap` | `int (*)(const camera_hw_device_t *, camera_stream_t *)` | 启用 Tile 重叠模式 |
| `disable_camera_ai_tile_overlap` | `int (*)(const camera_hw_device_t *, camera_stream_t *)` | 关闭 Tile 重叠 |
| `construct_default_request_settings` | `const camera_metadata_t* (*)(const camera_hw_device_t *, int type)` | 按 Template 生成默认 settings |
| `process_capture_request` | `int (*)(const camera_hw_device_t *, camera_capture_request_t *)` | 递交抓帧请求 |
| `dump` | `void (*)(const camera_hw_device_t *, int fd)` | 状态转储 |
| `flush` | `int (*)(const camera_hw_device_t *)` | 清空全部流水 |
| `signal_stream_flush` | `void (*)(const camera_hw_device_t *, uint32_t n, const camera_stream_t * const *streams)` | 特定流立即归还缓冲 |
| `is_reconfiguration_required` | `int (*)(const camera_hw_device_t *, const camera_metadata_t *old, const camera_metadata_t *new)` | 判断是否需重配 |

### 9.6 `camera_module_t` 模块级 ops（7 项）

| Ops | 原型 | 说明 |
| --- | --- | --- |
| `get_number_of_cameras` | `int (*)(void)` | 枚举摄像头数量 |
| `get_camera_info` | `int (*)(int id, camera_info_t *)` | 读取静态参数 |
| `set_camera_info` | `int (*)(int id, camera_info_t *)` | 写入 |
| `set_callbacks` | `int (*)(const camera_module_callbacks_t *)` | 注册模块级异步回调 |
| `set_torch_mode` | `int (*)(const char *camera_id, bool enabled)` | 闪光灯 |
| `notify_device_state_change` | `void (*)(uint64_t deviceState)` | 全局设备状态 |
| `get_camera_device_version` | `int (*)(int camera_id, uint32_t *version)` | 设备版本 |

### 9.7 回调与辅助

```c
typedef struct camera_callback_ops {
    void (*process_capture_result)(const struct camera_callback_ops *, const camera_capture_result_t *);
    void (*notify)(const struct camera_callback_ops *, const camera_notify_msg_t *);  /* AI / 事件 */
} camera_callback_ops_t;

typedef struct camera_module_callbacks {
    void (*camera_device_status_change)(const struct camera_module_callbacks*, int camera_id, int new_status);
    void (*torch_mode_status_change)   (const struct camera_module_callbacks*, const char* camera_id, int new_status);
} camera_module_callbacks_t;

static inline int camera_hw_device_open (const hw_module_t *m, camera_hw_device_t **dev);
static inline int camera_hw_device_close(camera_hw_device_t *dev);
static inline int get_splice_stream_width (camera_stream_t *);
static inline int get_splice_stream_height(camera_stream_t *);
```

### 9.8 错误码（`HARDWARE_DOMAIN / HARDWARE_MODULE_CAMERA`）

| 错误 | 说明 |
| --- | --- |
| `HARDWARE_CAMERA_OK` | 成功 |
| `INVALID_PARAM / NOT_INIT / MEMORY / TIMEOUT / NOT_FOUND / BUSY / IO / NOT_SUPPORTED / INVALID_STATE / OPERATION_FAILED` | 通用错误 |
| `DEVICE_LIMIT` (0x0017) | 设备数超限 |
| `CHANNEL_LIMIT` (0x0018) | 通道数超限 |
| `STREAM_LIMIT` (0x0019) | 码流数超限 |
| `STREAM_CONFIG` (0x001A) | 流配置非法 |
| `AI_NOT_AVAILABLE` (0x001B) | AI 能力不可用 |

### 9.9 JPEG Blob 标识

| 常量 | 值 | 用途 |
| --- | --- | --- |
| `CAMERA_JEPEG_BLOB_ID` | `0x00FF` | 标准 JPEG 数据段 |
| `CAMERA_JPEG_APP_SEGMENTS_BLOB_ID` | `0x0100` | APPn 附加段 |

---

## 10. WiFi HAL（`hardware/include/wifi.h`）

### 10.1 标识

| 项 | 值 |
| --- | --- |
| Module ID | `"wifi"` |
| Interface | `"wifi_hal"` |

### 10.2 数据结构

```c
typedef enum { WIFI_STA_STAT_IDLE, ..._CONNECTING, ..._CONNECTED, ..._ADDRESSED, ..._PSK_FAIL, ..._AP_NOT_FOUND } wifi_hal_status_t;

typedef struct { char ssid[33], bssid[18]; int rssi, frequency; char flags[128]; } wifi_scan_result_t;
typedef struct { char device_name[33], device_address[18]; int rssi, frequency; } wifi_p2p_device_info_t;
typedef struct { char group_ssid[33], group_passphrase[64], group_owner_address[18]; int group_id; } wifi_p2p_group_info_t;

#define WIFI_SCAN_MAX_RESULTS 64

typedef int (*wifi_connect_status_cb_t)(wifi_hal_status_t status);
```

### 10.3 `wifi_hw_device_t` ops

**STA**

| ops | 原型 | 功能 |
| --- | --- | --- |
| `load_config` | `int (*)(wifi_hw_device_t*, const char *cfg_path)` | 加载配置 |
| `sta_scan` | `int (*)(wifi_hw_device_t*)` | 启动扫描 |
| `sta_get_scan_results` | `int (*)(wifi_hw_device_t*, wifi_scan_result_t*, int *n)` | 读扫描结果 |
| `sta_connect` | `int (*)(wifi_hw_device_t*, const char *ssid, const char *pwd, int wait_ms)` | 连接 AP |
| `sta_disconnect` | `int (*)(wifi_hw_device_t*)` | 断开 |
| `sta_status` | `wifi_hal_status_t (*)(wifi_hw_device_t*)` | 状态查询 |
| `set_connect_status_cb` | `int (*)(wifi_hw_device_t*, wifi_connect_status_cb_t)` | 状态回调 |

**AP**

| ops | 原型 |
| --- | --- |
| `ap_start` / `ap_stop` | `int (*)(wifi_hw_device_t*)` |
| `ap_set_config` | `int (*)(wifi_hw_device_t*, const char *ssid, const char *pwd)` |
| `ap_get_config` | `int (*)(wifi_hw_device_t*, char *ssid, char *pwd)` |

**P2P**

| ops | 原型 |
| --- | --- |
| `p2p_discover_peers` / `p2p_stop_discover_peers` | `int (*)(wifi_hw_device_t*)` |
| `p2p_get_discovered_peers` | `int (*)(wifi_hw_device_t*, wifi_p2p_device_info_t*, int *n)` |
| `p2p_connect` / `p2p_disconnect` | `int (*)(wifi_hw_device_t*, const char *addr)` |
| `p2p_create_group` / `p2p_remove_group` | `int (*)(wifi_hw_device_t*)` |
| `p2p_is_group_owner` | `int (*)(wifi_hw_device_t*)` — 1/0 |
| `p2p_get_group_info` | `int (*)(wifi_hw_device_t*, wifi_p2p_group_info_t*)` |

### 10.4 入口

```c
static inline int wifi_hw_device_open (const hw_module_t *module, wifi_hw_device_t **device);
static inline int wifi_hw_device_close(wifi_hw_device_t *device);
int hw_wifi_module_init(void);
```

### 10.5 错误码

`HARDWARE_WIFI_OK` / `..._INVALID_PARAM/..._NOT_INIT/..._MEMORY/..._TIMEOUT/..._NOT_FOUND/..._BUSY/..._IO/..._NOT_SUPPORTED/..._INVALID_STATE/..._OPERATION_FAILED/..._PERMISSION_DENIED/..._INTERFACE_NOT_FOUND/..._INTERFACE_DOWN/..._CONFIG_FAILED/..._NETWORK_UNAVAILABLE`。

---

## 11. Ethernet HAL（`hardware/include/ethernet.h`）

### 11.1 标识

| 项 | 值 |
| --- | --- |
| Module ID | `"ethernet"` |
| Interface | `"ethernet_hal"` |

### 11.2 数据结构

```c
typedef enum { ETHERNET_STATUS_DOWN, ETHERNET_STATUS_UP, ETHERNET_STATUS_UNKNOWN } ethernet_status_t;
```

### 11.3 `ethernet_hal_device_t` ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `get_version` | `const char* (*)(ethernet_hal_device_t*)` | HAL 版本字符串 |
| `get_ip` | `int (*)(ethernet_hal_device_t*, const char *iface, char *ip, size_t len)` | 读 IPv4 |
| `get_ipv6` | `int (*)(ethernet_hal_device_t*, const char *iface, char *ipv6, size_t len)` | 读 IPv6 |
| `set_ip` | `int (*)(ethernet_hal_device_t*, const char *iface, const char *ip)` | 设 IP |
| `get_mac` | `int (*)(ethernet_hal_device_t*, const char *iface, char *mac, size_t len)` | 读 MAC |
| `get_netmask` / `set_netmask` | `int (*)(ethernet_hal_device_t*, const char *iface, char *nm, size_t len)` / set | 掩码 |
| `get_gateway` | `int (*)(ethernet_hal_device_t*, const char *iface, char *gw, size_t len)` | 读网关 |
| `set_default_gateway` | `int (*)(ethernet_hal_device_t*, const char *gw)` | 默认网关 |
| `get_dns` | `int (*)(ethernet_hal_device_t*, int flag, char *dns, size_t len)` | DNS 查询 |
| `get_current_interface` | `int (*)(ethernet_hal_device_t*, char *iface, size_t len)` | 当前网口 |
| `get_status` | `ethernet_status_t (*)(ethernet_hal_device_t*, const char *iface)` | 状态 |
| `set_up` / `set_down` | `int (*)(ethernet_hal_device_t*, const char *iface)` | 开关 |

### 11.4 入口

```c
static inline int ethernet_hal_device_open (const hw_module_t *m, ethernet_hal_device_t **d);
static inline int ethernet_hal_device_close(ethernet_hal_device_t *d);
int hw_ethernet_module_init(void);
```

### 11.5 错误码

同 `HARDWARE_WIFI_*` 对应族（替换前缀为 `HARDWARE_ETHERNET_`）。

---

## 12. Input 基础（`hardware/include/input.h`）

### 12.1 标识

| 项 | 值 |
| --- | --- |
| Module ID | `"input"` |

### 12.2 数据结构

```c
typedef void (*input_hal_event_callback_t)(int dev_id, const struct input_event *ev, void *user);

typedef struct input_device {
    struct hw_device_t common;
    const char *name;
    int fd;
    void *handle;
    int (*start)(struct input_device *dev);
    int (*stop) (struct input_device *dev);
    int (*set_callback)(struct input_device *dev, input_hal_event_callback_t cb, void *user);
} input_device_t;

typedef struct input_module {
    struct hw_module_t common;
    int (*open)(const struct input_module *m, input_device_t **device);
} input_module_t;

static inline int input_get_module(const hw_module_t **m, const char *inst);
```

### 12.3 错误码

`HARDWARE_INPUT_OK` / `..._INVALID_PARAM/..._NOT_INIT/..._MEMORY/..._NOT_FOUND/..._BUSY/..._NOT_SUPPORTED/..._OPERATION_FAILED`。

---

## 13. Input evdev（`hardware/include/input_evdev.h`）

### 13.1 实例名

| Instance | 说明 |
| --- | --- |
| `"evdev"` | 统一入口 |
| `"keyboard"` / `"touch"` / `"mouse"` | 子设备 |

### 13.2 数据结构

```c
enum evdev_state { eEVDEV_STATE_NONE = -1, eEVDEV_STATE_RELEASED, eEVDEV_STATE_PRESSED };

struct evdev_event {
    struct timeval time;
    uint16_t type, code;
    int32_t  value, xpos, ypos;
};
typedef int (*evdev_event_callback_t)(struct evdev_event *event);
```

### 13.3 `evdev_device_t` ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `sync` | `int (*)(evdev_device_t*, struct timeval *timeout)` | 同步事件 |
| `read` | `int (*)(evdev_device_t*, struct evdev_event*, struct timeval *timeout)` | 读事件 |
| `set_event_callback` | `int (*)(evdev_device_t*, evdev_event_callback_t)` | 注册回调 |

### 13.4 启停 API

```c
int hw_evdev_module_init(void);
int evdev_start(void (*on_event)(int dev_id, const struct input_event *ev, void *user), void *user);
int evdev_stop(void);
```

---

## 14. GPIO HAL（`hardware/include/gpio.h`）

### 14.1 标识

| 项 | 值 |
| --- | --- |
| Module ID | `"gpio"` |

### 14.2 `gpio_device_t` ops

| ops | 原型 | 参数 |
| --- | --- | --- |
| `set_direction` | `int (*)(gpio_device_t*, const char *name, uint32_t dir)` | `dir` 0=in/1=out |
| `set_output_level` | `int (*)(gpio_device_t*, const char *name, uint32_t level)` | 0=low/1=high |
| `get_input_level` | `int (*)(gpio_device_t*, const char *name, uint32_t *level)` | 读 |
| `set_drive_strength` | `int (*)(gpio_device_t*, const char *name, uint32_t strength)` | 0-3 |
| `enable_pull` | `int (*)(gpio_device_t*, const char *name, uint32_t enable)` | 0/1 |
| `set_pull_polarity` | `int (*)(gpio_device_t*, const char *name, uint32_t polarity)` | 0=down/1=up |
| `set_slew_rate` | `int (*)(gpio_device_t*, const char *name, uint32_t rate)` | 0=slow/1=fast |

### 14.3 入口

```c
int hw_gpio_module_init(void);
```

---

## 15. ADC HAL（`hardware/include/adc.h`）

### 15.1 标识

Module ID `"adc"`。

### 15.2 ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `read` | `int (*)(adc_device_t*, const char *name, uint8_t *buf, uint32_t bytes)` | 读通道 |

```c
int hw_adc_module_init(void);
```

---

## 16. PWM HAL（`hardware/include/pwm.h`）

### 16.1 标识

Module ID `"pwm"`。

### 16.2 数据结构

```c
typedef struct pwm_info { uint32_t period; uint32_t duty; } pwm_info_t;   /* 纳秒 */
```

### 16.3 ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `config` | `int (*)(pwm_device_t*, const char *name, struct pwm_info*)` | 设置周期与占空比 |
| `enable` | `int (*)(pwm_device_t*, const char *name, uint32_t en)` | 0/1 |

```c
int hw_pwm_module_init(void);
```

---

## 17. LED HAL（`hardware/include/led.h`）

### 17.1 标识

Module ID `"led"`。

### 17.2 ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `set_state` | `int (*)(led_device_t*, const char *name, uint32_t state)` | 0=off/1=on |
| `get_state` | `int (*)(led_device_t*, const char *name, uint32_t *state)` | 读 |
| `set_brightness` | `int (*)(led_device_t*, const char *name, uint32_t v)` | 亮度 |
| `get_brightness` | `int (*)(led_device_t*, const char *name, uint32_t *v)` | 读 |

```c
int hw_led_module_init(void);
```

---

## 18. Motor HAL（`hardware/include/motor.h`）

### 18.1 标识

| 项 | 值 |
| --- | --- |
| Module ID | `"motor"` |
| 设备名 | `"motor0"` / `"motor1"` |

### 18.2 数据结构

```c
typedef enum { eMOTOR_IS_STOP = 0, eMOTOR_IS_RUNNING } motor_status_t;

typedef struct motor_config { int pos, speed_step, steps_one_circle, total_steps, boundary_steps; } motor_config_t;

typedef struct motor_info   { motor_status_t status; int pos, speed_step, speed_angle,
                               steps_one_circle, total_steps, boundary_steps, attach_timer; } motor_info_t;
```

### 18.3 ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `config` | `int (*)(motor_device_t*, const char *name, struct motor_config*)` | 配置 |
| `set_step` | `int (*)(motor_device_t*, const char *name, int value)` | 设定步数 |
| `set_angle` | `int (*)(motor_device_t*, const char *name, int value)` | 设定角度 |
| `set_limit` | `int (*)(motor_device_t*, const char *name, bool enable, int value)` | 限位（值>0 顺时针，<0 逆时针） |
| `stop` | `int (*)(motor_device_t*, const char *name)` | 停止并等待 |
| `get_info` | `int (*)(motor_device_t*, const char *name, struct motor_info*)` | 状态 |

```c
int hw_motor_module_init(void);
```

---

## 19. Watchdog HAL（`hardware/include/wdt.h`）

### 19.1 标识

Module ID `"wdt"`。

### 19.2 ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `start` | `int (*)(wdt_device_t*, const char *name, struct timeval *timeout)` | 启动（默认 4s） |
| `feed` | `int (*)(wdt_device_t*, const char *name)` | 喂狗 |
| `stop` | `int (*)(wdt_device_t*, const char *name)` | 停止 |

```c
int hw_wdt_module_init(void);
```

---

## 20. IR HAL（`hardware/include/ir.h`）

### 20.1 标识

Module ID `"ir"`（含 IRCUT + IRLED 两族接口）。

### 20.2 数据结构

```c
typedef struct { int working_level; /* 0=low active, 1=high active */ } irled_param_t;
```

### 20.3 ops

**IRCUT**

| ops | 原型 | 功能 |
| --- | --- | --- |
| `ircut_init` | `int (*)(ir_device_t*, const char *cfg_path)` | 初始化 |
| `ircut_set` | `int (*)(ir_device_t*, int status)` | 0=day(彩色)/1=night(B&W) |
| `ircut_set_time` | `int (*)(ir_device_t*, int ms)` | 切换时间 |
| `ircut_get_time` | `int (*)(ir_device_t*, int *ms)` | 读 |

**IRLED**

| ops | 原型 | 功能 |
| --- | --- | --- |
| `irled_init` | `int (*)(ir_device_t*, const irled_param_t*)` | 初始化（可 NULL） |
| `irled_get_status` | `int (*)(ir_device_t*, int *status)` | 读 |
| `irled_set_status` | `int (*)(ir_device_t*, int status)` | 0=off/1=on |

```c
int hw_ir_module_init(void);
```

---

## 21. eFuse HAL（`hardware/include/efuse.h`）

### 21.1 标识

Module ID `"efuse"`。

### 21.2 ops

| ops | 原型 | 说明 |
| --- | --- | --- |
| `get_length` | `int (*)(efuse_device_t*, const char *name, const char *option, uint32_t *len)` | `option="global"/"customer"` |
| `get_id` | `int (*)(efuse_device_t*, const char *name, const char *option, char *buf)` | 读芯片 ID |
| `set_id` | `int (*)(efuse_device_t*, const char *name, const char *option, char *buf)` | 仅 `"customer"` 可写 |

```c
int hw_efuse_module_init(void);
```

---

## 22. MMC/SD HAL（`hardware/include/mmc.h`）

### 22.1 标识

| 项 | 值 |
| --- | --- |
| Module ID | `"mmc"` |
| 块设备 | `"/dev/mmcblk0"` (`MMC_DEVICE_NAME_BLK0`) |
| 文件系统 | `"vfat"` (`MMC_DEVICE_MOUNT_FS`)，支持 `exfat` |

### 22.2 ops

| ops | 原型 | 功能 |
| --- | --- | --- |
| `mount` | `int (*)(mmc_device_t*, const char *name, char *path, char *fs)` | 挂载 |
| `unmount` | `int (*)(mmc_device_t*, char *path)` | 卸载 |
| `format` | `int (*)(mmc_device_t*, const char *name, char *fs)` | 格式化 |
| `fsck` | `int (*)(mmc_device_t*, const char *name, char *fs)` | 文件系统检查 |

```c
int hw_mmc_module_init(void);
```

---

## 23. 通用调用骨架（所有 HAL 模块）

```c
#include "hardware.h"
#include "<module>.h"

const struct hw_module_t *module = NULL;
<module>_device_t         *dev   = NULL;

/* 1) 获取模块 */
if (hw_get_module(<MODULE>_HARDWARE_MODULE_ID, &module) != 0) return -1;

/* 2) 打开设备（两种风格：inline helper 或 module->methods->open） */
if (<module>_hw_device_open(module, &dev) != 0) return -1;

/* 3) 调用 ops */
int rc = dev->some_op(dev, arg1, arg2);

/* 4) 关闭 */
dev->common.close(&dev->common);
```

---

## 24. 相关分册

- [hardware-hal.md](hardware-hal.md) — 架构总览与扩展流程
- [framework.md](framework.md) — Framework 如何调用 HAL
- [新增 SoC](../skills/celestia-hardware.md) — 新增 SoC / HAL
- [guide/api-index.md](api-index.md) — 全局 API 索引
- [quick-start/bsp-and-products.md](../quick-start/bsp-and-products.md) — HAL 与 BSP 集成
