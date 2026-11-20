<p align="right"><b>English</b> | <a href="framework-api.md">中文</a></p>

# Framework API Reference (Public Queryable Interfaces)

> **Scope**: Public Framework API reference: per-service enums, structures, ops tables, and entry function signatures.
>
> For concepts, call flow, and boot timing see [framework.en.md](framework.en.md); for layering, contracts, and the runtime model see [Architecture overview](../architecture/overview.en.md); the [Glossary](../glossary.en.md) is the sole authority for terminology.

---

## 0. Quick reference index

| # | Module | Public header | `_open/close` entry | Main struct |
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
| 12 | Graphics Bitmap | `framework/ui/graphics/include/bitmap.h` | Direct functions | — |

> Runtime aggregate entry: `framework/runtime/include/runtime.h` `#include`s bootstrap/console/ota/otapack/reboot_info in one shot.

---

## 0.1 Common usage pattern (read first)

All Framework services follow the **same three-step pattern**:

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

**Return value conventions**:

- `0`: success
- `< 0`: error (encoded by `SYSTEM_ERROR_CODE(domain, module, code)`; see `system_error.h`)
- `> 0`: application data (e.g. byte count, frame count)

**Error code parsing pattern**:

```c
if (ret < 0) {
    int domain = (-ret >> 24) & 0x7F;
    int module = (-ret >> 16) & 0xFF;
    int code   = (-ret)        & 0xFFFF;
    LOGE("framework call fail: domain=%d module=0x%02x code=0x%04x", domain, module, code);
}
```

**Threading and lifetime**:

| Constraint | Description |
|---|---|
| Singleton | Multiple `*_open()` calls return the same handle; reference counting; matching `*_close()` calls |
| Ops callbacks | Invoked on Framework internal threads (capture, timer, network, etc.); **do not block or perform heavy work in callbacks**; **do not hold long-lived locks or call blocking methods on the same ops from a callback** |
| Handle validity | `svc` and `svc->xxx` remain valid until `svc_close`; do not reuse after close |
| Cross-thread | Handles may be shared across threads; ops methods are thread-safe (enforced by implementation); upper layers must avoid re-entrant callback deadlocks |
| Startup order | Recommended: `framework_init()` → `runtime_bootstrap_*` → other services `*_open` |

**Common errors and triage**:

| Return code (low 16 bits) | Scenario | What to check |
|---|---|---|
| `0x0001` INVALID_PARAM | Invalid pointer/parameter | Ensure structs and pointers are non-null |
| `0x0002` NOT_INIT | `*_open()` not called or `ops` is null | Check open return value and order |
| `0x0003` MEMORY | Allocation failed | Low system memory / handle pool exhausted |
| `0x0004` TIMEOUT | Timeout | Slow HAL/network; retry or adjust timeout |
| `0x0008` BUSY | Resource in use | stop/release before re-entry |
| `0x000A` NOT_SUPPORTED | Feature not built or unsupported on product | Check HAL capabilities for the SoC |
| `0x000B` INVALID_STATE | State machine error (e.g. start before prepare) | Check recorder/player state machines (Section 1.3) |

---

## 1. Multimedia service — `multimedia.h`

**Domain / module ID**: `FRAMEWORK_DOMAIN(0x01) / FRAMEWORK_MODULE_MULTIMEDIA(0x02)`
**Version macros**: `MULTIMEDIA_VERSION_MAJOR/MINOR/PATCH`, `MULTIMEDIA_VERSION`, `MULTIMEDIA_VERSION_STRING`
**Constants**: `MEDIA_MAX_CHN_NUM=3`, `MEDIA_MAX_RECT_BOX_COUNT=10`, `MEDIA_NUM_DATA_PTRS=4`, `MEDIA_AI_RECT_TARGET_MAX=64`

### 1.1 Default config file names

```c
#define MULTIMEDIA_CFG_VIDEO_FILE      "fw_video_config.json"
#define MULTIMEDIA_CFG_AUDIO_FILE      "fw_audio_config.ini"
#define MULTIMEDIA_CFG_OSD_FILE        "fw_osd_config.json"
#define MULTIMEDIA_CFG_AI_FILE         "fw_ai_config.ini"
#define MULTIMEDIA_CFG_OSD_FONT_FILE   "ak_font_16.bin"
#define MULTIMEDIA_CFG_OSD_LOGO_FILE   ""
```

### 1.2 Error codes (`multimedia_error_t = system_error_t`)

`MULTIMEDIA_OK`, `MULTIMEDIA_ERROR_INVALID_PARAM`, `_NOT_INIT`, `_MEMORY`, `_TIMEOUT`, `_NOT_FOUND`, `_ALREADY_EXISTS`, `_BUSY`, `_IO`, `_PERMISSION_DENIED`, `_NOT_SUPPORTED`, `_INVALID_STATE`, `_OPERATION_FAILED`

### 1.3 Enums

| Enum | Values | Description |
|------|----|------|
| `media_stream_state_t` | INIT/READY/PLAYING/PAUSED/STOPPED | Stream state |
| `media_type_t` | VIDEO/AUDIO/SUBTITLE | Media type |
| `media_video_codec_t` | H264/MJPEG/H265/MPEG4/NV12(0x100)/RGB24(0x101)/YV12(0x102) | Video codec |
| `media_audio_codec_t` | G726/MP3(2)/AAC(4)/PCM(6)/G711A(17)/G711U(18) | Audio codec |
| `media_ai_target_type_t` | HUMAN_SHAPE(0x1)/HUMAN_FACE(0x2)/PET(0x4)/VEHICLE(0x8)/NON_MOTOR_VEHICLE(0x10)/HUMAN_HAND(0x20)/PACKAGE(0x40)/CRY(0x80)/HUMAN_UPPER(0x100)/FACE_RECO(0x1000)/UPPER_RECO(0x2000)/MOTION(0x4000)/ALL(0xffffffff) | AI target bit mask |
| `media_audio_algorithm_t` | VAD/CRY/AEC/NS/ASLC/EQ/SSL | Audio algorithm type |
| `media_output_flags_t` | CLOUD_STORAGE(1<<0)/LIVE_STREAM(1<<1)/CARD_STORAGE(1<<2)/PREVIEW(1<<3)/TALK(1<<4)/SPEAKER(1<<5) | Output flag bits |
| `media_recorder_state_t` | IDLE/INITIALIZED/DATA_SOURCE_CONFIGURED/PREPARING/PREPARED/RECORDING/PAUSED/STOPPED/ERROR/RELEASED(99) | Recorder state machine |
| `media_player_state_t` | IDLE/INITIALIZED/PREPARING/PREPARED/STARTED/PAUSED/STOPPED/PLAYBACK_COMPLETE/ERROR/END(99) | Player state machine |
| `media_recorder_event_t` | PREPARED/STARTED/STOPPED/PAUSED/RESUMED/ERROR/INFO/MAX_DURATION_REACHED/MAX_FILE_SIZE_REACHED | Recorder events |
| `media_player_event_t` | PREPARED/STARTED/PAUSED/STOPPED/PLAYBACK_COMPLETE/ERROR/SEEK_COMPLETE/BUFFERING_START/BUFFERING_END/INFO | Player events |
| `media_player_scaling_mode_t` | SCALE_TO_FIT(1)/SCALE_TO_FIT_WITH_CROPPING | Video scaling |

### 1.4 Data structures

- Frame info: `media_frame_info_t`, `media_frame_pixel_t`, `media_frame_pcm_t`
- Encode parameters: `media_video_enc_param_t`, `media_audio_enc_param_t`
- Snapshot: `media_capture_picture_t`, `media_capture_picture_param_t`, `media_capture_rect_t`, `media_capture_cut_t`
- OSD/drawing: `media_rect_t`, `media_rgba_t`
- AI config: `media_ai_mode_config_t`, `media_ai_config_t`
- Video/audio info: `media_video_info_t`, `media_audio_info_t`
- Stream client: `media_stream_client_t`
- Recorder: `media_recorder_profile_t`, `media_dynamic_param_t`
- Player: `media_player_audio_attributes_t`
- Camera/audio: `media_stream_info_t`, `media_audio_src_info_t`, `media_audio_stream_info_t`
- Handles: `media_recorder_handle_t`, `media_player_handle_t`

### 1.5 Callback types

```c
typedef int (*media_frame_callback_t)(int handle, media_frame_info_t *frame, void *user_data);
typedef int (*media_msg_cb_t)(int handle, int msg_type, char *msg, int msg_len, void *user_data);
typedef int (*media_stream_callback_t)(media_recorder_handle_t, int stream_type, const void *frame_data, size_t frame_size, void *user_data);
typedef void (*media_recorder_event_callback_t)(media_recorder_handle_t, int event, int what, int extra, void *user_data);
typedef void (*media_player_event_callback_t)(media_player_handle_t, int event, int what, int extra, void *user_data);
typedef void (*media_player_video_show_callback_t)(int player_handle, media_frame_pixel_t *frame, void *user);
```

### 1.6 Ops structures

- `recorder_ops_t`: lifecycle (`create/destroy/reset/set_profile/get_profile/prepare/prepare_async/start/stop/pause/resume/is_paused`); data source (`set_output_file/set_output_stream/set_stream_callback`); encode parameters (`get_video_enc_param/get_audio_enc_param`); volume (`set_volume/get_volume`); dynamic parameters (`set/get_dynamic_param`); recording limits (`set_max_duration/set_max_file_size/set_orientation_hint`); state (`get_state/is_recording/get_last_error`); events (`set_event_callback`)
- `player_ops_t`: lifecycle; data source (`set_input_file/set_input_stream/set_realtime_source`); playback control (`prepare/prepare_async/start/stop/pause/resume/is_paused/set_playback_speed/get_playback_speed`); audio (`set_volume/get_volume/set_mute/get_mute/set_audio_attributes`); position (`seek/get_position/get_duration`); display (`set_window/get_video_width/get_video_height/set_video_scaling_mode/set_video_show_callback`); state; events
- `camera_ops_t`: stream config (`stream_get_configs/stream_get_id/stream_index_by_name/stream_index_by_role/stream_get_channel_size/stream_enable`); stream clients (`stream_reg_client/stream_get_frame/stream_unreg_client/stream_add_callback/stream_force_i_frame`); encode parameters (`stream_set_enc_param/stream_get_enc_param`); snapshot (`capture_picture`); effects (`get_effect/set_effect`); AI scene (`set_ai_scene/get_ai_scene/load_config/set_ai_enabled/ai_get_config/ai_set_config`)
- `audio_ops_t`: audio source enumeration (`audio_src_enum`); stream management (`stream_get_configs/load_config/stream_enable`); stream clients (`stream_reg_client/stream_get_frame/stream_unreg_client`); encode parameters; talk/playback (`talk_start/talk_send_data/talk_stop/play_notice/play_notice2`); volume (`out_set_volume/out_get_volume`); algorithms (`set_audio_algorithm_enable/set_audio_algorithm_param/get_max_amplitude`, legacy VAD/CRY/Echo parameters)
- `osd_ops_t`：`set_time_osd/set_text_osd`

### 1.7 Unified instance

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

### 1.8 Function list

```c
int multimedia_open(multimedia_t **multimedia);
int multimedia_close(multimedia_t *multimedia);
```

> Always follow `framework/services/multimedia/include/multimedia.h`. The symbols below are **internal implementation**, not public, and must not be referenced from application code:
> - `multimedia_is_camera_manager_initialized`
> - `multimedia_release_camera_manager`
> - `multimedia_teardown_osd_for_camera_close`
>
> To determine service state, use the `multimedia_open()` return code and your own `multimedia_t *` pointer (non-null means open). Framework services **do not** expose query APIs such as `<svc>_service_is_open()`.

### 1.9 Entry function reference

#### 1.9.1 `multimedia_open`

```c
int multimedia_open(multimedia_t **multimedia);
```

- **Purpose**: Open the multimedia service. The first call initializes all submodules (recorder/player/camera/audio/osd); later calls only increment the reference count.
- **Parameters**:
  - `multimedia` [out]: Receives the service singleton pointer, owned by the framework; the application **must not** `free` it.
- **Returns**: `0` on success; `< 0` is `MULTIMEDIA_ERROR_*`.
- **Common errors**: `_INVALID_PARAM` (null pointer), `_MEMORY`, `_OPERATION_FAILED` (HAL/config load failure).
- **Threads**: Any thread; protected internally by a mutex; re-entrant safe.
- **Prerequisites**: `framework_init()` has been called (SoC/log/HAL initialization complete).
- **Side effects**: Loads `MULTIMEDIA_CFG_*` config files; starts AV capture and the AI subsystem.

#### 1.9.2 `multimedia_close`

```c
int multimedia_close(multimedia_t *multimedia);
```

- **Purpose**: Decrement reference count; when it reaches 0, release resources (stop streams, release HAL, close callbacks).
- **Parameters**: `multimedia` [in]: Handle from `multimedia_open()`.
- **Returns**: `0` on success; `< 0` on error.
- **Threads**: Any thread.
- **Notes**: After this call **do not** use `multimedia->xxx->...`.

### 1.10 `recorder_ops_t` detailed function reference

> State machine: Section 1.3 `media_recorder_state_t`. In all functions, `handle` is the recorder handle returned by `create()`; `0` on success, `< 0` returns `MULTIMEDIA_ERROR_*`.

| Function | Signature | Purpose | Key parameters | Errors / state |
|---|---|---|---|---|
| `create` | `int create(media_recorder_handle_t *h)` | Create recorder instance | `h` [out] | `_MEMORY`; → `IDLE` |
| `destroy` | `int destroy(h)` | Destroy instance and release all resources | — | Idempotent |
| `reset` | `int reset(h)` | Reset to `IDLE` (keep instance) | — | `_INVALID_STATE` reset allowed from `ERROR` too |
| `set_profile` | `int set_profile(h, const media_recorder_profile_t *p)` | Set recording profile (resolution/bitrate/codec/output type) | `p` [in] | Only valid in `IDLE/INITIALIZED` |
| `get_profile` | `int get_profile(h, media_recorder_profile_t *p)` | Read currently active profile | `p` [out] | — |
| `prepare` | `int prepare(h, const media_recorder_profile_t *p)` | **Synchronously** prepare encoder/capture/muxer (`p` may be omitted to use prior `set_profile`) | `p` [in, nullable] | `_INVALID_STATE`; → `PREPARED` |
| `prepare_async` | `int prepare_async(h, const media_recorder_profile_t *p)` | **Asynchronously** prepare; fires `PREPARED` on completion | `p` [in, nullable] | → `PREPARING` → `PREPARED` |
| `start` | `int start(h)` | Start recording | — | Requires `PREPARED`; → `RECORDING` |
| `stop` | `int stop(h)` | Stop recording; close file/stream | — | Requires `RECORDING/PAUSED`; → `STOPPED` |
| `pause` | `int pause(h)` | Pause recording (file stays open) | — | Requires `RECORDING`; → `PAUSED` |
| `resume` | `int resume(h)` | Resume recording from pause | — | Requires `PAUSED`; → `RECORDING` |
| `is_paused` | `int is_paused(h, int *paused)` | Query pause state | `paused` [out] 0/1 | — |
| `set_output_file` | `int set_output_file(h, const char *file_path)` | Set output file or directory (directory mode auto-generates `YYYY-MM-DD/YYYYMMDD.HHMMSS.XXX.ext`) | `file_path` [in] | `_IO`; Only `INITIALIZED/DATA_SOURCE_CONFIGURED` |
| `set_output_stream` | `int set_output_stream(h, const char *url, const char *protocol)` | Set push URL and protocol (e.g. `"rtsp"`, `"rtmp"`) | `url`, `protocol` [in] | `_INVALID_PARAM` |
| `set_stream_callback` | `int set_stream_callback(h, int stream_type, media_stream_callback_t cb, void *user)` | Register post-encode frame callback (muxed H264/AAC, etc.) | `stream_type` uses `media_type_t`; `cb`/`user` | Callback on encode thread |
| `get_video_enc_param` | `int get_video_enc_param(h, media_video_enc_param_t *p)` | Get effective video encode parameters (policy-selected) | `p` [out] | — |
| `get_audio_enc_param` | `int get_audio_enc_param(h, media_audio_enc_param_t *p)` | Get effective audio encode parameters | `p` [out] | — |
| `set_volume` | `int set_volume(h, int volume)` | Recording input volume (0–100); not HAL output volume | `volume` 0–100 | `_INVALID_PARAM` |
| `get_volume` | `int get_volume(h, int *volume)` | Get recording input volume | `volume` [out] | — |
| `set_dynamic_param` | `int set_dynamic_param(h, const media_dynamic_param_t *p)` | Adjust bitrate/GOP, etc. at runtime | `p` [in] | Adjustable while running |
| `get_dynamic_param` | `int get_dynamic_param(h, media_dynamic_param_t *p)` | Read dynamic parameter snapshot | `p` [out] | — |
| `set_max_duration` | `int set_max_duration(h, long long max_duration_ms)` | Max duration per file; overrun fires `MAX_DURATION_REACHED` | Milliseconds; 0 = unlimited | — |
| `set_max_file_size` | `int set_max_file_size(h, long long bytes)` | Max bytes per file; overrun fires `MAX_FILE_SIZE_REACHED` | Bytes; 0 = unlimited | — |
| `set_orientation_hint` | `int set_orientation_hint(h, int degrees)` | Write MP4 rotation hint (0/90/180/270) | Degrees | `_INVALID_PARAM` |
| `get_state` | `int get_state(h, media_recorder_state_t *s)` | Read current state machine | `s` [out] | Valid in any state |
| `is_recording` | `int is_recording(h, int *recording)` | Whether in `RECORDING` | `recording` [out] 0/1 | — |
| `get_last_error` | `int get_last_error(h, multimedia_error_t *code)` | Read most recent internal error | `code` [out] | — |
| `set_event_callback` | `int set_event_callback(h, media_recorder_event_callback_t cb, void *user)` | Register state/error/limit event callback | `cb`/`user` | Callback on internal thread; **do not block** |

### 1.11 `player_ops_t` detailed function reference

> State machine: Section 1.3 `media_player_state_t`.

| Function | Signature | Purpose | Key parameters | Errors / state |
|---|---|---|---|---|
| `create` | `int create(media_player_handle_t *h)` | Create player instance | `h` [out] | `_MEMORY`; → `IDLE` |
| `destroy` | `int destroy(h)` | Destroy player | — | Idempotent |
| `reset` | `int reset(h)` | Reset to `IDLE` | — | — |
| `set_input_file` | `int set_input_file(h, const char *file_path)` | Set local media source | `file_path` [in] | `_IO` |
| `set_input_stream` | `int set_input_stream(h, const char *url, const char *protocol)` | Set network stream source (RTSP/HTTP, etc.) | `url`, `protocol` [in] | `_INVALID_PARAM` |
| `set_realtime_source` | `int set_realtime_source(h, int camera_id, int channel, int audio_source_id)` | Use camera/audio capture as input (live talk, preview) | `camera_id`, `channel`, `audio_source_id` | `_NOT_FOUND` |
| `prepare` | `int prepare(h)` | **Synchronously** prepare decode/render | — | → `PREPARED` |
| `prepare_async` | `int prepare_async(h)` | **Asynchronously** prepare; completion event `PREPARED` | — | → `PREPARING` → `PREPARED` |
| `start` | `int start(h)` | Start playback | — | Requires `PREPARED/PAUSED`; → `STARTED` |
| `stop` | `int stop(h)` | Stop playback | — | → `STOPPED` |
| `pause` | `int pause(h)` | Pause | — | → `PAUSED` |
| `resume` | `int resume(h)` | Resume playback | — | → `STARTED` |
| `is_paused` | `int is_paused(h, int *paused)` | Query pause state | `paused` [out] | — |
| `set_playback_speed` | `int set_playback_speed(h, float speed)` | Set playback speed (0.5/1.0/2.0, etc.) | `speed` > 0 | `_NOT_SUPPORTED` if decoder unsupported |
| `get_playback_speed` | `int get_playback_speed(h, float *speed)` | Get current playback speed | `speed` [out] | — |
| `set_volume` | `int set_volume(h, int volume)` | Player output volume (0–100, software mix) | 0–100 | — |
| `get_volume` | `int get_volume(h, int *volume)` | Get volume | `volume` [out] | — |
| `set_mute` | `int set_mute(h, int mute)` | Mute on/off | 0/1 | — |
| `get_mute` | `int get_mute(h, int *mute)` | Query mute | `mute` [out] | — |
| `set_audio_attributes` | `int set_audio_attributes(h, const media_player_audio_attributes_t *a)` | Set audio attributes (channels/sample rate, etc.) | `a` [in] | Only valid in `IDLE/INITIALIZED` |
| `seek` | `int seek(h, long long position_ms)` | Seek to playback position | ms | Fires `SEEK_COMPLETE` |
| `get_position` | `int get_position(h, long long *position_ms)` | Read current playback position | [out] ms | — |
| `get_duration` | `int get_duration(h, long long *duration_ms)` | Read total duration (streaming may be `-1`) | [out] ms | — |
| `set_window` | `int set_window(h, int window_handle)` | Bind render window (FB/overlay handle) | `window_handle` | `_NOT_SUPPORTED` no display use case |
| `get_video_width` | `int get_video_width(h, int *width)` | Read video width | [out] | Requires `PREPARED` |
| `get_video_height` | `int get_video_height(h, int *height)` | Read video height | [out] | Requires `PREPARED` |
| `set_video_scaling_mode` | `int set_video_scaling_mode(h, media_player_scaling_mode_t mode)` | Set scaling mode (letterbox/crop) | `mode` | — |
| `set_video_show_callback` | `int set_video_show_callback(h, media_player_video_show_callback_t cb, void *user)` | Register video frame render callback (external draw) | `cb`/`user` | Callback on render thread; **do not block** |
| `get_state` | `int get_state(h, media_player_state_t *s)` | Read state machine | [out] | — |
| `is_playing` | `int is_playing(h, int *playing)` | Whether playing | [out] 0/1 | — |
| `get_last_error` | `int get_last_error(h, multimedia_error_t *code)` | Read most recent error | [out] | — |
| `set_event_callback` | `int set_event_callback(h, media_player_event_callback_t cb, void *user)` | Event callback (PREPARED/STARTED/SEEK_COMPLETE/BUFFERING_*/ERROR…) | `cb`/`user` | Callback on internal thread |

### 1.12 `camera_ops_t` detailed function reference

> For **bypass** direct access to capture streams: AI, preview, re-encode, and other advanced cases. Prefer `recorder_ops_t` for normal recording.

| Function | Signature | Purpose | Key parameters / return | Notes |
|---|---|---|---|---|
| `stream_get_configs` | `int stream_get_configs(media_stream_info_t **streams, int *num)` | Get all stream metadata (from `fw_video_config.json`) | `streams` [out, internal buffer; do not free], `num` [out] | Read-only; callable anytime |
| `stream_get_id` | `int stream_get_id(int camera_id, int channel)` | Resolve `stream_id` from (camera, channel) | Returns `stream_id`; `< 0` if not found | — |
| `stream_index_by_name` | `int stream_index_by_name(const char *name, int *idx)` | Exact match on `stream_name` | `idx` [out] | `_NOT_FOUND` |
| `stream_index_by_role` | `int stream_index_by_role(const char *role, int *idx)` | Match by role (`main/preview/ai/qr`) | `idx` [out] | Auto-fallback by name/type |
| `stream_get_channel_size` | `int stream_get_channel_size(int stream, int *w, int *h)` | Get channel width/height | `w`/`h` [out] | From JSON `channel_config` |
| `stream_enable` | `int stream_enable(int stream, int enable)` | Enable/disable a stream | `enable` 0/1 | Enable/disable broadcast to all clients |
| `stream_reg_client` | `int stream_reg_client(int stream, int *client_handle, media_frame_callback_t cb, void *user)` | Register stream client (frame dispatch) | `client_handle` [out]; `cb` `NULL` callback uses pull mode | Pair with `stream_get_frame` |
| `stream_get_frame` | `int stream_get_frame(int client_handle, media_frame_info_t *frame)` | **Pull mode**: fetch one frame | `frame` [out]，references internal buffer | Caller must not `free(frame->data[*])` |
| `stream_unreg_client` | `int stream_unreg_client(int client_handle)` | Unregister client | — | Idempotent |
| `stream_add_callback` | `int stream_add_callback(int stream, int *client_handle, media_frame_callback_t cb, void *user)` | Add another callback client on existing stream | `client_handle` [out] | Simplified `stream_reg_client` |
| `stream_force_i_frame` | `int stream_force_i_frame(int stream)` | Force one I-frame (e.g. for join-in-progress) | — | Meaningful for H264/H265 only |
| `stream_set_enc_param` | `int stream_set_enc_param(int stream, media_video_enc_param_t *p)` | Set video encode parameters | `p` [in] | Effective while running |
| `stream_get_enc_param` | `int stream_get_enc_param(int stream, media_video_enc_param_t *p)` | Read video encode parameters | `p` [out] | — |
| `capture_picture` | `int capture_picture(int stream, const media_capture_picture_param_t *param, void **data, size_t *size)` | Synchronous snapshot | `param` [in], `data` [out]（internal buffer）, `size` [out] | Thread-safe; internal buffer reused by next `capture_picture`—copy promptly after use |
| `get_effect` | `int get_effect(int camera, int type, int *value)` | Read image effect (brightness/contrast/saturation, etc.) | `type`：implementation-defined; `value` [out] | — |
| `set_effect` | `int set_effect(int camera, int type, int value)` | Set image effect | Same as above | Takes effect immediately |
| `set_ai_scene` | `int set_ai_scene(int scene_id)` | Switch AI scene (multi-face/night, etc.) | `scene_id` | Affects subsequent AI detection |
| `get_ai_scene` | `int get_ai_scene(void)` | Read current AI scene | Returns `scene_id` | — |
| `load_config` | `int load_config(const char *video_cfg_path, const char *ai_cfg_path)` | Hot-reload video/AI config files | Two paths; either may be `NULL` | `_IO` |
| `set_ai_enabled` | `void set_ai_enabled(int enable)` | Global AI enable switch | `enable` 0/1 | No return value |
| `ai_get_config` | `int ai_get_config(media_ai_config_t *out, int *mode_count)` | Read AI config snapshot | `out`, `mode_count` [out] | — |
| `ai_set_config` | `int ai_set_config(media_ai_config_t *config)` | Apply AI config | `config` [in] | Adjustable while running |

### 1.13 `audio_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `audio_src_enum` | `int audio_src_enum(media_audio_src_info_t **srcs, int *num)` | Enumerate audio sources (mic/I2S, etc.) | Internal buffer; do not `free` |
| `stream_get_configs` | `int stream_get_configs(media_audio_stream_info_t **streams, int *num)` | Enumerate audio streams | Same as above |
| `load_config` | `int load_config(const char *audio_cfg_path)` | Hot-reload audio config | `_IO` |
| `stream_enable` | `int stream_enable(int stream, int enable)` | Enable/disable audio stream | — |
| `stream_reg_client` | `int stream_reg_client(int stream, int *client_handle, media_frame_callback_t cb, void *user)` | Register audio stream client | Callback may be `NULL` for pull mode |
| `stream_get_frame` | `int stream_get_frame(int client_handle, media_frame_info_t *frame)` | Pull one PCM/compressed audio frame | Frame buffer managed internally |
| `stream_unreg_client` | `int stream_unreg_client(int client_handle)` | Unregister | Idempotent |
| `stream_set_enc_param` | `int stream_set_enc_param(int stream, media_audio_enc_param_t *p)` | Set audio encode parameters | — |
| `stream_get_enc_param` | `int stream_get_enc_param(int stream, media_audio_enc_param_t *p)` | Read audio encode parameters | — |
| `talk_start` | `int talk_start(int dev)` | **Start talk (in/out together)**; AEC prerequisites | `dev` default `0`; Match input/output sample rates |
| `talk_send_data` | `int talk_send_data(int dev, media_frame_info_t *frame)` | Send PCM/encoded audio frame to peer | Call rate follows frame duration |
| `talk_stop` | `int talk_stop(int dev)` | End talk | — |
| `play_notice` | `int play_notice(int dev, const char *file_path)` | Play notice asynchronously (pcm/wav/mp3) | Non-blocking |
| `play_notice2` | `int play_notice2(int dev, const char *file_path, int wait_time_ms, int volume)` | Same, with max wait and separate volume | `wait_time_ms < 0` blocks until complete |
| `out_set_volume` | `int out_set_volume(int dev, int volume)` | **Hardware output volume** (HAL) | 0–100 |
| `out_get_volume` | `int out_get_volume(int dev, int *volume)` | Read hardware output volume | — |
| `set_audio_algorithm_enable` | `int set_audio_algorithm_enable(int dev, media_audio_algorithm_t algorithm, int enable)` | Enable/disable algorithm (VAD/CRY/**AEC**/NS/ASLC/EQ/SSL) | AEC only effective in talk mode |
| `set_audio_algorithm_param` | `int set_audio_algorithm_param(int dev, media_audio_algorithm_t alg, const char *name, const void *value, size_t size)` | Set algorithm parameter (blob) | `name` Defined by algorithm |
| `get_max_amplitude` | `int get_max_amplitude(int dev, int *amplitude)` | Read latest sample peak (VU meter) | `amplitude` [out] |

> The members below are **legacy**, kept for binary compatibility; new code must use `set_audio_algorithm_enable/_param` with `media_audio_algorithm_t`; not detailed in this document:
> `set_vad_param` / `get_vad_param` / `set_cry_param` / `get_cry_param` / `set_echo_param` / `get_echo_param`。

### 1.14 `osd_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `set_time_osd` | `int set_time_osd(int camera, int channel, int enable)` | Enable/disable timestamp overlay | Position/font from `fw_osd_config.json` |
| `set_text_osd` | `int set_text_osd(int camera, int channel, const char *text)` | Set/update channel text | `text` Must not exceed OSD max chars; UTF-8 |

---

## 2. Network service — `network.h`

**Domain / module ID**: `FRAMEWORK_DOMAIN / FRAMEWORK_MODULE_NETWORK(0x00)`
**Version macros**: `NETWORK_VERSION_*`
**Error code prefix**: `NETWORK_OK/NETWORK_ERROR_*`(including `_CONNECTION_FAILED/_DEVICE_NOT_AVAILABLE/_NETWORK_UNREACHABLE/_INTERFACE_NOT_FOUND/_CONFIG_ERROR/_NOT_IMPLEMENTED`）

### 2.1 Enums

- `network_wifi_state_e`：DISABLED/DISABLING/ENABLED/ENABLING/UNKNOWN
- `network_wifi_station_state_e`：DISCONNECTED/CONNECTING/CONNECTED/DISCONNECTING
- `network_wifi_ap_state_e`：DISABLED/ENABLING/ENABLED/DISABLING
- `network_wifi_p2p_state_e`：DISABLED/ENABLING/ENABLED/DISABLING/DISCOVERING/CONNECTING/CONNECTED
- `network_wifi_status_e`：IDLE/CONNECTED/CONNECTING/ADDRESSED/PSK_FAIL/AP_NOT_FOUND
- `network_wifi_event_type_e`：SCAN_RESULTS_AVAILABLE/STATE_CHANGED/CONNECTION_CHANGED/LINK_SPEED_CHANGED/RSSI_CHANGED/NETWORK_CONNECTED/NETWORK_DISCONNECTED/NETWORK_CONNECTION_FAILED
- `network_rtsp_param_type_e`：IPV4/IPV6/MAX
- `network_rtsp_audio_type_e`：G726/AAC(4)/PCM_ALAW(17)/PCM_ULAW(18)
- `network_rtsp_video_type_e`：H264/MJPEG/H265/MAX

### 2.2 Structures

- Configuration: `network_wifi_station_config_t{ssid[33], passphrase[64], security_type}`, `network_wifi_ap_config_t{ssid, passphrase, security_type, channel, max_clients}`
- Scan results: `network_wifi_scan_result_t{ssid, bssid[18], frequency, rssi, flags}`
- AP clients: `network_wifi_ap_client_t{mac_address[18], ip_address[16], rssi}`
- Events: `network_wifi_event_t{type, data, data_len}`
- RTSP：`network_rtsp_video_enc_param_t`, `network_rtsp_audio_enc_param_t`, `network_rtsp_frame_info_t`

### 2.3 Callbacks

```c
typedef void (*network_wifi_event_callback_t)(int event, void *data);
typedef int  (*network_wifi_status_callback_t)(network_wifi_status_e status);
```

### 2.4 Ops structures

| Ops | Main functions |
|-----|---------|
| `network_wifi_station_ops_t` | `init/deinit/connect/disconnect/reconnect/get_state/get_ssid/get_bssid/get_rssi/get_link_speed/get_ip_address/scan_start/scan_get_results/scan_stop` |
| `network_wifi_ap_ops_t` | `init/deinit/set_config/get_config/start/stop/is_started/get_state/get_connected_clients/get_max_clients` |
| `network_wifi_p2p_ops_t` | `init/deinit/discover_peers/stop_discover_peers/get_discovered_peers/connect/disconnect/get_state/create_group/remove_group/is_group_owner/get_group_info` |
| `network_wifi_ops_t` | Manager: `init/deinit/enable/disable/is_enabled/get_state`; Event：`register_event_callback/unregister_event_callback/set_status_callback`; Info: `get_ip/get_mac`; Submodules: `station/ap/p2p` |
| `network_ethernet_ops_t` | `get_ip/get_ipv6/get_mac/get_ip_by_name/get_ipv6_by_name/get_mac_by_name` |
| `network_ntp_ops_t` | `init/deinit/set_server` |
| `network_ping_ops_t` | `ping_ipv4/ping_ipv6` |
| `network_rtsp_ops_t` | `init/deinit/start/stop/set_video_param_callback/set_audio_param_callback/set_dev_param_callback` |

### 2.5 Unified instance

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

### 2.6 Entry function reference

#### 2.6.1 `network_open`

```c
int network_open(network_t **network);
```

- **Purpose**: Open network service (WiFi Manager + Ethernet + NTP + Ping + RTSP control); first call initializes drivers and event thread.
- **Parameters**: `network` [out] Service singleton pointer.
- **Returns**: `0` on success; `< 0` is `NETWORK_ERROR_*`.
- **Threads**:Any thread; internal reference counting.
- **Prerequisites**: `framework_init()` done; on products without WiFi, `wifi` submodule may be `NULL`.

#### 2.6.2 `network_close`

```c
int network_close(network_t *network);
```

- **Purpose**: Decrement reference; last close releases event thread and network submodules.
- **Returns**: `0` / `< 0`。

### 2.7 `network_wifi_ops_t` (Manager) detailed function reference

| Function | Signature | Purpose | Return / notes |
|---|---|---|---|
| `init` | `int init(void)` | Initialize WiFi manager (driver, event queue) | Idempotent; repeated calls return same result |
| `deinit` | `int deinit(void)` | Deinitialize; release resources | Disconnect station/stop AP before exit |
| `enable` | `int enable(void)` | Power on WiFi RF | `_DEVICE_NOT_AVAILABLE` no chip |
| `disable` | `int disable(void)` | Power off WiFi | Auto-disconnects station/stops AP |
| `is_enabled` | `int is_enabled(int *enabled)` | Query RF power state | `enabled` [out] 0/1 |
| `get_state` | `int get_state(network_wifi_state_e *state)` | Read Manager state machine | `DISABLED/DISABLING/ENABLED/ENABLING/UNKNOWN` |
| `register_event_callback` | `int register_event_callback(network_wifi_event_callback_t cb)` | Register WiFi event callback (scan/connect/RSSI, etc.) | Multiple different `cb` allowed |
| `unregister_event_callback` | `int unregister_event_callback(network_wifi_event_callback_t cb)` | Unregister callback | Match by address |
| `set_status_callback` | `int set_status_callback(network_wifi_status_callback_t cb)` | Register connection status callback (IDLE/CONNECTING, etc.) | Only one; new call overwrites |
| `get_ip` | `int get_ip(char *ip)` | Get current WiFi IPv4 (`"a.b.c.d"`) | Buffer ≥ 16 |
| `get_mac` | `int get_mac(char *mac)` | Get MAC (`"xx:xx:xx:xx:xx:xx"`) | Buffer ≥ 18 |
| Sub-ops | `station / ap / p2p` | Three submodules; see Sections 2.8–2.10 | — |

### 2.8 `network_wifi_station_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` | `int init(void)` | Initialize station mode | Call after `enable()` |
| `deinit` | `int deinit(void)` | Deinitialize | — |
| `connect` | `int connect(const network_wifi_station_config_t *config)` | Connect to SSID | `config->ssid/passphrase/security_type` |
| `disconnect` | `int disconnect(void)` | Disconnect | — |
| `reconnect` | `int reconnect(void)` | Reconnect with last parameters | `_NOT_INIT` if no history |
| `get_state` | `int get_state(network_wifi_station_state_e *state)` | Read station state machine | — |
| `get_ssid` | `int get_ssid(char *ssid, size_t len)` | Current AP SSID | `len ≥ 33` |
| `get_bssid` | `int get_bssid(char *bssid, size_t len)` | Current AP BSSID | `len ≥ 18` |
| `get_rssi` | `int get_rssi(int *rssi)` | Signal strength (dBm) | `rssi` [out] |
| `get_link_speed` | `int get_link_speed(int *speed_mbps)` | Current link speed | Mbps |
| `get_ip_address` | `int get_ip_address(char *ip, size_t len)` | Get DHCP-assigned IP | `len ≥ 16` |
| `scan_start` | `int scan_start(void)` | Start scan; results via `SCAN_RESULTS_AVAILABLE` event | Asynchronous |
| `scan_get_results` | `int scan_get_results(network_wifi_scan_result_t *results, int *num)` | Synchronously fetch scan results | `num` in=capacity, out=actual count |
| `scan_stop` | `int scan_stop(void)` | Stop scan | Idempotent |

### 2.9 `network_wifi_ap_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` / `deinit` | — | AP mode lifecycle | — |
| `set_config` | `int set_config(const network_wifi_ap_config_t *c)` | Set SSID/passphrase/channel/max clients | Set before start |
| `get_config` | `int get_config(network_wifi_ap_config_t *c)` | Read current config | — |
| `start` | `int start(void)` | Start AP | `_CONFIG_ERROR` if `set_config` not called |
| `stop` | `int stop(void)` | Stop AP | — |
| `is_started` | `int is_started(int *started)` | Query whether running | `started` [out] 0/1 |
| `get_state` | `int get_state(network_wifi_ap_state_e *state)` | Read AP state machine | — |
| `get_connected_clients` | `int get_connected_clients(network_wifi_ap_client_t *clients, int *num)` | Enumerate connected STAs | `num` in=capacity, out=actual count |
| `get_max_clients` | `int get_max_clients(int *max_clients)` | Read max connections (from config) | — |

### 2.10 `network_wifi_p2p_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` / `deinit` | — | P2P lifecycle | — |
| `discover_peers` | `int discover_peers(void)` | Start discovery | Results via WiFi events |
| `stop_discover_peers` | `int stop_discover_peers(void)` | Stop discovery | — |
| `get_discovered_peers` | `int get_discovered_peers(void *devices, int *num_devices)` | Get discovered device list | `devices` type defined by implementation |
| `connect` | `int connect(const char *device_address)` | Connect to P2P device | MAC string |
| `disconnect` | `int disconnect(void)` | Disconnect | — |
| `get_state` | `int get_state(network_wifi_p2p_state_e *state)` | Read P2P state machine | — |
| `create_group` | `int create_group(void)` | Create P2P group (as GO) | — |
| `remove_group` | `int remove_group(void)` | Remove group | — |
| `is_group_owner` | `int is_group_owner(int *is_owner)` | Whether local device is GO | `is_owner` [out] 0/1 |
| `get_group_info` | `int get_group_info(void *info)` | Read group info | `info` type defined by implementation |

### 2.11 `network_ethernet_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `get_ip` | `int get_ip(char *ip)` | Default Ethernet IPv4 | Buffer ≥ 16 |
| `get_ipv6` | `int get_ipv6(char *ip)` | Default interface IPv6 | Buffer ≥ 46 |
| `get_mac` | `int get_mac(char *mac)` | Default interface MAC | Buffer ≥ 18 |
| `get_ip_by_name` | `int get_ip_by_name(const char *iface, char *ip, size_t len)` | Read IPv4 by interface name | `iface` e.g. `"eth0" |
| `get_ipv6_by_name` | `int get_ipv6_by_name(const char *iface, char *ip, size_t len)` | Read IPv6 by interface name | — |
| `get_mac_by_name` | `int get_mac_by_name(const char *iface, char *mac, size_t len)` | Read MAC by interface name | — |

### 2.12 `network_ntp_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` | `int init(void)` | Initialize NTP client (background thread) | Idempotent |
| `deinit` | `int deinit(void)` | Deinitialize | — |
| `set_server` | `int set_server(char *server_ip)` | Set NTP server (IP or hostname) | Triggers one sync immediately |

### 2.13 `network_ping_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `ping_ipv4` | `int ping_ipv4(const char *dest, int ttl, int sequence, int timeout)` | Synchronous ICMP ping | `timeout` ms; returns RTT (ms) or `<0` on error |
| `ping_ipv6` | `int ping_ipv6(const char *dest, int ttl, int sequence, int timeout)` | IPv6 variant | Same as above |

### 2.14 `network_rtsp_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` / `deinit` | — | RTSP control lifecycle | — |
| `start` | `int start(void)` | Start RTSP service | Requires `init` first |
| `stop` | `int stop(void)` | Stop RTSP service | — |
| `set_video_param_callback` | `int set_video_param_callback(void *cb)` | Inject video encode parameter supply callback (signature defined by gateway) | Links to Multimedia |
| `set_audio_param_callback` | `int set_audio_param_callback(void *cb)` | Same for audio encode parameters | — |
| `set_dev_param_callback` | `int set_dev_param_callback(void *cb)` | Device description callback (vendor/version/SN) | — |

---

## 3. Notification service — `notification.h`

**Module ID**: `FRAMEWORK_MODULE_NOTIFICATION(0x01)`

### 3.1 Enums

- `alarm_type_t`：MD/MAN_BODY/VOICE/FACE/CRY/PREROLL/VEHICLE_GEN/VEHICLE_MOTOR/VEHICLE_BIKE/ALERTAREA/TRANSGRESSION/PET/PACKAGE/OFF_DUTY/PFS/RPS/FENCE/INOUT/WANDER/CALL_TO_SERVER_S/CALL_TO_SERVER_E/NUM/ALL(0xffffffff)
- `alarm_state_t`：END(0)/START(1)
- `message_type_t`：SLEEP/WAKEUP/INVERSION/LSSL/TALK_IN/TALK_OUT/NET_CONNECT/NUM/ALL
- `input_device_type_t`：UNKNOWN/KEYPAD/TOUCH/MOUSE/ENCODER/SENSOR_PIR
- `input_action_t`：DOWN/UP/REPEAT/LONGPRESS/DOUBLE_CLICK/TRIPLE_CLICK
- `input_keycode_t`: full Linux-style keycode list (digits/letters/HOME/BACK/MENU/OK/ENTER/ESC/arrows/POWER/CONFIG/RESET/TALK/MODE_SWITCH/volume/media/camera/brightness/TORCH/color keys/FUNC_1..3)

### 3.2 Structures

```c
#define ALARM_DETAIL_LEN 1024
typedef struct { alarm_type_t type; long long timestamp_ms; alarm_state_t state; char detail[ALARM_DETAIL_LEN]; int detail_len; } alarm_t;
typedef struct { message_type_t type; long long timestamp_ms; char *data; int data_len; } message_t;
typedef struct { uint64_t timestamp_ms; input_device_type_t device; int device_id; input_keycode_t keycode; int linux_code; input_action_t action; int value; int meta_state; } input_event_t;
```

### 3.3 Callbacks

```c
typedef int  (*alarm_callback_t)(alarm_t *alarm, int msg_len, void *user_data);
typedef int  (*message_callback_t)(message_t *msg, void *user_data);
typedef int  (*input_listener_cb)(const input_event_t *ev, void *user_data);
```

### 3.4 Ops structures

| Ops | Functions |
|-----|------|
| `alarm_ops_t` | `init/uninit/post/detach/reg_client/unreg_client` |
| `message_ops_t` | `init/uninit/post/reg_client/unreg_client` |
| `message_queue_ops_t` | `init/uninit/create/destroy/read/write` |
| `input_ops_t` | `init/deinit/subscribe/unsubscribe/set_focus/set_debounce_ms/set_repeat_ms` |

### 3.5 Unified instance

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

### 3.6 Entry function reference

#### 3.6.1 `notification_open`

```c
int notification_open(notification_t **notification);
```

- **Purpose**: Open notification service (alarms/messages/queues/input events).
- **Returns**: `0` on success; `< 0` on error.
- **Threads**: Any thread.

#### 3.6.2 `notification_close`

```c
int notification_close(notification_t *notification);
```

- **Purpose**: Decrement reference count; last close releases dispatcher thread.

### 3.7 `alarm_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` | `int init(void)` | Initialize alarm dispatcher | Idempotent |
| `uninit` | `int uninit(void)` | Deinitialize | — |
| `post` | `int post(alarm_type_t type, char *data, int data_len)` | Post alarm event (from capture/AI/HAL) | `data` may be `NULL`; `data_len ≤ ALARM_DETAIL_LEN` |
| `detach` | `int detach(alarm_type_t type)` | Clear subscribers for type (debug) | — |
| `reg_client` | `int reg_client(alarm_type_t type, alarm_callback_t cb, void *user, int min_interval)` | Subscribe to alarm type | `min_interval` Min interval (ms) for debounce |
| `unreg_client` | `int unreg_client(alarm_type_t type, alarm_callback_t cb)` | Unsubscribe | Match `type + cb` |

### 3.8 `message_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` / `uninit` | — | Message dispatcher lifecycle | — |
| `post` | `int post(message_type_t type, char *data, int data_len)` | Publish system message (SLEEP/WAKEUP/TALK_IN, etc.) | — |
| `reg_client` | `int reg_client(message_type_t type, message_callback_t cb, void *user)` | Subscribe to message type | — |
| `unreg_client` | `int unreg_client(message_type_t type, message_callback_t cb)` | Unsubscribe | — |

### 3.9 `message_queue_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` / `uninit` | — | Module lifecycle | — |
| `create` | `int create(const char *queue_name)` | Create named queue; returns handle (`> 0`) | Same name returns same handle |
| `destroy` | `int destroy(const char *queue_name)` | Destroy queue | Idempotent |
| `read` | `int read(int handle, char *msg, int *msg_len, long long *msg_time_ms)` | Blocking read one message | `msg_len` in=capacity, out=actual; `msg_time_ms` may be `NULL` |
| `write` | `int write(int handle, char *msg, int msg_len)` | Write one message to queue | On full queue, drop/overwrite is implementation-defined |

### 3.10 `input_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` | `int init(void)` | Initialize input dispatch (keypad/touch/mouse/sensor) | — |
| `deinit` | `int deinit(void)` | Deinitialize | — |
| `subscribe` | `int subscribe(int priority, uint32_t keymask, input_listener_cb cb, void *user)` | Subscribe to input; **high-priority** listener may consume events | `keymask` selects `input_keycode_t` bits |
| `unsubscribe` | `int unsubscribe(input_listener_cb cb, void *user)` | Unsubscribe | Match `cb+user` |
| `set_focus` | `int set_focus(void *token)` | Bind focus token; only that token receives events | `NULL` clears focus |
| `set_debounce_ms` | `int set_debounce_ms(int ms)` | Set key debounce (ms) | — |
| `set_repeat_ms` | `int set_repeat_ms(int ms)` | Set long-press repeat interval | `0` disables |

---

## 4. Device service — `device.h`

**Module ID**: `FRAMEWORK_MODULE_DEVICE(0x04)`
**Version macros / structures**: `DEVICE_VERSION_*`, `device_version_info_t`, `device_get_version()`, `device_get_version_string()`

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

- Commands: `ptz_cmd_t`（UP/DOWN/LEFT/RIGHT/LEFT_UP/RIGHT_UP/LEFT_DOWN/RIGHT_DOWN/STOP/UP_STEP/DOWN_STEP/LEFT_STEP/RIGHT_STEP/ZOOM_NEAR/ZOOM_FAR/FOCUS_NEAR/FOCUS_FAR/IRIS_OPEN/IRIS_CLOSE）
- States: `ptz_state_t`（STOP/RUNNING）
- Tracking: `ptz_track_mode_t`（DISABLE/MOTION/BODY/PET/VEHICLE）
- Position: `ptz_position_t{pan, tilt, fov_h, fov_v, zoom_value, state}`
- Presets: `ptz_preset_info_t{id, name[20], pan, tilt, zoom}`

`ptz_ops_t`：`init/deinit/control/set_position/get_position/set_speed/get_speed/get_state/reset/set_fov/preset_set/preset_goto/preset_delete/preset_get_count/preset_get_list/preset_get_by_name/tracking_set_enable/tracking_get_enable/tracking_set_mode/tracking_get_mode/tracking_set_temporary_switch/tracking_handle_motion/tracking_handle_human`

### 4.3 Power

- Enums: `power_manager_suspend_mode_t`（NORMAL/DEEP）
- Hooks: `power_manager_hook_t`

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

### 4.6 Unified instance

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

### 4.7 Submodule error codes

- LED：`DEVICE_ERROR_LED_NOT_FOUND/_INVALID_TYPE/_INVALID_MODE/_SCENE_CONFLICT`
- PTZ：`_PTZ_NOT_READY/_POSITION_LIMIT/_MOTOR_FAULT/_PRESET_FULL/_PRESET_NOT_FOUND/_TRACKING_FAILED`
- Night vision: `_NV_SENSOR_FAILED/_IRCUT_FAILED/_IRLED_FAILED/_SWITCH_FAILED/_ISP_NOT_READY`
- Backlight: `_BL_PWM_FAILED/_GPIO_FAILED/_BRIGHTNESS_INVALID`

### 4.8 Entry function reference

```c
int  device_open(device_t **device);
int  device_close(device_t *device);
uint32_t device_get_version(void);
const char *device_get_version_string(void);
```

- `device_open`：First call initializes hardware control submodules; some may be `NULL` per product form factor.
- `device_close`：Decrement reference count and release.
- `device_get_version` / `_string`：Compile-time version; no handle required; callable anytime.

### 4.9 `led_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` | `int init(void)` | Initialize LED manager | Idempotent |
| `deinit` | `void deinit(void)` | Deinitialize | No return value |
| `set_scene` | `int set_scene(const led_scene_config_t *config)` | Activate LED by scene (with priority) | Higher priority overrides lower |
| `clear_scene` | `int clear_scene(led_scene_t scene)` | Clear a scene | Falls back to next priority scene |
| `get_active_scene` | `int get_active_scene(led_scene_t *scene)` | Read active scene | — |
| `set_led` | `int set_led(led_type_t type, led_mode_t mode, uint32_t brightness)` | Control single LED directly | `brightness` 0–255 |
| `set_blink` | `int set_blink(led_type_t type, uint32_t on_ms, uint32_t off_ms)` | Set blink parameters | — |
| `get_led` | `int get_led(led_type_t type, led_mode_t *mode, uint32_t *brightness)` | Read current state | — |

### 4.10 `ptz_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` / `deinit` | — | Lifecycle | — |
| `control` | `int control(ptz_cmd_t cmd, int speed)` | Pan-tilt/zoom/focus/iris control | `speed` 0–100，ignored for `_STEP` |
| `set_position` | `int set_position(const ptz_position_t *pos)` | Move to absolute position | May hit `_POSITION_LIMIT` |
| `get_position` | `int get_position(ptz_position_t *pos)` | Read current position/FOV/zoom | — |
| `set_speed` / `get_speed` | `int (int speed)` / `int (int *speed)` | Default move speed (continuous) | 0–100 |
| `get_state` | `int get_state(ptz_state_t *state)` | `STOP` / `RUNNING` | — |
| `reset` | `int reset(void)` | Return to mechanical center | Blocks until done |
| `set_fov` | `int set_fov(double fov_h, double fov_v)` | Record FOV (for UI coordinate mapping) | Degrees |
| `preset_set` | `int preset_set(int preset_id, const char *name)` | Save current position as preset | `preset_id` 0–N，limit per product spec |
| `preset_goto` | `int preset_goto(int preset_id)` | Goto preset | Completes asynchronously |
| `preset_delete` | `int preset_delete(int preset_id)` | Delete preset | — |
| `preset_get_count` | `int preset_get_count(void)` | Saved count (return value) | — |
| `preset_get_list` | `int preset_get_list(ptz_preset_info_t *list, int max_count)` | Enumerate presets | Count returned by function |
| `preset_get_by_name` | `int preset_get_by_name(const char *name, ptz_preset_info_t *info)` | Find by name | `_PRESET_NOT_FOUND` |
| `tracking_set_enable` | `int tracking_set_enable(int enable)` | Target tracking master switch | — |
| `tracking_get_enable` | `int tracking_get_enable(void)` | Query enable | Returns 0/1 |
| `tracking_set_mode` | `int tracking_set_mode(ptz_track_mode_t mode)` | Tracking mode (MOTION/BODY/PET/VEHICLE) | — |
| `tracking_set_temporary_switch` | `int tracking_set_temporary_switch(int enable)` | Temporarily pause/resume (keeps mode) | — |
| `tracking_handle_motion` | `int tracking_handle_motion(void *md_info)` | Inject motion detection for tracking | `md_info` from AI/MD callback |
| `tracking_handle_human` | `int tracking_handle_human(const void *human_output, int rect_num, int video_w, int video_h)` | Inject human detection results | Same as above |

### 4.11 `power_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` / `deinit` | — | Lifecycle | — |
| `suspend` | `int suspend(power_manager_suspend_mode_t mode)` | Enter NORMAL/DEEP sleep | Blocks until wake |
| `resume` | `int resume(void)` | Post-wake cleanup | Usually internal |
| `get_last_wakeup_source` | `int get_last_wakeup_source(int *source)` | Read last wake source | Source encoding implementation-defined |
| `register_hook` | `int register_hook(power_manager_hook_t pre, power_manager_hook_t post)` | Register pre-sleep/post-wake hooks | `pre/post` may be `NULL` |
| `unregister_hook` | `int unregister_hook(power_manager_hook_t pre, power_manager_hook_t post)` | Unregister | — |

### 4.12 `nv_ops_t` (night vision) detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` / `deinit` | — | Lifecycle | — |
| `set_param` | `int set_param(nightvision_photosensitive_mode_t, nightvision_mode_t)` | Set photosensor/night vision mode | — |
| `get_param` | `int get_param(... *photosensitive, ... *mode)` | Read current parameters | — |
| `set_normal_switch` / `get_normal_switch` | `int (nightvision_normal_switch_t)` / `int (... *)` | Normal day/night switch (DAY/NIGHT/AUTO) | — |
| `set_color_switch` / `get_color_switch` | `int (nightvision_color_switch_t)` / `int (... *)` | Color night vision (GRAY/COLOR/SMART) | — |
| `get_day_night` | `int get_day_night(void)` | Return current state (`nightvision_state_t`) | — |
| `set_alarm_status` | `int set_alarm_status(int alarm_status)` | Report alarm state for fill-light/color switch | — |

### 4.13 `backlight_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` | `int init(const char *instance, const char *name)` | Select backend (PWM/GPIO, etc.) and initialize | `instance` `instance` hardware instance; `name` backend |
| `deinit` | `int deinit(void)` | Deinitialize | — |
| `set_value` | `int set_value(uint32_t value)` | Set brightness (0–max) | `_BRIGHTNESS_INVALID` |
| `release` | `int release(void)` | Release control (allow other users) | — |

---

## 5. Storage service — `storage.h`

**Module ID**: `FRAMEWORK_MODULE_STORAGE(0x07)`

### 5.1 Enums / callbacks

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

### 5.2 Ops structures

| Ops | Functions |
|-----|------|
| `storage_tfcard_ops_t` | `init(dir)/deinit/status_subscribe/status_unsubscribe(handle)/get_status/get_capacity(sync)/get_used_space(sync)/get_mounted_dir/format(sync)` |
| `storage_flash_ops_t`  | `write_from_file/write_from_memory/read_to_memory/write_start/write_input/write_stop` |

### 5.3 Unified instance

```c
typedef struct {
    storage_tfcard_ops_t *tfcard;
    storage_flash_ops_t  *flash;
    void                 *reserved[8];
} storage_t;

int storage_open(storage_t **storage);
int storage_close(storage_t *storage);
```

### 5.4 Dedicated error codes

- File: `_FILE_NOT_FOUND/_FILE_OPEN_FAILED/_FILE_READ_FAILED/_FILE_WRITE_FAILED`
- Flash：`_FLASH_ERASE_FAILED/_WRITE_FAILED/_READ_FAILED/_BAD_BLOCK`
- TF card: `_TFCARD_NOT_MOUNTED/_FULL/_READ_ONLY/_FORMAT_FAILED`

### 5.5 Entry function reference

```c
int storage_open(storage_t **storage);
int storage_close(storage_t *storage);
```

- `storage_open`：First call starts TF monitor thread and Flash writer; `tfcard` may be `NULL` without TF slot.
- `storage_close`：Decrement reference; last close stops monitor thread.

### 5.6 `storage_tfcard_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` | `int init(const char *dir)` | Init TF card module; set default mount dir | `dir` default `/mnt/tf` |
| `deinit` | `int deinit(void)` | Deinitialize | — |
| `status_subscribe` | `int status_subscribe(storage_tfcard_status_callback_t cb)` | Subscribe to TF card status changes | Returns `handle > 0` for unsubscribe |
| `status_unsubscribe` | `int status_unsubscribe(int handle)` | Unsubscribe | Idempotent |
| `get_status` | `storage_tfcard_status_e get_status(void)` | Read latest status（`NORMAL/EJECTED/READONLY/FORMAT/DISABLED/ERROR`） | Non-blocking |
| `get_capacity` | `int get_capacity(int sync)` | Read total capacity (MB) | `sync=1` refresh; `0` use cache |
| `get_used_space` | `int get_used_space(int sync)` | Read used space (MB) | Same as above |
| `get_mounted_dir` | `const char *get_mounted_dir(void)` | Read mount directory | String owned by module; do not `free` |
| `format` | `void format(int sync)` | Format TF card | `sync=1` blocks; `0` async background |

### 5.7 `storage_flash_ops_t` detailed function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `write_from_file` | `int write_from_file(const char *dev_dst, const char *file_src, int is_fs_upgrade)` | Write file to Flash partition (`/dev/mtdN` or name) | `is_fs_upgrade=1` filesystem upgrade mode |
| `write_from_memory` | `int write_from_memory(const char *dev_dst, const void *buf, const int buf_size)` | Write memory buffer once | — |
| `read_to_memory` | `int read_to_memory(const char *dev_dst, void *buf, const int buf_size)` | Read partition to buffer | Returns bytes read |
| `write_start` | `int write_start(const char *dev_dst, void **handle)` | Chunked write session start | `handle` [out] |
| `write_input` | `int write_input(void *handle, const void *buf, const int buf_size)` | Write one chunk | Accumulates over calls |
| `write_stop` | `int write_stop(void *handle)` | End chunked write; finalize verify | Idempotent finalize |

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

### 6.1 Function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `runtime_bootstrap_init` | `int runtime_bootstrap_init(const char *config_path)` | Pre-load `settings.json` (before open, for early daemon state) | `config_path` may be `NULL` for default |
| `runtime_bootstrap_open` | `int runtime_bootstrap_open(runtime_bootstrap_t **bootstrap)` | Get runtime ops handle | Singleton |
| `runtime_bootstrap_close` | `int runtime_bootstrap_close(runtime_bootstrap_t *bootstrap)` | Release handle reference | — |
| `runtime_bootstrap_get_state` | `system_state_e get_state(void)` | Read system state (init/sleep/run) | Any thread |
| `runtime_bootstrap_set_state` | `int set_state(system_state_e state)` | Set system state; run lifecycle hooks | Suggested order: `init → run → sleep → run` |
| `load_config` (ops) | `int bootstrap->load_config(const char *config_path)` | Hot-reload runtime config | `0` / `< 0` |
| `framework_init` | `int framework_init(int soc_mode, int log_level)` | **One-shot init** (SoC/log/thread/HAL); call earliest in app | `soc_mode` is SoC-defined; `log_level` see `log.h` |

---

## 7. Console — `console.h`

**Color codes**: `CONSOLE_ERROR_COLOR`, `_WARN_COLOR`, `_MSG_COLOR`, `_DEBUG_COLOR`, `_INPUT_COLOR`, `_HIGHLIGHT_COLOR`, `CONSOLE_MODE_OFF`
**Macros**: `console_warn(fmt, ...)`, `DEBUG_HERE`
**Constants**: `COMMAND_STRING_LEN_MAX=32`, `COMMAND_PARAMS_NUM_MAX=16`, `COMMAND_PARAMS_LEN_MAX=256`

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

### 7.1 Function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `console_init` | `int console_init(const char *pCMDPath)` | Bind command pipe (e.g. `/tmp/celestia.cmd`); start receiver thread | Idempotent |
| `console_deinit` | `int console_deinit(void)` | Stop thread; close pipe | — |
| `console_print` | `int console_print(const char *titleColor, const char *title, const char *strColor, const char *subTitle, const char *fmt, ...)` | Structured log print with color/title | Use with `console_warn(fmt, ...)` macro |
| `console_loop` | `int console_loop(void)` | Blocking message loop (debug tools; not for production) | Usually `console_init` runs async |
| `console_cmd_register` | `int console_cmd_register(const char *cmd, FN_COMMAND fn, const char *help)` | Register CLI handler | `fn(pParams, pcAck)` Handles input and fills reply |

---

## 8. OTA — `ota.h` + `otapack.h`

### 8.1 Runtime API (`ota.h`)

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

### 8.2 Firmware package format (`otapack.h`)

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

> Header length `256` is **frozen**; do not change after first release; only append fields inside `Head`.

### 8.3 Runtime function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `ota_init` | `int ota_init(void)` | Init OTA subsystem (downloader/installer/state machine) | Idempotent; Usually called by runtime |
| `ota_start` | `int ota_start(ota_info_t info)` | **Async** start upgrade (download → verify → install) | Query via `ota_get_status` / `ota_get_progress_percent` |
| `ota_stop` | `int ota_stop(void)` | Cancel current upgrade | Idempotent; May not abort in `INSTALLING` |
| `ota_get_status` | `ota_status_e ota_get_status(void)` | Read status (IDLE/DOWNLOADING/INSTALLING/FAIL/DONE) | Non-blocking |
| `ota_get_progress_percent` | `int ota_get_progress_percent(void)` | Download/install progress (0–100) | — |

> In `otapack.h`, `DEFAULT_CIPHER/DEFAULT_MD/DEFAULT_KEY` and `encryptStr[]/mdStr[]` are for **packaging tools and upgrader only**; applications **must not** read default key constants.

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

### 9.1 Function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `reboot_info_init` | `int reboot_info_init(void)` | Restore last reboot reason from storage | Call early in boot |
| `reboot_info_set` | `int reboot_info_set(reboot_reason_e reason, const char *msg)` | Record reboot reason (before reboot) | `msg` ≤ 256 chars |
| `reboot_info_get` | `int reboot_info_get(reboot_param_t *info)` | Read last reboot reason, time, and note | — |
| `reboot_info_refresh_powerdown_time` | `int reboot_info_refresh_powerdown_time(void)` | Flush timestamp to storage (periodic snapshot; approximates power-down time) | Call periodically |

---

## 10. RTSP Gateway — `rtsp_protocol.h`

**Module ID**: `FRAMEWORK_MODULE_GATEWAY(0x08)`

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

### 10.1 Entry and server ops function reference

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `rtsp_protocol_open` | `int rtsp_protocol_open(rtsp_protocol_t **rtsp)` | Open RTSP Gateway; If `multimedia` not in context, calls `multimedia_open` and sets `multimedia_owned=true` | App may preset `multimedia` to share singleton |
| `rtsp_protocol_close` | `int rtsp_protocol_close(rtsp_protocol_t *rtsp)` | Close Gateway; Also closes internal multimedia when `multimedia_owned=true` | — |
| `server->start` | `int start(rtsp_protocol_context_t *ctx)` | Start RTSP service，Start listening on port | Port/codec from `fw_video_config.json` |
| `server->stop` | `int stop(rtsp_protocol_context_t *ctx)` | Stop RTSP service | — |
| `server->get_status` | `int get_status(rtsp_protocol_context_t *ctx, bool *is_running)` | Query run state | `is_running` [out] |

---

## 11. Nebula Gateway — `nebula_protocol.h`

**Module ID**: `FRAMEWORK_MODULE_NEBULA_PROTOCOL(0x09)`
**Version**: `NEBULA_PROTOCOL_VERSION_*`

### 11.1 Ops

| Ops | Functions |
|-----|------|
| `nebula_network_ops_t` | `init/deinit/start/stop/send_to_all_clients/send_to_local_client/start_stream/stop_stream/link_server_init/link_server_deinit` |
| `nebula_http_ops_t`    | `init/deinit/start/stop` |
| `nebula_pair_ops_t`    | `init/deinit/scan_connect_run/play_file` |

### 11.2 Unified instance

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

### 11.3 Entry and ops function reference

#### 11.3.1 Entry

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `nebula_protocol_open` | `int nebula_protocol_open(nebula_protocol_t **nebula)` | Open Nebula Gateway (private protocol + HTTP + provisioning) | Singleton |
| `nebula_protocol_close` | `int nebula_protocol_close(nebula_protocol_t *nebula)` | Close Gateway | Reference counting |

#### 11.3.2 `nebula_network_ops_t`

| Function | Purpose | Notes |
|---|---|---|
| `init` / `deinit` | Private protocol core lifecycle | — |
| `start` / `stop` | Start/stop protocol stack (TCP/TLS listen, heartbeat) | — |
| `send_to_all_clients` | `int (const void *data, size_t len)` Broadcast to all connected clients | — |
| `send_to_local_client` | `int (const void *data, size_t len)` Send to LAN clients only | — |
| `start_stream` / `stop_stream` | `int (nebula_network_handle_t handle)` Start/stop AV stream for client | `handle` from Gateway |
| `link_server_init` / `link_server_deinit` | Init/release link service (device-to-cloud keep-alive) | Optional; product flag |

#### 11.3.3 `nebula_http_ops_t`

| Function | Purpose | Notes |
|---|---|---|
| `init` / `deinit` | HTTP control-plane lifecycle | — |
| `start` / `stop` | Start/stop onboard HTTP (config/snapshot/diagnostics) | Default port in config |

#### 11.3.4 `nebula_pair_ops_t`

| Function | Signature | Purpose | Notes |
|---|---|---|---|
| `init` / `deinit` | — | Provisioning/QR pairing lifecycle | — |
| `scan_connect_run` | `int scan_connect_run(void *arg)` | Start QR provisioning (scan QR → WiFi → cloud) | `arg` provisioning params; **Blocks** until done or timeout |
| `play_file` | `int play_file(const char *name)` | Play provisioning voice prompt (e.g. provisioning success/failure) | `name` resource id |

---

## 12. Graphics Bitmap — `bitmap.h`

Bitmap load and pixel utilities (see `framework/ui/graphics/include/bitmap.h` for structures; placeholder for GUI docs).

---

## 13. Other public headers

| Header | Role | Description |
|-------|------|------|
| `framework/runtime/include/runtime_error.h` | Runtime error codes | Unified errors for bootstrap/console/ota/reboot_info |
| `framework/runtime/include/runtime.h` | Aggregate include | One-shot include of bootstrap/console/ota/otapack/reboot_info |
| `framework/services/storage/include/tfcard_utils.h` | TF card helpers | Storage-internal; **avoid app dependency** (use `storage.h` for app APIs) |
| `framework/ui/lvgl/include/lv_port.h`, `lv_port_utils.h` | LVGL port | For UI submodules |
| `framework/ui/common/include/port.h`, `gui_error.h` | GUI common | UI internal abstraction |

---

## 14. Public API usage constraints

1. **Use Framework services only via `xxx_open() → ops → xxx_close()`**; Singleton guaranteed by Framework.
2. Applications **must not `#include` any `hardware/**/*.h`** (public or private); expose capabilities via Framework only.
3. **Cross-layer errors use `system_error_t`** (`SYSTEM_ERROR_CODE(domain, module, error)` in `system/core/include/system_error.h`).
4. All `reserved[N]` in `ops` are for binary-compatible extension; New functions must replace `reserved` slots only.
5. This document covers **only upper-layer (app/framework) callable APIs**; Internal hooks, legacy, and auto-invoke APIs are out of scope.

---

## 15. References

- Concepts and call flow: [framework.en.md](framework.en.md)
- Extending functionality: [Adding a Framework service](../skills/celestia-framework.en.md)
- System Core: [system-core.en.md](system-core.en.md)
- HAL: [hardware-hal.en.md](hardware-hal.en.md)
- API index: [guide/api-index.en.md](api-index.en.md)