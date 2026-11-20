<p align="right"><b>English</b> | <a href="system-core-api.md">中文</a></p>

# System Core API Reference (Public Lookup Interfaces)

> **Scope of this document**: System Core public lookup interface reference: error-code bit layout, logging, configuration, threads, time, memory, containers, and utility functions.
>
> For concepts, responsibilities, and boot placement see [system-core.en.md](system-core.en.md); for layering, contracts, and the runtime model see [Architecture overview](../architecture/overview.en.md); terminology is defined solely in the [Glossary](../glossary.en.md).

---

## 0. Quick reference index

| # | Header | Module ID (error domain) | Description |
|---|--------|-------------------|------|
| 1 | `system_core.h`        | — | Single entry point; trimmed via `SYSTEM_CORE_IMPORT_*` macros |
| 2 | `system_error.h`       | `SYSTEM_DOMAIN(0x00)` / all domains | Unified error-code encoding and base error codes |
| 3 | `system_types.h`       | — | Base types, function pointers, `SYSTEM_true/false` |
| 4 | `system_version.h`     | — | System Core version macros |
| 5 | `log.h`                | `SYSTEM_MODULE_LOG(0x02)` | Unified logging system |
| 6 | `system_config.h`     | `SYSTEM_MODULE_CONFIG(0x01)` | INI / regex / JSON configuration management |
| 7 | `thread.h`             | `SYSTEM_MODULE_THREAD(0x03)` | Threads and read–write locks |
| 8 | `gthread.h`            | — | GLib-style thread/mutex abstraction |
| 9 | `system_time.h`        | `SYSTEM_MODULE_TIME(0x06)` | System time |
| 10 | `timer.h`             | `SYSTEM_MODULE_TIME(0x06)` | Software timer tasks |
| 11 | `watchdog.h`          | `SYSTEM_MODULE_WATCHDOG(0x09)` | Watchdog |
| 12 | `system_memory.h`     | `SYSTEM_MODULE_MEMORY(0x04)` | Memory allocation and DMA memory |
| 13 | `md5.h`               | — | MD5 algorithm |
| 14 | `base64.h`            | `SYSTEM_MODULE_CRYPTO(0x07)` | Base64 encode/decode |
| 15 | `crc32.h`             | `SYSTEM_MODULE_CRYPTO(0x07)` | CRC32 checksum |
| 16 | `string_utils.h`      | `SYSTEM_MODULE_UTILS(0x08)` | String utilities |
| 17 | `geometry.h`          | `SYSTEM_MODULE_UTILS(0x08)` | Geometry/rectangle utilities |
| 18 | `image_convert.h`     | `SYSTEM_MODULE_UTILS(0x08)` | YUV/RGB conversion |
| 19 | `system_pixel_format.h` | — | Pixel formats and `rect_t` |
| 20 | `system_file.h`       | — | File read/write wrappers |
| 21 | `list.h`              | — | Kernel-style doubly linked list |
| 22 | `ring_buffer.h`       | `SYSTEM_MODULE_MEMORY(0x04)` | Ring message buffer |
| 23 | `circular_buffer.h`   | — | Multi-reader broadcast ring buffer |
| 24 | `system_macros.h`     | — | General macros |
| 25 | `system_assert.h`     | — | Assertion/expectation macros |
| 26 | `system_reentry.h`    | — | Reentrancy guard macros |
| 27 | `system_debug.h`      | — | Debug checkpoints |

---

## 1. Error codes — `system_error.h`

### 1.1 Types and bit layout

```c
typedef int32_t system_error_t;   /* 负值 = 错误；0 = 成功 */
/* Bit layout:  [31]=sign  [24..30]=domain  [16..23]=module  [0..15]=error */
```

### 1.2 Domain / Module

| Macro | Value | Description |
|---|----|------|
| `SYSTEM_DOMAIN`    | 0x00 | system/core layer |
| `FRAMEWORK_DOMAIN` | 0x01 | framework layer |
| `HARDWARE_DOMAIN`  | 0x02 | hardware layer |

System modules: `SYSTEM_MODULE_CORE(0x00)`, `CONFIG(0x01)`, `LOG(0x02)`, `THREAD(0x03)`, `MEMORY(0x04)`, `STORAGE(0x05)`, `TIME(0x06)`, `CRYPTO(0x07)`, `UTILS(0x08)`, `WATCHDOG(0x09)`, `SOC_PLATFORM(0x0A)`

Framework modules: `NETWORK(0x00)`, `NOTIFICATION(0x01)`, `MULTIMEDIA(0x02)`, `RUNTIME(0x03)`, `DEVICE(0x04)`, `UI(0x05)`, `AI(0x06)`, `STORAGE(0x07)`, `GATEWAY(0x08)`, `NEBULA_PROTOCOL(0x09)`

Hardware modules: `WIFI(0x00)`, `AUDIO(0x01)`, `CAMERA(0x02)`, `VISION(0x03)`, `DISPLAY(0x04)`, `DRIVER(0x05)`, `GPU(0x06)`, `OSD(0x07)`, `INPUT(0x08)`, `AI(0x09)`, `ETHERNET(0x0A)`

### 1.3 Common error codes (0x0000–0x0016)

`ERROR_SUCCESS / _INVALID_PARAM / _NOT_INIT / _MEMORY / _TIMEOUT / _NOT_FOUND / _ALREADY_EXISTS / _BUSY / _IO / _PERMISSION_DENIED / _NOT_SUPPORTED / _INVALID_STATE / _OPERATION_FAILED / _DEVICE_NOT_AVAILABLE / _CONFIG_ERROR / _NETWORK / _CONNECTION_FAILED / _QUEUE_FULL / _QUEUE_EMPTY / _CLIENT_FULL / _INVALID_TYPE / _NETWORK_UNREACHABLE / _INTERFACE_NOT_FOUND`

> From 0x0017 onward are module-private error codes.

### 1.4 Macros and functions

```c
#define SYSTEM_ERROR_CODE(domain, module, error) \
    (-((int32_t)((domain) << 24) | ((module) << 16) | (error)))

#define SYSTEM_ERROR_GET_DOMAIN(err)  /* 提取 domain */
#define SYSTEM_ERROR_GET_MODULE(err)  /* 提取 module */
#define SYSTEM_ERROR_GET_ERROR(err)   /* 提取 error   */

static inline system_error_t system_error_from_errno(int errno_value);
const char *                  system_error_to_string(system_error_t error);
static inline int             system_error_is_domain(system_error_t err, uint32_t domain);
static inline int             system_error_is_module(system_error_t err, uint32_t domain, uint32_t module);
```

> Module-specific error codes (`LOG_ERROR_*`, `CONFIG_ERROR_*`, `THREAD_ERROR_*`, etc.) are **defined inline in each module’s public header** (`log.h`, `system_config.h`, `system_thread.h`, …), assembled using the `SYSTEM_DOMAIN` / `SYSTEM_MODULE_*` tables from this header; `system_error.h` only centralizes domain/module code tables and common `ERROR_*`.

### 1.5 Predefined System Core layer errors

`SYSTEM_SUCCESS(=0)`, `SYSTEM_FAILED`, `SYSTEM_ERROR_*` (Core-domain macros for all common `ERROR_*` codes),
file-related: `SYSTEM_ERROR_FILE_NOT_FOUND / _FILE_OPEN_FAILED / _FILE_READ_FAILED / _FILE_WRITE_FAILED`.

---

## 2. Base types and version

### 2.1 `system_types.h`

```c
#define SYSTEM_true   (1)
#define SYSTEM_false  (0)
#define TRUE          1
#define FALSE         0
#define SYSTEM_SUCCESS 0

typedef int          (*SYSTEM_COMPARE_FUNCTION)(const void *a, const void *b);
typedef int          (*SYSTEM_COMPARE_DATA_FUNCTION)(const void *a, const void *b, void *user_data);
typedef int          (*SYSTEM_EQUAL_FUNCTION)(const void *a, const void *b);
typedef void         (*SYSTEM_DESTROY_NOTIFY_FUNCTION)(void *data);
typedef void         (*SYSTEM_FUNC_FUNCTION)(void *data, void *user_data);
typedef unsigned int (*SYSTEM_HASH_FUNCTION)(const void *key);
typedef void         (*SYSTEM_HASH_FOREACH_FUNCTION)(void *key, void *value, void *user_data);
```

### 2.2 `system_version.h`

```c
#define SYSTEM_CORE_VERSION_MAJOR 0
#define SYSTEM_CORE_VERSION_MINOR 12
#define SYSTEM_CORE_VERSION_PATCH 0
#define SYSTEM_CORE_VERSION_CODE  /* (M<<16)|(m<<8)|p */
#define SYSTEM_CORE_MAKE_VERSION(major, minor, patch) /* ... */

static inline const char *system_core_version_string(void);
```

### 2.3 `system_core.h` aggregate entry

Control switches (all enabled by default):
`SYSTEM_CORE_IMPORT_ALL`, `_CONFIG`, `_LOG`, `_THREAD`, `_TIME`, `_TIMER`, `_UTILS`, `_IMAGE_CONVERT`, `_BASE64`, `_CRC32`, `_MD5`, `_CIRCULAR_BUFFER`, `_MEMORY`, `_WATCHDOG`.

Base macros: `MAX(x,y)`, `MIN(x,y)`, `PARAM_IN_RANGE(v, min, max)`.

> ⚠️ The aggregate is **partial**: even when `SYSTEM_CORE_IMPORT_ALL=1`, `system_core.h` does not pull in `system_macros.h`, `system_assert.h`, `system_debug.h`, `system_reentry.h`, `system_file.h`, `system_pixel_format.h`, `list.h`, or `ring_buffer.h`; include those headers directly. The thread module actually imports `system_thread.h` (not `thread.h`). Build artifacts include both `libsystem_core.so` and `libsystem_core.a`.

---

## 3. Logging — `log.h`

### 3.1 Levels

```c
enum COMMON_LOG_LEVEL { LOG_FATAL = -1, LOG_ERROR, LOG_WARNING, LOG_INFO, LOG_DEBUG, LOG_VERBOSE };
```

### 3.2 Structures and callbacks

```c
typedef struct {
    int Level;
    char File[128]; char Function[64]; char KeyWord[64];
    int StartLine, EndLine;
    long long StartT, EndT;
} log_filter_info;

typedef int (*LOG_PRINT_CB)(int level, char *str_t, long lTimeUS, char *log, int iLogLen);
typedef int (*LOG_SAVE_CB) (int level, char *str_t, long lTimeUS, char *log, int iLogLen);
```

### 3.3 Functions

```c
int   log_print_base(int level, const char *tag, const char *file,
                     const char *fun, int line, const char *fmt, ...)
      __attribute__((format(printf, 6, 7)));
int   log_print_set_filter(log_filter_info *pLogFilterInfo);
int   log_print_set_cb(LOG_PRINT_CB cb);
int   log_print_set_save_file_cb(LOG_SAVE_CB cb);
char *log_get_save_path(void);
void  log_set_level(int log_level);
int   log_write_init(void);
int   log_write_deinit(void);
```

### 3.4 Function reference

| Function | Purpose | Key parameters / return | Thread |
|---|---|---|---|
| `log_print_base` | Low-level print API (macros `log_error/warn/info/debug/verbose` expand here) | `level`: see Section 3.1; `tag`: module tag; `file/fun/line`: typically `__FILE__/__FUNCTION__/__LINE__`; `fmt` and variadic args follow `printf`; returns bytes actually written | Any thread; internally locked |
| `log_print_set_filter` | Set filter criteria (level/keyword/line range/time window) | `pLogFilterInfo` [in] must be non-NULL; returns `LOG_OK/_INVALID_PARAM` | Any; only one global filter at a time |
| `log_print_set_cb` | Register **print output** callback (take over screen/serial output) | `cb` `NULL` restores default; returns 0/< 0 | Callback invoked from printing threads; **must not block** |
| `log_print_set_save_file_cb` | Register **persist-to-disk** callback (take over log file writes) | Same as above | Callback invoked from internal writer thread |
| `log_get_save_path` | Read current log save directory | Returns constant string; caller **must not** `free` | Any |
| `log_set_level` | Set global minimum log level | Messages below this level are silently dropped | Any |
| `log_write_init` | Initialize async persist subsystem (open files, start writer thread) | Returns 0 / error | Call once; repeated calls are idempotent |
| `log_write_deinit` | Stop writer thread and flush buffers | — | Call before exit |

### 3.5 Helper macros

`log_error(fmt, ...)`, `log_warn(fmt, ...)`, `log_info(fmt, ...)`, `log_debug(fmt, ...)`, `log_verbose(fmt, ...)`
Define `#define LOG_TAG "YOUR_TAG"` before use.

### 3.6 Error codes

`LOG_OK`, `LOG_ERROR_INVALID_PARAM / _NOT_INIT / _MEMORY / _OPERATION_FAILED / _INVALID_LEVEL(0x0017) / _FILE_OPEN_FAILED(0x0018) / _WRITE_FAILED(0x0019) / _BUFFER_FULL(0x001A)`.

---

## 4. Configuration management — `system_config.h`

### 4.1 Base types

```c
typedef struct _dictionary_ dictionary;
typedef system_error_t      config_error_t;
```

### 4.2 Native iniparser API

```c
int            iniparser_getnsec(const dictionary *d);
const char *   iniparser_getsecname(const dictionary *d, int n);
void           iniparser_dump_ini(const dictionary *d, FILE *f);
void           iniparser_dumpsection_ini(const dictionary *d, const char *s, FILE *f);
void           iniparser_dump(const dictionary *d, FILE *f);
int            iniparser_getsecnkeys(const dictionary *d, const char *s);
const char **  iniparser_getseckeys(const dictionary *d, const char *s, const char **keys);
const char *   iniparser_getstring(const dictionary *d, const char *key, const char *def);
int            iniparser_getint(const dictionary *d, const char *key, int notfound);
long int       iniparser_getlongint(const dictionary *d, const char *key, long int notfound);
double         iniparser_getdouble(const dictionary *d, const char *key, double notfound);
int            iniparser_getboolean(const dictionary *d, const char *key, int notfound);
int            iniparser_set(dictionary *ini, const char *entry, const char *val);
int            iniparser_seti(dictionary *ini, const char *entry, int val);
int            iniparser_set_double(dictionary *ini, const char *entry, double val);
void           iniparser_unset(dictionary *ini, const char *entry);
int            iniparser_find_entry(const dictionary *ini, const char *entry);
dictionary *   iniparser_load(const char *ininame);
void           iniparser_freedict(dictionary *d);
int            iniparser_save_file(dictionary *ini, const char *pFileName);
dictionary *   dictionary_new(size_t size);
void           iniparser_set_error_callback(int (*errback)(const char *, ...));
```

### 4.3 Extended helper API

```c
int    iniparser_setl(dictionary *ini, const char *entry, long int val);
int    iniparser_setb(dictionary *ini, const char *entry, int val);
int    iniparser_add_section(dictionary *ini, const char *section);
void   iniparser_remove_section(dictionary *ini, const char *section);
char * iniparser_getstring_dup(const dictionary *ini, const char *key, const char *def);

int          iniparser_dict_getint(const dictionary *ini, const char *section, const char *key, int notfound);
const char * iniparser_dict_getstring(const dictionary *ini, const char *section, const char *key, const char *def);
long int     iniparser_dict_getlongint(const dictionary *ini, const char *section, const char *key, long int notfound);
double       iniparser_dict_getdouble(const dictionary *ini, const char *section, const char *key, double notfound);
int          iniparser_dict_getboolean(const dictionary *ini, const char *section, const char *key, int notfound);
int          iniparser_dict_setint(dictionary *ini, const char *section, const char *key, int val);

int iniparser_file_getstring(const char *file, const char *section, const char *key, char *buf, int len, const char *def);
int iniparser_file_getint   (const char *file, const char *section, const char *key, int def);
int iniparser_file_setstring(const char *file, const char *section, const char *key, const char *str);
int iniparser_file_setint   (const char *file, const char *section, const char *key, int val);
```

### 4.4 Regex and JSON

```c
typedef enum config_regex_action_e   { /* MATCH_ALL/ONE, DELETE_*, REPLACE_* (pattern/compiled) */ };
typedef enum config_regex_split_option_e { NONE, FRONT, TAIL, MIDDLE, FULL };
typedef enum config_regex_exec_option_e  { EXEC_ALL, EXEC_ONE };

char * config_regex_read_file(char *filename, int *file_size);
int    config_regex_get_string(char *conf, char *key, char **value);
int    config_regex_get_int   (char *conf, char *key, int  *value);

int    config_regex_match(regex_t *regex_in, char *pattern, int flags,
                          char *buffer, char *result, int result_len,
                          char *split, int split_option, int exec_option);
int    config_regex_replace(regex_t *regex_in, char *pattern, int flags,
                            char *buffer, char *result, int result_len,
                            char *replacement, int exec_option);
int    config_regex_extract(char *buffer, char *result, int result_len,
                            int regex_count, /* pattern, flags, ... */ ...);

char * cJSON_RemoveComments(const char *json_string);
#ifdef CONFIG_THIRD_PARTY_CJSON
int cJSON_GetItemInt    (struct cJSON *object, const char *string, int defaultValue);
int cJSON_GetItemString (struct cJSON *object, const char *string, char *dst, int dstSize, const char *defaultValue);
int cJSON_GetArrayItemInt(struct cJSON *object, const char *string, int *array, int arraySize);
#endif
```

Legacy compatibility: `ERGEX_HING_BUFFFULL / CONF_LEN_RES / CONF_LEN_PATTERN / CONF_LEN_ERROR / CONF_REG_STYLE / ACTION_PTRN_* / ACTION_COMP_* / SPLIT_* / EXEC_*` plus inline wrappers `read_file / get_conf_str / get_conf_int / regexpr_match / regexpr_replace` and macro `regexpr_get(...)` (disable via `CONFIG_DISABLE_REGEX_LEGACY_APIS`).

### 4.5 Function reference (common subset)

> Full `iniparser_*` follows iniparser 3.x semantics: `d` is the dictionary returned by `iniparser_load`, `key` is like `"section:keyname"`; all read APIs return `def`/`notfound` when the key is missing.

| Function | Purpose | Parameters / return | Notes |
|---|---|---|---|
| `iniparser_load` | Load a `.ini` file into a dictionary | `ininame` [in] path; returns dictionary pointer or `NULL` | Caller frees with `iniparser_freedict` |
| `iniparser_freedict` | Free dictionary | `d` [in] | Idempotent; `NULL`-safe |
| `iniparser_getstring` | Read string (`def` fallback) | Returns read-only pointer owned by dictionary | Invalid after dictionary is destroyed |
| `iniparser_getint` / `_getlongint` / `_getdouble` / `_getboolean` | Read int/float/bool | Returns `notfound` on parse failure | — |
| `iniparser_set` / `_seti` / `_set_double` | Write string/int/float | `entry` like `"sect:key"`; new section needs any key set first | Not persisted until `save_file` |
| `iniparser_save_file` | Write dictionary to file | `ini`, `pFileName` | Returns `-1` on failure |
| `iniparser_dict_get*` | Access via `(section, key)` without manual string concat | `defaultValue/notfound` as above | Recommended |
| `iniparser_file_getstring` / `_getint` / `_setstring` / `_setint` | One-shot load → get/set → save helper | `buf`/`len` buffer and capacity | Avoid manual `dictionary*` management |
| `config_regex_get_string` / `_get_int` | For **non-INI** custom formats: extract fields via regex in text | `conf` body, `key` match pattern; `value` is `malloc` string (caller `free`) | Small files only; use `iniparser` for large files |
| `config_regex_match` | Search buffer with compiled regex | `split_option/exec_option` see enums | Thread-safe (separate `regex_t`) |
| `config_regex_replace` | Replace | Same as above | — |
| `cJSON_RemoveComments` | Strip `//` / `/* */` comments; relaxed JSON → strict JSON | Returns `malloc` string | Preprocess before cJSON parse |

### 4.6 Path management

```c
typedef enum { CONFIG_PATH_TYPE_CONFIG = 0, DATA, LOG, TMP, MAX } config_path_type_t;

#define CONFIG_DEFAULT_CONFIG_PATH "/etc/config/"
#define CONFIG_DEFAULT_DATA_PATH   "/var/data/"
#define CONFIG_DEFAULT_LOG_PATH    "/var/log/"
#define CONFIG_DEFAULT_TMP_PATH    "/tmp/"

const char *config_get_path_by_type(int type);
int         config_set_path        (int type, const char *path);
int         config_build_path      (int type, const char *filename, char *out_path, size_t out_size);
int         config_build_file_path (const char *filename, char *out_path, size_t out_size);
```

| Function | Purpose | Parameters / return |
|---|---|---|
| `config_get_path_by_type` | Query directory for a path type | Returns constant string (e.g. `"/etc/config/"`); do not `free` |
| `config_set_path` | Override default path for a type (test/custom) | `path` must end with `/`; returns 0/< 0 |
| `config_build_path` | Concatenate `<type_path>/<filename>` into `out_path` | `out_size` is buffer capacity; returns bytes written or < 0 |
| `config_build_file_path` | Detect filename type (by prefix) then concatenate | Returns 0/< 0 |

### 4.7 Error codes

`CONFIG_OK / _INVALID_PARAM / _NOT_INIT / _MEMORY / _NOT_FOUND / _CONFIG_ERROR / _OPERATION_FAILED / _INVALID_PATH(0x0017) / _FILE_NOT_FOUND(0x0018) / _PARSE_FAILED(0x0019) / _KEY_NOT_FOUND(0x001A) / _SECTION_NOT_FOUND(0x001B) / _WRITE_FAILED(0x001C)`.

---

## 5. Threads and locks — `thread.h`

### 5.1 Types

```c
typedef int (*thread_function_t)(void *param);
typedef void *rw_lock_t;
typedef system_error_t thread_error_t;
```

### 5.2 Functions

```c
int thread_init(void);
int thread_deinit(void);

int thread_create(int *handle, thread_function_t function, void *param,
                  const char *name, int stack_size, int priority);
int thread_destroy(int handle);
int thread_get_info(char *buffer, int buffer_len);
int thread_check_in(int handle);

rw_lock_t *rw_lock_create(void);
void       rw_lock_destroy(rw_lock_t *lock);
void       rw_lock_read_lock   (rw_lock_t *lock);
int        rw_lock_try_read_lock(rw_lock_t *lock);
void       rw_lock_read_unlock (rw_lock_t *lock);
void       rw_lock_write_lock  (rw_lock_t *lock);
int        rw_lock_try_write_lock(rw_lock_t *lock);
void       rw_lock_write_unlock(rw_lock_t *lock);
```

### 5.3 Function reference

| Function | Purpose | Parameters / return | Thread |
|---|---|---|---|
| `thread_init` | Initialize thread subsystem (internal bookkeeping) | `0` / `< 0` | Call once early in process |
| `thread_deinit` | Deinitialize | — | Call before exit |
| `thread_create` | Create thread running `function(param)` | `handle` [out] thread handle (valid if > 0); `name` display name; `stack_size` bytes (0 = default); `priority` 0–99 (or negative = default) | Any |
| `thread_destroy` | Request thread exit and reclaim handle | Handle from `thread_create` | Blocks until `function` returns |
| `thread_get_info` | Dump status of all threads (diagnostics) | `buffer`/`buffer_len` | Any |
| `thread_check_in` | Periodic liveness check from thread body | Used with watchdog | Call in each monitored thread loop |
| `rw_lock_create` / `rw_lock_destroy` | Create/destroy read–write lock | Returns `rw_lock_t *`; destroy only when no holders | Lock object may be shared across threads |
| `rw_lock_read_lock` / `_read_unlock` | Shared read lock/unlock | Multiple readers allowed | Blocking |
| `rw_lock_try_read_lock` | Try read lock | `0` on success, non-zero if contended | Non-blocking |
| `rw_lock_write_lock` / `_write_unlock` | Exclusive write lock/unlock | Waits for all readers to release | Blocking |
| `rw_lock_try_write_lock` | Try write lock | Same as try_read | Non-blocking |

### 5.4 Helper macros

```c
THREAD_RUN_DEF  (ThreadName)
THREAD_RUN_START(ThreadName, function, param, stack_size, priority)
THREAD_RUN_STOP (ThreadName)
THREAD_RUN_CHECK(ThreadName)
```

### 5.5 Error codes

`THREAD_OK / _INVALID_PARAM / _NOT_INIT / _MEMORY / _OPERATION_FAILED / _CREATE_FAILED(0x0017) / _JOIN_FAILED(0x0018) / _MUTEX_INIT_FAILED(0x0019) / _MUTEX_LOCK_FAILED(0x001A) / _COND_INIT_FAILED(0x001B)`.

---

## 6. GThread abstraction — `gthread.h`

```c
typedef void *(*GThreadFunc)(void *data);
typedef void  *GThread;
typedef union  _GMutex    GMutex;    /* { void*p; unsigned int i[2]; } */
typedef struct _GRecMutex GRecMutex;
typedef struct _GRWLock   GRWLock;
typedef struct _GCond     GCond;

GThread *g_thread_new(const char *name, GThreadFunc func, void *data);
void    *g_thread_join(GThread *thread);

void g_mutex_init  (GMutex *mutex);
void g_mutex_clear (GMutex *mutex);
void g_mutex_lock  (GMutex *mutex);
void g_mutex_unlock(GMutex *mutex);
int  g_mutex_trylock(GMutex *mutex);
```

### 6.1 Function reference

| Function | Purpose | Parameters / return | Notes |
|---|---|---|---|
| `g_thread_new` | Create and start thread | `name`, `func(data)`, `data` | Returns `NULL` on failure |
| `g_thread_join` | Wait for thread end and reclaim | Returns `func` return value | Each thread joinable once |
| `g_mutex_init` / `_clear` | Mutex init/destroy | `GMutex *` stack or static object | Do not use mutex before init |
| `g_mutex_lock` / `_unlock` | Lock/unlock | — | Not recursive; second lock on same thread deadlocks |
| `g_mutex_trylock` | Try lock | Non-zero = locked, 0 = already held | Non-blocking |

> Note: GThread abstraction is mainly for GLib-style interoperability; new code should prefer `thread.h`.

---

## 7. Time and timers — `system_time.h` / `timer.h`

### 7.1 `system_time.h`

```c
int   system_get_time(unsigned long long *out_time);
int   system_set_time(long long input_time, int tolerance_value);
int   system_timed(void);

long long system_get_sys_run_time64(void);
long long system_get_sys_run_time_s(void);
long long system_get_sys_run_time_ns(void);
long long system_get_date_time64(void);
long long system_get_local_time64(void);
char *    system_get_hms_time64(void);
long long system_string_to_seconds(char *chTime);
long long system_string_to_secondsT(char *chTime);
long long system_time_to_seconds(int iYear, int iMon, int iDay, int iH, int iM, int iS);
char *    system_seconds_to_stringT(time_t secs);
char *    system_get_date_to_string(void);
int       system_get_year_mon_day(int *pYear, int *pMon, int *pDay);
int       system_sleep_ms(unsigned int ms);

#define SYSTEM_PROCESS_TIME(process)  /* 测量 process 的执行耗时，单位 us */
```

#### Function reference

| Function | Purpose | Parameters / return | Notes |
|---|---|---|---|
| `system_get_time` | Read current UTC time (seconds) | `out_time` [out] | 0 / error |
| `system_set_time` | Set system time (UTC seconds) | `tolerance_value`: allowed NTP skew threshold (seconds); set only if exceeded; `0` forces set | Requires root |
| `system_timed` | Check whether system time is reasonable (synchronized) | Returns 0 = not synced; 1 = synced | Any |
| `system_get_sys_run_time64/_s/_ns` | Monotonic time since boot (ms / s / ns) | Unaffected by NTP/setTime | Use for deltas |
| `system_get_date_time64` | UTC date/time (seconds) | — | Convenience equivalent to `system_get_time` |
| `system_get_local_time64` | Local date/time (app timezone) | — | Follows `TZ` env var |
| `system_get_hms_time64` | Returns `"HH:MM:SS"` string | Internal static buffer | Not thread-safe (copy after use) |
| `system_string_to_seconds` / `_secondsT` | Parse `"YYYY-MM-DD HH:MM:SS"` to seconds | Returns < 0 on format error | `_T` variant accepts `T` separator |
| `system_time_to_seconds` | Seconds from components (construction helper) | 4-digit year | — |
| `system_seconds_to_stringT` | Seconds → `"YYYY-MM-DDTHH:MM:SS"` | Internal static buffer | Not thread-safe |
| `system_get_date_to_string` | Current date `"YYYY-MM-DD"` | Internal static buffer | Same as above |
| `system_get_year_mon_day` | Write `*pYear/*pMon/*pDay` | 0 / error | — |
| `system_sleep_ms` | Millisecond sleep (`nanosleep` wrapper) | Does not resume sleep after signal | — |

### 7.2 `timer.h`

```c
typedef void *timer_task_handle_t;
typedef void *timer_manager_handle_t;

int                     timer_init(void);
int                     timer_deinit(void);
timer_manager_handle_t *timer_manager_create(void);
void                    timer_manager_destroy(timer_manager_handle_t *manager);
timer_task_handle_t    *timer_task_create(int (*callback)(void *), void *arg, int interval_ms);
int                     timer_task_add   (timer_manager_handle_t *manager, timer_task_handle_t *task);
int                     timer_task_remove(timer_manager_handle_t *manager, timer_task_handle_t *task);
timer_task_handle_t    *timer_task_add_common   (int (*callback)(void *), void *arg, int interval_ms);
int                     timer_task_remove_common(timer_task_handle_t *task);
```

Legacy: `TimerTaskH / TimerManagerH`, `timer_manager_new/free`, `timer_create_task / add_task / remove_task / add_task_common / remove_task_common`.

#### Function reference

| Function | Purpose | Parameters / return | Notes |
|---|---|---|---|
| `timer_init` / `timer_deinit` | Subsystem init/deinit | 0 / < 0 | Once per process |
| `timer_manager_create` | Create task scheduler manager | Handle or `NULL` on failure | Starts internal timer thread |
| `timer_manager_destroy` | Destroy manager (wait for thread, free all tasks) | — | Idempotent |
| `timer_task_create` | Create periodic task (not added to manager) | `callback(arg)` returning non-zero cancels task; `interval_ms` period | Handle or `NULL` on failure |
| `timer_task_add` | Attach task to manager and start scheduling | 0 / < 0 | — |
| `timer_task_remove` | Remove from manager and destroy task | — | Blocks until callback will not run again |
| `timer_task_add_common` | Create and register on shared global manager | Same as `create + add` | No need to hold manager yourself |
| `timer_task_remove_common` | Remove from shared manager | — | — |

### 7.3 Error codes (Time domain)

`TIME_OK / _INVALID_PARAM / _NOT_INIT / _TIMEOUT / _OPERATION_FAILED / _INVALID_FORMAT(0x0017) / _TIMER_CREATE_FAILED(0x0018) / _TIMER_START_FAILED(0x0019) / _TIMER_STOP_FAILED(0x001A)`.

---

## 8. Watchdog — `watchdog.h`

```c
int  watchdog_init(int timeout_sec);
int  watchdog_deinit(void);
void watchdog_reboot_device(void);
int  watchdog_delay_reboot_device(int delay_time);
int  watchdog_set_enable(int enable);
int  watchdog_get_enable(void);
```

| Function | Purpose | Parameters / return | Notes |
|---|---|---|---|
| `watchdog_init` | Open kernel `/dev/watchdog`, set timeout | `timeout_sec` applied to hardware; 0 / error | Must be after `thread_init` |
| `watchdog_deinit` | Close watchdog device | — | Clean shutdown to avoid reset |
| `watchdog_reboot_device` | Trigger reboot immediately (magic write) | Does not return | Use with `reboot_info_set()` |
| `watchdog_delay_reboot_device` | Reboot after delay (ms) | `delay_time` ms; 0 / < 0 | Gives upper layers time to finish |
| `watchdog_set_enable` | Enable/disable feed thread | `enable` 0/1; returns 0 | When disabled, feed manually or accept timeout |
| `watchdog_get_enable` | Query feed thread state | Returns 0/1 | — |

Error codes: `WATCHDOG_OK / _INVALID_PARAM / _NOT_INIT / _TIMEOUT / _INIT_FAILED(0x0017) / _FEED_FAILED(0x0018) / _WATCHDOG_TIMEOUT(0x0019)`.

---

## 9. Memory management — `system_memory.h`

```c
void *system_malloc_do(int iSize, const char *pfun, int iLine);
#define system_malloc(x)  system_malloc_do((x), __FUNCTION__, __LINE__)
int   system_free(void *p);
int   system_mem_init(void);
int   system_mem_deinit(void);
int   system_mem_get_info(char *pBuff, int iBufLen);

/* DMA */
void *system_dma_alloc(uint32_t size);
int   system_dma_alloc_ex(uint32_t size, void **vaddr, uint64_t *paddr);
int   system_dma_free(void *vaddr);
int   system_dma_vaddr2paddr(void *vaddr, uint64_t *paddr);
```

| Function | Purpose | Parameters / return | Notes |
|---|---|---|---|
| `system_malloc(x)` | Macro: allocation with `__FUNCTION__/__LINE__` tracking | `x` bytes; `void *` / `NULL` | Like `malloc` with tracking |
| `system_malloc_do` | Low-level implementation (usually via macro) | Same as above | — |
| `system_free` | Free memory | 0 / < 0 | `NULL`-safe |
| `system_mem_init` / `_deinit` | Optional memory statistics subsystem | 0 / < 0 | Usually called early in runtime |
| `system_mem_get_info` | Write allocation statistics (string dump) | `pBuff`/`iBufLen` caller-allocated | For diagnostics |
| `system_dma_alloc` | Allocate physically contiguous DMA memory | Virtual address or `NULL` | Free with `system_dma_free` |
| `system_dma_alloc_ex` | Return both `vaddr` and `paddr` | 0 / < 0 | For handing off to hardware |
| `system_dma_free` | Free DMA memory | `vaddr` must be from `system_dma_alloc*` | — |
| `system_dma_vaddr2paddr` | `vaddr` → `paddr` | `paddr` [out] | — |

Error codes: `MEMORY_OK / _INVALID_PARAM / _NOT_INIT / _MEMORY / _OPERATION_FAILED / _ALLOC_FAILED(0x0017) / _FREE_FAILED(0x0018) / _BUFFER_OVERFLOW(0x0019) / _BUFFER_UNDERFLOW(0x001A) / _INVALID_POINTER(0x001B)`.

---

## 10. Encryption/decryption and checksums

### 10.1 Base64 — `base64.h`

```c
int base64_encode(const uint8_t *input, size_t input_length, uint8_t *output, size_t output_capacity);
int base64_decode(const uint8_t *input, size_t input_length, uint8_t *output, size_t output_capacity);
```

| Function | Purpose | Parameters / return | Notes |
|---|---|---|---|
| `base64_encode` | Standard Base64 encode (no line breaks) | `output_capacity` ≥ `4 * ((input_length + 2) / 3) + 1`; bytes written or < 0 | NUL-terminated output |
| `base64_decode` | Standard Base64 decode | `output_capacity` ≥ `input_length * 3 / 4 + 1`; decoded bytes or < 0 | Trailing newline OK; illegal chars rejected |

### 10.2 CRC32 — `crc32.h`

```c
typedef struct { uint32_t value; } crc32_context_t;
int      crc32_context_init   (crc32_context_t *ctx);
int      crc32_context_update (crc32_context_t *ctx, const void *data, size_t length);
int      crc32_context_final  (crc32_context_t *ctx, uint32_t *crc);
uint32_t crc32_compute        (const void *data, size_t length);

/* Legacy */
typedef crc32_context_t CRC32_CTX;
static inline int      crc32_init  (CRC32_CTX *ctx);
static inline int      crc32_update(CRC32_CTX *ctx, const void *buf, size_t size);
static inline int      crc32_final (CRC32_CTX *ctx, unsigned int *crc);
static inline unsigned int crc32   (const void *buf, size_t size);
```

| Function | Purpose | Notes |
|---|---|---|
| `crc32_context_init` | Init CRC context (`value=0xFFFFFFFF`) | Must init before reusing ctx |
| `crc32_context_update` | Accumulate data into ctx | May call multiple times |
| `crc32_context_final` | Finish and output CRC | `crc` [out] |
| `crc32_compute` | One-shot compute | Same as init+update+final |
| `crc32` (legacy) | One-shot for old API | Same as above |

### 10.3 MD5 — `md5.h`

```c
#define HASHLEN     16
#define HASHHEXLEN  32
typedef char HASH[HASHLEN];
typedef char HASHHEX[HASHHEXLEN + 1];
typedef struct { UINT4 state[4]; UINT4 count[2]; unsigned char buffer[64]; } MD5_CTX;
typedef enum { EN_MD5AUTH_ALGO_MD5, _SESS, _INVALID } EN_MD5AUTH_ALGO;

void MD5Init  (MD5_CTX *context);
void MD5Update(MD5_CTX *context, unsigned char *input, unsigned int inputLen);
void MD5Final (unsigned char digest[16], MD5_CTX *context);

void MD5Auth_CvtHex             (HASH bin, HASHHEX hex);
void MD5Auth_DigestCalcHA1      (const char *authId, const char *password, unsigned char alg,
                                 const char *realm, const char *nonce, const char *cnonce, HASHHEX sessionKey);
void MD5Auth_DigestCalcResponse (HASHHEX ha1, const char *nonce, const char *nc, const char *cnonce,
                                 const char *qop, const char *method, const char *uri, HASHHEX entity, HASHHEX response);
int  MD5Auth_BuildAuthResonseWithMD5(char *resp, unsigned short maxlen, unsigned char alg,
                                     const char *auth, const char *realm, const char *pass, const char *nonce,
                                     const char *nc, const char *cnonce, const char *qop,
                                     const char *method, const char *uri);
```

| Function | Purpose | Notes |
|---|---|---|
| `MD5Init` / `MD5Update` / `MD5Final` | Standard MD5 streaming triple | `digest[16]` is 128-bit output |
| `MD5Auth_CvtHex` | 16-byte bin → 32-byte hex string | `hex` is `HASHHEX` (33 bytes incl. `\0`) |
| `MD5Auth_DigestCalcHA1` | HTTP Digest: compute HA1 | Output in `sessionKey` |
| `MD5Auth_DigestCalcResponse` | Compute HTTP Digest Response | — |
| `MD5Auth_BuildAuthResonseWithMD5` | Build full `Authorization` header | `resp` buffer at least `maxlen` bytes |

### 10.4 Crypto error codes

`CRYPTO_OK / _INVALID_PARAM / _MEMORY / _OPERATION_FAILED / _INVALID_DATA(0x0017) / _ENCODE_FAILED(0x0018) / _DECODE_FAILED(0x0019) / _CHECKSUM_MISMATCH(0x001A)`.

---

## 11. Utility functions

### 11.1 Strings — `string_utils.h`

```c
int string_toupper (char *str);
int string_find    (const char *buf, int buf_len, const char *finding_str, int finding_len, int direction);
int process_find   (const char *process_name);
```

| Function | Purpose | Parameters / return |
|---|---|---|
| `string_toupper` | In-place upper case | `str` must be writable (not literal); 0 / < 0 |
| `string_find` | Search substring in arbitrary `buf` (may be binary) | `direction` 0=forward / 1=reverse; offset or < 0 |
| `process_find` | Find process by name | pid if exists, else `0`/ < 0 |

Error codes: `STRING_UTILS_OK / _INVALID_PARAM / _OPERATION_FAILED / _TOO_LONG(0x0019)`.

### 11.2 Geometry — `geometry.h`

```c
typedef struct { double x, y, width, height; } uv_rect_t;

int   rect_right       (const rect_t *rect);
int   rect_bottom      (const rect_t *rect);
int   rect_intersection(const rect_t *src1, const rect_t *src2, rect_t *dst);
int   rect_to_uv_rect  (const rect_t *src, int max_width, int max_height, uv_rect_t *dst);
float percent_to_real_value(float min, float middle, float max, int percent);
double percent_to_value    (int percent, const double *value_list, int value_count);
```

| Function | Purpose | Parameters / return |
|---|---|---|
| `rect_right` / `rect_bottom` | Right/bottom edge (= `x + width` / `y + height`) | Integer coordinates |
| `rect_intersection` | Intersection of two rectangles | **Returns 1=overlap, 0=none**; `dst` zeroed if none |
| `rect_to_uv_rect` | Pixel rect → normalized `[0,1]` UV | `max_width/max_height` = source image size |
| `percent_to_real_value` | Three-segment linear map (`min → middle → max`) | `percent` 0–100 |
| `percent_to_value` | Map percent to `value_list` index | `value_count` = array length |

### 11.3 Pixel format — `system_pixel_format.h`

Defines `rect_t` and system pixel format enums (see header for details).

### 11.4 Image conversion — `image_convert.h`

```c
int  yuv420sp_roi_resize              (unsigned char *yuv420sp_in, unsigned short w, unsigned short h,
                                       unsigned short x1, unsigned short y1, unsigned short rw, unsigned short rh,
                                       unsigned short ow, unsigned short oh, unsigned char *yuv420sp_out);
int  yuv420sp_roi_copy                (unsigned char *yuv420sp_in, unsigned short w, unsigned short h,
                                       unsigned short ox, unsigned short oy, unsigned short ow, unsigned short oh,
                                       unsigned char *yuv420sp_out);
int  yuv420sp_roi_resize_rgbline_version1(/* ... */);
int  yuv420sp_roi_resize_rgbline_version2(/* ... */);
int  rgbline_to_yuv_420sp             (unsigned char *pin, int width, int height, unsigned char *pout);
void save_rgbline_and_yuv             (void *rgb_data, const char *save_path, int h, int w, int count);
int  rgbline_roi_resize_version1      (/* ... */);
int  rgbline_roi_resize_version2      (/* ... */);
int  rgbline_roi_resize_version3      (/* ... */);
```

Error codes: `IMAGE_CONVERT_OK / _INVALID_PARAM / _OPERATION_FAILED / _YUV_CONVERT_FAILED(0x001A)`.

### 11.5 File utilities — `system_file.h`

```c
int  system_file_read      (const char *file_path, char *buffer, size_t buffer_len);
int  system_file_write     (const char *file_path, const char *data, size_t data_len);
int  system_file_read_all  (const char *file_path, char *buffer, size_t max_len);
int  system_get_file_path  (char *pInput, char *pOutput, int iOutLen);
int  system_get_file_name  (char *pInput, char *pOutput, int iOutLen);
int  system_get_file_size  (char *pc_file);
```

| Function | Purpose | Parameters / return |
|---|---|---|
| `system_file_read` | Read up to `buffer_len` bytes from file | Bytes read or < 0 |
| `system_file_write` | Write `data` to file (overwrite) | Bytes written or < 0 |
| `system_file_read_all` | Read entire file into `buffer` (`max_len` cap) | File size or < 0 |
| `system_get_file_path` | Extract directory from full path | `pOutput/iOutLen` caller buffer |
| `system_get_file_name` | Extract file name (with extension) | Same as above |
| `system_get_file_size` | File size in bytes | < 0 on failure |

---

## 12. Containers

### 12.1 `list.h` (Linux-style intrusive doubly linked list)

Macros / inline functions:
- `struct list_head { next, prev };`
- `LIST_HEAD(name)`, `LIST_HEAD_INIT(name)`, `INIT_LIST_HEAD(list)`
- `offset_of(type, member)`, `container_of(ptr, type, member)`, `LIST_POISON1/2`
- `list_add / list_add_tail / list_del / list_del_init / list_replace / list_replace_init`
- `list_is_last / list_empty / list_empty_careful`
- `list_splice / list_splice_tail / list_splice_init`
- `list_entry / list_first_entry`, `list_for_each`, `list_for_each_safe`

### 12.2 `ring_buffer.h` (single-consumer message ring buffer)

```c
#define MSG_CNT_MAX 100
#define RING_BUFFER_MSG_COUNT_DEFAULT 32

typedef struct { char *buf; int buf_len; } ring_buffer_buf_t;
typedef struct {
    ring_buffer_buf_t *msgs;
    int head, tail, count, max_count;
    GMutex mutex;
} ring_buffer_t;

int ring_buffer_buf_create  (ring_buffer_buf_t *buf);
int ring_buffer_buf_clear   (ring_buffer_buf_t *buf);
int ring_buffer_buf_copy_in (ring_buffer_buf_t *buf, const char *data, int data_len);
int ring_buffer_buf_copy_out(ring_buffer_buf_t *buf, char *data, int *data_len);

int ring_buffer_init   (ring_buffer_t *rb, int buf_count);
int ring_buffer_deinit (ring_buffer_t *rb);
int ring_buffer_msg_push_wait(ring_buffer_t *rb, const char *data, int data_len, int timeout_ms, int *wait_flag);
int ring_buffer_msg_push     (ring_buffer_t *rb, const char *data, int data_len);
int ring_buffer_msg_pop      (ring_buffer_t *rb, char *data, int *data_len);
```

| Function | Purpose | Parameters / return | Thread |
|---|---|---|---|
| `ring_buffer_buf_create` / `_clear` | Create/clear one message buffer (fixed capacity `buf_len`) | Internal `malloc` | Caller must lock |
| `ring_buffer_buf_copy_in` / `_copy_out` | Copy `data` in/out of one message buffer | Returns actual bytes | Caller must lock |
| `ring_buffer_init` | Init ring message queue (`buf_count` slots) | Default `RING_BUFFER_MSG_COUNT_DEFAULT=32`, max `MSG_CNT_MAX=100` | Call once |
| `ring_buffer_deinit` | Free all message slots | — | No active producer/consumer |
| `ring_buffer_msg_push` | Non-blocking enqueue | Full → `_QUEUE_FULL` | Producer thread |
| `ring_buffer_msg_push_wait` | Blocking enqueue (`timeout_ms`; `wait_flag` for external abort) | 0 / `_TIMEOUT` / `_QUEUE_FULL` | Producer thread |
| `ring_buffer_msg_pop` | Blocking dequeue into `data` (capacity `*data_len`) | `*data_len` [in/out]; blocks when empty | Single-consumer |

### 12.3 `circular_buffer.h` (multi-reader broadcast ring buffer)

```c
typedef struct circular_buffer_handle_s *circular_buffer_handle_t;

circular_buffer_handle_t circular_buffer_create (unsigned int mem_size);
int                      circular_buffer_destroy(circular_buffer_handle_t handle);

unsigned char *circular_buffer_get_write_buffer(circular_buffer_handle_t handle, unsigned int write_len);
int            circular_buffer_put_write_buffer(circular_buffer_handle_t handle, unsigned int write_len);

int  circular_buffer_reader_create (circular_buffer_handle_t handle, const char *name);
int  circular_buffer_reader_destroy(circular_buffer_handle_t handle, unsigned int reader_id);

int  circular_buffer_get_data      (circular_buffer_handle_t handle, unsigned int reader_id,
                                    unsigned char **data_buf, unsigned int *data_size, unsigned int timeout_ms);

#define CIRCULAR_BUFFER_HAS_GET_DATA_COPY 1
int  circular_buffer_get_data_copy(circular_buffer_handle_t handle, unsigned int reader_id,
                                   unsigned char *out_buf, unsigned int out_buf_size,
                                   unsigned int *out_data_len, unsigned int timeout_ms);
```

| Function | Purpose | Parameters / return | Thread |
|---|---|---|---|
| `circular_buffer_create` | Create multi-reader broadcast buffer (`mem_size` bytes) | Handle / `NULL` | One writer, many readers |
| `circular_buffer_destroy` | Destroy handle (evicts all readers first) | 0 / < 0 | No active readers |
| `circular_buffer_get_write_buffer` | Get contiguous write region (`write_len` bytes) | Returns `buf *`; blocks if insufficient space | **Single producer** |
| `circular_buffer_put_write_buffer` | Commit write (broadcast to all readers) | 0 / < 0 | Same as above |
| `circular_buffer_reader_create` | Register reader, assign `reader_id` | `reader_id ≥ 0` or < 0; `name` for diagnostics | Concurrent create OK |
| `circular_buffer_reader_destroy` | Unregister reader | 0 / < 0 | Idempotent |
| `circular_buffer_get_data` | Zero-copy read: pointer + length (valid until next `get_data`) | `timeout_ms=0` non-blocking; `UINT32_MAX` block forever | One thread per `reader_id` |
| `circular_buffer_get_data_copy` | Copy into `out_buf` (caller owns memory) | Exceeds `out_buf_size` → `_BUFFER_OVERFLOW` | Same as above |

---

## 13. Helper macros (non-API headers)

### 13.1 `system_macros.h`

Common general macros; included by default from `system_types.h`. See header for full list.

### 13.2 `system_assert.h`

```c
SYSTEM_ASSERT(exp)                                /* 失败则 exit(1) */
SYSTEM_ASSERT_TRUE(cond) / SYSTEM_ASSERT_FALSE(cond)

SYSTEM_TEST(cond, verbose)                         /* 测试条件，返回 SYSTEM_true/false */
SYSTEM_EXPECT(cond)                                /* verbose 版本 */
SYSTEM_EXPECT_RETURN(cond)
SYSTEM_EXPECT_RETURN_VAL(cond, ret)
SYSTEM_EXPECT_JUMP(cond, location)
SYSTEM_EXPECT_BREAK(cond) / SYSTEM_EXPECT_CONTINUE(cond)

SYSTEM_EXPECT_VERBOSE_RETURN(cond)
SYSTEM_EXPECT_VERBOSE_RETURN_VAL(cond, ret)
SYSTEM_EXPECT_VERBOSE_JUMP(cond, location)
SYSTEM_EXPECT_VERBOSE_BREAK(cond) / SYSTEM_EXPECT_VERBOSE_CONTINUE(cond)

SYSTEM_CHECK0_RET_ERROR(expr)                      /* expr 非 0 即打印并 return */
```

### 13.3 `system_reentry.h`

```c
REENTRY_INIT(ModuleName)       /* 定义静态计数器 */
REENTRY_START(ModuleName)      /* 计数器 +1 */
REENTRY_STOP(ModuleName)       /* 计数器 = 0 */
REENTRY_CHECK(ModuleName)                          /* 若已在用则 log_error 并 return 0 */
REENTRY_CHECK_NO_PRINT(ModuleName)                 /* 同上，但不打日志 */
REENTRY_GET_INIT_STATUS(ModuleName)                /* 返回计数器当前值 */
REENTRY_CHECK_INIT_RETURN(ModuleName)              /* 未初始化则 log_error 并 return -1 */
REENTRY_CHECK_INIT_RETURN_NULL(ModuleName)         /* 未初始化则 log_error 并 return NULL */
```

Requires `#include "log.h"` and `#define LOG_TAG "..."` before use.

### 13.4 `system_debug.h`

```c
SYSTEM_CHECK_POINT()   /* 打印 "--  @  file:line" */
```

---

## 14. Dependency constraints

1. System Core may depend only **downward** on BSP/Kernel and the standard C library; **must not** depend on Framework / Hardware HAL / Application.
2. Module error codes must be encoded with `SYSTEM_ERROR_CODE(domain, module, error)`; do not return raw `errno`—convert with `system_error_from_errno()`.
3. All public APIs live in `system/core/include/*.h`; `system/core/src/**/*.c` is private implementation—other layers must not include it.

---

## 15. References

- Concepts and call flow: [system-core.en.md](system-core.en.md)
- Framework: [framework.en.md](framework.en.md)
- HAL: [hardware-hal.en.md](hardware-hal.en.md)
- Where Application uses these APIs: [application.en.md](application.en.md)
- Global API index: [api-index.en.md](api-index.en.md)