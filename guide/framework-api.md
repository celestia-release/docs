<p align="right"><a href="framework-api.en.md">English</a> | <b>中文</b></p>

# Framework API 参考手册（对外可查接口）

> **本文范围**：Framework 对外可查接口参考：逐服务给出枚举、结构、ops 与入口函数签名。
>
> 概念、调用流与 Boot 时机见 [framework.md](framework.md)；分层、契约与运行期模型见 [架构总览](../architecture/overview.md)；术语以 [术语与命名对照表](../glossary.md) 为唯一解释源。

---

## 0. 速查目录

| # | 模块 | 公开头文件 | `_open/close` 入口 | 主 struct |
|---|------|-----------|--------------------|-----------|
| 1 | Multimedia | `framework/services/multimedia/include/multimedia.h` | `multimedia_open/close` | `multimedia_t` |
| 2 | Network    | `framework/services/network/include/network.h`       | `network_open/close`    | `network_t`    |
| 3 | Notification | `framework/services/notification/include/notification.h` | `notification_open/close` | `notification_t` |
| 4 | Device     | `framework/services/device/include/device.h`         | `device_open/close`     | `device_t`     |
| 5 | Storage    | `framework/services/storage/include/storage.h`       | `storage_open/close`    | `storage_t`    |
| 6 | Runtime Bootstrap | `framework/runtime/include/runtime_bootstrap.h` | `runtime_bootstrap_open/close` | `runtime_bootstrap_t` |
| 7 | Console    | `framework/runtime/include/console.h`                | `console_init/deinit`   | `command_pkt` |
| 8 | OTA        | `framework/runtime/include/ota.h` / `otapack.h`      | `ota_init/start/stop`   | `ota_info_t`, `OTA_FW_HEADINFO` |
| 9 | Reboot Info | `framework/runtime/include/reboot_info.h`           | `reboot_info_init`      | `reboot_param_t` |
| 10 | RTSP Gateway | `framework/gateway/rtsp_protocol/include/rtsp_protocol.h` | `rtsp_protocol_open/close` | `rtsp_protocol_t` |
| 11 | Nebula Gateway | `framework/gateway/nebula_protocol/include/nebula_protocol.h` | `nebula_protocol_open/close` | `nebula_protocol_t` |
| 12 | Graphics Bitmap | `framework/ui/graphics/include/bitmap.h` | 直接函数 | — |

> Runtime 汇总入口：`framework/runtime/include/runtime.h` 将 bootstrap/console/ota/otapack/reboot_info 一次性 `#include`。

---

## 0.1 通用使用范式（强烈建议先读）

所有 Framework 服务遵循**统一三段式**：

```c
/* 1) 打开服务，拿到单例句柄与 ops 集合 */
svc_t *svc = NULL;
int ret = svc_open(&svc);
if (ret < 0 || svc == NULL) {
    /* 错误处理：解析 SYSTEM_ERROR_CODE 或直接上报 */
    return ret;
}

/* 2) 通过 ops 指针调用能力 */
ret = svc->some_ops->do_something(arg1, arg2);

/* 3) 关闭（引用计数内部维护；最后一个 close 才真正释放） */
svc_close(svc);
```

**错误返回约定**：

- `0`：成功
- `< 0`：错误（由 `SYSTEM_ERROR_CODE(domain, module, code)` 编码，见 `system_error.h`）
- `> 0`：业务数据（例如字节数、帧数等）

**错误码解析范式**：

```c
if (ret < 0) {
    int domain = (-ret >> 24) & 0x7F;
    int module = (-ret >> 16) & 0xFF;
    int code   = (-ret)        & 0xFFFF;
    LOGE("framework call fail: domain=%d module=0x%02x code=0x%04x", domain, module, code);
}
```

**线程与生命周期**：

| 约束 | 说明 |
|---|---|
| 单例 | `*_open()` 多次调用返回同一句柄；内部引用计数，对应次数的 `*_close()` |
| Ops 回调 | 在 Framework 内部线程触发（如采集线程、定时器线程、网络线程）；**禁止在回调内做阻塞/耗时操作**；**禁止在回调里持有长锁或反调用同一 ops 的阻塞方法** |
| 句柄有效期 | `svc` 与 `svc->xxx` 在 `svc_close` 之前始终有效；关闭后不可复用 |
| 跨线程 | 句柄可被多线程共享；ops 方法自身线程安全（底层实现保证），上层需避免重入回调死锁 |
| 启动次序 | 推荐：`framework_init()` → `runtime_bootstrap_*` → 其它业务服务 `*_open` |

**常见错误与排查**：

| 返回码（低 16 位） | 场景 | 排查方向 |
|---|---|---|
| `0x0001` INVALID_PARAM | 指针/参数非法 | 检查传入结构体与指针非空 |
| `0x0002` NOT_INIT | 未 `*_open()` 或 `ops` 为空 | 检查 open 返回值与次序 |
| `0x0003` MEMORY | 分配失败 | 系统内存不足 / 句柄用尽 |
| `0x0004` TIMEOUT | 超时 | 底层 HAL / 网络响应慢；加重试或调整超时 |
| `0x0008` BUSY | 资源占用 | 先 stop/release 再重入 |
| `0x000A` NOT_SUPPORTED | 能力未编译/机型不支持 | 查对应 SoC 的 HAL 能力 |
| `0x000B` INVALID_STATE | 状态机错误（如未 prepare 就 start） | 检查录像/播放状态机 §1.3 |

---

## 1. Multimedia 服务 — `multimedia.h`

**模块域/模块号**：`FRAMEWORK_DOMAIN(0x01) / FRAMEWORK_MODULE_MULTIMEDIA(0x02)`
**版本宏**：`MULTIMEDIA_VERSION_MAJOR/MINOR/PATCH`、`MULTIMEDIA_VERSION`、`MULTIMEDIA_VERSION_STRING`
**常量**：`MEDIA_MAX_CHN_NUM=3`、`MEDIA_MAX_RECT_BOX_COUNT=10`、`MEDIA_NUM_DATA_PTRS=4`、`MEDIA_AI_RECT_TARGET_MAX=64`

### 1.1 默认配置文件名

```c
#define MULTIMEDIA_CFG_VIDEO_FILE      "fw_video_config.json"
#define MULTIMEDIA_CFG_AUDIO_FILE      "fw_audio_config.ini"
#define MULTIMEDIA_CFG_OSD_FILE        "fw_osd_config.json"
#define MULTIMEDIA_CFG_AI_FILE         "fw_ai_config.ini"
#define MULTIMEDIA_CFG_OSD_FONT_FILE   "ak_font_16.bin"
#define MULTIMEDIA_CFG_OSD_LOGO_FILE   ""
```

### 1.2 错误码（`multimedia_error_t = system_error_t`）

`MULTIMEDIA_OK`、`MULTIMEDIA_ERROR_INVALID_PARAM`、`_NOT_INIT`、`_MEMORY`、`_TIMEOUT`、`_NOT_FOUND`、`_ALREADY_EXISTS`、`_BUSY`、`_IO`、`_PERMISSION_DENIED`、`_NOT_SUPPORTED`、`_INVALID_STATE`、`_OPERATION_FAILED`

### 1.3 枚举

| 枚举 | 值 | 说明 |
|------|----|------|
| `media_stream_state_t` | INIT/READY/PLAYING/PAUSED/STOPPED | 流状态 |
| `media_type_t` | VIDEO/AUDIO/SUBTITLE | 媒体类型 |
| `media_video_codec_t` | H264/MJPEG/H265/MPEG4/NV12(0x100)/RGB24(0x101)/YV12(0x102) | 视频编码 |
| `media_audio_codec_t` | G726/MP3(2)/AAC(4)/PCM(6)/G711A(17)/G711U(18) | 音频编码 |
| `media_ai_target_type_t` | HUMAN_SHAPE(0x1)/HUMAN_FACE(0x2)/PET(0x4)/VEHICLE(0x8)/NON_MOTOR_VEHICLE(0x10)/HUMAN_HAND(0x20)/PACKAGE(0x40)/CRY(0x80)/HUMAN_UPPER(0x100)/FACE_RECO(0x1000)/UPPER_RECO(0x2000)/MOTION(0x4000)/ALL(0xffffffff) | AI 目标位掩码 |
| `media_audio_algorithm_t` | VAD/CRY/AEC/NS/ASLC/EQ/SSL | 音频算法类型 |
| `media_output_flags_t` | CLOUD_STORAGE(1<<0)/LIVE_STREAM(1<<1)/CARD_STORAGE(1<<2)/PREVIEW(1<<3)/TALK(1<<4)/SPEAKER(1<<5) | 输出组合位 |
| `media_recorder_state_t` | IDLE/INITIALIZED/DATA_SOURCE_CONFIGURED/PREPARING/PREPARED/RECORDING/PAUSED/STOPPED/ERROR/RELEASED(99) | 录像状态机 |
| `media_player_state_t` | IDLE/INITIALIZED/PREPARING/PREPARED/STARTED/PAUSED/STOPPED/PLAYBACK_COMPLETE/ERROR/END(99) | 播放状态机 |
| `media_recorder_event_t` | PREPARED/STARTED/STOPPED/PAUSED/RESUMED/ERROR/INFO/MAX_DURATION_REACHED/MAX_FILE_SIZE_REACHED | 录像事件 |
| `media_player_event_t` | PREPARED/STARTED/PAUSED/STOPPED/PLAYBACK_COMPLETE/ERROR/SEEK_COMPLETE/BUFFERING_START/BUFFERING_END/INFO | 播放事件 |
| `media_player_scaling_mode_t` | SCALE_TO_FIT(1)/SCALE_TO_FIT_WITH_CROPPING | 视频缩放 |

### 1.4 数据结构

- 帧信息：`media_frame_info_t`、`media_frame_pixel_t`、`media_frame_pcm_t`
- 编码参数：`media_video_enc_param_t`、`media_audio_enc_param_t`
- 抓图：`media_capture_picture_t`、`media_capture_picture_param_t`、`media_capture_rect_t`、`media_capture_cut_t`
- OSD/绘图：`media_rect_t`、`media_rgba_t`
- AI 配置：`media_ai_mode_config_t`、`media_ai_config_t`
- 视频/音频信息：`media_video_info_t`、`media_audio_info_t`
- 流客户端：`media_stream_client_t`
- 录像：`media_recorder_profile_t`、`media_dynamic_param_t`
- 播放：`media_player_audio_attributes_t`
- 相机/音频：`media_stream_info_t`、`media_audio_src_info_t`、`media_audio_stream_info_t`
- 句柄：`media_recorder_handle_t`、`media_player_handle_t`

### 1.5 回调类型

```c
typedef int (*media_frame_callback_t)(int handle, media_frame_info_t *frame, void *user_data);
typedef int (*media_msg_cb_t)(int handle, int msg_type, char *msg, int msg_len, void *user_data);
typedef int (*media_stream_callback_t)(media_recorder_handle_t, int stream_type, const void *frame_data, size_t frame_size, void *user_data);
typedef void (*media_recorder_event_callback_t)(media_recorder_handle_t, int event, int what, int extra, void *user_data);
typedef void (*media_player_event_callback_t)(media_player_handle_t, int event, int what, int extra, void *user_data);
typedef void (*media_player_video_show_callback_t)(int player_handle, media_frame_pixel_t *frame, void *user);
```

### 1.6 Ops 结构

- `recorder_ops_t`：生命周期（`create/destroy/reset/set_profile/get_profile/prepare/prepare_async/start/stop/pause/resume/is_paused`）、数据源（`set_output_file/set_output_stream/set_stream_callback`）、编码参数（`get_video_enc_param/get_audio_enc_param`）、音量（`set_volume/get_volume`）、动态参数（`set/get_dynamic_param`）、录像限制（`set_max_duration/set_max_file_size/set_orientation_hint`）、状态（`get_state/is_recording/get_last_error`）、事件（`set_event_callback`）
- `player_ops_t`：生命周期、数据源（`set_input_file/set_input_stream/set_realtime_source`）、播控（`prepare/prepare_async/start/stop/pause/resume/is_paused/set_playback_speed/get_playback_speed`）、音频（`set_volume/get_volume/set_mute/get_mute/set_audio_attributes`）、位置（`seek/get_position/get_duration`）、显示（`set_window/get_video_width/get_video_height/set_video_scaling_mode/set_video_show_callback`）、状态、事件
- `camera_ops_t`：流配置（`stream_get_configs/stream_get_id/stream_index_by_name/stream_index_by_role/stream_get_channel_size/stream_enable`）、流客户端（`stream_reg_client/stream_get_frame/stream_unreg_client/stream_add_callback/stream_force_i_frame`）、编码参数（`stream_set_enc_param/stream_get_enc_param`）、抓图（`capture_picture`）、特效（`get_effect/set_effect`）、AI 场景（`set_ai_scene/get_ai_scene/load_config/set_ai_enabled/ai_get_config/ai_set_config`）
- `audio_ops_t`：音频源枚举（`audio_src_enum`）、流管理（`stream_get_configs/load_config/stream_enable`）、流客户端（`stream_reg_client/stream_get_frame/stream_unreg_client`）、编码参数、Talk/Playback（`talk_start/talk_send_data/talk_stop/play_notice/play_notice2`）、音量（`out_set_volume/out_get_volume`）、算法（`set_audio_algorithm_enable/set_audio_algorithm_param/get_max_amplitude`、legacy VAD/CRY/Echo 参数）
- `osd_ops_t`：`set_time_osd/set_text_osd`

### 1.7 统一实例

```c
typedef struct {
    recorder_ops_t *recorder;
    player_ops_t   *player;
    camera_ops_t   *camera;
    audio_ops_t    *audio;
    osd_ops_t      *osd;
    void          *reserved[8];
} multimedia_t;
```

### 1.8 函数清单

```c
int multimedia_open(multimedia_t **multimedia);
int multimedia_close(multimedia_t *multimedia);
```

> 严格以 `framework/services/multimedia/include/multimedia.h` 为准。以下符号均为**内部实现**，不对外，不得在应用层引用：
> - `multimedia_is_camera_manager_initialized`
> - `multimedia_release_camera_manager`
> - `multimedia_teardown_osd_for_camera_close`
>
> 判断服务状态请直接使用 `multimedia_open()` 返回码与自管 `multimedia_t *` 指针（非空即已打开）；Framework 各服务**不存在** `<svc>_service_is_open()` 这类查询接口。

### 1.9 入口函数详述

#### 1.9.1 `multimedia_open`

```c
int multimedia_open(multimedia_t **multimedia);
```

- **功能**：打开多媒体服务，首次调用会完成全部子模块（recorder/player/camera/audio/osd）初始化，后续调用仅累加引用计数。
- **参数**：
  - `multimedia` [out]：用于接收服务单例指针，由框架持有，应用**不得**`free`。
- **返回**：`0` 成功；`< 0` 为 `MULTIMEDIA_ERROR_*`。
- **常见错误**：`_INVALID_PARAM`（指针为空）、`_MEMORY`、`_OPERATION_FAILED`（HAL/配置加载失败）。
- **线程**：任意线程；框架内部以互斥锁保护，重入安全。
- **前置**：`framework_init()` 已调用（SoC/log/HAL 初始化完成）。
- **副作用**：加载 `MULTIMEDIA_CFG_*` 配置文件；启动音视频采集与 AI 子系统。

#### 1.9.2 `multimedia_close`

```c
int multimedia_close(multimedia_t *multimedia);
```

- **功能**：减少引用计数；降为 0 时真正释放资源（停流、释放 HAL、关闭回调）。
- **参数**：`multimedia` [in]：`multimedia_open()` 返回的句柄。
- **返回**：`0` 成功；`< 0` 错误。
- **线程**：任意线程。
- **注意**：调用后**禁止**继续使用 `multimedia->xxx->...`。

### 1.10 `recorder_ops_t` 详细函数说明

> 状态机参考 §1.3 `media_recorder_state_t`。所有函数参数中的 `handle` 均为 `create()` 返回的录像器句柄；`0` 成功，`< 0` 返回 `MULTIMEDIA_ERROR_*`。

| 函数 | 签名 | 功能 | 关键参数 | 错误 / 状态 |
|---|---|---|---|---|
| `create` | `int create(media_recorder_handle_t *h)` | 创建录像器实例 | `h` [out] | `_MEMORY`；→ `IDLE` |
| `destroy` | `int destroy(h)` | 销毁实例并释放所有资源 | — | 幂等 |
| `reset` | `int reset(h)` | 重置到 `IDLE` 状态（保留实例） | — | `_INVALID_STATE` 从 `ERROR` 也可 reset |
| `set_profile` | `int set_profile(h, const media_recorder_profile_t *p)` | 设置录像参数（分辨率/码率/编码/输出类型） | `p` [in] | 仅 `IDLE/INITIALIZED` 有效 |
| `get_profile` | `int get_profile(h, media_recorder_profile_t *p)` | 读取当前生效参数 | `p` [out] | — |
| `prepare` | `int prepare(h, const media_recorder_profile_t *p)` | **同步**准备编码器/采集/封装（可省略 `p` 使用前次 `set_profile`） | `p` [in, nullable] | `_INVALID_STATE`；→ `PREPARED` |
| `prepare_async` | `int prepare_async(h, const media_recorder_profile_t *p)` | **异步**准备，完成时触发 `PREPARED` 事件 | `p` [in, nullable] | → `PREPARING` → `PREPARED` |
| `start` | `int start(h)` | 开始录像 | — | 需 `PREPARED`；→ `RECORDING` |
| `stop` | `int stop(h)` | 停止录像，关闭文件/码流 | — | 需 `RECORDING/PAUSED`；→ `STOPPED` |
| `pause` | `int pause(h)` | 暂停录像（不关闭文件） | — | 需 `RECORDING`；→ `PAUSED` |
| `resume` | `int resume(h)` | 从暂停恢复录像 | — | 需 `PAUSED`；→ `RECORDING` |
| `is_paused` | `int is_paused(h, int *paused)` | 查询暂停状态 | `paused` [out] 0/1 | — |
| `set_output_file` | `int set_output_file(h, const char *file_path)` | 设置输出文件或目录（目录时自动生成 `YYYY-MM-DD/YYYYMMDD.HHMMSS.XXX.ext`） | `file_path` [in] | `_IO`；仅 `INITIALIZED/DATA_SOURCE_CONFIGURED` |
| `set_output_stream` | `int set_output_stream(h, const char *url, const char *protocol)` | 设置推流 URL 与协议（如 `"rtsp"`, `"rtmp"`） | `url`, `protocol` [in] | `_INVALID_PARAM` |
| `set_stream_callback` | `int set_stream_callback(h, int stream_type, media_stream_callback_t cb, void *user)` | 注册编码后帧回调（H264/AAC 等已封装帧） | `stream_type` 取 `media_type_t`；`cb`/`user` | 回调在编码线程触发 |
| `get_video_enc_param` | `int get_video_enc_param(h, media_video_enc_param_t *p)` | 获取实际视频编码参数（由策略选择） | `p` [out] | — |
| `get_audio_enc_param` | `int get_audio_enc_param(h, media_audio_enc_param_t *p)` | 获取实际音频编码参数 | `p` [out] | — |
| `set_volume` | `int set_volume(h, int volume)` | 录音输入音量（0–100），非 HAL 输出音量 | `volume` 0–100 | `_INVALID_PARAM` |
| `get_volume` | `int get_volume(h, int *volume)` | 获取录音输入音量 | `volume` [out] | — |
| `set_dynamic_param` | `int set_dynamic_param(h, const media_dynamic_param_t *p)` | 运行时调整码率/关键帧等 | `p` [in] | 运行中可调 |
| `get_dynamic_param` | `int get_dynamic_param(h, media_dynamic_param_t *p)` | 读取动态参数快照 | `p` [out] | — |
| `set_max_duration` | `int set_max_duration(h, long long max_duration_ms)` | 单文件最大时长，超出触发 `MAX_DURATION_REACHED` 事件 | 毫秒；0 = 无限 | — |
| `set_max_file_size` | `int set_max_file_size(h, long long bytes)` | 单文件最大字节数，超出触发 `MAX_FILE_SIZE_REACHED` 事件 | 字节；0 = 无限 | — |
| `set_orientation_hint` | `int set_orientation_hint(h, int degrees)` | 写入 mp4 旋转提示（0/90/180/270） | 度 | `_INVALID_PARAM` |
| `get_state` | `int get_state(h, media_recorder_state_t *s)` | 读取当前状态机 | `s` [out] | 任意状态均可 |
| `is_recording` | `int is_recording(h, int *recording)` | 是否处于 `RECORDING` | `recording` [out] 0/1 | — |
| `get_last_error` | `int get_last_error(h, multimedia_error_t *code)` | 读取最近一次内部错误 | `code` [out] | — |
| `set_event_callback` | `int set_event_callback(h, media_recorder_event_callback_t cb, void *user)` | 注册状态/错误/限额事件回调 | `cb`/`user` | 回调在内部线程触发；**禁止阻塞** |

### 1.11 `player_ops_t` 详细函数说明

> 状态机参考 §1.3 `media_player_state_t`。

| 函数 | 签名 | 功能 | 关键参数 | 错误 / 状态 |
|---|---|---|---|---|
| `create` | `int create(media_player_handle_t *h)` | 创建播放器实例 | `h` [out] | `_MEMORY`；→ `IDLE` |
| `destroy` | `int destroy(h)` | 销毁播放器 | — | 幂等 |
| `reset` | `int reset(h)` | 重置到 `IDLE` | — | — |
| `set_input_file` | `int set_input_file(h, const char *file_path)` | 设置本地媒体源 | `file_path` [in] | `_IO` |
| `set_input_stream` | `int set_input_stream(h, const char *url, const char *protocol)` | 设置网络流源（RTSP/HTTP 等） | `url`, `protocol` [in] | `_INVALID_PARAM` |
| `set_realtime_source` | `int set_realtime_source(h, int camera_id, int channel, int audio_source_id)` | 将相机/音频采集作为输入源（实时对讲、预览） | `camera_id`, `channel`, `audio_source_id` | `_NOT_FOUND` |
| `prepare` | `int prepare(h)` | **同步**准备解码/渲染 | — | → `PREPARED` |
| `prepare_async` | `int prepare_async(h)` | **异步**准备，完成事件 `PREPARED` | — | → `PREPARING` → `PREPARED` |
| `start` | `int start(h)` | 开始播放 | — | 需 `PREPARED/PAUSED`；→ `STARTED` |
| `stop` | `int stop(h)` | 停止播放 | — | → `STOPPED` |
| `pause` | `int pause(h)` | 暂停 | — | → `PAUSED` |
| `resume` | `int resume(h)` | 恢复播放 | — | → `STARTED` |
| `is_paused` | `int is_paused(h, int *paused)` | 查询暂停状态 | `paused` [out] | — |
| `set_playback_speed` | `int set_playback_speed(h, float speed)` | 设置倍速（0.5/1.0/2.0 等） | `speed` > 0 | `_NOT_SUPPORTED` 若解码器不支持 |
| `get_playback_speed` | `int get_playback_speed(h, float *speed)` | 获取当前倍速 | `speed` [out] | — |
| `set_volume` | `int set_volume(h, int volume)` | 播放器输出音量（0–100，软件混音） | 0–100 | — |
| `get_volume` | `int get_volume(h, int *volume)` | 获取音量 | `volume` [out] | — |
| `set_mute` | `int set_mute(h, int mute)` | 静音开关 | 0/1 | — |
| `get_mute` | `int get_mute(h, int *mute)` | 查询静音 | `mute` [out] | — |
| `set_audio_attributes` | `int set_audio_attributes(h, const media_player_audio_attributes_t *a)` | 设置音频属性（通道/采样率等） | `a` [in] | 仅 `IDLE/INITIALIZED` |
| `seek` | `int seek(h, long long position_ms)` | 定位到指定播放位置 | ms | 触发 `SEEK_COMPLETE` 事件 |
| `get_position` | `int get_position(h, long long *position_ms)` | 读取当前播放位置 | [out] ms | — |
| `get_duration` | `int get_duration(h, long long *duration_ms)` | 读取总时长（流媒体可能为 `-1`） | [out] ms | — |
| `set_window` | `int set_window(h, int window_handle)` | 绑定渲染窗口（FB/overlay handle） | `window_handle` | `_NOT_SUPPORTED` 无显示场景 |
| `get_video_width` | `int get_video_width(h, int *width)` | 读取视频宽度 | [out] | 需 `PREPARED` |
| `get_video_height` | `int get_video_height(h, int *height)` | 读取视频高度 | [out] | 需 `PREPARED` |
| `set_video_scaling_mode` | `int set_video_scaling_mode(h, media_player_scaling_mode_t mode)` | 设置缩放模式（等比/裁剪） | `mode` | — |
| `set_video_show_callback` | `int set_video_show_callback(h, media_player_video_show_callback_t cb, void *user)` | 注册视频帧渲染回调（外部自绘） | `cb`/`user` | 回调在渲染线程；**禁止阻塞** |
| `get_state` | `int get_state(h, media_player_state_t *s)` | 读取状态机 | [out] | — |
| `is_playing` | `int is_playing(h, int *playing)` | 是否播放中 | [out] 0/1 | — |
| `get_last_error` | `int get_last_error(h, multimedia_error_t *code)` | 读取最近一次错误 | [out] | — |
| `set_event_callback` | `int set_event_callback(h, media_player_event_callback_t cb, void *user)` | 事件回调（PREPARED/STARTED/SEEK_COMPLETE/BUFFERING_*/ERROR…） | `cb`/`user` | 回调在内部线程 |

### 1.12 `camera_ops_t` 详细函数说明

> 用于**旁路**直接访问采集流，用于 AI、预览、二次编码等高级场景。常规录像应优先使用 `recorder_ops_t`。

| 函数 | 签名 | 功能 | 关键参数 / 返回 | 备注 |
|---|---|---|---|---|
| `stream_get_configs` | `int stream_get_configs(media_stream_info_t **streams, int *num)` | 获取所有码流元信息（来源 `fw_video_config.json`） | `streams` [out, 内部缓冲不可 free]、`num` [out] | 只读，可随时调用 |
| `stream_get_id` | `int stream_get_id(int camera_id, int channel)` | 由 (camera, channel) 反查 stream_id | 返回 `stream_id`，`< 0` 未找到 | — |
| `stream_index_by_name` | `int stream_index_by_name(const char *name, int *idx)` | 按 `stream_name` 精确匹配 | `idx` [out] | `_NOT_FOUND` |
| `stream_index_by_role` | `int stream_index_by_role(const char *role, int *idx)` | 按角色（`main/preview/ai/qr`）匹配 | `idx` [out] | 自动 fallback name/type |
| `stream_get_channel_size` | `int stream_get_channel_size(int stream, int *w, int *h)` | 获取通道宽/高 | `w`/`h` [out] | 来自 JSON `channel_config` |
| `stream_enable` | `int stream_enable(int stream, int enable)` | 启停某码流 | `enable` 0/1 | 启停会广播给所有 client |
| `stream_reg_client` | `int stream_reg_client(int stream, int *client_handle, media_frame_callback_t cb, void *user)` | 注册流客户端（分发帧） | `client_handle` [out]；`cb` 可为 `NULL` 时走 pull 模式 | 与 `stream_get_frame` 配对 |
| `stream_get_frame` | `int stream_get_frame(int client_handle, media_frame_info_t *frame)` | **pull 模式**主动拉取一帧 | `frame` [out]，引用内部缓冲 | 调用方不得 `free(frame->data[*])` |
| `stream_unreg_client` | `int stream_unreg_client(int client_handle)` | 注销客户端 | — | 幂等 |
| `stream_add_callback` | `int stream_add_callback(int stream, int *client_handle, media_frame_callback_t cb, void *user)` | 在已存在的流上追加一个回调 client | `client_handle` [out] | 等价于简化版 `stream_reg_client` |
| `stream_force_i_frame` | `int stream_force_i_frame(int stream)` | 强制生成一个 I 帧（用于切入观看） | — | 仅对 H264/H265 有意义 |
| `stream_set_enc_param` | `int stream_set_enc_param(int stream, media_video_enc_param_t *p)` | 设置视频编码参数 | `p` [in] | 运行中可生效 |
| `stream_get_enc_param` | `int stream_get_enc_param(int stream, media_video_enc_param_t *p)` | 读取视频编码参数 | `p` [out] | — |
| `capture_picture` | `int capture_picture(int stream, const media_capture_picture_param_t *param, void **data, size_t *size)` | 同步抓图 | `param` [in]、`data` [out]（内部缓冲）、`size` [out] | 线程安全；内部缓冲由后续 `capture_picture` 复用，使用完毕请立即拷贝 |
| `get_effect` | `int get_effect(int camera, int type, int *value)` | 读取图像特效（亮度/对比度/饱和度…） | `type`：内部定义；`value` [out] | — |
| `set_effect` | `int set_effect(int camera, int type, int value)` | 设置图像特效 | 同上 | 实时生效 |
| `set_ai_scene` | `int set_ai_scene(int scene_id)` | 切换 AI 场景（多人脸/夜视等） | `scene_id` | 影响后续 AI 检测 |
| `get_ai_scene` | `int get_ai_scene(void)` | 读取当前 AI 场景 | 返回 `scene_id` | — |
| `load_config` | `int load_config(const char *video_cfg_path, const char *ai_cfg_path)` | 热加载视频/AI 配置文件 | 两个路径，可任一为 `NULL` | `_IO` |
| `set_ai_enabled` | `void set_ai_enabled(int enable)` | 全局 AI 使能开关 | `enable` 0/1 | 无返回 |
| `ai_get_config` | `int ai_get_config(media_ai_config_t *out, int *mode_count)` | 读取 AI 配置快照 | `out`、`mode_count` [out] | — |
| `ai_set_config` | `int ai_set_config(media_ai_config_t *config)` | 下发 AI 配置 | `config` [in] | 运行中可调 |

### 1.13 `audio_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `audio_src_enum` | `int audio_src_enum(media_audio_src_info_t **srcs, int *num)` | 枚举音频源（麦克风/I2S 等） | 内部缓冲，不可 `free` |
| `stream_get_configs` | `int stream_get_configs(media_audio_stream_info_t **streams, int *num)` | 枚举音频流 | 同上 |
| `load_config` | `int load_config(const char *audio_cfg_path)` | 热加载音频配置 | `_IO` |
| `stream_enable` | `int stream_enable(int stream, int enable)` | 启/停音频流 | — |
| `stream_reg_client` | `int stream_reg_client(int stream, int *client_handle, media_frame_callback_t cb, void *user)` | 注册音频流客户端 | 回调可为 `NULL` 走 pull 模式 |
| `stream_get_frame` | `int stream_get_frame(int client_handle, media_frame_info_t *frame)` | pull 一帧 PCM/压缩音频 | 帧缓冲由内部管理 |
| `stream_unreg_client` | `int stream_unreg_client(int client_handle)` | 注销 | 幂等 |
| `stream_set_enc_param` | `int stream_set_enc_param(int stream, media_audio_enc_param_t *p)` | 设置音频编码参数 | — |
| `stream_get_enc_param` | `int stream_get_enc_param(int stream, media_audio_enc_param_t *p)` | 读取音频编码参数 | — |
| `talk_start` | `int talk_start(int dev)` | **启动对讲（同时开启 in/out）**；AEC 前置条件 | `dev` 默认 `0`；匹配输入输出采样率 |
| `talk_send_data` | `int talk_send_data(int dev, media_frame_info_t *frame)` | 向远端发送 PCM/编码音频帧 | 调用频率由帧时长决定 |
| `talk_stop` | `int talk_stop(int dev)` | 结束对讲 | — |
| `play_notice` | `int play_notice(int dev, const char *file_path)` | 异步播放提示音（pcm/wav/mp3） | 非阻塞 |
| `play_notice2` | `int play_notice2(int dev, const char *file_path, int wait_time_ms, int volume)` | 同上，附加最大等待与独立音量 | `wait_time_ms < 0` 阻塞至完成 |
| `out_set_volume` | `int out_set_volume(int dev, int volume)` | **硬件输出音量**（HAL 层） | 0–100 |
| `out_get_volume` | `int out_get_volume(int dev, int *volume)` | 读取硬件输出音量 | — |
| `set_audio_algorithm_enable` | `int set_audio_algorithm_enable(int dev, media_audio_algorithm_t algorithm, int enable)` | 开关算法（VAD/CRY/**AEC**/NS/ASLC/EQ/SSL） | AEC 仅 talk 模式有效 |
| `set_audio_algorithm_param` | `int set_audio_algorithm_param(int dev, media_audio_algorithm_t alg, const char *name, const void *value, size_t size)` | 设置算法参数（blob） | `name` 由算法定义 |
| `get_max_amplitude` | `int get_max_amplitude(int dev, int *amplitude)` | 读取最近一次采样峰值（VU meter） | `amplitude` [out] |

> 以下成员为**历史遗留**，保留仅为二进制兼容，新代码一律使用 `set_audio_algorithm_enable/_param` 配合 `media_audio_algorithm_t`，本文档不再详述：
> `set_vad_param` / `get_vad_param` / `set_cry_param` / `get_cry_param` / `set_echo_param` / `get_echo_param`。

### 1.14 `osd_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `set_time_osd` | `int set_time_osd(int camera, int channel, int enable)` | 启/停时间戳叠加 | 位置/字体来自 `fw_osd_config.json` |
| `set_text_osd` | `int set_text_osd(int camera, int channel, const char *text)` | 设置/更新通道文本 | `text` 不得超过 OSD 最大字符数，UTF-8 |

---

## 2. Network 服务 — `network.h`

**模块域/号**：`FRAMEWORK_DOMAIN / FRAMEWORK_MODULE_NETWORK(0x00)`
**版本宏**：`NETWORK_VERSION_*`
**错误码前缀**：`NETWORK_OK/NETWORK_ERROR_*`（含 `_CONNECTION_FAILED/_DEVICE_NOT_AVAILABLE/_NETWORK_UNREACHABLE/_INTERFACE_NOT_FOUND/_CONFIG_ERROR/_NOT_IMPLEMENTED`）

### 2.1 枚举

- `network_wifi_state_e`：DISABLED/DISABLING/ENABLED/ENABLING/UNKNOWN
- `network_wifi_station_state_e`：DISCONNECTED/CONNECTING/CONNECTED/DISCONNECTING
- `network_wifi_ap_state_e`：DISABLED/ENABLING/ENABLED/DISABLING
- `network_wifi_p2p_state_e`：DISABLED/ENABLING/ENABLED/DISABLING/DISCOVERING/CONNECTING/CONNECTED
- `network_wifi_status_e`：IDLE/CONNECTED/CONNECTING/ADDRESSED/PSK_FAIL/AP_NOT_FOUND
- `network_wifi_event_type_e`：SCAN_RESULTS_AVAILABLE/STATE_CHANGED/CONNECTION_CHANGED/LINK_SPEED_CHANGED/RSSI_CHANGED/NETWORK_CONNECTED/NETWORK_DISCONNECTED/NETWORK_CONNECTION_FAILED
- `network_rtsp_param_type_e`：IPV4/IPV6/MAX
- `network_rtsp_audio_type_e`：G726/AAC(4)/PCM_ALAW(17)/PCM_ULAW(18)
- `network_rtsp_video_type_e`：H264/MJPEG/H265/MAX

### 2.2 结构

- 配置：`network_wifi_station_config_t{ssid[33], passphrase[64], security_type}`、`network_wifi_ap_config_t{ssid, passphrase, security_type, channel, max_clients}`
- 扫描结果：`network_wifi_scan_result_t{ssid, bssid[18], frequency, rssi, flags}`
- AP 客户端：`network_wifi_ap_client_t{mac_address[18], ip_address[16], rssi}`
- 事件：`network_wifi_event_t{type, data, data_len}`
- RTSP：`network_rtsp_video_enc_param_t`、`network_rtsp_audio_enc_param_t`、`network_rtsp_frame_info_t`

### 2.3 回调

```c
typedef void (*network_wifi_event_callback_t)(int event, void *data);
typedef int  (*network_wifi_status_callback_t)(network_wifi_status_e status);
```

### 2.4 Ops 结构

| Ops | 主要函数 |
|-----|---------|
| `network_wifi_station_ops_t` | `init/deinit/connect/disconnect/reconnect/get_state/get_ssid/get_bssid/get_rssi/get_link_speed/get_ip_address/scan_start/scan_get_results/scan_stop` |
| `network_wifi_ap_ops_t` | `init/deinit/set_config/get_config/start/stop/is_started/get_state/get_connected_clients/get_max_clients` |
| `network_wifi_p2p_ops_t` | `init/deinit/discover_peers/stop_discover_peers/get_discovered_peers/connect/disconnect/get_state/create_group/remove_group/is_group_owner/get_group_info` |
| `network_wifi_ops_t` | Manager：`init/deinit/enable/disable/is_enabled/get_state`；Event：`register_event_callback/unregister_event_callback/set_status_callback`；信息：`get_ip/get_mac`；子模块：`station/ap/p2p` |
| `network_ethernet_ops_t` | `get_ip/get_ipv6/get_mac/get_ip_by_name/get_ipv6_by_name/get_mac_by_name` |
| `network_ntp_ops_t` | `init/deinit/set_server` |
| `network_ping_ops_t` | `ping_ipv4/ping_ipv6` |
| `network_rtsp_ops_t` | `init/deinit/start/stop/set_video_param_callback/set_audio_param_callback/set_dev_param_callback` |

### 2.5 统一实例

```c
typedef struct network_t {
    network_wifi_ops_t     *wifi;
    network_ethernet_ops_t *ethernet;
    network_ntp_ops_t      *ntp;
    network_ping_ops_t     *ping;
    network_rtsp_ops_t     *rtsp;
    void                   *reserved[8];
} network_t;

int network_open(network_t **network);
int network_close(network_t *network);
```

### 2.6 入口函数详述

#### 2.6.1 `network_open`

```c
int network_open(network_t **network);
```

- **功能**：打开网络服务（WiFi Manager + Ethernet + NTP + Ping + RTSP 控制），首次调用完成底层驱动与事件线程初始化。
- **参数**：`network` [out] 服务单例指针。
- **返回**：`0` 成功；`< 0` 为 `NETWORK_ERROR_*`。
- **线程**：任意；内部引用计数。
- **前置**：`framework_init()` 完成；若机型无 WiFi 硬件，`wifi` 子模块可能为 `NULL`。

#### 2.6.2 `network_close`

```c
int network_close(network_t *network);
```

- **功能**：减少引用；最后一次关闭时释放事件线程、关闭网络子模块。
- **返回**：`0` / `< 0`。

### 2.7 `network_wifi_ops_t`（Manager） 详细函数说明

| 函数 | 签名 | 功能 | 返回 / 备注 |
|---|---|---|---|
| `init` | `int init(void)` | 初始化 WiFi 管理器（驱动、事件队列） | 幂等，多次调用返回同一结果 |
| `deinit` | `int deinit(void)` | 反初始化，释放资源 | 退出前应断开 station/关 AP |
| `enable` | `int enable(void)` | 上电 WiFi 射频 | `_DEVICE_NOT_AVAILABLE` 无芯片 |
| `disable` | `int disable(void)` | 掉电 WiFi | 会自动断开 station/关 AP |
| `is_enabled` | `int is_enabled(int *enabled)` | 查询射频开关状态 | `enabled` [out] 0/1 |
| `get_state` | `int get_state(network_wifi_state_e *state)` | 读取 Manager 状态机 | `DISABLED/DISABLING/ENABLED/ENABLING/UNKNOWN` |
| `register_event_callback` | `int register_event_callback(network_wifi_event_callback_t cb)` | 注册 WiFi 事件回调（扫描/连接/RSSI…） | 可多次注册不同 `cb` |
| `unregister_event_callback` | `int unregister_event_callback(network_wifi_event_callback_t cb)` | 注销回调 | 按地址匹配 |
| `set_status_callback` | `int set_status_callback(network_wifi_status_callback_t cb)` | 注册连接状态变化回调（IDLE/CONNECTING/…） | 仅一个；再次设置会覆盖 |
| `get_ip` | `int get_ip(char *ip)` | 获取当前 WiFi 接口 IPv4（`"a.b.c.d"`） | 缓冲区 ≥ 16 |
| `get_mac` | `int get_mac(char *mac)` | 获取 MAC（`"xx:xx:xx:xx:xx:xx"`） | 缓冲区 ≥ 18 |
| 子 ops | `station / ap / p2p` | 三个子模块，见 §2.8–§2.10 | — |

### 2.8 `network_wifi_station_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` | `int init(void)` | 初始化 station 模式 | 需 `enable()` 后调用 |
| `deinit` | `int deinit(void)` | 反初始化 | — |
| `connect` | `int connect(const network_wifi_station_config_t *config)` | 连接到指定 SSID | `config->ssid/passphrase/security_type` |
| `disconnect` | `int disconnect(void)` | 断开连接 | — |
| `reconnect` | `int reconnect(void)` | 用上次参数重连 | `_NOT_INIT` 若无历史 |
| `get_state` | `int get_state(network_wifi_station_state_e *state)` | 读取 station 状态机 | — |
| `get_ssid` | `int get_ssid(char *ssid, size_t len)` | 当前 AP 的 SSID | `len ≥ 33` |
| `get_bssid` | `int get_bssid(char *bssid, size_t len)` | 当前 AP 的 BSSID | `len ≥ 18` |
| `get_rssi` | `int get_rssi(int *rssi)` | 信号强度 (dBm) | `rssi` [out] |
| `get_link_speed` | `int get_link_speed(int *speed_mbps)` | 当前链路速率 | Mbps |
| `get_ip_address` | `int get_ip_address(char *ip, size_t len)` | 获取 DHCP 分配的 IP | `len ≥ 16` |
| `scan_start` | `int scan_start(void)` | 启动扫描，结果经 `SCAN_RESULTS_AVAILABLE` 事件回传 | 异步 |
| `scan_get_results` | `int scan_get_results(network_wifi_scan_result_t *results, int *num)` | 同步取扫描结果 | `num` 入参=容量，出参=实际数 |
| `scan_stop` | `int scan_stop(void)` | 停止扫描 | 幂等 |

### 2.9 `network_wifi_ap_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` / `deinit` | — | AP 模式生命周期 | — |
| `set_config` | `int set_config(const network_wifi_ap_config_t *c)` | 设置 SSID/密码/信道/最大客户端数 | 启动前设置 |
| `get_config` | `int get_config(network_wifi_ap_config_t *c)` | 读取当前配置 | — |
| `start` | `int start(void)` | 启动 AP | `_CONFIG_ERROR` 若未 set_config |
| `stop` | `int stop(void)` | 停止 AP | — |
| `is_started` | `int is_started(int *started)` | 查询是否运行 | `started` [out] 0/1 |
| `get_state` | `int get_state(network_wifi_ap_state_e *state)` | 读取 AP 状态机 | — |
| `get_connected_clients` | `int get_connected_clients(network_wifi_ap_client_t *clients, int *num)` | 枚举已连接 STA | `num` 入参=容量，出参=实际数 |
| `get_max_clients` | `int get_max_clients(int *max_clients)` | 读取最大连接数（来自 config） | — |

### 2.10 `network_wifi_p2p_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` / `deinit` | — | P2P 生命周期 | — |
| `discover_peers` | `int discover_peers(void)` | 启动搜索 | 结果经 WiFi 事件回传 |
| `stop_discover_peers` | `int stop_discover_peers(void)` | 停止搜索 | — |
| `get_discovered_peers` | `int get_discovered_peers(void *devices, int *num_devices)` | 取已发现设备列表 | `devices` 类型由底层实现定义 |
| `connect` | `int connect(const char *device_address)` | 连接指定 P2P 设备 | MAC 字符串 |
| `disconnect` | `int disconnect(void)` | 断开连接 | — |
| `get_state` | `int get_state(network_wifi_p2p_state_e *state)` | 读取 P2P 状态机 | — |
| `create_group` | `int create_group(void)` | 创建 P2P 组（作为 GO） | — |
| `remove_group` | `int remove_group(void)` | 销毁组 | — |
| `is_group_owner` | `int is_group_owner(int *is_owner)` | 自己是否为 GO | `is_owner` [out] 0/1 |
| `get_group_info` | `int get_group_info(void *info)` | 读取组信息 | `info` 类型由底层实现定义 |

### 2.11 `network_ethernet_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `get_ip` | `int get_ip(char *ip)` | 默认有线网卡的 IPv4 | 缓冲 ≥ 16 |
| `get_ipv6` | `int get_ipv6(char *ip)` | 默认网卡 IPv6 | 缓冲 ≥ 46 |
| `get_mac` | `int get_mac(char *mac)` | 默认网卡 MAC | 缓冲 ≥ 18 |
| `get_ip_by_name` | `int get_ip_by_name(const char *iface, char *ip, size_t len)` | 按网卡名读取 IPv4 | `iface` 如 `"eth0"` |
| `get_ipv6_by_name` | `int get_ipv6_by_name(const char *iface, char *ip, size_t len)` | 按网卡名读取 IPv6 | — |
| `get_mac_by_name` | `int get_mac_by_name(const char *iface, char *mac, size_t len)` | 按网卡名读取 MAC | — |

### 2.12 `network_ntp_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` | `int init(void)` | 初始化 NTP 客户端（启动后台线程） | 幂等 |
| `deinit` | `int deinit(void)` | 反初始化 | — |
| `set_server` | `int set_server(char *server_ip)` | 设置 NTP 服务器（IP 或域名） | 立即触发一次同步 |

### 2.13 `network_ping_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `ping_ipv4` | `int ping_ipv4(const char *dest, int ttl, int sequence, int timeout)` | 同步 ICMP ping | `timeout` 毫秒；返回 RTT (ms) 或 `<0` 错误 |
| `ping_ipv6` | `int ping_ipv6(const char *dest, int ttl, int sequence, int timeout)` | IPv6 版 | 同上 |

### 2.14 `network_rtsp_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` / `deinit` | — | RTSP 控制生命周期 | — |
| `start` | `int start(void)` | 启动 RTSP 服务 | 需先 init |
| `stop` | `int stop(void)` | 停止 RTSP 服务 | — |
| `set_video_param_callback` | `int set_video_param_callback(void *cb)` | 注入视频编码参数供给回调（由 gateway 内部定义签名） | 用于联动 Multimedia |
| `set_audio_param_callback` | `int set_audio_param_callback(void *cb)` | 同上，音频编码参数 | — |
| `set_dev_param_callback` | `int set_dev_param_callback(void *cb)` | 设备描述参数回调（设备厂牌/版本/SN） | — |

---

## 3. Notification 服务 — `notification.h`

**模块号**：`FRAMEWORK_MODULE_NOTIFICATION(0x01)`

### 3.1 枚举

- `alarm_type_t`：MD/MAN_BODY/VOICE/FACE/CRY/PREROLL/VEHICLE_GEN/VEHICLE_MOTOR/VEHICLE_BIKE/ALERTAREA/TRANSGRESSION/PET/PACKAGE/OFF_DUTY/PFS/RPS/FENCE/INOUT/WANDER/CALL_TO_SERVER_S/CALL_TO_SERVER_E/NUM/ALL(0xffffffff)
- `alarm_state_t`：END(0)/START(1)
- `message_type_t`：SLEEP/WAKEUP/INVERSION/LSSL/TALK_IN/TALK_OUT/NET_CONNECT/NUM/ALL
- `input_device_type_t`：UNKNOWN/KEYPAD/TOUCH/MOUSE/ENCODER/SENSOR_PIR
- `input_action_t`：DOWN/UP/REPEAT/LONGPRESS/DOUBLE_CLICK/TRIPLE_CLICK
- `input_keycode_t`：完整 Linux-style keycode 列表（数字/字母/HOME/BACK/MENU/OK/ENTER/ESC/方向/POWER/CONFIG/RESET/TALK/MODE_SWITCH/音量/媒体/相机/亮度/TORCH/颜色按键/FUNC_1..3）

### 3.2 结构

```c
#define ALARM_DETAIL_LEN 1024
typedef struct { alarm_type_t type; long long timestamp_ms; alarm_state_t state; char detail[ALARM_DETAIL_LEN]; int detail_len; } alarm_t;
typedef struct { message_type_t type; long long timestamp_ms; char *data; int data_len; } message_t;
typedef struct { uint64_t timestamp_ms; input_device_type_t device; int device_id; input_keycode_t keycode; int linux_code; input_action_t action; int value; int meta_state; } input_event_t;
```

### 3.3 回调

```c
typedef int  (*alarm_callback_t)(alarm_t *alarm, int msg_len, void *user_data);
typedef int  (*message_callback_t)(message_t *msg, void *user_data);
typedef int  (*input_listener_cb)(const input_event_t *ev, void *user_data);
```

### 3.4 Ops 结构

| Ops | 函数 |
|-----|------|
| `alarm_ops_t` | `init/uninit/post/detach/reg_client/unreg_client` |
| `message_ops_t` | `init/uninit/post/reg_client/unreg_client` |
| `message_queue_ops_t` | `init/uninit/create/destroy/read/write` |
| `input_ops_t` | `init/deinit/subscribe/unsubscribe/set_focus/set_debounce_ms/set_repeat_ms` |

### 3.5 统一实例

```c
typedef struct notification_t {
    alarm_ops_t         *alarm;
    message_ops_t       *message;
    message_queue_ops_t *queue;
    input_ops_t         *input;
    void                *reserved[8];
} notification_t;

int notification_open(notification_t **notification);
int notification_close(notification_t *notification);
```

### 3.6 入口函数详述

#### 3.6.1 `notification_open`

```c
int notification_open(notification_t **notification);
```

- **功能**：打开通知服务（告警/消息/消息队列/输入事件）。
- **返回**：`0` 成功；`< 0` 错误。
- **线程**：任意。

#### 3.6.2 `notification_close`

```c
int notification_close(notification_t *notification);
```

- **功能**：引用计数递减；最后一次关闭释放内部 dispatcher 线程。

### 3.7 `alarm_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` | `int init(void)` | 初始化告警分发器 | 幂等 |
| `uninit` | `int uninit(void)` | 反初始化 | — |
| `post` | `int post(alarm_type_t type, char *data, int data_len)` | 投递告警事件（由采集/AI/HAL 侧触发） | `data` 可为 `NULL`；`data_len ≤ ALARM_DETAIL_LEN` |
| `detach` | `int detach(alarm_type_t type)` | 清空指定类型的订阅者（调试用） | — |
| `reg_client` | `int reg_client(alarm_type_t type, alarm_callback_t cb, void *user, int min_interval)` | 订阅某类告警 | `min_interval` 最小间隔（毫秒），用于抑制抖动 |
| `unreg_client` | `int unreg_client(alarm_type_t type, alarm_callback_t cb)` | 取消订阅 | 按 `type + cb` 匹配 |

### 3.8 `message_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` / `uninit` | — | 消息分发器生命周期 | — |
| `post` | `int post(message_type_t type, char *data, int data_len)` | 发布系统消息（SLEEP/WAKEUP/TALK_IN…） | — |
| `reg_client` | `int reg_client(message_type_t type, message_callback_t cb, void *user)` | 订阅某类消息 | — |
| `unreg_client` | `int unreg_client(message_type_t type, message_callback_t cb)` | 取消订阅 | — |

### 3.9 `message_queue_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` / `uninit` | — | 模块生命周期 | — |
| `create` | `int create(const char *queue_name)` | 创建命名队列，返回 handle（`> 0`） | 同名已存在返回同一 handle |
| `destroy` | `int destroy(const char *queue_name)` | 销毁队列 | 幂等 |
| `read` | `int read(int handle, char *msg, int *msg_len, long long *msg_time_ms)` | 阻塞读取一条消息 | `msg_len` 入参=容量，出参=实际；`msg_time_ms` 可为 `NULL` |
| `write` | `int write(int handle, char *msg, int msg_len)` | 向队列写入一条消息 | 队列满时丢弃/覆盖由实现决定 |

### 3.10 `input_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` | `int init(void)` | 初始化输入分发（聚合 keypad/touch/mouse/sensor） | — |
| `deinit` | `int deinit(void)` | 反初始化 | — |
| `subscribe` | `int subscribe(int priority, uint32_t keymask, input_listener_cb cb, void *user)` | 订阅输入事件；**高优先级** Listener 可消费事件 | `keymask` 按位选择 `input_keycode_t` |
| `unsubscribe` | `int unsubscribe(input_listener_cb cb, void *user)` | 取消订阅 | 按 `cb+user` 匹配 |
| `set_focus` | `int set_focus(void *token)` | 绑定焦点令牌，仅该 token 下 listener 收事件 | `NULL` 解除焦点 |
| `set_debounce_ms` | `int set_debounce_ms(int ms)` | 设置按键去抖（毫秒） | — |
| `set_repeat_ms` | `int set_repeat_ms(int ms)` | 设置长按重复触发间隔 | `0` 禁用 |

---

## 4. Device 服务 — `device.h`

**模块号**：`FRAMEWORK_MODULE_DEVICE(0x04)`
**版本宏/结构**：`DEVICE_VERSION_*`、`device_version_info_t`、`device_get_version()`、`device_get_version_string()`

### 4.1 LED

```c
typedef enum { LED_TYPE_STATUS_RED, LED_TYPE_STATUS_BLUE, LED_TYPE_IR, LED_TYPE_PILOT, LED_TYPE_MAX } led_type_t;
typedef enum { LED_MODE_OFF, LED_MODE_ON, LED_MODE_BLINK, LED_MODE_BREATHE, LED_MODE_MAX } led_mode_t;
typedef enum { LED_PRIORITY_LOW, NORMAL, HIGH, CRITICAL, MAX } led_priority_t;
typedef enum { LED_SCENE_SYSTEM, NORMAL, ALARM, RECORDING, NETWORK, USER, MAX } led_scene_t;
#define LED_BRIGHTNESS_MIN 0  /* _MAX=255, _DEFAULT=128, LED_BLINK_DEFAULT_ON_MS=500, _OFF_MS=500 */
typedef struct { uint32_t on_ms; uint32_t off_ms; } led_blink_params_t;
typedef struct { led_type_t type; led_mode_t mode; uint32_t brightness; led_blink_params_t blink; } led_control_t;
typedef struct { led_scene_t scene; led_priority_t priority; led_control_t red_led, blue_led; bool active; } led_scene_config_t;
```

`led_ops_t`：`init/deinit/set_scene/clear_scene/get_active_scene/set_led/set_blink/get_led`

### 4.2 PTZ

- 命令：`ptz_cmd_t`（UP/DOWN/LEFT/RIGHT/LEFT_UP/RIGHT_UP/LEFT_DOWN/RIGHT_DOWN/STOP/UP_STEP/DOWN_STEP/LEFT_STEP/RIGHT_STEP/ZOOM_NEAR/ZOOM_FAR/FOCUS_NEAR/FOCUS_FAR/IRIS_OPEN/IRIS_CLOSE）
- 状态：`ptz_state_t`（STOP/RUNNING）
- 跟踪：`ptz_track_mode_t`（DISABLE/MOTION/BODY/PET/VEHICLE）
- 位置：`ptz_position_t{pan, tilt, fov_h, fov_v, zoom_value, state}`
- 预置点：`ptz_preset_info_t{id, name[20], pan, tilt, zoom}`

`ptz_ops_t`：`init/deinit/control/set_position/get_position/set_speed/get_speed/get_state/reset/set_fov/preset_set/preset_goto/preset_delete/preset_get_count/preset_get_list/preset_get_by_name/tracking_set_enable/tracking_get_enable/tracking_set_mode/tracking_get_mode/tracking_set_temporary_switch/tracking_handle_motion/tracking_handle_human`

### 4.3 Power

- 枚举：`power_manager_suspend_mode_t`（NORMAL/DEEP）
- 钩子：`power_manager_hook_t`

`power_ops_t`：`init/deinit/suspend/resume/get_last_wakeup_source/register_hook/unregister_hook`

### 4.4 Night Vision

- `nightvision_photosensitive_mode_t`：HD/SOFT
- `nightvision_mode_t`：NORMAL/COLOR
- `nightvision_normal_switch_t`：DAY/NIGHT/AUTO
- `nightvision_color_switch_t`：GRAY/COLOR/SMART
- `nightvision_state_t`：DAY/NIGHT_IR/NIGHT_WHITE_LED/NULL

`nv_ops_t`：`init/deinit/set_param/get_param/set_normal_switch/get_normal_switch/set_color_switch/get_color_switch/get_day_night/set_alarm_status`

### 4.5 Backlight

`backlight_ops_t`：`init(instance, name)/deinit/set_value/release`

### 4.6 统一实例

```c
typedef struct {
    led_ops_t       *led;
    ptz_ops_t       *ptz;
    power_ops_t     *power;
    nv_ops_t        *nv;
    backlight_ops_t *backlight;
    void            *reserved[10];
} device_t;

int device_open(device_t **device);
int device_close(device_t *device);
```

### 4.7 子模块错误码

- LED：`DEVICE_ERROR_LED_NOT_FOUND/_INVALID_TYPE/_INVALID_MODE/_SCENE_CONFLICT`
- PTZ：`_PTZ_NOT_READY/_POSITION_LIMIT/_MOTOR_FAULT/_PRESET_FULL/_PRESET_NOT_FOUND/_TRACKING_FAILED`
- 夜视：`_NV_SENSOR_FAILED/_IRCUT_FAILED/_IRLED_FAILED/_SWITCH_FAILED/_ISP_NOT_READY`
- 背光：`_BL_PWM_FAILED/_GPIO_FAILED/_BRIGHTNESS_INVALID`

### 4.8 入口函数详述

```c
int  device_open(device_t **device);
int  device_close(device_t *device);
uint32_t device_get_version(void);
const char *device_get_version_string(void);
```

- `device_open`：首次调用初始化所有硬件控制子模块；非空机型可能部分子模块为 `NULL`（按产品形态）。
- `device_close`：引用计数递减释放。
- `device_get_version` / `_string`：编译期版本号；无句柄要求，随时可调。

### 4.9 `led_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` | `int init(void)` | 初始化 LED 管理器 | 幂等 |
| `deinit` | `void deinit(void)` | 反初始化 | 无返回 |
| `set_scene` | `int set_scene(const led_scene_config_t *config)` | 按场景（含优先级）激活 LED | 高优先级覆盖低优先级 |
| `clear_scene` | `int clear_scene(led_scene_t scene)` | 撤销某场景 | 撤销后回落到下一优先级场景 |
| `get_active_scene` | `int get_active_scene(led_scene_t *scene)` | 读取当前激活场景 | — |
| `set_led` | `int set_led(led_type_t type, led_mode_t mode, uint32_t brightness)` | 直接控制单个 LED | `brightness` 0–255 |
| `set_blink` | `int set_blink(led_type_t type, uint32_t on_ms, uint32_t off_ms)` | 设置闪烁参数 | — |
| `get_led` | `int get_led(led_type_t type, led_mode_t *mode, uint32_t *brightness)` | 读取当前状态 | — |

### 4.10 `ptz_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` / `deinit` | — | 生命周期 | — |
| `control` | `int control(ptz_cmd_t cmd, int speed)` | 方向/步进/缩放/聚光/光圈控制 | `speed` 0–100，对 `_STEP` 无效 |
| `set_position` | `int set_position(const ptz_position_t *pos)` | 直接到达绝对位置 | 可能超限触发 `_POSITION_LIMIT` |
| `get_position` | `int get_position(ptz_position_t *pos)` | 读取当前位置/FOV/zoom | — |
| `set_speed` / `get_speed` | `int (int speed)` / `int (int *speed)` | 默认转动速度（持续移动用） | 0–100 |
| `get_state` | `int get_state(ptz_state_t *state)` | `STOP` / `RUNNING` | — |
| `reset` | `int reset(void)` | 回到机械中位 | 阻塞到完成 |
| `set_fov` | `int set_fov(double fov_h, double fov_v)` | 记录当前 FOV（用于 UI 坐标映射） | 度 |
| `preset_set` | `int preset_set(int preset_id, const char *name)` | 保存当前位为预置点 | `preset_id` 0–N，上限见产品规格 |
| `preset_goto` | `int preset_goto(int preset_id)` | 跳转到预置点 | 异步完成 |
| `preset_delete` | `int preset_delete(int preset_id)` | 删除预置点 | — |
| `preset_get_count` | `int preset_get_count(void)` | 已保存数量（返回值即是） | — |
| `preset_get_list` | `int preset_get_list(ptz_preset_info_t *list, int max_count)` | 枚举预置点 | 实际数量由返回值给出 |
| `preset_get_by_name` | `int preset_get_by_name(const char *name, ptz_preset_info_t *info)` | 按名称查找 | `_PRESET_NOT_FOUND` |
| `tracking_set_enable` | `int tracking_set_enable(int enable)` | 目标跟踪总开关 | — |
| `tracking_get_enable` | `int tracking_get_enable(void)` | 查询开关 | 返回 0/1 |
| `tracking_set_mode` | `int tracking_set_mode(ptz_track_mode_t mode)` | 跟踪模式（MOTION/BODY/PET/VEHICLE） | — |
| `tracking_set_temporary_switch` | `int tracking_set_temporary_switch(int enable)` | 临时暂停/恢复（保留模式） | — |
| `tracking_handle_motion` | `int tracking_handle_motion(void *md_info)` | 注入运动检测结果驱动跟踪 | `md_info` 由 AI/MD 回调传入 |
| `tracking_handle_human` | `int tracking_handle_human(const void *human_output, int rect_num, int video_w, int video_h)` | 注入人体检测结果 | 同上 |

### 4.11 `power_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` / `deinit` | — | 生命周期 | — |
| `suspend` | `int suspend(power_manager_suspend_mode_t mode)` | 进入 NORMAL/DEEP 睡眠 | 阻塞到唤醒才返回 |
| `resume` | `int resume(void)` | 唤醒后收尾 | 多由内部调用 |
| `get_last_wakeup_source` | `int get_last_wakeup_source(int *source)` | 读取最近一次唤醒源 | 源编码由底层定义 |
| `register_hook` | `int register_hook(power_manager_hook_t pre, power_manager_hook_t post)` | 注册睡前/唤醒后钩子 | `pre/post` 可为 `NULL` |
| `unregister_hook` | `int unregister_hook(power_manager_hook_t pre, power_manager_hook_t post)` | 反注册 | — |

### 4.12 `nv_ops_t`（夜视） 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` / `deinit` | — | 生命周期 | — |
| `set_param` | `int set_param(nightvision_photosensitive_mode_t, nightvision_mode_t)` | 设置光敏/夜视模式 | — |
| `get_param` | `int get_param(... *photosensitive, ... *mode)` | 读取当前参数 | — |
| `set_normal_switch` / `get_normal_switch` | `int (nightvision_normal_switch_t)` / `int (... *)` | 普通日夜切换模式（DAY/NIGHT/AUTO） | — |
| `set_color_switch` / `get_color_switch` | `int (nightvision_color_switch_t)` / `int (... *)` | 彩色夜视模式（GRAY/COLOR/SMART） | — |
| `get_day_night` | `int get_day_night(void)` | 返回当前判定状态（见 `nightvision_state_t`） | — |
| `set_alarm_status` | `int set_alarm_status(int alarm_status)` | 上报告警状态用于切换补光/色彩 | — |

### 4.13 `backlight_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` | `int init(const char *instance, const char *name)` | 选择后端（PWM/GPIO 等）初始化 | `instance` 硬件实例号，`name` 后端名 |
| `deinit` | `int deinit(void)` | 反初始化 | — |
| `set_value` | `int set_value(uint32_t value)` | 设置亮度（0–亮度上限） | `_BRIGHTNESS_INVALID` |
| `release` | `int release(void)` | 释放控制权（允许其他组件占用） | — |

---

## 5. Storage 服务 — `storage.h`

**模块号**：`FRAMEWORK_MODULE_STORAGE(0x07)`

### 5.1 枚举 / 回调

```c
typedef enum {
    STORAGE_TFCARD_STATUS_ERROR    = -1,
    STORAGE_TFCARD_STATUS_DISABLED =  0,
    STORAGE_TFCARD_STATUS_NORMAL,
    STORAGE_TFCARD_STATUS_EJECTED,
    STORAGE_TFCARD_STATUS_READONLY,
    STORAGE_TFCARD_STATUS_FORMAT,
} storage_tfcard_status_e;

typedef int (*storage_tfcard_status_callback_t)(storage_tfcard_status_e status);
```

### 5.2 Ops 结构

| Ops | 函数 |
|-----|------|
| `storage_tfcard_ops_t` | `init(dir)/deinit/status_subscribe/status_unsubscribe(handle)/get_status/get_capacity(sync)/get_used_space(sync)/get_mounted_dir/format(sync)` |
| `storage_flash_ops_t`  | `write_from_file/write_from_memory/read_to_memory/write_start/write_input/write_stop` |

### 5.3 统一实例

```c
typedef struct {
    storage_tfcard_ops_t *tfcard;
    storage_flash_ops_t  *flash;
    void                 *reserved[8];
} storage_t;

int storage_open(storage_t **storage);
int storage_close(storage_t *storage);
```

### 5.4 专用错误码

- 文件：`_FILE_NOT_FOUND/_FILE_OPEN_FAILED/_FILE_READ_FAILED/_FILE_WRITE_FAILED`
- Flash：`_FLASH_ERASE_FAILED/_WRITE_FAILED/_READ_FAILED/_BAD_BLOCK`
- TF 卡：`_TFCARD_NOT_MOUNTED/_FULL/_READ_ONLY/_FORMAT_FAILED`

### 5.5 入口函数详述

```c
int storage_open(storage_t **storage);
int storage_close(storage_t *storage);
```

- `storage_open`：首次调用启动 TF 卡监控线程、初始化 Flash 写入器；若机型无 TF，`tfcard` 可为 `NULL`。
- `storage_close`：引用计数递减；最后一次关闭停止监控线程。

### 5.6 `storage_tfcard_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` | `int init(const char *dir)` | 初始化 TF 卡模块，设置默认挂载目录 | `dir` 默认 `/mnt/tf` |
| `deinit` | `int deinit(void)` | 反初始化 | — |
| `status_subscribe` | `int status_subscribe(storage_tfcard_status_callback_t cb)` | 订阅 TF 卡状态变更 | 返回 `handle > 0` 用于取消 |
| `status_unsubscribe` | `int status_unsubscribe(int handle)` | 取消订阅 | 幂等 |
| `get_status` | `storage_tfcard_status_e get_status(void)` | 读取最新状态（`NORMAL/EJECTED/READONLY/FORMAT/DISABLED/ERROR`） | 非阻塞 |
| `get_capacity` | `int get_capacity(int sync)` | 读取总容量（MB） | `sync=1` 强制刷新，`0` 用缓存 |
| `get_used_space` | `int get_used_space(int sync)` | 读取已用空间（MB） | 同上 |
| `get_mounted_dir` | `const char *get_mounted_dir(void)` | 读取当前挂载目录 | 字符串属模块所有，不可 `free` |
| `format` | `void format(int sync)` | 格式化 TF 卡 | `sync=1` 阻塞；`0` 后台异步 |

### 5.7 `storage_flash_ops_t` 详细函数说明

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `write_from_file` | `int write_from_file(const char *dev_dst, const char *file_src, int is_fs_upgrade)` | 将文件写入到 Flash 分区（`/dev/mtdN` 或分区名） | `is_fs_upgrade=1` 切换到文件系统升级模式 |
| `write_from_memory` | `int write_from_memory(const char *dev_dst, const void *buf, const int buf_size)` | 将内存 buffer 一次写入 | — |
| `read_to_memory` | `int read_to_memory(const char *dev_dst, void *buf, const int buf_size)` | 从分区读取到 buffer | 返回实际读取字节 |
| `write_start` | `int write_start(const char *dev_dst, void **handle)` | 分块写入的会话起点 | `handle` [out] |
| `write_input` | `int write_input(void *handle, const void *buf, const int buf_size)` | 分块写入一段数据 | 多次调用累加 |
| `write_stop` | `int write_stop(void *handle)` | 结束分块写入，完成校验 | 幂等收尾 |

---

## 6. Runtime Bootstrap — `runtime_bootstrap.h`

```c
typedef enum { system_state_init = 0, system_state_sleep = 1, system_state_run = 2 } system_state_e;

typedef struct runtime_bootstrap {
    int (*load_config)(const char *config_path);
} runtime_bootstrap_t;

int runtime_bootstrap_init(const char *config_path);
int runtime_bootstrap_open(runtime_bootstrap_t **bootstrap);
int runtime_bootstrap_close(runtime_bootstrap_t *bootstrap);
system_state_e runtime_bootstrap_get_state(void);
int runtime_bootstrap_set_state(system_state_e state);

/* 供上层统一初始化：SoC、Log、线程、HAL 模块一次性就绪 */
int framework_init(int soc_mode, int log_level);
```

### 6.1 函数详述

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `runtime_bootstrap_init` | `int runtime_bootstrap_init(const char *config_path)` | 先行加载 `settings.json`（可在 open 之前调用，便于守护进程早期读取状态） | `config_path` 可为 `NULL` 使用默认路径 |
| `runtime_bootstrap_open` | `int runtime_bootstrap_open(runtime_bootstrap_t **bootstrap)` | 获取 runtime 操作句柄 | 单例 |
| `runtime_bootstrap_close` | `int runtime_bootstrap_close(runtime_bootstrap_t *bootstrap)` | 释放句柄引用 | — |
| `runtime_bootstrap_get_state` | `system_state_e get_state(void)` | 读取系统状态（init/sleep/run） | 任意线程 |
| `runtime_bootstrap_set_state` | `int set_state(system_state_e state)` | 切换系统状态，触发对应生命周期钩子 | 顺序建议：`init → run → sleep → run` |
| `load_config` (ops) | `int bootstrap->load_config(const char *config_path)` | 热加载 runtime 配置 | `0` / `< 0` |
| `framework_init` | `int framework_init(int soc_mode, int log_level)` | **一键初始化**（SoC / Log / Thread / HAL 一次就绪），应用最早调用 | `soc_mode` 由 SoC 定义；`log_level` 见 `log.h` |

---

## 7. Console — `console.h`

**颜色码**：`CONSOLE_ERROR_COLOR`、`_WARN_COLOR`、`_MSG_COLOR`、`_DEBUG_COLOR`、`_INPUT_COLOR`、`_HIGHLIGHT_COLOR`、`CONSOLE_MODE_OFF`
**宏**：`console_warn(fmt, ...)`、`DEBUG_HERE`
**常量**：`COMMAND_STRING_LEN_MAX=32`、`COMMAND_PARAMS_NUM_MAX=16`、`COMMAND_PARAMS_LEN_MAX=256`

```c
typedef struct {
    int n;
    char cmd[COMMAND_STRING_LEN_MAX];
    char p[COMMAND_PARAMS_NUM_MAX][COMMAND_PARAMS_LEN_MAX];
} command_pkt;

typedef int (*FN_COMMAND)(const command_pkt *pParams, char *pcAck);

int console_init(const char *pCMDPath);
int console_deinit(void);
int console_print(const char *pcTitleColor, const char *pcTitle,
                  const char *pcStringColor, const char *pcSubTitle,
                  const char *pcFormat, ...);
int console_loop(void);
int console_cmd_register(const char *pcCommand, FN_COMMAND fnFunction, const char *pcHelp);
```

### 7.1 函数详述

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `console_init` | `int console_init(const char *pCMDPath)` | 绑定命令管道（如 `/tmp/celestia.cmd`），启动后台接收线程 | 幂等 |
| `console_deinit` | `int console_deinit(void)` | 停止线程，关闭管道 | — |
| `console_print` | `int console_print(const char *titleColor, const char *title, const char *strColor, const char *subTitle, const char *fmt, ...)` | 带颜色/标题的结构化日志打印 | 可结合 `console_warn(fmt, ...)` 宏 |
| `console_loop` | `int console_loop(void)` | 阻塞式消息循环（调试工具场景；正常业务勿用） | 通常 `console_init` 已异步跑 |
| `console_cmd_register` | `int console_cmd_register(const char *cmd, FN_COMMAND fn, const char *help)` | 注册命令行处理函数 | `fn(pParams, pcAck)` 处理输入并返回应答 |

---

## 8. OTA — `ota.h` + `otapack.h`

### 8.1 运行时 API（`ota.h`）

```c
typedef struct { char cDownloadUrl[256]; int iFwVersion; } ota_info_t;

typedef enum {
    OTA_STATUS_IDLE        = 0,
    OTA_STATUS_DOWNLIADING = 1,
    OTA_STATUS_INSTALLING  = 2,
    OTA_STATUS_FAIL        = 3,
    OTA_STATUS_DONE        = 4
} ota_status_e;

int          ota_init(void);
int          ota_start(ota_info_t info);
int          ota_stop(void);
ota_status_e ota_get_status(void);
int          ota_get_progress_percent(void);
```

### 8.2 固件包格式（`otapack.h`）

```c
#define DEFAULT_CIPHER "AES-128-CBC"
#define DEFAULT_MD     "SHA1"
#define DEFAULT_KEY    "cipherkeycipherkey"

typedef enum { ENCRYPT_NONE, ENCRYPT_AES_CBC128, ENCRYPT_AES_CBC192, ENCRYPT_AES_CBC256, ENCRYPT_MAX = 64 } ENCRYPT_TYPE_E;
typedef enum { MD_NONE, MD_SHA1, MD_SHA224, MD_SHA256, MD_SHA384, MD_SHA512, MD_MAX = 64 } MSG_DIGEST_TYPE_E;
typedef enum { CHECK_TYPE_FILELEN = 0, CHECK_TYPE_CRC32 = 1 } CHECK_TYPE_E;

extern const char* encryptStr[ENCRYPT_MAX];
extern const char* mdStr[MD_MAX];

#pragma pack(push, 1)
typedef union {
    struct {
        unsigned char  Magic[8];
        unsigned int   upgradeType;
        unsigned int   checkType;
        unsigned int   filelen;
        unsigned int   checkSum;
        unsigned char  method_md;
        unsigned char  method_cipher;
        unsigned int   iSrcFileLen;
    } Head;
    unsigned char headinfo[256];
} OTA_FW_HEADINFO;
#pragma pack(pop)
```

> 头部长度 `256` **锁定**，首次发布后不允许修改，仅可向 `Head` 内追加成员。

### 8.3 运行时函数详述

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `ota_init` | `int ota_init(void)` | 初始化 OTA 子系统（下载器 / 安装器 / 状态机） | 幂等；通常由 runtime 代为调用 |
| `ota_start` | `int ota_start(ota_info_t info)` | **异步**启动升级（下载 → 校验 → 安装） | 过程中可 `ota_get_status` / `ota_get_progress_percent` 查询 |
| `ota_stop` | `int ota_stop(void)` | 取消当前升级任务 | 幂等；若已进入 `INSTALLING` 可能不可中止 |
| `ota_get_status` | `ota_status_e ota_get_status(void)` | 读取状态（IDLE/DOWNLOADING/INSTALLING/FAIL/DONE） | 非阻塞 |
| `ota_get_progress_percent` | `int ota_get_progress_percent(void)` | 下载/安装进度（0–100） | — |

> `otapack.h` 中 `DEFAULT_CIPHER/DEFAULT_MD/DEFAULT_KEY` 与 `encryptStr[]/mdStr[]` 仅供**打包工具与升级器内部**使用，应用层**禁止**直接读取默认密钥常量。

---

## 9. Reboot Info — `reboot_info.h`

```c
typedef enum {
    REBOOT_REASON_DUT_COMMON            = 1000,
    REBOOT_REASON_DUT_POWER_DOWN        = 1001,
    REBOOT_REASON_DUT_CRASH             = 1002,
    REBOOT_REASON_DUT_HWATCHDOG         = 1003,
    REBOOT_REASON_DUT_FACTORY_RESET     = 1004,
    REBOOT_REASON_DUT_ABNORMAL_REBOOT   = 1005,
    REBOOT_REASON_SDK_COMMON            = 2000,
    REBOOT_REASON_SDK_SCHEDULED_MAINTENANCE = 2001,
    REBOOT_REASON_SDK_SWATCHDOG         = 2002,
    REBOOT_REASON_CLOUD_COMMON          = 3000,
    REBOOT_REASON_CLOUD_REBOOT          = 3001,
    REBOOT_REASON_CLOUD_UPGRADE         = 3002,
    REBOOT_REASON_CLOUD_FACTORY_RESET   = 3003,
} reboot_reason_e;

typedef struct {
    reboot_reason_e reason;
    time_t          ts;
    char            message[256];
} reboot_param_t;

int reboot_info_set(reboot_reason_e reason, const char *msg);
int reboot_info_get(reboot_param_t *info);
int reboot_info_init(void);
int reboot_info_refresh_powerdown_time(void);
```

### 9.1 函数详述

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `reboot_info_init` | `int reboot_info_init(void)` | 从存储中恢复上一次重启原因记录 | 开机早期调用 |
| `reboot_info_set` | `int reboot_info_set(reboot_reason_e reason, const char *msg)` | 记录本次重启原因（重启前调用） | `msg` ≤ 256 字符 |
| `reboot_info_get` | `int reboot_info_get(reboot_param_t *info)` | 读取上一次重启原因及时间/备注 | — |
| `reboot_info_refresh_powerdown_time` | `int reboot_info_refresh_powerdown_time(void)` | 将当前时间戳刷写到存储（用于定期快照，硬件掉电取近似关机时间） | 建议周期性调用 |

---

## 10. RTSP Gateway — `rtsp_protocol.h`

**模块号**：`FRAMEWORK_MODULE_GATEWAY(0x08)`

```c
typedef enum { RTSP_STREAM_TYPE_LIVE = 0, RTSP_STREAM_TYPE_PLAYBACK = 1 } rtsp_stream_type_t;

typedef struct {
    int used;
    rtsp_stream_type_t stream_type;
    int channel;
    int video_handle;
    int audio_handle;
} rtsp_stream_handle_t;

typedef struct rtsp_protocol_server_ops {
    int (*start)(rtsp_protocol_context_t *ctx);
    int (*stop)(rtsp_protocol_context_t *ctx);
    int (*get_status)(rtsp_protocol_context_t *ctx, bool *is_running);
    void *reserved[5];
} rtsp_protocol_server_ops_t;

typedef struct rtsp_protocol_ops {
    rtsp_protocol_server_ops_t *server;
    void *reserved[7];
} rtsp_protocol_ops_t;

typedef struct rtsp_protocol_context {
    rtsp_protocol_ops_t *ops;
    multimedia_t        *multimedia;
    rtsp_stream_handle_t stream_handles[8];
    int                  record_start_time;
    bool                 initialized;
    bool                 multimedia_owned;
    void                *reserved[6];
} rtsp_protocol_t;

int rtsp_protocol_open(rtsp_protocol_t **rtsp);
int rtsp_protocol_close(rtsp_protocol_t *rtsp);
```

### 10.1 入口与 Server Ops 函数详述

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `rtsp_protocol_open` | `int rtsp_protocol_open(rtsp_protocol_t **rtsp)` | 打开 RTSP Gateway；若 `multimedia` 未在 context 注入，会自行 `multimedia_open` 并设置 `multimedia_owned=true` | 应用可预先赋值 `multimedia` 以共享单例 |
| `rtsp_protocol_close` | `int rtsp_protocol_close(rtsp_protocol_t *rtsp)` | 关闭 Gateway；`multimedia_owned=true` 时一并关闭内部 multimedia | — |
| `server->start` | `int start(rtsp_protocol_context_t *ctx)` | 启动 RTSP 服务，开始监听端口 | 端口/编码来自 `fw_video_config.json` |
| `server->stop` | `int stop(rtsp_protocol_context_t *ctx)` | 停止 RTSP 服务 | — |
| `server->get_status` | `int get_status(rtsp_protocol_context_t *ctx, bool *is_running)` | 查询运行状态 | `is_running` [out] |

---

## 11. Nebula Gateway — `nebula_protocol.h`

**模块号**：`FRAMEWORK_MODULE_NEBULA_PROTOCOL(0x09)`
**版本**：`NEBULA_PROTOCOL_VERSION_*`

### 11.1 Ops

| Ops | 函数 |
|-----|------|
| `nebula_network_ops_t` | `init/deinit/start/stop/send_to_all_clients/send_to_local_client/start_stream/stop_stream/link_server_init/link_server_deinit` |
| `nebula_http_ops_t`    | `init/deinit/start/stop` |
| `nebula_pair_ops_t`    | `init/deinit/scan_connect_run/play_file` |

### 11.2 统一实例

```c
typedef struct {
    nebula_network_ops_t *network;
    nebula_http_ops_t    *http;
    nebula_pair_ops_t    *pair;
    void                 *reserved[8];
} nebula_protocol_t;

int nebula_protocol_open(nebula_protocol_t **nebula);
int nebula_protocol_close(nebula_protocol_t *nebula);
```

### 11.3 入口与 Ops 函数详述

#### 11.3.1 入口

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `nebula_protocol_open` | `int nebula_protocol_open(nebula_protocol_t **nebula)` | 打开 Nebula Gateway（私有协议 + HTTP + 配网） | 单例 |
| `nebula_protocol_close` | `int nebula_protocol_close(nebula_protocol_t *nebula)` | 关闭 Gateway | 引用计数 |

#### 11.3.2 `nebula_network_ops_t`

| 函数 | 功能 | 备注 |
|---|---|---|
| `init` / `deinit` | 私有协议核心生命周期 | — |
| `start` / `stop` | 启/停协议栈（TCP/TLS 监听、心跳） | — |
| `send_to_all_clients` | `int (const void *data, size_t len)` 广播数据到全部已连接客户端 | — |
| `send_to_local_client` | `int (const void *data, size_t len)` 仅发送到局域网客户端 | — |
| `start_stream` / `stop_stream` | `int (nebula_network_handle_t handle)` 启/停某客户端的音视频流 | `handle` 来自 Gateway 分配 |
| `link_server_init` / `link_server_deinit` | 初始化/释放"链接服务"（设备到云 Keep-alive） | 可选，按产品开关 |

#### 11.3.3 `nebula_http_ops_t`

| 函数 | 功能 | 备注 |
|---|---|---|
| `init` / `deinit` | HTTP 控制面生命周期 | — |
| `start` / `stop` | 启动/停止设备内置 HTTP 服务（配置/抓图/诊断页） | 默认端口见配置 |

#### 11.3.4 `nebula_pair_ops_t`

| 函数 | 签名 | 功能 | 备注 |
|---|---|---|---|
| `init` / `deinit` | — | 配网/扫码配对生命周期 | — |
| `scan_connect_run` | `int scan_connect_run(void *arg)` | 启动"扫码配网"流程（识别二维码→连接 WiFi→上报云端） | `arg` 为配网参数指针；**阻塞**直到流程结束或超时 |
| `play_file` | `int play_file(const char *name)` | 播放配网语音提示（"配网成功"、"连接失败"等） | `name` 资源名 |

---

## 12. Graphics Bitmap — `bitmap.h`

位图加载与像素操作工具（详细结构请直接参考 `framework/ui/graphics/include/bitmap.h`；本文件仅作 GUI 场景占位）。

---

## 13. 其他公共头文件

| 头文件 | 定位 | 说明 |
|-------|------|------|
| `framework/runtime/include/runtime_error.h` | Runtime 错误码 | 供 bootstrap/console/ota/reboot_info 统一错误返回 |
| `framework/runtime/include/runtime.h` | 汇总 include | 一次性引入 bootstrap/console/ota/otapack/reboot_info |
| `framework/services/storage/include/tfcard_utils.h` | TF 卡辅助 | 供 storage 内部使用，**不建议应用直接依赖**（如需应用层能力，走 `storage.h`） |
| `framework/ui/lvgl/include/lv_port.h`, `lv_port_utils.h` | LVGL 适配 | 供 UI 子模块使用 |
| `framework/ui/common/include/port.h`, `gui_error.h` | GUI 公共 | UI 内部抽象 |

---

## 14. 对外接口使用约束

1. **只能通过 `xxx_open() → ops 调用 → xxx_close()`** 三段式使用 Framework 服务；单例由 Framework 内部保证。
2. **应用层不得 `#include` 任何 `hardware/**/*.h`**（公开或私有 HAL 头均禁止），相关能力必须通过 Framework 服务暴露。
3. **跨层错误码统一使用 `system_error_t`**（`system/core/include/system_error.h` 中的 `SYSTEM_ERROR_CODE(domain, module, error)` 编码）。
4. 所有 `ops` 结构中的 `reserved[N]` 用于二进制兼容扩展；新增函数只能填到 `reserved` 位上。
5. 文档**只描述上层（应用/上层框架）调用的接口**；框架内部钩子、legacy 接口、auto-invoke 接口不在本文覆盖范围。

---

## 15. 参考

- 概念与调用流：[framework.md](framework.md)
- 扩展新功能：[新增 Framework 服务](../skills/celestia-framework.md)
- System Core：[system-core.md](system-core.md)
- HAL：[hardware-hal.md](hardware-hal.md)
- API 索引：[guide/api-index.md](api-index.md)
