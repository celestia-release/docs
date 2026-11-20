---
name: celestia-aos-port
description: >-
  Celestia Linux 产品移植 AOS（RTOS）skill：把原本跑 Linux 的产品线（.so 动态链接、
  多进程假设）移植到 Aos 用户态（单静态 ELF）。
  覆盖：Linux↔Aos 差异与对应做法、三条隔离原则、AOS POSIX 支持接口清单（可用/链得上不可用/
  未实现/头有实现无四类 + 行为注意事项）、新增 POSIX 兼容接口的规范（compat 垫片三类情形、
  __wrap 替换 stub、SDK 升级迁移规则）、.so→.a 静态链接改造、静态 dlopen 表、
  HAL 对照 platform/sample、BSP 内核补丁、RomFS+config.lfs、最终链接 aos_user.ld、
  启动链路（app_entry→app_main→module_start）、Linux 回归与成对合并纪律、Aos 排障。
  新 AOS/RTOS 产品移植、aos 兼容层增补、Aos 链接/上板排障时使用。
---

<p align="right"><a href="celestia-aos-port.en.md">English</a> | <b>中文</b></p>

# Linux 产品移植 AOS（RTOS）skill

## 适用场景

- 新芯片 / 新 SDK 只提供 Aos 用户态（只产 `.a`，最终链成单个静态 ELF），而产品业务代码原生于 Linux。
- 既有 SoC 的 SDK 从 Linux 切到 Aos 版本。

**目标形态**：应用与 framework 源码与 Linux 产品**同源**；Aos 差异只落在 project overlay 的 POSIX 兼容层 + 芯片 HAL；Linux 产品编译行为零变化。

---

## 一、Linux ↔ Aos 差异与对应做法

移植前先建立差异地图，后续每一步都是落实其中一行的「做法」列。下表以 Sky39AV130 V1.01 实测为准，换 SDK 逐行重核。

| 维度 | Linux 产品假设 | Aos 注意与做法 |
|------|---------------|----------------|
| 链接形态 | 多个 `.so` 动态链接，`dlopen` 运行时加载 | 全部静态链成**单 ELF**：各层 Makefile 只产 `.a`；`dlopen` 由静态表顶替（步骤 4） |
| 进程模型 | `fork` / `exec` / `system` / `popen` 可用 | 一律不使用：worker 改 `thread_create`；shell 调用改真 syscall（如 `system("mount …")` → `mount()` + 格式化 API） |
| 线程栈 | glibc pthread，默认栈 8MB | 用户任务栈以 **64KB 为量级**：大局部数组、深递归逐个审计，改静态区或堆分配 |
| 内存 | 进程隔离，按进程回收 | 单地址空间：缓冲统一走静态区或芯片 DMA HAL（`ak_mem_*` 类），不假设自动回收 |
| 文件系统 | ext4/fat 可读写，`truncate` 可用 | RomFS **只读** + config.lfs（LittleFS）+ akfat（TF）；`ftruncate` 缩短按**失败路径处理**（返回 `EOPNOTSUPP` 是约定行为，不是 bug） |
| 调试手段 | gdb、strace、shell | 串口日志 + ucontext 崩溃栈：日志只打事实（sensor_id、路径、ret），`dladdr` 为崩溃栈占位 |
| C++ | glibc + libstdc++.so | CLIB 静态 libstdc++.a / libgcc.a **支持异常**：Aos 分支保留异常语义，勿加 `-fno-exceptions`（`-fno-rtti` 可保留） |
| 启动 | init → 启动脚本拉起 `<x>_main` | 内核直接装载 RomFS `usr/user.strip.elf`，入口 `app_entry`：BSP 打包见步骤 6 |
| 编译 ABI | 各目标可独立配置 | **全 ELF 编译标志一致**（enum ABI）：所有 `.o`、SDK `.a` 与 libgcc 的 `-mcpu/-mfloat-abi` 须与产品 `config.mk` / SDK `rules.mk` 相同；`hardware/Makefile` Aos 分支用 `CFLAGS +=`（`:=` 会冲掉 `-mcpu`） |

---

## 二、三条原则（违反即返工）

| 原则 | 说明 |
|------|------|
| **公用代码少改** | `ipc_cam`、framework 服务保持 OS/产品无关；Aos 差异放 `hardware/**/hal/<SoC>/` 与 project overlay 的 POSIX 增补 |
| **应用不按 OS 分叉** | 禁止在 `application` / `framework` 内为 OS 改 include 或加 `#if aos`；POSIX 缺口由 compat 垫片头在**包含路径上接住** |
| **隔离集中** | POSIX 增补只在 `project/<p>/system/<SoC>/<V>/platform/{include/alios/compat,src/aos}`；开关只在 `project/<p>/config.mk`；`system/core` 保持零 diff |

---

## 三、AOS POSIX 支持的接口清单

实现在 SDK 闭源预编译库（`platform/lib/alios/usyscall.a` / `uspace_posix.a` / `uspace_libc.a` / `uspace_aos.a`，逐符号 `nm` 实测）。移植前对业务代码用到的每个 POSIX 接口按下表分类，再按分类处理。

### ✅ 可直接使用

| 类别 | 接口 |
|------|------|
| 线程 / 同步 | `pthread_create` / `join` / mutex / cond / rwlock |
| 文件 / 目录 | `open/read/write/stat/opendir/readdir`、`statvfs/statfs`（接入前核实 struct 布局） |
| 网络 | socket 全套（lwip）、`poll`、`epoll_create1/ctl/wait`、`getaddrinfo` |
| 时间 | `usleep`、`nanosleep`、`CLOCK_REALTIME`、`CLOCK_MONOTONIC` |
| 其他 | `prctl(PR_SET_NAME)`、`mount/umount`、`reboot`、VFS 网关（`vfs_*`） |

此类**源码保持原样**——`#include <pthread.h>` 等标准头不动，缺的声明/宏由 compat 垫片头补。

### ⚠️ 链接得上，但按不可用对待

`alarm`、`setsid`、`umask`、`syscall`、`getgrnam`、`getpwnam`、`syslog`、`_isatty`、`getuid`、`readlink`、`shmget/shmat/shmdt/shmctl`、`get_nprocs*`、`get_phys_pages`、`get_avphys_pages`、`pthread_cancel`、`pthread_kill`、`pthread_atfork`、`timer_getoverrun`。

做法：换等价实现——`syslog` → `log_*`；`shmget` 共享内存 → 单进程内全局/静态区；`pthread_cancel` → 协作式退出标志。

### ❌ 完全没有实现

`fork`、`exec`、`system`、`popen`。

做法：业务代码中**必须消除**——`system("mount/mkfs …")` 改真 syscall（Mira 实例：`driver/blkdev/mmc.c` 的 Aos 分支用 `mount/umount` + `Driver_QuickFormat`）；`fork` 出 worker 的模式改 `thread_create`。

### ⚠️ 头文件有、实现是 stub

`getifaddrs` / `freeifaddrs` / `gai_strerror`（lwIP 提供 `ifaddrs.h`，实现为 printf 桩）。

做法（按优先级）：① 取本机 IPv4 下沉到芯片网络 HAL，用 `ioctl(SIOCGIFADDR)`；② 兜底用 compat `__wrap` 替换（见下节规范）。

### 行为注意事项（能编过，按约定写）

| 项 | 约定与做法 |
|------|-----------|
| `mmap` prot | 实现按**单个** prot 值匹配，`PROT_READ\|PROT_WRITE` 组合会落到只写：不用 mmap 做读写映射，显示/寄存器类访问走芯片 ioctl |
| `ftruncate` | 变长方向可用（`lseek`+写 0）；缩短返回 `EOPNOTSUPP`：调用方设计上接受失败，不依赖缩短 |
| `strncpy` 重映射 | compat `string.h` 将 `strncpy` 重映射到 `string_strncpy`（截断 + 保证 NUL）：新增依赖终止行为的代码直接用它；不要新增依赖 POSIX「不保证终止」语义的代码 |
| lwIP 错误码 | `e == 119` 类魔法数来自 lwIP：代码中宏化后再判断 |
| 线程栈 | 64KB 量级（内核 ustack patch）：`main.c` 等处的大缓冲挪静态区；新线程按需指定栈大小 |

---

## 四、新增 POSIX 兼容接口的规范

落点固定两处（源码一律**不进 SDK 树**）：

```text
project/<p>/system/<SoC>/<V>/platform/
├── include/alios/compat/   ← 垫片头（自足 + include_next 两种）
├── src/aos/                ← 实现（*.c，Makefile wildcard → libaos_posix.a）
└── lib/alios/              ← SDK 闭源库（只读，只 nm 不改）
```

- include 链：**compat → SDK `alios/external` → 工具链 libc**（垫片头 `#include_next`）。
- compat 必须是**第一个 `-I`**：产品 `config.mk`（`CORE_COMPAT_INC`）与工程 `rules.mk`（`AOS_COMPAT_INC`）双保险。
- 新增头/源**零构建改动**（compat 靠第一个 `-I`；src 靠 wildcard）——唯 `__wrap` 情形需在产品 Makefile 的 `USER_LINK_FLAGS` 补一行。
- SDK 原生已有的头（如 `sys/{prctl,ioctl,socket}.h`）**不加垫片**，直接用；垫片遮蔽 SDK 原生头是升级时的隐患。

### 三类情形（判定 → 动作）

| 情形 | 判定方法 | 头（compat/） | 实现（src/aos/） |
|------|----------|---------------|------------------|
| ① SDK 有实现、只缺声明/宏 | `nm` 能找到符号 | 同名垫片 `include_next` + 只补缺（如 `sys/reboot.h` 补声明、`sys/mount.h` 补 `MS_*` 宏） | **不加** |
| ② SDK 是 stub、需替换 | 反汇编确认是 printf 桩 | 垫片头补声明 | **`__wrap_<name>`** + 产品 Makefile `-Wl,--wrap=<name>`；**禁止同名强定义**（会与 `lwip_usyscall.o` 重复定义） |
| ③ SDK 完全没有 | `nm platform/lib/alios/*.a` 全查无 | 自足头（不 `include_next`） | 直接定义原名（`dlopen` 表、`realpath`、`ftruncate`） |

### 操作流程（新增一个接口）

1. `nm` 判定三类情形；
2. **写头**：情形①②与标准头同名（`include_next`），情形③自足；声明进头，不在 `.c` 里裸写；
3. **写实现**到 `platform/src/aos/`：情形②命名 `__wrap_<name>` 并补 `-Wl,--wrap=<name>`；情形①不加实现；
4. **更新账本**：`include/alios/compat/README.md` 与 `src/aos/README.md` 的规则 / 文件清单同步更新（两份 README 即接口账本）；
5. **验证**：Aos 产品全量编译 + `nm -u user.elf` 符号闭合 + Linux 产品全量回归（compat 头在非 `KERNEL_AOS` 时必须落到系统头，Linux 行为零变化）。

### 禁止事项

- 不改 `application` / `framework` 的 include，不在这两层加 OS 宏；
- 不做标准函数的行为改写（`strncpy` 重映射是唯一已记录的存量例外，不扩散）；
- 不为某个调用方写「专用头」——compat 只补标准接口的缺口，业务差异进 HAL 或产品层；
- 实现不做全局副作用（如改日志全局缓冲这类全产品行为变化）。

### SDK 升级迁移规则（如 V1.01 → V1.02）

把 `include/alios/compat/` 与 `src/aos/` 迁入新 overlay 前**逐文件重审**：

- 新 SDK 原生提供的头从 compat 删除（垫片会遮蔽原生头）；
- 情形②的 stub 替换重新反汇编确认新库仍是 stub；
- 以 `nm` 重测符号↔库映射（stub 归属可能随库版本变化）。

---

## 五、移植步骤

### 步骤 0　准备

| 项 | 操作 | 完成标准 |
|----|------|----------|
| 工具链 | SDK 自带（如 `tools/toolchain/arm-alios-eabi-*.tar.gz`）；产品 `setup_toolchain.sh` 优先 `/opt/...`，否则解压到 `out/<p>/toolchain/`；`build.sh` 仅当 `CONFIG_OS_NAME=aos` 且脚本存在时 source | `arm-none-eabi-gcc -dumpmachine` 能跑 |
| SDK 树 | `system/<SoC>/<V>/` 只读；确认 `platform/lib/*.a`、`platform/dep_lib/*.a`、`platform/lib/alios/uspace_posix.a`、`aos_user.ld`、`platform/sample/` 存在 | 路径齐全 |
| 原厂对照 | 用厂商 `auto_build.sh` 打一版对照镜像（人工，不代跑） | `image/` 有内核、rootfs、config 分区 |
| POSIX 子集实测 | 按第三节清单逐类 `nm` / 反汇编，产出本产品的「可用 / 不可用 / 无」三列表 | 列入工程文档 |

### 步骤 1　工程骨架

从最接近的现有 Aos 产品复制工程目录写法，逐项替换 SoC / 板型；**不改 `application/app_main/`、不改厂商树**。

| 落点 | 内容 |
|------|------|
| `project/<p>/config.mk` | `CONFIG_OS_NAME=aos`、`CONFIG_SOC_PLATFORM/SOC_SDK_VERSION/BOARD`、`CONFIG_APP_MAIN_TARGET=<x>_main`、`CONFIG_APP_MAIN_DYNAMIC_CORE_SO=n`、按产品开 `CONFIG_APP_PACKAGES_IPCAMERA=y` + `CONFIG_APP_RTSP=y`，GUI / 云 / 播放等关 `n`；`ifeq ($(CONFIG_OS_NAME),aos)` 段内加 `CFLAGS += -DKERNEL_AOS` 与 `CORE_COMPAT_INC`（第一个 `-I`） |
| `project/<p>/system/<SoC>/<V>/rules.mk` | `include` 厂商 `rules.mk` 后补：`CROSS_COMPILE`、`BUILD_SHARED=n`、`AOS_COMPAT_INC`（第一个 `-I`）、`-DKERNEL_AOS`、芯片宏（如 `-D__CHIP_AK3918AV130_SERIES`）。`build.sh` 存在工程 rules.mk 时优先选用 |
| `project/<p>/setup_toolchain.sh` | 仅本产品的工具链安装 |
| 板级 overlay `board/<Board>/` | DTB、defconfig、`resource/etc/config/`（settings.json / fw_video_config.json / ISP conf）、Flash 布局（NOR：`root.romfs` + `config.lfs`） |

> ⚠️ `CONFIG_OS_NAME` 全文统一写 `aos`（产品层与板级一致）；板级若需独立的 SDK 侧 OS 标识，用独立变量名，不与产品层 `CONFIG_OS_NAME` 撞名。

### 步骤 2　静态链接改造（.so → .a）

`CONFIG_OS_NAME` 是 Makefile 门闩。Linux 产品保持 `linux` 生成 `.so`；Aos 只 `$(AR) rcs` 产 `.a`，不传 `-shared/-fPIC/-rpath/-soname`，install 只拷 `.a` 和头。

- 各层 Makefile 已有 `CONFIG_OS_NAME` 分支的直接核对；遗漏处（如 `rtsp_server`）补 Aos 分支；
- 应用层唯一要改的 Makefile：`packages/ipcamera/ipc_cam/Makefile`（Aos 产 `libipc_cam_app.a`；`BUILD_SHARED=n` 时去 `-fPIC`，加 `-DIPC_CAM_NEBULA_STATIC=1`）;
- third_party 只开必需库（cjson / iniparser / mbedtls），`CONFIG_THIRD_PARTY_*_SHARED=n`；ffmpeg / curl / openssl / GUI 库关闭；
- C++ 库（zxing）Aos 分支保留异常支持，勿加 `-fno-exceptions`。

### 步骤 3　业务代码 POSIX 清理

对 `ipc_cam` 与本产品打开的 framework 模块（multimedia / network / rtsp / storage / device / notification / runtime），按第三节接口清单逐文件过：

| 清单类别 | 做法 |
|----------|------|
| ✅ 可直接使用 | 源码不动，标准头保持原样 |
| ❌ fork / exec / system / popen | 消除：改真 syscall 或 `thread_create` |
| ⚠️ stub | 取址下沉芯片 HAL（`SIOCGIFADDR`）；兜底走 `__wrap` |
| ⚠️ 链得上不可用 | 换实现：`syslog` → `log_*`、shm → 静态区 |
| 行为注意事项 | 大栈审计、mmap 组合 prot 消除 |

最终口径：`ipc_cam` 与打开的 framework 编译单元不直接包含 `pthread.h / unistd.h / dlfcn.h / sys/socket.h / sys/ioctl.h / sys/epoll.h / sys/mman.h` 之外的未审计路径，不调用禁用清单符号；头只来自 `system/core` 公开接口或 compat 垫片。验证用 `nm -u` + 源码 grep。

### 步骤 4　动态加载机制的替换（app_main 类入口零改动）

Linux 产品普遍依赖 `.so` 插件机制（`dlopen`/`dlsym` 按配置加载业务模块）；Aos 无动态链接器，移植时必须为**每个 `dlopen` 调用点**选定一种静态替代：

| 方案 | 适用 | 做法 |
|------|------|------|
| 入口直连 | 单应用、加载关系固定 | 链接期直接调用模块导出函数，删掉 dlopen 路径 |
| **静态注册表**（推荐，改动最小） | 要保持「按配置加载」的代码结构 | 提供同名 `dlopen/dlsym/dlclose/dlerror` 实现：配置里的 `so_path` 只作查找键，命中返回预登记的函数指针表 |
| 链接桩 | 可选/已关闭的模块 | 特性关 `n` + stub 提供符号，调用点不感知 |

采用静态注册表时（Celestia 实例：`platform/src/aos/aos_dl.c` 登记 `module_init` / `module_start` → 单应用 `ipc_cam`），有三点在任何静态链接场景通用：

1. **防 `--gc-sections` 丢符号**：被 `dlopen` 间接引用的库，链接器看不到引用关系——该库用 `--whole-archive` 链入，导出符号所在对象标 `used` 属性（如 `s_keep_ptrs`）。丢失只在上板运行时暴露（`dlsym` 返回空），链接期无告警；
2. **配置字段降级不改结构**：`settings.json` 的 `type=so`、`so_path` 字段保留原样，路径字符串降级为查找键，上层解析代码零改动；
3. **错误路径保持 Linux 排障习惯**：查不到注册项时 `dlerror` 返回说明字符串，不打断既有错误处理。

主程序（`application/app_main/`）保持**零改动**——`dlopen` 符号由兼容层提供，启动顺序与 Linux 产品一致：`main` → 打开 framework → `dlopen` + `module_init` → `module_start` 阻塞保活。

### 步骤 5　HAL（对照 platform/sample）

- 落点 `hardware/**/hal/<SoC>/`（无 `hal/<SoC>/` 的扁平驱动写源文件内 Aos 分支）；公开头 `hardware/include/*.h` 不改；
- 控制流从对应 `platform/sample/*` 抄（open → set attr → get frame/stream → release/close）；sample 的 `main` / 参数解析不链入；
- 成熟度分级：**L0** 仅链接符号 → **L1** 部分 MPI → **L2** sample 顺序对齐 → **L3** 上板实测；仅「能链过」不算 HAL 完成；
- 禁止空桩充数：framework 关特性的链接桩（`*_stub.c`）不算 HAL 能力；
- 与已有 Linux SoC 同名的 MPI（`ak_vi/ak_venc/ak_svp2/ak_ai/ak_aenc/ak_ao`）一致时直接复用同一套实现，芯片宏（`__CHIP_*_SERIES`）控制结构体布局差异；无对应头的分支只编同族 API 分支。

### 步骤 6　BSP 与镜像

厂商 tar 不改；差异只打在 `out/` 解压后的副本（`build_soc_system.sh` staging 补丁）。四类内核补丁（新 SoC 照抄思路）：

| 补丁 | 作用 |
|------|------|
| `*_patch_kernel_out_bin_check` | 内核 `kernel.bin` 输出在 `$BUILD_SRC_DIR/alios_kernel@.../binary/` 而 SDK 只查 `kernel/out/`：python3 行块替换 + 必要时拷入；**且须 `env BUILD_DIR="$SDK_BUILD"` 重定向**，否则产物落 `build/` 根 |
| `*_patch_kernel_uspace_trees` | RHINO `APP_ENTRY_TASK_NAME` 宏 + 64KB ustack：扫描 out 下全部 uspace 副本并挂 staging `build_kernel.sh` |
| `*_patch_kernel_host_python` | 宿主 python2 脚本改 python3 |
| `*_patch_kernel_build_scripts` | 去掉会冲掉配置的 `distclean` |

镜像规则：用户 ELF 装入 RomFS `usr/user.strip.elf`；JSON / ISP conf 进 `config.lfs`（LittleFS）；RomFS 打包前去掉 `etc/config`；驱动编进内核镜像，rootfs 不做 `insmod`；`out/<p>/` 根只留 `build/ install/ burntool/`，可烧录文件在 `out/<p>/build/system/<SoC>/<V>/image/`。

### 步骤 7　启动链路与上板验收

Aos 启动链（与 Linux 产品的 init 脚本链不同，上板定位按此走）：

```text
内核（RHINO）→ 装载 RomFS usr/user.strip.elf → app_entry
  → app_main main()（framework 句柄就绪）
  → dlopen(so_path)（静态表命中）→ module_init → module_start（阻塞保活，拉起采集/录像/RTSP）
```

| 层级 | 标准 |
|------|------|
| 整编 | `./build.sh --product-name <p> --jobs N`（**勿用 `-j`**；U-Boot `-j1`）零错误 |
| 用户态 ELF | `readelf -h` ELF32 / 对应架构；`readelf -d` 无 `NEEDED`；`nm -u` 无 `dlopen/getifaddrs/fork/shmget` 未定义；`module_start` 为 `T` |
| Linux 回归 | **每个共用仓改动后跑一次 Linux 产品全量编译**（luna 或 nova）——所有宏守卫、compat 头改动必须以此验证 |
| 上板 | 7.1 启动出 `module_start` → 7.2/7.3 出帧 / 音频 → 7.4 RTSP → 7.5 配置分区持久化 → 7.7 TF / 7.8 WiFi |

---

## 六、合入纪律

1. **application 与 framework 必须成对合并**：`ipc_cam_rtsp.c` 调 `rtsp_protocol_log_suggested_urls()`（跨仓依赖），单仓合入会断所有产品的链接。
2. framework 的回退必须 amend / squash 进原 commit 再合并（工作区回退不算）；跨层依赖改动各留独立提交并注明影响 Linux。
3. 合并前各跑一次 Linux 产品全量编译 + Aos 产品全量编译。
4. `system/core` 保持零 diff；被否决的改动不进 commit。

---

## 七、排障：AOS 常见问题

| 现象 | 处理 |
|------|------|
| 内核编译报 `fixdep: error opening depfile` | 构建循环被中断留下的内核脏增量（编译本身成功）：`build.sh -c -k` 清 soc_platform 层重建即愈，不是代码问题 |
| 链接报 `getifaddrs` 重复定义 | SDK stub 在 `lwip_usyscall.o`，同名强符号必撞：删强定义，改 `__wrap` + `-Wl,--wrap=` |
| third_party 模块行为异常（如 mbedtls 随机数） | 各层 OS 标识变量不一致所致：`CONFIG_OS_NAME` 全文统一为 `aos`，板级不另设撞名变量；统一后重编 |
| 内核报 `kernel build failed` 但 bin 已生成 | SDK 只查 `kernel/out/`：确认 out bin check 补丁 + `BUILD_DIR` 重定向生效（步骤 6） |
| `ftruncate` 缩短文件失败 | VFS 无 truncate，返回 `EOPNOTSUPP` 是约定行为：调用方按失败路径处理 |
| Aos 链接丢 `module_init` / `module_start` | `--whole-archive` 覆盖 `libipc_cam_app.a` + `s_keep_ptrs` 标 `used` |
| 最终链接 relocation / ABI 错 | 编译标志不一致（enum ABI）：所有 `.o` 与 libgcc 须同 `-mcpu/-mfloat-abi`；查各层 Aos 分支 CFLAGS 是否被 `:=` 冲掉 |
| C++ 库链不上异常符号 | Aos 分支误加 `-fno-exceptions`：CLIB libstdc++ 支持异常，去掉即可 |
| 整编并行失败 | 用 `--jobs N`，勿用裸 `-j`（U-Boot 固定 `-j1`） |

---

## 八、快速检查清单（新移植产品）

- [ ] 步骤 0：工具链 + SDK 库/sample 齐全 + 原厂对照镜像 + POSIX 三列表实测
- [ ] 步骤 1：config.mk / rules.mk / setup_toolchain.sh / 板级 overlay；`CONFIG_OS_NAME` 全文一致
- [ ] 步骤 2：Aos 全链只产 `.a`；`install/` 无 Celestia `.so`；C++ 库未去异常
- [ ] 步骤 3：禁用符号（fork / exec / system / popen / 不可用清单）清零；stub 有替代路径
- [ ] 步骤 4：`nm user.elf` `module_start` 为 `T`；app_main 无功能改动
- [ ] 步骤 5：HAL ≥ L2（sample 顺序），上板 L3 绑定 7.x
- [ ] 步骤 6：内核四补丁；`root.romfs` + `config.lfs`；可烧录文件在 `build/system/<SoC>/<V>/image/`
- [ ] 步骤 7：Linux 产品全量回归 + Aos 全量 + `nm -u` 口径
- [ ] compat/README 与 src/aos/README 已随新增接口更新
