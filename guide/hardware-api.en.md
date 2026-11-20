<p align="right"><b>English</b> | <a href="hardware-api.md">中文</a></p>

# Hardware HAL API Reference (Public Queryable Interfaces)

> **Scope of this document**: Hardware HAL public queryable interface reference: per-module IDs, data structures, ops, and error codes.
>
> For HAL contracts and the process for adding modules see [hardware-hal.en.md](hardware-hal.en.md); for layering, contracts, and the runtime model see [Architecture overview](../architecture/overview.en.md); terminology is defined solely in the [Glossary](../glossary.en.md).

---

## 0. Quick reference index

| Category | Module | Module ID | Primary structure | Section |
| --- | --- | --- | --- | --- |
| Platform | SoC Platform | — | Direct-call API | §1 |
| Base contract | hardware.h | — | `hw_module_t`/`hw_device_t` | §2 |
| Display/Graphics | Display | `display` | `display_device_t` | §3 |
|  | OSD | `osd` | `osd_hw_device_t` | §4 |
|  | GPU (2D) | `gpu` | `gpu_device_t` | §5 |
| Vision | ISP | `vision_isp` | `isp_device_t` | §6 |
|  | Motion Detect | `motion_detect` | `motion_detect_device_t` | §7 |
| Multimedia | Audio | `audio_hal` | `audio_hw_device_t` | §8 |
|  | Camera | `camera` | `camera_hw_device_t` | §9 |
| Network | WiFi | `wifi` | `wifi_hw_device_t` | §10 |
|  | Ethernet | `ethernet` | `ethernet_hal_device_t` | §11 |
| Input | Input (base) | `input` | `input_device_t` | §12 |
|  | Input evdev | `input`/`evdev` | `evdev_device_t` | §13 |
| Drivers | GPIO | `gpio` | `gpio_device_t` | §14 |
|  | ADC | `adc` | `adc_device_t` | §15 |
|  | PWM | `pwm` | `pwm_device_t` | §16 |
|  | LED | `led` | `led_device_t` | §17 |
|  | Motor | `motor` | `motor_device_t` | §18 |
|  | WDT | `wdt` | `wdt_device_t` | §19 |
|  | IR (IRCUT/LED) | `ir` | `ir_device_t` | §20 |
|  | eFuse | `efuse` | `efuse_device_t` | §21 |
|  | MMC/SD | `mmc` | `mmc_device_t` | §22 |

> **Conventions**:
> - All `hw_*_module_init()` functions are registered at BSP layer via constructors in the HAL module; Framework typically does not call them directly.
> - All `*_device_t->common.close(&dev->common)` is equivalent to the corresponding `xxx_hw_device_close()` inline helper.
> - Error code domain: `SYSTEM_ERROR_CODE(HARDWARE_DOMAIN, HARDWARE_MODULE_<X>, <CODE>)`; see `system_error.h` for details.

---

## 0.1 HAL common function call contract (shared by all ops)

To avoid repeating the same notes in every module, the following contract **applies by default to all HAL ops in §3–§22 below**. Module sections call out differences **only when they exist**.

### 0.1.1 Return value conventions

| Return type | Meaning |
|---|---|
| `int` | `0` = success; `< 0` = `SYSTEM_ERROR_CODE(HARDWARE_DOMAIN, HARDWARE_MODULE_<X>, code)`; `> 0` is application data (e.g. `ping_ipv4` RTT, `preset_get_count`) |
| `uint32_t` | Actual measured/read value (e.g. level, brightness, sample rate); no error channel; ops may return `0` on invalid parameters—see each module |
| `bool` / `const char *` | Query interfaces; return value is the result (`false`/`NULL` on read failure) |

### 0.1.2 Parameter conventions

- **Input pointers** (e.g. `const <T> *cfg`): owned by the caller; ops read only for the duration of the call and **must not** retain the pointer after return.
- **Output pointers** (e.g. `<T> *out`): allocated by the caller; ops write into them; when `NULL` is allowed, that is stated explicitly.
- **Internally buffered output** (e.g. `void **data` / `char *`): pointer owned by the module; caller **must not** `free()`; content is valid until the **next call to the same op** or `device.close` (each module section notes exceptions).
- **Name parameter** `const char *name`: hardware instance name (e.g. `"gpio_led0"`, `"pwm_bl"`, `"motor0"`), defined by SoC BSP DTS / config.
- **Common errors**: null pointer or uninitialized device → `_INVALID_PARAM` / `_NOT_INIT`.

### 0.1.3 Lifecycle and call order

```
hw_get_module("<id>", &mod)       // reentrant; repeated calls return the same module
  → mod->methods->open(mod, "<inst>", &dev)   // each call returns an independent dev instance
      → dev->ops(...)             // call only between open and close
  → dev->common.close(&dev->common)
```

- `open` and `close` must be **paired**; failure to pair causes resource leaks or low-level handle crashes.
- Different `inst` names map to independent instances; repeated `open` on the same `inst` is module-defined (most return `_BUSY`).

### 0.1.4 Thread safety

| Scenario | Constraint |
|---|---|
| `hw_get_module` | Fully thread-safe; internal locking |
| `module->methods->open` / `close` | **No** guarantee of concurrency safety between open/close; upper layer (usually Framework) serializes |
| ops on the same `dev` | **Single-threaded** by default; lock at caller if multiple threads invoke the same `dev` |
| ops on different `dev` | Thread-safe (independent) |
| HAL callbacks (notify / event) | Fired on HAL internal threads; **do not** block in callbacks or call blocking ops on the same `dev` in the reverse direction |

### 0.1.5 Default error code families

Each module follows the common `HARDWARE_<MODULE>_OK / _INVALID_PARAM / _NOT_INIT / _MEMORY / _TIMEOUT / _NOT_FOUND / _BUSY / _IO / _PERMISSION_DENIED / _NOT_SUPPORTED / _INVALID_STATE / _OPERATION_FAILED` family; module-specific codes are listed in the corresponding section (e.g. §8.9 / §9.8).

### 0.1.6 Module `hw_*_module_init()` functions

- Invoked automatically early in process startup by HAL shared-library constructors (`__attribute__((constructor))`) or BSP link scripts to register `hw_module_t` in the global table.
- Framework / Application **must not call directly**; function prototypes are listed for white-box reference only.

---

## 1. SoC Platform (`hardware/include/soc_platform.h`)

Platform abstraction layer; invoked at boot by `framework_init()` to hide chip differences.

### 1.1 Direct-call API

| Function | Prototype | Purpose |
| --- | --- | --- |
| `soc_platform_init` | `int soc_platform_init(int mode)` | Initialize SoC platform; `mode` 0=normal, 1=debug |
| `soc_platform_set_log_level` | `int soc_platform_set_log_level(int level)` | Set platform log level 1=ERR / 2=WARN / 3=NOTICE / 4=NORMAL / 5=INFO / 6=DEBUG |

### 1.2 Error codes

| Macro | Description |
| --- | --- |
| `SOC_PLATFORM_OK` | Success |
| `SOC_PLATFORM_ERROR_INVALID_PARAM` | Invalid parameter |
| `SOC_PLATFORM_ERROR_NOT_SUPPORTED` | Feature not supported |
| `SOC_PLATFORM_ERROR_PLATFORM_NOT_SUPPORTED` | Current chip platform not supported |
| `SOC_PLATFORM_ERROR_HAL_INIT_FAILED` | HAL initialization failed |
| `SOC_PLATFORM_ERROR_HAL_DEINIT_FAILED` | HAL deinitialization failed |

> The actual low-level implementation is provided by `hardware/soc_platform/hal_soc_platform_api.h` (per SoC project directory, with `hal/` subdivisions `AnyCloud39AV200/`, `AnyCloud39AV100_V2/`, and `Sky39AV130/`); `soc_platform/` also includes `hal_ipu_api.h` and weak-symbol stub `hal_ipu_weak_stub.c` (unimplemented SoC IPU interfaces fall back to the weak stub). See [quick-start/bsp-and-products.en.md](../quick-start/bsp-and-products.en.md).

---

## 2. HAL base contract (`hardware/include/hardware.h`)

All HAL module structures are based on this:

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

### 2.1 Public API

| Function | Prototype | Description |
| --- | --- | --- |
| `hw_get_module` | `int hw_get_module(const char *id, const struct hw_module_t **module)` | Get module by ID |
| `hw_get_module_by_class` | `int hw_get_module_by_class(const char *class_id, const char *inst, const struct hw_module_t **module)` | Class + instance name |
| `hw_register_module` | `int hw_register_module(const struct hw_module_t *module)` | Register module with system (BSP side) |
| `module->methods->open(module, name, &device)` | — | Open device instance |
| `device->close(device)` | — | Close device instance |

### 2.2 Common macros

| Macro | Purpose |
| --- | --- |
| `HARDWARE_MODULE_TAG` | Fixed value for `hw_module_t.tag` |
| `HARDWARE_DEVICE_TAG` | Fixed value for `hw_device_t.tag` |
| `HARDWARE_MODULE_API_VERSION(maj, min)` | Encode module version |
| `HARDWARE_DEVICE_API_VERSION(maj, min)` | Encode device version |
| `HAL_MODULE_INFO_SYM` | Module symbol name (macro built from `module_id`) |
| `TO_HW_DEVICE_T_OPEN(p)` | Safe cast to `hw_device_t**` |

---

## 3. Display HAL (`hardware/include/display.h`)

### 3.1 Identification

| Item | Value |
| --- | --- |
| Module ID | `"display"` (`DISPLAY_HARDWARE_MODULE_ID`) |
| Instance | `"fb"` (`DISPLAY_DEVICE_INSTANCE_FB`), device `/dev/fb0` |

### 3.2 Data structures

```c
typedef struct display_buffer {
    void     *vaddr;   /* 虚拟地址 */
    uint64_t  paddr;   /* 物理地址 */
    uint32_t  size;    /* 字节数 */
} display_buffer_t;
```

### 3.3 `display_device_t` ops

| ops | Prototype | Description |
| --- | --- | --- |
| `get_resolution` | `int (*)(display_device_t*, uint32_t *w, uint32_t *h)` | Query resolution |
| `get_offline_buffer` | `int (*)(display_device_t*, display_buffer_t*)` | Get off-screen buffer (Nova direct write) |
| `get_pixel_format` | `int (*)(display_device_t*, pixel_format_t*)` | Pixel format |
| `get_bpp` | `int (*)(display_device_t*, uint32_t*)` | Bits per pixel |
| `flush` | `int (*)(display_device_t*)` | Flush to screen |

### 3.4 Module entry

```c
int hw_display_module_init(void);
int hw_display_get_module(const struct hw_module_t **module, const char *inst);
```

### 3.5 Error codes (partial)

`HARDWARE_DISPLAY_OK` / `..._INVALID_PARAM` / `..._NOT_INIT` / `..._MEMORY` / `..._IO` / `..._NOT_FOUND` / `..._INVALID_STATE` / `..._OPERATION_FAILED` / `..._NOT_SUPPORTED` / `..._DEVICE_BUSY` / `..._MAP_FAILED` / `..._FLUSH_FAILED`.

---

## 4. OSD HAL (`hardware/include/osd.h`)

### 4.1 Identification and constants

| Item | Value |
| --- | --- |
| Module ID | `"osd"` |
| Interface | `"osd_hw_if"` |
| `OSD_OBJ_COUNT` | 8 (max objects per channel) |
| `OSD_TEXT_LEN` | 100 |
| `OSD_BOX_MAX_NUM` | 50 |

### 4.2 Data structures

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

| ops | Prototype | Purpose |
| --- | --- | --- |
| `init` | `int (*)(const osd_hw_device_t*, osd_attr_t*, const char *font_file)` | Initialize + load font |
| `deinit` | `int (*)(const osd_hw_device_t*)` | Deinitialize |
| `draw_bmp` | `int (*)(const osd_hw_device_t*, int nChn, int nObj, osd_rect_info*, const unsigned char *pData, int iDataSize)` | Draw bitmap |
| `clean_obj` | `int (*)(const osd_hw_device_t*, int nChn, int nObj)` | Clear object; `nObj=-1` clears all |
| `copy_cavas_to_vi` | `int (*)(const osd_hw_device_t*, int nChn)` | Canvas → VI hardware OSD |
| `set_font` | `int (*)(const osd_hw_device_t*, int nChn, osd_font_info*)` | Set font |
| `get_text_wh` | `int (*)(const osd_hw_device_t*, int nChn, const char*, int len, int *w, int *h)` | Compute text width/height |
| `draw_text` | `int (*)(const osd_hw_device_t*, int nChn, int nObj, int x, int y, const char*, int len)` | Draw text |
| `set_attr` | `int (*)(const osd_hw_device_t*, int nChn, osd_chn_info*)` | Set channel attributes |
| `set_enable` | `int (*)(const osd_hw_device_t*, int nChn, int nObj, int enable)` | Enable object |
| `draw_box` | `int (*)(const osd_hw_device_t*, int nChn, pp_box_group_t*)` | Draw detection box group |
| `adjust_box_coordinates` | `int (*)(const osd_hw_device_t*, pp_box_group_t*, int video_w, int video_h)` | Normalized coordinates → pixel coordinates |

### 4.4 Entry / convenience

```c
int osd_hw_device_open(const hw_module_t *module, osd_hw_device_t **device);
int osd_hw_device_close(osd_hw_device_t *device);
int hw_osd_module_init(void);
```

### 4.5 Error codes

`HARDWARE_OSD_OK` / `..._INVALID_PARAM` / `..._NOT_INIT` / `..._MEMORY` / `..._NOT_FOUND` / `..._NOT_SUPPORTED` / `..._OPERATION_FAILED`.

---

## 5. GPU 2D HAL (`hardware/include/gpu.h`)

### 5.1 Identification

| Item | Value |
| --- | --- |
| Module ID | `"gpu"` |
| Device | `"gpu"` |
| `IMAGE_OPACITY_MIN/MAX` | 2 / 253 |

### 5.2 Data structures

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

| ops | Prototype | Purpose |
| --- | --- | --- |
| `image_copy` | `int (*)(gpu_device_t*, image_desc* in, image_desc* out)` | Copy |
| `image_scale` | `int (*)(gpu_device_t*, image_desc* in, image_desc* out)` | Scale |
| `image_transform` | `int (*)(gpu_device_t*, image_desc* in, image_desc* out, transform_t t)` | Flip/rotate |
| `image_blend` | `int (*)(gpu_device_t*, image_desc* fg, image_desc* bg, image_desc* out, blend_config*)` | Blend (`out=NULL` in-place) |
| `image_blend_colorkey` | `int (*)(gpu_device_t*, image_desc* in, image_desc* out, image_colorkey*)` | Color-key blend |
| `image_fill` | `int (*)(gpu_device_t*, image_desc* out, uint32_t color)` | Solid fill |
| `check_input_format` | `bool (*)(system_pixel_format_t)` | Input pixel format support |
| `check_output_format` | `bool (*)(system_pixel_format_t)` | Output pixel format support |
| `supported_min_width` | `int32_t (*)(void)` | Minimum width |
| `supported_min_height` | `int32_t (*)(void)` | Minimum height |

### 5.4 Entry

```c
int hw_gpu_module_init(void);
```

### 5.5 Error codes

`HARDWARE_GPU_OK` / `..._INVALID_PARAM` / `..._NOT_SUPPORTED` / `..._MEMORY` / `..._OPERATION_FAILED`.

---

## 6. ISP HAL (`hardware/include/isp.h`)

### 6.1 Identification

| Item | Value |
| --- | --- |
| Module ID | `"vision_isp"` |
| Interface | `"vision_isp_hw_if"` |
| Dimension constants | `ISP_MD_DIMENSION_H_MAX=64`, `ISP_MD_DIMENSION_V_MAX=36` |

### 6.2 Data structures

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

| ops | Prototype | Purpose |
| --- | --- | --- |
| `init` | `int (*)(isp_device_t*, void *config)` | Initialize |
| `deinit` | `int (*)(isp_device_t*)` | Deinitialize |
| `set_param` | `int (*)(isp_device_t*, int key, long value)` | Set parameter |
| `get_frame` | `int (*)(isp_device_t*, void *frame_out)` | Get frame data |
| `switch_mode` | `int (*)(isp_device_t*, isp_mode_t)` | Day/night mode switch |
| `set_soft_ps_param` | `int (*)(isp_device_t*, const isp_soft_ps_param_t*)` | Soft photosensor parameters |
| `get_soft_ps_level` | `int (*)(isp_device_t*, int pre_status, int *day_night)` | Soft photosensor detection result |

### 6.4 `isp_module_t` ops

```c
int (*open)(const isp_module_t*, isp_device_t**);
int  hw_isp_module_init(void);
```

### 6.5 Direct-call API (for Framework/Driver)

| Function | Prototype |
| --- | --- |
| `isp_switch_mode` | `int isp_switch_mode(int dev_id, isp_mode_t mode)` |
| `isp_set_soft_ps_param` | `int isp_set_soft_ps_param(int dev_id, const isp_soft_ps_param_t *param)` |
| `isp_get_soft_ps_level` | `int isp_get_soft_ps_level(int dev_id, int pre_status, int *day_night)` |

### 6.6 Error codes

`HARDWARE_ISP_OK` / `..._INVALID_PARAM` / `..._NOT_INIT` / `..._MEMORY` / `..._NOT_FOUND` / `..._NOT_SUPPORTED` / `..._OPERATION_FAILED` (domain reuses `HARDWARE_MODULE_VISION`).

---

## 7. Motion Detect HAL (`hardware/include/motion_detect.h`)

### 7.1 Identification

| Item | Value |
| --- | --- |
| Module ID | `"motion_detect"` |
| Interface | `"motion_detect_hw_if"` |

### 7.2 Data structures

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

| ops | Prototype | Purpose |
| --- | --- | --- |
| `init` / `deinit` | `int (*)(motion_detect_device_t*)` | Initialize/deinitialize |
| `set_config` | `int (*)(motion_detect_device_t*, const motion_detect_config_t*)` | Configure |
| `get_config` | `int (*)(motion_detect_device_t*, motion_detect_config_t*)` | Read |
| `set_callback` | `int (*)(motion_detect_device_t*, motion_detect_callback_t, void *user)` | Set callback |
| `set_day_night_callback` | `int (*)(motion_detect_device_t*, motion_detect_day_night_callback_t)` | Register day/night query |
| `handle_isp_md` | `int (*)(motion_detect_device_t*, const void *md_info)` | Receive ISP MD info |
| `mrd_init` / `mrd_destroy` | MRD operation entry | Return/destroy handle |
| `mrd_set_filters` | `int (*)(..., unsigned short flt_big, unsigned short flt_small)` | Large/small filter |
| `mrd_set_motion_region_size` | `int (*)(..., int size_min, int size_max)` | Region size range |
| `mrd_set_mode` | `int (*)(..., unsigned char mode)` | Multi-frame/single-frame |
| `mrd_process` | `int (*)(..., unsigned short md_stat[][64], int len, motion_detect_box_t *boxes)` | Run detection |

### 7.4 Entry

```c
int hw_motion_detect_module_init(void);
/* motion_detect_module_t->open(module, &device) opens an instance */
```

### 7.5 Error codes

`HARDWARE_MOTION_DETECT_OK` and `..._INVALID_PARAM/..._NOT_INIT/..._MEMORY/..._NOT_FOUND/..._NOT_SUPPORTED/..._OPERATION_FAILED` (domain shared with `HARDWARE_MODULE_VISION`).

---

## 8. Audio HAL (`hardware/include/audio.h`)

> The only public header for this module is `hardware/include/audio.h`.
> SoC-side `hal_audio_api.h` / `audio_hw.h` and similar are internal implementation and not public.

### 8.1 Identification and version

| Item | Value |
| --- | --- |
| Module ID | `AUDIO_HARDWARE_MODULE_ID = "audio_hal"` |
| Interface | `AUDIO_HARDWARE_INTERFACE = "audio_hw_if"` |
| Module API version | `AUDIO_MODULE_API_VERSION_CURRENT = HARDWARE_MODULE_API_VERSION(0,1)` |
| Device API version | `AUDIO_DEVICE_API_VERSION_CURRENT = HARDWARE_DEVICE_API_VERSION(0,1)` |
| Module init | `int hw_audio_module_init(void)` |

### 8.2 Enums and constants

| Enum / constant | Values |
| --- | --- |
| `audio_devices_t` | `NONE=0x0` / `OUT_SPEAKER=0x01` / `OUT_LINE=0x02` / `OUT_USB_DEVICE=0x04` / `OUT_BUS=0x05` / `IN_AMIC=0x100` / `IN_DMIC=0x101` / `IN_LINE=0x102` / `IN_BUS0=0x103` / `IN_BUS1=0x104` / `IN_USB_DEVICE=0x105` / `IN_LOOPBACK=0x40000` |
| `audio_mode_t` | `NONE / NORMAL / MUSIC / CALL / MAX` |
| `audio_effect_type_t` | `NONE=0` mask bits: `SOUND_MASK=0xff` / `DETECT_MASK=0xff00` / `SSL_MASK=0xf0000`; common bits: `ASLC / EQ / AEC / NS / RESAMPLE / VAD / CPU_CRY / NPU_CRY / SSL / LSSL / CSSL` |
| `audio_channel_mask_t` | `IN_MONO=0x010000 / IN_STEREO=0x020000 / IN_TRI / IN_QUAD / OUT_MONO=0x01000000 / OUT_STEREO=0x02000000`; `AUDIO_CHANNEL_COUNT_MAX=4` |
| `audio_format_t` | PCM/MP3/AMR/AAC/WMA/ALAW/ULAW/OPUS/FLAC/… and `PCM_16_BIT / PCM_8_BIT` combinations; format masks `MAIN_MASK=0xFF000000`, `SUB_MASK=0x00FFFFFF` |
| `audio_dual_mono_mode_t` | `OFF / LR / LL / RR` |
| `audio_timestretch_stretch_mode_t` | `DEFAULT / VOICE` |
| `audio_timestretch_fallback_mode_t` | `CUT_REPEAT / DEFAULT / MUTE / FAIL` |
| Playback rate range | `SPEED_MIN=0.01f / MAX=20.0f`; `PITCH_MIN=0.25f / MAX=4.0f`; `NORMAL=1.0f` |

### 8.3 Data structures

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

### 8.4 `audio_stream_t` common stream ops (13 items)

| Ops | Prototype | Description |
| --- | --- | --- |
| `get_sample_rate` | `uint32_t (*)(const audio_stream_t *)` | Sample rate Hz |
| `set_sample_rate` | `int (*)(audio_stream_t *, uint32_t)` | Currently reserved; use `set_parameters` |
| `get_buffer_size` | `size_t (*)(const audio_stream_t *)` | Buffer size in bytes, frame-aligned |
| `get_channels` | `audio_channel_mask_t (*)(const audio_stream_t *)` | Channel configuration |
| `get_format` | `audio_format_t (*)(const audio_stream_t *)` | Format |
| `set_format` | `int (*)(audio_stream_t *, audio_format_t)` | Currently reserved; use `set_parameters` |
| `dump` | `int (*)(const audio_stream_t *, char *path, char *name, int enable)` | Stream dump for debug |
| `get_device` | `audio_devices_t (*)(const audio_stream_t *)` | Currently bound device |
| `set_device` | `int (*)(audio_stream_t *, audio_devices_t)` | Currently reserved; use `set_parameters` |
| `set_parameters` | `int (*)(const audio_stream_t *, const char *name, void *value, unsigned len)` | Keys: `Volume / AIParam / Vad / NpuCry / CpuCry / Lssl / AIReboot`, etc. |
| `get_parameters` | `char* (*)(const audio_stream_t *, const char *keys)` | Heap string; caller `free`s |
| `add_audio_effect` | `int (*)(const audio_stream_t *, audio_effect_type_t)` | Add effect |
| `remove_audio_effect` | `int (*)(const audio_stream_t *, audio_effect_type_t)` | Remove effect |

### 8.5 `audio_stream_out_t` output stream ops (10 items beyond `common`)

| Ops | Description |
| --- | --- |
| `set_volume(out, float)` | Hardware volume 0.0~1.0 |
| `write(out, buf, bytes)` | Write PCM; non-blocking if callback bound |
| `set_callback(out, cb, cookie)` | Register non-blocking write/drain completion callback |
| `pause(out) / resume(out)` | Required for offline playback |
| `set_event_callback(out, cb, cookie)` | Events such as codec format change |
| `get_audio_description_mix_level(out, *dB)` | AD secondary stream mix level |
| `set_audio_description_mix_level(out, dB)` | Same (set) |
| `get_playback_rate_parameters(out, *rate)` | Speed/pitch/TimeStretch parameters |
| `set_playback_rate_parameters(out, *rate)` | Same (set) |

### 8.6 `audio_stream_in_t` input stream ops

| Ops | Description |
| --- | --- |
| `set_volume(in, float)` | Capture gain 0.0~1.0 (reserved) |
| `read(in, buf, bytes)` | Read PCM, including post-effect processing |

### 8.7 `audio_hw_device_t` device-level ops (12 items)

| Ops | Description |
| --- | --- |
| `set_callback_ops(dev, *cb)` | Register `process_capture_result` + `notify` |
| `get_supported_devices(dev)` | Returns supported `audio_devices_t` bitmask |
| `get_master_volume(dev, *vol)` | Master volume (Primary HAL) |
| `set_mode(dev, mode)` | `NORMAL/MUSIC/CALL`; mode switch reloads effect chain |
| `set_mic_mute(dev, bool)` / `get_mic_mute(dev, *bool)` | MIC mute control |
| `set_parameters(stream, name, value, len)` / `get_parameters(dev, keys)` | Global key/value |
| `get_input_buffer_size(dev)` | Recommended input buffer size |
| `open_output_stream(dev, handle, devices, *cfg, **out)` | Open output stream |
| `close_output_stream(dev, out)` | Close |
| `open_input_stream(dev, handle, devices, *cfg, **in)` | Open input stream |
| `close_input_stream(dev, in)` | Close |

### 8.8 Callbacks and helpers

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

### 8.9 Error codes (`HARDWARE_DOMAIN / HARDWARE_MODULE_AUDIO`)

| Error | Description |
| --- | --- |
| `HARDWARE_AUDIO_OK` | Success (= `SYSTEM_SUCCESS`) |
| `INVALID_PARAM / NOT_INIT / MEMORY / TIMEOUT / NOT_FOUND / BUSY / IO / PERMISSION / NOT_SUPPORTED / INVALID_STATE / OPERATION_FAILED` | Common errors |
| `STREAM_OPEN_FAILED` (0x0017) | Stream open failed |
| `STREAM_PARAM_INVALID` (0x0018) | Invalid stream parameters |
| `CODEC_UNSUPPORTED` (0x0019) | Codec not supported |
| `DEVICE_BUSY` (0x001A) | Device busy |
| `DEVICE_NOT_AVAILABLE` (0x001B) | Device not available |

---

## 9. Camera HAL (`hardware/include/camera.h`)

> The only public header for this module is `hardware/include/camera.h`.
> SoC-side `hal_camera_api.h` / `camera_hw.h` and similar are internal implementation and not public.

### 9.1 Identification and version

| Item | Value |
| --- | --- |
| Module ID | `CAMERA_HARDWARE_MODULE_ID = "camera"` |
| Interface | `CAMERA_HARDWARE_INTERFACE = "camera_hw_if"` |
| Module API version | `CAMERA_MODULE_API_VERSION_CURRENT = HARDWARE_MODULE_API_VERSION(1,0)` |
| Device API version | `CAMERA_DEVICE_API_VERSION_1_0 = HARDWARE_DEVICE_API_VERSION(1,0)` |
| Module init | `int hw_camera_module_init(void)` |

### 9.2 Core enums

| Enum | Values |
| --- | --- |
| `camera_stream_rotation_t` | `0° / 90° / 180° / 270°` counterclockwise |
| `camera_codec_mode_t` | `AVC / HEVC / MJPEG / YUV / RAW` |
| `camera_codec_profile_t` | AVC: `BASE / MAIN / HIGH / C_BASE`; HEVC: `MAIN / MAIN_STILL / MAIN_INTRA / MAIN10_HDR10 / MAIN10_HDR10Plus`; JPEG: `JPEG_PROFILE` |
| `camera_codec_level_t` | AVC L1.0~6.2, HEVC L1~6.2 |
| `camera_rc_mode_t` | `CBR / VBR / AVBR / CVBR / FIXQP / QPMAP` |
| `camera_stream_format_t` | `SINGLE / SPLICE_VER / SPLICE_HOR` |
| `camera_id_t` | `CAMERA_ID_1 … CAMERA_ID_5` / `CAMERA_ID_MAX=5` |
| `camera_channel_id_t` | `MAIN / SUB / THIRD / FOURTH / MAX` |
| `camera_buffer_status_t` | `OK / ERROR` |
| `camera_request_template_t` | `NONE / PREVIEW / VIDEO_RECORD / VIDEO_CALL / QR_SCAN / PICTURE / METADATA / AI / CUSTOM` |
| `camera_ai_performance_t` | `LOW (QVGA) / MEDIUM (VGA) / HIGH (720P) / VERY_HIGH / ULTRA` |
| `torch_mode_status_t` | `NOT_AVAILABLE / AVAILABLE_OFF / AVAILABLE_ON` |

### 9.3 AI type bit masks (`camera_ai_type_t`)

| Category | Bits | Meaning |
| --- | --- | --- |
| Masks | `MAIN_MASK=0xFFFFFFF0 / SUB_MASK=0x0F / DETECT_MASK=0x03FFFFF0 / DETECT_CLASSIFY_MASK=0x03FC0000 / RECOGNIZE_MASK=0xFC000000` | Grouping |
| Detection | `HUMAN_SHAPE 0x10 / HUMAN_FACE 0x20 / UPPER_BODY 0x40 / GESTURE 0x80 / PET 0x100 / PACKAGE 0x200 / FIRE 0x400 / VEHICLE 0x1000 / NON_MOTOR_VEHICLE 0x2000 / MOTION 0x4000` | Main detection classes |
| Classification | `CLASSIFY_GESTURE 0x040000 / CLASSIFY_ANTI_SPOOFING_FACE 0x080000` | Classification derived from detection |
| Recognition | `RECOGNIZE_HUMAN_FACE 0x04000000 / RECOGNIZE_UPPER_BODY 0x08000000` | Feature recognition |
| Subtypes | Non-motor vehicles `MOTORCYCLE/BICYCLE/TRICYCLE`; vehicles `BUS/CAR/TRUCK`; gestures `OK/SIX/STOP/THUMB_UP/TWO`; anti-spoofing `LIVING/NON_LIVING`; pets `CAT/DOG` | Sub types under each main detection class |

### 9.4 Key data structures

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

### 9.5 `camera_hw_device_t` device-level ops (14 items)

| Ops | Prototype | Description |
| --- | --- | --- |
| `set_callback_ops` | `int (*)(const camera_hw_device_t *, const camera_callback_ops_t *)` | Register `process_capture_result` + `notify` |
| `configure_streams` | `int (*)(const camera_hw_device_t *, camera_stream_configuration_t *)` | Configure input/output streams |
| `reconfigure_streams` | `int (*)(const camera_hw_device_t *, camera_stream_configuration_t *)` | Reconfigure (channel add/remove, encoding parameter changes) |
| `set_parameters` | `int (*)(const camera_hw_device_t *, const char *name, void *value, unsigned len)` | Keys: `AiConfig`, … |
| `get_parameters` | `char* (*)(const camera_hw_device_t *, const char *keys)` | Heap string |
| `add_camera_ai` | `int (*)(const camera_hw_device_t *, camera_ai_type_t)` | Enable AI capability |
| `remove_camera_ai` | `int (*)(const camera_hw_device_t *, camera_ai_type_t)` | Disable AI capability |
| `enable_camera_ai_tile_overlap` | `int (*)(const camera_hw_device_t *, camera_stream_t *)` | Enable tile overlap mode |
| `disable_camera_ai_tile_overlap` | `int (*)(const camera_hw_device_t *, camera_stream_t *)` | Disable tile overlap |
| `construct_default_request_settings` | `const camera_metadata_t* (*)(const camera_hw_device_t *, int type)` | Build default settings by template |
| `process_capture_request` | `int (*)(const camera_hw_device_t *, camera_capture_request_t *)` | Submit capture request |
| `dump` | `void (*)(const camera_hw_device_t *, int fd)` | State dump |
| `flush` | `int (*)(const camera_hw_device_t *)` | Flush entire pipeline |
| `signal_stream_flush` | `void (*)(const camera_hw_device_t *, uint32_t n, const camera_stream_t * const *streams)` | Immediately return buffers for specific streams |
| `is_reconfiguration_required` | `int (*)(const camera_hw_device_t *, const camera_metadata_t *old, const camera_metadata_t *new)` | Determine if reconfiguration is required |

### 9.6 `camera_module_t` module-level ops (7 items)

| Ops | Prototype | Description |
| --- | --- | --- |
| `get_number_of_cameras` | `int (*)(void)` | Enumerate camera count |
| `get_camera_info` | `int (*)(int id, camera_info_t *)` | Read static parameters |
| `set_camera_info` | `int (*)(int id, camera_info_t *)` | Write |
| `set_callbacks` | `int (*)(const camera_module_callbacks_t *)` | Register module-level async callbacks |
| `set_torch_mode` | `int (*)(const char *camera_id, bool enabled)` | Torch/flash |
| `notify_device_state_change` | `void (*)(uint64_t deviceState)` | Global device state |
| `get_camera_device_version` | `int (*)(int camera_id, uint32_t *version)` | Device version |

### 9.7 Callbacks and helpers

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

### 9.8 Error codes (`HARDWARE_DOMAIN / HARDWARE_MODULE_CAMERA`)

| Error | Description |
| --- | --- |
| `HARDWARE_CAMERA_OK` | Success |
| `INVALID_PARAM / NOT_INIT / MEMORY / TIMEOUT / NOT_FOUND / BUSY / IO / NOT_SUPPORTED / INVALID_STATE / OPERATION_FAILED` | Common errors |
| `DEVICE_LIMIT` (0x0017) | Device count exceeded |
| `CHANNEL_LIMIT` (0x0018) | Channel count exceeded |
| `STREAM_LIMIT` (0x0019) | Stream count exceeded |
| `STREAM_CONFIG` (0x001A) | Invalid stream configuration |
| `AI_NOT_AVAILABLE` (0x001B) | AI capability unavailable |

### 9.9 JPEG blob identifiers

| Constant | Value | Purpose |
| --- | --- | --- |
| `CAMERA_JEPEG_BLOB_ID` | `0x00FF` | Standard JPEG data segment |
| `CAMERA_JPEG_APP_SEGMENTS_BLOB_ID` | `0x0100` | APPn extension segments |

---

## 10. WiFi HAL (`hardware/include/wifi.h`)

### 10.1 Identification

| Item | Value |
| --- | --- |
| Module ID | `"wifi"` |
| Interface | `"wifi_hal"` |

### 10.2 Data structures

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

| ops | Prototype | Purpose |
| --- | --- | --- |
| `load_config` | `int (*)(wifi_hw_device_t*, const char *cfg_path)` | Load configuration |
| `sta_scan` | `int (*)(wifi_hw_device_t*)` | Start scan |
| `sta_get_scan_results` | `int (*)(wifi_hw_device_t*, wifi_scan_result_t*, int *n)` | Read scan results |
| `sta_connect` | `int (*)(wifi_hw_device_t*, const char *ssid, const char *pwd, int wait_ms)` | Connect to AP |
| `sta_disconnect` | `int (*)(wifi_hw_device_t*)` | Disconnect |
| `sta_status` | `wifi_hal_status_t (*)(wifi_hw_device_t*)` | Status query |
| `set_connect_status_cb` | `int (*)(wifi_hw_device_t*, wifi_connect_status_cb_t)` | Status callback |

**AP**

| ops | Prototype |
| --- | --- |
| `ap_start` / `ap_stop` | `int (*)(wifi_hw_device_t*)` |
| `ap_set_config` | `int (*)(wifi_hw_device_t*, const char *ssid, const char *pwd)` |
| `ap_get_config` | `int (*)(wifi_hw_device_t*, char *ssid, char *pwd)` |

**P2P**

| ops | Prototype |
| --- | --- |
| `p2p_discover_peers` / `p2p_stop_discover_peers` | `int (*)(wifi_hw_device_t*)` |
| `p2p_get_discovered_peers` | `int (*)(wifi_hw_device_t*, wifi_p2p_device_info_t*, int *n)` |
| `p2p_connect` / `p2p_disconnect` | `int (*)(wifi_hw_device_t*, const char *addr)` |
| `p2p_create_group` / `p2p_remove_group` | `int (*)(wifi_hw_device_t*)` |
| `p2p_is_group_owner` | `int (*)(wifi_hw_device_t*)` — 1/0 |
| `p2p_get_group_info` | `int (*)(wifi_hw_device_t*, wifi_p2p_group_info_t*)` |

### 10.4 Entry

```c
static inline int wifi_hw_device_open (const hw_module_t *module, wifi_hw_device_t **device);
static inline int wifi_hw_device_close(wifi_hw_device_t *device);
int hw_wifi_module_init(void);
```

### 10.5 Error codes

`HARDWARE_WIFI_OK` / `..._INVALID_PARAM/..._NOT_INIT/..._MEMORY/..._TIMEOUT/..._NOT_FOUND/..._BUSY/..._IO/..._NOT_SUPPORTED/..._INVALID_STATE/..._OPERATION_FAILED/..._PERMISSION_DENIED/..._INTERFACE_NOT_FOUND/..._INTERFACE_DOWN/..._CONFIG_FAILED/..._NETWORK_UNAVAILABLE`.

---

## 11. Ethernet HAL (`hardware/include/ethernet.h`)

### 11.1 Identification

| Item | Value |
| --- | --- |
| Module ID | `"ethernet"` |
| Interface | `"ethernet_hal"` |

### 11.2 Data structures

```c
typedef enum { ETHERNET_STATUS_DOWN, ETHERNET_STATUS_UP, ETHERNET_STATUS_UNKNOWN } ethernet_status_t;
```

### 11.3 `ethernet_hal_device_t` ops

| ops | Prototype | Purpose |
| --- | --- | --- |
| `get_version` | `const char* (*)(ethernet_hal_device_t*)` | HAL version string |
| `get_ip` | `int (*)(ethernet_hal_device_t*, const char *iface, char *ip, size_t len)` | Read IPv4 |
| `get_ipv6` | `int (*)(ethernet_hal_device_t*, const char *iface, char *ipv6, size_t len)` | Read IPv6 |
| `set_ip` | `int (*)(ethernet_hal_device_t*, const char *iface, const char *ip)` | Set IP |
| `get_mac` | `int (*)(ethernet_hal_device_t*, const char *iface, char *mac, size_t len)` | Read MAC |
| `get_netmask` / `set_netmask` | `int (*)(ethernet_hal_device_t*, const char *iface, char *nm, size_t len)` / set | Netmask |
| `get_gateway` | `int (*)(ethernet_hal_device_t*, const char *iface, char *gw, size_t len)` | Read gateway |
| `set_default_gateway` | `int (*)(ethernet_hal_device_t*, const char *gw)` | Default gateway |
| `get_dns` | `int (*)(ethernet_hal_device_t*, int flag, char *dns, size_t len)` | DNS query |
| `get_current_interface` | `int (*)(ethernet_hal_device_t*, char *iface, size_t len)` | Current interface |
| `get_status` | `ethernet_status_t (*)(ethernet_hal_device_t*, const char *iface)` | Status |
| `set_up` / `set_down` | `int (*)(ethernet_hal_device_t*, const char *iface)` | Bring up/down |

### 11.4 Entry

```c
static inline int ethernet_hal_device_open (const hw_module_t *m, ethernet_hal_device_t **d);
static inline int ethernet_hal_device_close(ethernet_hal_device_t *d);
int hw_ethernet_module_init(void);
```

### 11.5 Error codes

Same family as `HARDWARE_WIFI_*` (replace prefix with `HARDWARE_ETHERNET_`).

---

## 12. Input base (`hardware/include/input.h`)

### 12.1 Identification

| Item | Value |
| --- | --- |
| Module ID | `"input"` |

### 12.2 Data structures

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

### 12.3 Error codes

`HARDWARE_INPUT_OK` / `..._INVALID_PARAM/..._NOT_INIT/..._MEMORY/..._NOT_FOUND/..._BUSY/..._NOT_SUPPORTED/..._OPERATION_FAILED`.

---

## 13. Input evdev (`hardware/include/input_evdev.h`)

### 13.1 Instance names

| Instance | Description |
| --- | --- |
| `"evdev"` | Unified entry |
| `"keyboard"` / `"touch"` / `"mouse"` | Sub-devices |

### 13.2 Data structures

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

| ops | Prototype | Purpose |
| --- | --- | --- |
| `sync` | `int (*)(evdev_device_t*, struct timeval *timeout)` | Synchronize events |
| `read` | `int (*)(evdev_device_t*, struct evdev_event*, struct timeval *timeout)` | Read event |
| `set_event_callback` | `int (*)(evdev_device_t*, evdev_event_callback_t)` | Register callback |

### 13.4 Start/stop API

```c
int hw_evdev_module_init(void);
int evdev_start(void (*on_event)(int dev_id, const struct input_event *ev, void *user), void *user);
int evdev_stop(void);
```

---

## 14. GPIO HAL (`hardware/include/gpio.h`)

### 14.1 Identification

| Item | Value |
| --- | --- |
| Module ID | `"gpio"` |

### 14.2 `gpio_device_t` ops

| ops | Prototype | Parameters |
| --- | --- | --- |
| `set_direction` | `int (*)(gpio_device_t*, const char *name, uint32_t dir)` | `dir` 0=in/1=out |
| `set_output_level` | `int (*)(gpio_device_t*, const char *name, uint32_t level)` | 0=low/1=high |
| `get_input_level` | `int (*)(gpio_device_t*, const char *name, uint32_t *level)` | Read |
| `set_drive_strength` | `int (*)(gpio_device_t*, const char *name, uint32_t strength)` | 0-3 |
| `enable_pull` | `int (*)(gpio_device_t*, const char *name, uint32_t enable)` | 0/1 |
| `set_pull_polarity` | `int (*)(gpio_device_t*, const char *name, uint32_t polarity)` | 0=down/1=up |
| `set_slew_rate` | `int (*)(gpio_device_t*, const char *name, uint32_t rate)` | 0=slow/1=fast |

### 14.3 Entry

```c
int hw_gpio_module_init(void);
```

---

## 15. ADC HAL (`hardware/include/adc.h`)

### 15.1 Identification

Module ID `"adc"`.

### 15.2 ops

| ops | Prototype | Purpose |
| --- | --- | --- |
| `read` | `int (*)(adc_device_t*, const char *name, uint8_t *buf, uint32_t bytes)` | Read channel |

```c
int hw_adc_module_init(void);
```

---

## 16. PWM HAL (`hardware/include/pwm.h`)

### 16.1 Identification

Module ID `"pwm"`.

### 16.2 Data structures

```c
typedef struct pwm_info { uint32_t period; uint32_t duty; } pwm_info_t;   /* 纳秒 */
```

### 16.3 ops

| ops | Prototype | Purpose |
| --- | --- | --- |
| `config` | `int (*)(pwm_device_t*, const char *name, struct pwm_info*)` | Set period and duty cycle |
| `enable` | `int (*)(pwm_device_t*, const char *name, uint32_t en)` | 0/1 |

```c
int hw_pwm_module_init(void);
```

---

## 17. LED HAL (`hardware/include/led.h`)

### 17.1 Identification

Module ID `"led"`.

### 17.2 ops

| ops | Prototype | Purpose |
| --- | --- | --- |
| `set_state` | `int (*)(led_device_t*, const char *name, uint32_t state)` | 0=off/1=on |
| `get_state` | `int (*)(led_device_t*, const char *name, uint32_t *state)` | Read |
| `set_brightness` | `int (*)(led_device_t*, const char *name, uint32_t v)` | Brightness |
| `get_brightness` | `int (*)(led_device_t*, const char *name, uint32_t *v)` | Read |

```c
int hw_led_module_init(void);
```

---

## 18. Motor HAL (`hardware/include/motor.h`)

### 18.1 Identification

| Item | Value |
| --- | --- |
| Module ID | `"motor"` |
| Device names | `"motor0"` / `"motor1"` |

### 18.2 Data structures

```c
typedef enum { eMOTOR_IS_STOP = 0, eMOTOR_IS_RUNNING } motor_status_t;

typedef struct motor_config { int pos, speed_step, steps_one_circle, total_steps, boundary_steps; } motor_config_t;

typedef struct motor_info   { motor_status_t status; int pos, speed_step, speed_angle,
                               steps_one_circle, total_steps, boundary_steps, attach_timer; } motor_info_t;
```

### 18.3 ops

| ops | Prototype | Purpose |
| --- | --- | --- |
| `config` | `int (*)(motor_device_t*, const char *name, struct motor_config*)` | Configure |
| `set_step` | `int (*)(motor_device_t*, const char *name, int value)` | Set step count |
| `set_angle` | `int (*)(motor_device_t*, const char *name, int value)` | Set angle |
| `set_limit` | `int (*)(motor_device_t*, const char *name, bool enable, int value)` | Limit (value>0 clockwise, <0 counterclockwise) |
| `stop` | `int (*)(motor_device_t*, const char *name)` | Stop and wait |
| `get_info` | `int (*)(motor_device_t*, const char *name, struct motor_info*)` | Status |

```c
int hw_motor_module_init(void);
```

---

## 19. Watchdog HAL (`hardware/include/wdt.h`)

### 19.1 Identification

Module ID `"wdt"`.

### 19.2 ops

| ops | Prototype | Purpose |
| --- | --- | --- |
| `start` | `int (*)(wdt_device_t*, const char *name, struct timeval *timeout)` | Start (default 4s) |
| `feed` | `int (*)(wdt_device_t*, const char *name)` | Feed watchdog |
| `stop` | `int (*)(wdt_device_t*, const char *name)` | Stop |

```c
int hw_wdt_module_init(void);
```

---

## 20. IR HAL (`hardware/include/ir.h`)

### 20.1 Identification

Module ID `"ir"` (IRCUT + IRLED interface families).

### 20.2 Data structures

```c
typedef struct { int working_level; /* 0=low active, 1=high active */ } irled_param_t;
```

### 20.3 ops

**IRCUT**

| ops | Prototype | Purpose |
| --- | --- | --- |
| `ircut_init` | `int (*)(ir_device_t*, const char *cfg_path)` | Initialize |
| `ircut_set` | `int (*)(ir_device_t*, int status)` | 0=day (color)/1=night (B&W) |
| `ircut_set_time` | `int (*)(ir_device_t*, int ms)` | Switch time |
| `ircut_get_time` | `int (*)(ir_device_t*, int *ms)` | Read |

**IRLED**

| ops | Prototype | Purpose |
| --- | --- | --- |
| `irled_init` | `int (*)(ir_device_t*, const irled_param_t*)` | Initialize (may be NULL) |
| `irled_get_status` | `int (*)(ir_device_t*, int *status)` | Read |
| `irled_set_status` | `int (*)(ir_device_t*, int status)` | 0=off/1=on |

```c
int hw_ir_module_init(void);
```

---

## 21. eFuse HAL (`hardware/include/efuse.h`)

### 21.1 Identification

Module ID `"efuse"`.

### 21.2 ops

| ops | Prototype | Description |
| --- | --- | --- |
| `get_length` | `int (*)(efuse_device_t*, const char *name, const char *option, uint32_t *len)` | `option="global"/"customer"` |
| `get_id` | `int (*)(efuse_device_t*, const char *name, const char *option, char *buf)` | Read chip ID |
| `set_id` | `int (*)(efuse_device_t*, const char *name, const char *option, char *buf)` | Writable only for `"customer"` |

```c
int hw_efuse_module_init(void);
```

---

## 22. MMC/SD HAL (`hardware/include/mmc.h`)

### 22.1 Identification

| Item | Value |
| --- | --- |
| Module ID | `"mmc"` |
| Block device | `"/dev/mmcblk0"` (`MMC_DEVICE_NAME_BLK0`) |
| Filesystem | `"vfat"` (`MMC_DEVICE_MOUNT_FS`); `exfat` supported |

### 22.2 ops

| ops | Prototype | Purpose |
| --- | --- | --- |
| `mount` | `int (*)(mmc_device_t*, const char *name, char *path, char *fs)` | Mount |
| `unmount` | `int (*)(mmc_device_t*, char *path)` | Unmount |
| `format` | `int (*)(mmc_device_t*, const char *name, char *fs)` | Format |
| `fsck` | `int (*)(mmc_device_t*, const char *name, char *fs)` | Filesystem check |

```c
int hw_mmc_module_init(void);
```

---

## 23. Common call skeleton (all HAL modules)

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

## 24. Related documents

- [hardware-hal.en.md](hardware-hal.en.md) — Architecture overview and extension workflow
- [framework.en.md](framework.en.md) — How Framework calls HAL
- [Adding a SoC](../skills/celestia-hardware.en.md) — Adding SoC / HAL
- [guide/api-index.en.md](api-index.en.md) — Global API index
- [quick-start/bsp-and-products.en.md](../quick-start/bsp-and-products.en.md) — HAL and BSP integration