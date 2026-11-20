<p align="right"><a href="system-core-api.en.md">English</a> | <b>中文</b></p>

# System Core API 参考手册（对外可查接口）

> **本文范围**：System Core 对外可查接口参考：错误码位布、日志、配置、线程、时间、内存、容器与工具函数。
>
> 概念、职责与 Boot 位置见 [system-core.md](system-core.md)；分层、契约与运行期模型见 [架构总览](../architecture/overview.md)；术语以 [术语与命名对照表](../glossary.md) 为唯一解释源。

---

## 0. 速查目录

| # | 头文件 | 模块号（error 域） | 说明 |
|---|--------|-------------------|------|
| 1 | `system_core.h`        | — | 单一入口，按 `SYSTEM_CORE_IMPORT_*` 宏裁剪 |
| 2 | `system_error.h`       | `SYSTEM_DOMAIN(0x00)` / 所有域 | 统一错误码编码与基础错误码 |
| 3 | `system_types.h`       | — | 基础类型、函数指针、`SYSTEM_true/false` |
| 4 | `system_version.h`     | — | System Core 版本号宏 |
| 5 | `log.h`                | `SYSTEM_MODULE_LOG(0x02)` | 统一日志系统 |
| 6 | `system_config.h`     | `SYSTEM_MODULE_CONFIG(0x01)` | INI / 正则 / JSON 配置管理 |
| 7 | `thread.h`             | `SYSTEM_MODULE_THREAD(0x03)` | 线程与读写锁 |
| 8 | `gthread.h`            | — | GLib 风格线程/互斥抽象 |
| 9 | `system_time.h`        | `SYSTEM_MODULE_TIME(0x06)` | 系统时间 |
| 10 | `timer.h`             | `SYSTEM_MODULE_TIME(0x06)` | 软件定时器任务 |
| 11 | `watchdog.h`          | `SYSTEM_MODULE_WATCHDOG(0x09)` | 看门狗 |
| 12 | `system_memory.h`     | `SYSTEM_MODULE_MEMORY(0x04)` | 内存分配与 DMA 内存 |
| 13 | `md5.h`               | — | MD5 算法 |
| 14 | `base64.h`            | `SYSTEM_MODULE_CRYPTO(0x07)` | Base64 编解码 |
| 15 | `crc32.h`             | `SYSTEM_MODULE_CRYPTO(0x07)` | CRC32 校验 |
| 16 | `string_utils.h`      | `SYSTEM_MODULE_UTILS(0x08)` | 字符串工具 |
| 17 | `geometry.h`          | `SYSTEM_MODULE_UTILS(0x08)` | 几何/矩形工具 |
| 18 | `image_convert.h`     | `SYSTEM_MODULE_UTILS(0x08)` | YUV/RGB 转换 |
| 19 | `system_pixel_format.h` | — | 像素格式与 `rect_t` |
| 20 | `system_file.h`       | — | 文件读写封装 |
| 21 | `list.h`              | — | 内核风格双向链表 |
| 22 | `ring_buffer.h`       | `SYSTEM_MODULE_MEMORY(0x04)` | 环形消息缓冲 |
| 23 | `circular_buffer.h`   | — | 多读者广播型环形缓冲 |
| 24 | `system_macros.h`     | — | 通用宏 |
| 25 | `system_assert.h`     | — | 断言/期望宏 |
| 26 | `system_reentry.h`    | — | 重入保护宏 |
| 27 | `system_debug.h`      | — | 调试检查点 |

---

## 1. 错误码 — `system_error.h`

### 1.1 类型与位布局

```c
typedef int32_t system_error_t;   /* 负值 = 错误；0 = 成功 */
/* Bit layout:  [31]=sign  [24..30]=domain  [16..23]=module  [0..15]=error */
```

### 1.2 Domain / Module

| 宏 | 值 | 说明 |
|---|----|------|
| `SYSTEM_DOMAIN`    | 0x00 | system/core 层 |
| `FRAMEWORK_DOMAIN` | 0x01 | framework 层 |
| `HARDWARE_DOMAIN`  | 0x02 | hardware 层 |

System 模块：`SYSTEM_MODULE_CORE(0x00)`、`CONFIG(0x01)`、`LOG(0x02)`、`THREAD(0x03)`、`MEMORY(0x04)`、`STORAGE(0x05)`、`TIME(0x06)`、`CRYPTO(0x07)`、`UTILS(0x08)`、`WATCHDOG(0x09)`、`SOC_PLATFORM(0x0A)`

Framework 模块：`NETWORK(0x00)`、`NOTIFICATION(0x01)`、`MULTIMEDIA(0x02)`、`RUNTIME(0x03)`、`DEVICE(0x04)`、`UI(0x05)`、`AI(0x06)`、`STORAGE(0x07)`、`GATEWAY(0x08)`、`NEBULA_PROTOCOL(0x09)`

Hardware 模块：`WIFI(0x00)`、`AUDIO(0x01)`、`CAMERA(0x02)`、`VISION(0x03)`、`DISPLAY(0x04)`、`DRIVER(0x05)`、`GPU(0x06)`、`OSD(0x07)`、`INPUT(0x08)`、`AI(0x09)`、`ETHERNET(0x0A)`

### 1.3 通用错误码（0x0000–0x0016）

`ERROR_SUCCESS / _INVALID_PARAM / _NOT_INIT / _MEMORY / _TIMEOUT / _NOT_FOUND / _ALREADY_EXISTS / _BUSY / _IO / _PERMISSION_DENIED / _NOT_SUPPORTED / _INVALID_STATE / _OPERATION_FAILED / _DEVICE_NOT_AVAILABLE / _CONFIG_ERROR / _NETWORK / _CONNECTION_FAILED / _QUEUE_FULL / _QUEUE_EMPTY / _CLIENT_FULL / _INVALID_TYPE / _NETWORK_UNREACHABLE / _INTERFACE_NOT_FOUND`

> 0x0017 起为模块私有错误码。

### 1.4 宏与函数

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

> 各模块专用错误码（`LOG_ERROR_*`、`CONFIG_ERROR_*`、`THREAD_ERROR_*` 等）**内联定义在各模块公共头**（`log.h`、`system_config.h`、`system_thread.h` …）中，引用本头的 `SYSTEM_DOMAIN` / `SYSTEM_MODULE_*` 表组装；`system_error.h` 只集中域码 / 模块码表与通用 `ERROR_*`。

### 1.5 System Core 层预定义错误

`SYSTEM_SUCCESS(=0)`、`SYSTEM_FAILED`、`SYSTEM_ERROR_*`（对所有 `ERROR_*` 通用码做 Core 域宏），
文件类：`SYSTEM_ERROR_FILE_NOT_FOUND / _FILE_OPEN_FAILED / _FILE_READ_FAILED / _FILE_WRITE_FAILED`。

---

## 2. 基础类型与版本

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

### 2.3 `system_core.h` 聚合入口

控制开关（默认全部开启）：
`SYSTEM_CORE_IMPORT_ALL`、`_CONFIG`、`_LOG`、`_THREAD`、`_TIME`、`_TIMER`、`_UTILS`、`_IMAGE_CONVERT`、`_BASE64`、`_CRC32`、`_MD5`、`_CIRCULAR_BUFFER`、`_MEMORY`、`_WATCHDOG`。

基础宏：`MAX(x,y)`、`MIN(x,y)`、`PARAM_IN_RANGE(v, min, max)`。

> ⚠️ 聚合为**部分聚合**：即使 `SYSTEM_CORE_IMPORT_ALL=1`，`system_core.h` 也不会带入 `system_macros.h`、`system_assert.h`、`system_debug.h`、`system_reentry.h`、`system_file.h`、`system_pixel_format.h`、`list.h`、`ring_buffer.h`，使用这些头须直接 `#include`。线程模块实际引入的是 `system_thread.h`（非 `thread.h`）。构建产物同时含 `libsystem_core.so` 与 `libsystem_core.a`。

---

## 3. 日志 — `log.h`

### 3.1 级别

```c
enum COMMON_LOG_LEVEL { LOG_FATAL = -1, LOG_ERROR, LOG_WARNING, LOG_INFO, LOG_DEBUG, LOG_VERBOSE };
```

### 3.2 结构与回调

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

### 3.3 函数

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

### 3.4 函数详述

| 函数 | 功能 | 关键参数 / 返回 | 线程 |
|---|---|---|---|
| `log_print_base` | 底层打印接口（宏 `log_error/warn/info/debug/verbose` 展开到此） | `level`：见 §3.1；`tag`：模块标签；`file/fun/line`：通常用 `__FILE__/__FUNCTION__/__LINE__`；`fmt` 及变参遵循 `printf`；返回实际写入字节数 | 任意线程；内部带锁 |
| `log_print_set_filter` | 设置过滤条件（级别/关键字/行号区间/时间窗口） | `pLogFilterInfo` [in] 非空；返回 `LOG_OK/_INVALID_PARAM` | 任意；当前仅一条全局过滤 |
| `log_print_set_cb` | 注册**打印输出**回调（接管屏幕/串口输出） | `cb` 为 `NULL` 则恢复默认；返回 0/< 0 | 回调会在各打印线程调用；**禁止阻塞** |
| `log_print_set_save_file_cb` | 注册**落盘**回调（接管日志文件写入） | 同上 | 回调在内部写盘线程调用 |
| `log_get_save_path` | 读取当前日志保存目录 | 返回常量字符串，调用方**不可**`free` | 任意 |
| `log_set_level` | 设置全局最低日志级别 | 低于该级别将被静默丢弃 | 任意 |
| `log_write_init` | 初始化异步落盘子系统（打开文件、启动写线程） | 返回 0 / 错误 | 单次调用；多次为幂等 |
| `log_write_deinit` | 停止落盘线程并冲刷缓冲 | — | 退出前调用 |

### 3.5 辅助宏

`log_error(fmt, ...)`、`log_warn(fmt, ...)`、`log_info(fmt, ...)`、`log_debug(fmt, ...)`、`log_verbose(fmt, ...)`
使用前需 `#define LOG_TAG "YOUR_TAG"`。

### 3.6 错误码

`LOG_OK`、`LOG_ERROR_INVALID_PARAM / _NOT_INIT / _MEMORY / _OPERATION_FAILED / _INVALID_LEVEL(0x0017) / _FILE_OPEN_FAILED(0x0018) / _WRITE_FAILED(0x0019) / _BUFFER_FULL(0x001A)`。

---

## 4. 配置管理 — `system_config.h`

### 4.1 基础类型

```c
typedef struct _dictionary_ dictionary;
typedef system_error_t      config_error_t;
```

### 4.2 原生 iniparser API

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

### 4.3 扩展辅助 API

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

### 4.4 正则与 JSON

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

Legacy 兼容：`ERGEX_HING_BUFFFULL / CONF_LEN_RES / CONF_LEN_PATTERN / CONF_LEN_ERROR / CONF_REG_STYLE / ACTION_PTRN_* / ACTION_COMP_* / SPLIT_* / EXEC_*` 以及 inline 包装器 `read_file / get_conf_str / get_conf_int / regexpr_match / regexpr_replace` 和宏 `regexpr_get(...)`（通过 `CONFIG_DISABLE_REGEX_LEGACY_APIS` 关闭）。

### 4.5 函数详述（常用子集）

> 完整 `iniparser_*` 遵循 iniparser 3.x 语义：`d` 为 `iniparser_load` 返回的词典，`key` 形如 `"section:keyname"`；所有读取接口在 key 不存在时返回 `def`/`notfound`。

| 函数 | 功能 | 参数 / 返回 | 备注 |
|---|---|---|---|
| `iniparser_load` | 加载 `.ini` 文件到词典 | `ininame` [in] 路径；返回词典指针或 `NULL` | 词典由调用方 `iniparser_freedict` 释放 |
| `iniparser_freedict` | 释放词典 | `d` [in] | 幂等；`NULL` 安全 |
| `iniparser_getstring` | 读字符串（`def` 兜底） | 返回只读指针，内存归词典所有 | 词典销毁后指针失效 |
| `iniparser_getint` / `_getlongint` / `_getdouble` / `_getboolean` | 读整/浮点/布尔 | 解析失败返回 `notfound` | — |
| `iniparser_set` / `_seti` / `_set_double` | 写入字符串/整数/浮点 | `entry` 形如 `"sect:key"`；新建 section 需先 `set` 任一键 | 未 `save_file` 不落盘 |
| `iniparser_save_file` | 将词典写入文件 | `ini`, `pFileName` | 失败返回 `-1` |
| `iniparser_dict_get*` | 按 `(section, key)` 两段访问，避免手工拼接字符串 | `defaultValue/notfound` 同上 | 推荐使用 |
| `iniparser_file_getstring` / `_getint` / `_setstring` / `_setint` | **一次性** load → get/set → save 的便捷封装 | `buf`/`len` 缓冲与容量 | 避免手工管理 `dictionary*` |
| `config_regex_get_string` / `_get_int` | 对**非 INI**自研格式：用正则在文本中提取字段 | `conf` 文本体，`key` 匹配模式；`value` 返回 `malloc` 字符串（调用方 `free`） | 仅适合小文件；大文件用 `iniparser` |
| `config_regex_match` | 对已编译 regex 在 buffer 中搜索 | `split_option/exec_option` 见枚举 | 线程安全（`regex_t` 独立） |
| `config_regex_replace` | 替换 | 同上 | — |
| `cJSON_RemoveComments` | 剔除 `// / /* */` 注释，将宽松 JSON 转严格 JSON | 返回 `malloc` 字符串 | 便于在 cJSON 解析前预处理 |

### 4.6 路径管理

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

| 函数 | 功能 | 参数 / 返回 |
|---|---|---|
| `config_get_path_by_type` | 查询某路径类型对应的目录 | 返回常量字符串（如 `"/etc/config/"`），不可 `free` |
| `config_set_path` | 覆盖某类型默认路径（测试/定制场景） | `path` 必须以 `/` 结尾；返回 0/< 0 |
| `config_build_path` | 拼接 `<type_path>/<filename>` 到 `out_path` | `out_size` 给出缓冲容量；返回写入字节或 `< 0` |
| `config_build_file_path` | 自动识别 filename 属于何种类型（按前缀）再拼接 | 返回 0/< 0 |

### 4.7 错误码

`CONFIG_OK / _INVALID_PARAM / _NOT_INIT / _MEMORY / _NOT_FOUND / _CONFIG_ERROR / _OPERATION_FAILED / _INVALID_PATH(0x0017) / _FILE_NOT_FOUND(0x0018) / _PARSE_FAILED(0x0019) / _KEY_NOT_FOUND(0x001A) / _SECTION_NOT_FOUND(0x001B) / _WRITE_FAILED(0x001C)`。

---

## 5. 线程与锁 — `thread.h`

### 5.1 类型

```c
typedef int (*thread_function_t)(void *param);
typedef void *rw_lock_t;
typedef system_error_t thread_error_t;
```

### 5.2 函数

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

### 5.3 函数详述

| 函数 | 功能 | 参数 / 返回 | 线程 |
|---|---|---|---|
| `thread_init` | 初始化线程子系统（内部 bookkeeping） | `0` / `< 0` | 进程早期调用一次 |
| `thread_deinit` | 反初始化 | — | 退出前调用 |
| `thread_create` | 创建线程并绑定 `function(param)` | `handle` [out] 线程句柄（句柄 > 0 有效）；`name` 显示名；`stack_size` 字节（0 走默认）；`priority` 0–99（或负值默认） | 任意 |
| `thread_destroy` | 请求线程退出并回收句柄 | 句柄从 `thread_create` 获得 | 阻塞至 `function` 返回 |
| `thread_get_info` | 读取当前所有线程的状态 dump（用于诊断） | `buffer`/`buffer_len` | 任意 |
| `thread_check_in` | 在线程体内周期性上报存活 | 用于 watchdog 关联 | 每个被监控线程循环中调用 |
| `rw_lock_create` / `rw_lock_destroy` | 创建/销毁读写锁 | 返回 `rw_lock_t *`；销毁前须无持有者 | 锁对象可跨线程共享 |
| `rw_lock_read_lock` / `_read_unlock` | 共享读加/解锁 | 多读者可并存 | 阻塞式 |
| `rw_lock_try_read_lock` | 尝试读锁 | 成功返回 `0`，竞争返回非 `0` | 非阻塞 |
| `rw_lock_write_lock` / `_write_unlock` | 独占写加/解锁 | 等待所有读者释放 | 阻塞式 |
| `rw_lock_try_write_lock` | 尝试写锁 | 同 try_read | 非阻塞 |

### 5.4 辅助宏

```c
THREAD_RUN_DEF  (ThreadName)
THREAD_RUN_START(ThreadName, function, param, stack_size, priority)
THREAD_RUN_STOP (ThreadName)
THREAD_RUN_CHECK(ThreadName)
```

### 5.5 错误码

`THREAD_OK / _INVALID_PARAM / _NOT_INIT / _MEMORY / _OPERATION_FAILED / _CREATE_FAILED(0x0017) / _JOIN_FAILED(0x0018) / _MUTEX_INIT_FAILED(0x0019) / _MUTEX_LOCK_FAILED(0x001A) / _COND_INIT_FAILED(0x001B)`。

---

## 6. GThread 抽象 — `gthread.h`

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

### 6.1 函数详述

| 函数 | 功能 | 参数 / 返回 | 备注 |
|---|---|---|---|
| `g_thread_new` | 创建并启动线程 | `name`、`func(data)`、`data` | 失败返回 `NULL` |
| `g_thread_join` | 等待线程结束，回收 | 返回线程 `func` 的返回值 | 每线程只能 join 一次 |
| `g_mutex_init` / `_clear` | 互斥锁初始化/销毁 | `GMutex *` 栈对象或静态对象 | 未 init 的 mutex 不得使用 |
| `g_mutex_lock` / `_unlock` | 加/解锁 | — | 不可递归；同线程二次 lock 死锁 |
| `g_mutex_trylock` | 尝试加锁 | 返回非 0 = 成功加锁，0 = 已占用 | 非阻塞 |

> 注：GThread 抽象主要用于与 GLib 风格代码互操作；新代码推荐使用 `thread.h`。

---

## 7. 时间与定时器 — `system_time.h` / `timer.h`

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

#### 函数详述

| 函数 | 功能 | 参数 / 返回 | 备注 |
|---|---|---|---|
| `system_get_time` | 读取当前 UTC 时间（秒） | `out_time` [out] | 0 / 错误 |
| `system_set_time` | 设置系统时间（UTC 秒） | `tolerance_value`：允许与 NTP 偏差阈值（秒，超出才真正设置），`0` 强制 | 需 root |
| `system_timed` | 检查系统时间是否合理（已同步） | 返回 0 = 未同步；1 = 已同步 | 任意 |
| `system_get_sys_run_time64/_s/_ns` | 单调时间（开机起经过的 ms / s / ns） | 不受 NTP/setTime 影响 | 用于差值测量 |
| `system_get_date_time64` | UTC 日期时间（秒） | — | 与 `system_get_time` 等价的便捷封装 |
| `system_get_local_time64` | 本地日期时间（应用时区） | — | 遵循 `TZ` 环境变量 |
| `system_get_hms_time64` | 返回 `"HH:MM:SS"` 字符串 | 内部静态缓冲 | 非线程安全（拷贝后用） |
| `system_string_to_seconds` / `_secondsT` | 将 `"YYYY-MM-DD HH:MM:SS"` 转秒数 | 返回 `< 0` 格式错误 | `_T` 版本支持 `T` 分隔 |
| `system_time_to_seconds` | 由分量拼秒数（方便构造） | 年为 4 位 | — |
| `system_seconds_to_stringT` | 秒数 → `"YYYY-MM-DDTHH:MM:SS"` | 内部静态缓冲 | 非线程安全 |
| `system_get_date_to_string` | 当前日期 `"YYYY-MM-DD"` | 内部静态缓冲 | 同上 |
| `system_get_year_mon_day` | 分别写入 `*pYear/*pMon/*pDay` | 0 / 错误 | — |
| `system_sleep_ms` | 毫秒级睡眠（`nanosleep` 封装） | 被信号打断不续睡 | — |

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

Legacy：`TimerTaskH / TimerManagerH`、`timer_manager_new/free`、`timer_create_task / add_task / remove_task / add_task_common / remove_task_common`。

#### 函数详述

| 函数 | 功能 | 参数 / 返回 | 备注 |
|---|---|---|---|
| `timer_init` / `timer_deinit` | 子系统初始化/反初始化 | 0 / < 0 | 进程级一次即可 |
| `timer_manager_create` | 创建一个任务调度管理器 | 返回句柄，失败 `NULL` | 内部起定时线程 |
| `timer_manager_destroy` | 销毁管理器（等待线程退出，释放全部 task） | — | 幂等 |
| `timer_task_create` | 创建单个周期任务（不添加到 manager） | `callback(arg)` 返回非 0 时任务自动取消；`interval_ms` 周期（ms） | 返回句柄，失败 `NULL` |
| `timer_task_add` | 将任务挂到 manager 开始调度 | 0 / < 0 | — |
| `timer_task_remove` | 从 manager 移除并销毁 task | — | 阻塞至回调不再执行 |
| `timer_task_add_common` | 用"全局共享 manager"一步创建并注册 | 等价于 `create + add` | 不需要自行持有 manager |
| `timer_task_remove_common` | 从共享 manager 移除 | — | — |

### 7.3 错误码（Time 域）

`TIME_OK / _INVALID_PARAM / _NOT_INIT / _TIMEOUT / _OPERATION_FAILED / _INVALID_FORMAT(0x0017) / _TIMER_CREATE_FAILED(0x0018) / _TIMER_START_FAILED(0x0019) / _TIMER_STOP_FAILED(0x001A)`。

---

## 8. 看门狗 — `watchdog.h`

```c
int  watchdog_init(int timeout_sec);
int  watchdog_deinit(void);
void watchdog_reboot_device(void);
int  watchdog_delay_reboot_device(int delay_time);
int  watchdog_set_enable(int enable);
int  watchdog_get_enable(void);
```

| 函数 | 功能 | 参数 / 返回 | 备注 |
|---|---|---|---|
| `watchdog_init` | 打开内核 `/dev/watchdog`，设置超时 | `timeout_sec` 生效到硬件；返回 0 / 错误 | 必须在 `thread_init` 之后 |
| `watchdog_deinit` | 关闭 watchdog 设备 | — | 合理关闭以防重启 |
| `watchdog_reboot_device` | 立即触发重启（内部写魔数） | 不返回 | 配合 `reboot_info_set()` 用 |
| `watchdog_delay_reboot_device` | 延迟若干毫秒后重启 | `delay_time` ms；返回 0 / < 0 | 给上层留收尾时间 |
| `watchdog_set_enable` | 打开/关闭喂狗线程 | `enable` 0/1；返回 0 | 关闭后需自行喂狗或接受超时 |
| `watchdog_get_enable` | 查询当前喂狗线程状态 | 返回 0/1 | — |

错误码：`WATCHDOG_OK / _INVALID_PARAM / _NOT_INIT / _TIMEOUT / _INIT_FAILED(0x0017) / _FEED_FAILED(0x0018) / _WATCHDOG_TIMEOUT(0x0019)`。

---

## 9. 内存管理 — `system_memory.h`

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

| 函数 | 功能 | 参数 / 返回 | 备注 |
|---|---|---|---|
| `system_malloc(x)` | 宏：带 `__FUNCTION__/__LINE__` 追踪的内存分配 | `x` 字节；返回 `void *` / `NULL` | 与 `malloc` 等价但有追踪 |
| `system_malloc_do` | 底层实现（通常走宏调用） | 同上 | — |
| `system_free` | 释放内存 | 返回 0 / < 0 | `NULL` 安全 |
| `system_mem_init` / `_deinit` | 可选的内存统计子系统 | 0 / < 0 | 通常由 runtime 早期调用 |
| `system_mem_get_info` | 写入当前分配统计（字符串 dump） | `pBuff`/`iBufLen` 调用方分配 | 诊断用 |
| `system_dma_alloc` | 分配一段 DMA 连续内存（物理连续） | 返回虚拟地址或 `NULL` | 用 `system_dma_free` 释放 |
| `system_dma_alloc_ex` | 同时返回 `vaddr` + `paddr` | 0 / < 0 | 便于直接下发到硬件 |
| `system_dma_free` | 释放 DMA 内存 | `vaddr` 必须是 `system_dma_alloc*` 返回值 | — |
| `system_dma_vaddr2paddr` | 由 `vaddr` 反查 `paddr` | `paddr` [out] | — |

错误码：`MEMORY_OK / _INVALID_PARAM / _NOT_INIT / _MEMORY / _OPERATION_FAILED / _ALLOC_FAILED(0x0017) / _FREE_FAILED(0x0018) / _BUFFER_OVERFLOW(0x0019) / _BUFFER_UNDERFLOW(0x001A) / _INVALID_POINTER(0x001B)`。

---

## 10. 加/解密与校验

### 10.1 Base64 — `base64.h`

```c
int base64_encode(const uint8_t *input, size_t input_length, uint8_t *output, size_t output_capacity);
int base64_decode(const uint8_t *input, size_t input_length, uint8_t *output, size_t output_capacity);
```

| 函数 | 功能 | 参数 / 返回 | 备注 |
|---|---|---|---|
| `base64_encode` | 标准 Base64 编码（无换行） | `output_capacity` 需 ≥ `4 * ((input_length + 2) / 3) + 1`；返回写入字节或 `< 0` | 输出尾部写 `\0` |
| `base64_decode` | 标准 Base64 解码 | `output_capacity` 需 ≥ `input_length * 3 / 4 + 1`；返回解码后实际字节或 `< 0` | 容忍末尾换行，不容忍非法字符 |

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

| 函数 | 功能 | 备注 |
|---|---|---|
| `crc32_context_init` | 初始化 CRC 上下文（`value=0xFFFFFFFF`） | 复用 ctx 前必须 init |
| `crc32_context_update` | 累加一段数据到 ctx | 可多次调用 |
| `crc32_context_final` | 结束，输出 CRC | `crc` [out] |
| `crc32_compute` | 一次性计算 | 等价 `init+update+final` |
| `crc32` (legacy) | 兼容旧 API 的一次性计算 | 同上 |

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

| 函数 | 功能 | 备注 |
|---|---|---|
| `MD5Init` / `MD5Update` / `MD5Final` | 标准 MD5 流式计算三段式 | `digest[16]` 为 128 位输出 |
| `MD5Auth_CvtHex` | 将 16 字节 bin 转 32 字节 hex 字符串 | `hex` 为 `HASHHEX`（33 字节，含 `\0`） |
| `MD5Auth_DigestCalcHA1` | HTTP Digest 认证：计算 HA1 | 输出写入 `sessionKey` |
| `MD5Auth_DigestCalcResponse` | 计算 HTTP Digest Response | — |
| `MD5Auth_BuildAuthResonseWithMD5` | 构造完整 `Authorization` 头 | `resp` 至少 `maxlen` 字节缓冲 |

### 10.4 Crypto 错误码

`CRYPTO_OK / _INVALID_PARAM / _MEMORY / _OPERATION_FAILED / _INVALID_DATA(0x0017) / _ENCODE_FAILED(0x0018) / _DECODE_FAILED(0x0019) / _CHECKSUM_MISMATCH(0x001A)`。

---

## 11. 工具函数

### 11.1 字符串 — `string_utils.h`

```c
int string_toupper (char *str);
int string_find    (const char *buf, int buf_len, const char *finding_str, int finding_len, int direction);
int process_find   (const char *process_name);
```

| 函数 | 功能 | 参数 / 返回 |
|---|---|---|
| `string_toupper` | 原地将字符串转大写 | `str` 必须为可写（非字面量），返回 0 / < 0 |
| `string_find` | 在任意 `buf` 内搜索子串（可二进制） | `direction` 0=正向 / 1=反向；返回偏移或 `< 0` |
| `process_find` | 按进程名搜索是否存在 | 存在返回 pid，不存在返回 `0`/ `< 0` |

错误码：`STRING_UTILS_OK / _INVALID_PARAM / _OPERATION_FAILED / _TOO_LONG(0x0019)`。

### 11.2 几何 — `geometry.h`

```c
typedef struct { double x, y, width, height; } uv_rect_t;

int   rect_right       (const rect_t *rect);
int   rect_bottom      (const rect_t *rect);
int   rect_intersection(const rect_t *src1, const rect_t *src2, rect_t *dst);
int   rect_to_uv_rect  (const rect_t *src, int max_width, int max_height, uv_rect_t *dst);
float percent_to_real_value(float min, float middle, float max, int percent);
double percent_to_value    (int percent, const double *value_list, int value_count);
```

| 函数 | 功能 | 参数 / 返回 |
|---|---|---|
| `rect_right` / `rect_bottom` | 返回矩形右/下边界（= `x + width` / `y + height`） | 整型坐标 |
| `rect_intersection` | 计算两矩形交集 | **返回 1=有重叠、0=无重叠**；无重叠时 `dst` 置零 |
| `rect_to_uv_rect` | 将像素级矩形归一化为 `[0,1]` UV 坐标 | `max_width/max_height` 为原图尺寸 |
| `percent_to_real_value` | 三段线性映射（`min → middle → max`） | `percent` 0–100 |
| `percent_to_value` | 以百分比索引映射到 `value_list` | `value_count` 数组长度 |

### 11.3 像素格式 — `system_pixel_format.h`

定义 `rect_t` 以及系统像素格式枚举（详细请直接参阅头文件）。

### 11.4 图像转换 — `image_convert.h`

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

错误码：`IMAGE_CONVERT_OK / _INVALID_PARAM / _OPERATION_FAILED / _YUV_CONVERT_FAILED(0x001A)`。

### 11.5 文件工具 — `system_file.h`

```c
int  system_file_read      (const char *file_path, char *buffer, size_t buffer_len);
int  system_file_write     (const char *file_path, const char *data, size_t data_len);
int  system_file_read_all  (const char *file_path, char *buffer, size_t max_len);
int  system_get_file_path  (char *pInput, char *pOutput, int iOutLen);
int  system_get_file_name  (char *pInput, char *pOutput, int iOutLen);
int  system_get_file_size  (char *pc_file);
```

| 函数 | 功能 | 参数 / 返回 |
|---|---|---|
| `system_file_read` | 从文件读取最多 `buffer_len` 字节 | 返回实际读取字节或 `< 0` |
| `system_file_write` | 将 `data` 写入文件（覆盖） | 返回实际写入字节或 `< 0` |
| `system_file_read_all` | 将整个文件读入 `buffer`（`max_len` 限制） | 返回文件大小或 `< 0` |
| `system_get_file_path` | 从完整路径中提取目录部分 | `pOutput/iOutLen` 为调用方缓冲 |
| `system_get_file_name` | 提取文件名（含扩展） | 同上 |
| `system_get_file_size` | 返回文件字节大小 | 失败返回 `< 0` |

---

## 12. 容器

### 12.1 `list.h`（Linux 风格侵入式双向链表）

宏/内联函数：
- `struct list_head { next, prev };`
- `LIST_HEAD(name)`、`LIST_HEAD_INIT(name)`、`INIT_LIST_HEAD(list)`
- `offset_of(type, member)`、`container_of(ptr, type, member)`、`LIST_POISON1/2`
- `list_add / list_add_tail / list_del / list_del_init / list_replace / list_replace_init`
- `list_is_last / list_empty / list_empty_careful`
- `list_splice / list_splice_tail / list_splice_init`
- `list_entry / list_first_entry`、`list_for_each`、`list_for_each_safe`

### 12.2 `ring_buffer.h`（单读者消息环形缓冲）

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

| 函数 | 功能 | 参数 / 返回 | 线程 |
|---|---|---|---|
| `ring_buffer_buf_create` / `_clear` | 单个消息缓冲区创建/清空（定长缓冲，容量由 `buf_len` 指定） | 内部 `malloc` | 使用者加锁 |
| `ring_buffer_buf_copy_in` / `_copy_out` | 将 `data` 拷入/拷出单个消息缓冲 | 返回实际字节 | 使用者加锁 |
| `ring_buffer_init` | 初始化环形消息队列（分配 `buf_count` 条消息槽） | 默认 `RING_BUFFER_MSG_COUNT_DEFAULT=32`，上限 `MSG_CNT_MAX=100` | 单次调用 |
| `ring_buffer_deinit` | 释放所有消息槽 | — | 确保无生产者/消费者 |
| `ring_buffer_msg_push` | 非阻塞投递一条消息 | 满则返回 `_QUEUE_FULL` | 生产者线程 |
| `ring_buffer_msg_push_wait` | 阻塞投递（超时 `timeout_ms`；`wait_flag` 可用于外部打断） | 返回 0 / `_TIMEOUT` / `_QUEUE_FULL` | 生产者线程 |
| `ring_buffer_msg_pop` | 阻塞取出一条消息到 `data`（按 `*data_len` 容量） | `*data_len` [in/out]；队空则阻塞 | 单消费者场景 |

### 12.3 `circular_buffer.h`（多读者广播型环形缓冲）

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

| 函数 | 功能 | 参数 / 返回 | 线程 |
|---|---|---|---|
| `circular_buffer_create` | 创建多读者广播缓冲（容量 `mem_size` 字节） | 返回句柄 / `NULL` | 一写多读 |
| `circular_buffer_destroy` | 销毁 handle（会先踢出所有 reader） | 0 / < 0 | 调用前确保无活跃 reader |
| `circular_buffer_get_write_buffer` | 获取一块连续写入区（`write_len` 字节） | 返回 `buf *`；不足空间阻塞 | **单生产者** |
| `circular_buffer_put_write_buffer` | 提交写入（广播到所有 reader） | 0 / < 0 | 同上 |
| `circular_buffer_reader_create` | 注册一个 reader，分配 `reader_id` | 返回 `reader_id ≥ 0` 或 `< 0`；`name` 用于诊断 | 可并发创建多个 |
| `circular_buffer_reader_destroy` | 注销 reader | 0 / < 0 | 幂等 |
| `circular_buffer_get_data` | 零拷贝读取：取指针 + 长度（生命周期到下次 `get_data`） | `timeout_ms=0` 非阻塞；`UINT32_MAX` 永久阻塞 | 一个 `reader_id` 单线程调用 |
| `circular_buffer_get_data_copy` | 将数据拷贝到 `out_buf`（调用方拥有内存） | 超出 `out_buf_size` 返回 `_BUFFER_OVERFLOW` | 同上 |

---

## 13. 辅助宏（非函数头文件）

### 13.1 `system_macros.h`

常见通用宏；被 `system_types.h` 默认 `#include`。具体内容以头文件为准。

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

使用前需 `#include "log.h"` 并 `#define LOG_TAG "..."`。

### 13.4 `system_debug.h`

```c
SYSTEM_CHECK_POINT()   /* 打印 "--  @  file:line" */
```

---

## 14. 依赖约束

1. System Core 只能**向下**依赖 BSP/Kernel 与标准 C 库；**禁止**依赖 Framework / Hardware HAL / Application。
2. 各模块的错误码必须通过 `SYSTEM_ERROR_CODE(domain, module, error)` 编码；不要返回裸 `errno`，应使用 `system_error_from_errno()` 转换。
3. 所有公共 API 都暴露在 `system/core/include/*.h`；`system/core/src/**/*.c` 为私有实现，禁止其他层包含。

---

## 15. 参考

- 概念与调用流：[system-core.md](system-core.md)
- Framework：[framework.md](framework.md)
- HAL：[hardware-hal.md](hardware-hal.md)
- Application 使用这些 API 的位置：[application.md](application.md)
- 全局 API 索引：[guide/api-index.md](api-index.md)
