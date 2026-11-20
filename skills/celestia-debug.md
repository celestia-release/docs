---
name: celestia-debug
description: >-
  Celestia 上板调试、闭环验证、内存/DMA 排障与 Bug 分类 skill：串口 log 判 PASS/FAIL；
  按崩溃/预览编码/内存DMA/启动驱动/App·SO/构建stale/出流网络 七类归类，给出典型 log、
  根因与解法，并路由到 celestia-application / framework / hardware / project / scripts 等层 skill。
  含 Nova 分区溢出（Page more than end page / unknown-block）与通话 stub 解码归类。
  当需要跑板验证、贴 log 排障、定位崩溃/超时/OOM、或不知问题落在哪一层时使用。
---

<p align="right"><a href="celestia-debug.en.md">English</a> | <b>中文</b></p>

# Celestia 上板调试、闭环验证与 Bug 分类 skill

> 定位层级见 [celestia](celestia.md)；编译见 [celestia-scripts](celestia-scripts.md)。
> 闭环：build → 烧录 → 串口抓 log → 按下表**归类** → 套用解法 / 下钻层 skill → 复测 PASS。
> AI 工作表：[roles.md §5](../quick-start/roles.md#5-调试与维护)。

## 核心原则

1. 编译通过 ≠ 功能生效；有运行期表现必须上板看 log。
2. **先归类再改代码**：用下方「log 关键词」命中 Bug 类，再进对应解法表。
3. 以本层为主；证据（返回码/log/tests）明确才下钻下层 skill。
4. 改完重跑闭环，直到 PASS/FAIL 判据满足。

串口：Luna 默认 `115200` 8N1（`picocom -b 115200 /dev/ttyUSB0`）。

## PASS/FAIL 判据

| 场景 | PASS | FAIL 关键词 |
|---|---|---|
| 预览 | 连续出帧 ≥20s | `invalid inbuf` / `QBUF fail` / select timeout / sensor 反复上下电 |
| 录像 | 文件增大、正常收尾 | live 期 config reload、写文件失败 |
| 崩溃 | 无 signal | `signal=11` / `fault_addr=` |
| 启动 | 主进程起来、外设节点在 | panic、insmod 失败、主进程名错 |
| 出流 | RTSP/nebula ≥20s 稳 | 断流、端口起不来、编码失败 |

---

## Bug 分类总表（贴 log 先查这里）

| 类 | 典型 log / 现象 | 主 skill | 辅 skill |
|---|---|---|---|
| **A 崩溃** | `signal=11`、`fault_addr=`、segfault、backtrace | 按栈归层 | [celestia](celestia.md) |
| **B 预览/采集/编码** | `invalid inbuf`、`QBUF fail`、`select timeout`、`num_slices`、不出帧 | [celestia-hardware](celestia-hardware.md) / [celestia-framework](celestia-framework.md) | — |
| **C 内存/DMA/OOM** | ion/dma alloc fail、`QBUF`+meminfo 正常、OOM、花屏 | 本节「内存/DMA」 | [celestia-project](celestia-project.md)、[celestia-third-party](celestia-third-party.md) |
| **D 启动/驱动/Board** | `/dev/*` 缺失、insmod 失败、拉错主进程、overlay 无效、`unknown-block`、烧录 `Page more than end page` | [celestia-project](celestia-project.md) | [celestia-hardware](celestia-hardware.md)、[celestia-third-party](celestia-third-party.md) |
| **E App/GUI/SO** | `dlopen` 失败、不上桌面、图标空白、IOTCamera 形态秒退 | [celestia-application](celestia-application.md) | — |
| **F 构建/stale/配置** | 改源不生效、`Failed to get … module: -1`、config.h 不对 | [celestia-scripts](celestia-scripts.md) | [celestia-hardware](celestia-hardware.md)、[celestia-project](celestia-project.md) |
| **G 出流/网络/云** | RTSP/nebula 无流、云连不上、`stream_reg_client` 失败 | [celestia-framework](celestia-framework.md) | [celestia-application](celestia-application.md) |

不确定类 → 先 [celestia](celestia.md) 五层路由，再回本表。

---

## A. 崩溃 / 死机

| 现象 / log | 根因 | 解法 | skill |
|---|---|---|---|
| 开 AI 后 `signal=11 fault_addr=0x90` | `camera_manager` close 与 `ai_enable` 竞态 | 生命周期 mutex；`ai_enable` 路径本地捕获 device 指针 | framework / hardware |
| 任意 `signal=11` + backtrace | 空指针 / UAF / 越界 | 抓栈 → 归层 → 核对本层句柄生命周期与锁 | [celestia](celestia.md) → 层 skill |
| `signal=11` 在 `free`/`malloc` 内，`fault_addr` 接近 `0xffffffff`（如 `free+0x50` / `0xfffffffd`）| uclibc `free()` 先读 `mem-8` 的 chunk 头：`fault_addr≈0xffffffff` ⇒ 传进去的是**小整数**（`free(5)`→`0xfffffffd`），**不是**真 double-free。两种来源：① 指针字段被当 int 字段读（结构体布局不一致，走 F 类 stale）；② 堆被更早的越界写坏，free-list 的 `fd` 被覆盖成小整数 | **先看 `r0`**（崩溃处理器已打印）：`r0` 是小整数→①/上游传错指针；`r0` 是合法堆指针→②，故障在更早的越界写，与崩溃点无关。再看 `tid` + `stack[]` 定栈 | 按 `stack[]` 归层 |
| 拍照/录像后进程挂死 | teardown 关电或 live reload 清零 priv | 对称 teardown；live 期禁止 `load_config` 清 live | framework / hardware |
| watchdog / 整机复位 | 驱动死锁或内核 oops | `dmesg`；先 D 类查 ko，再 HAL ioctl | project / hardware |

**抓证**：串口完整 backtrace + 崩溃前 50 行 + 产品名 + 复现步骤。

---

## B. 预览 / 采集 / 编码

| 现象 / log | 根因 | 解法 | skill |
|---|---|---|---|
| `invalid inbuf cap=524288` 丢帧 | `max_frame_payload` 在 `codec_mode` 前算，NV12 退回 512KB | **先解析 `codec_mode` 再算 payload** | framework / hardware |
| `QBUF fail` / 重入失败 | capture/record teardown 破坏 live 通路 | 对称 teardown；勿在 live 期 reload | framework / hardware |
| CHN0 `select timeout` | `num_slices≥2` 但 VENC 未 bind 到 VI（slice 通道不向用户态出 YUV）| AV100_V2 走 bind + `venc_get_stream`；未实现的 SoC 保持 `num_slices=1` | hardware |
| sensor 不出帧 / `/dev/video0` 无 | ko 未载或 HAL 未 open | 先 D 类 insmod，再 `hw_get_module` | project → hardware |
| `vi 3dnr memsize … 5040 KB` 异常 | 3DNR 常量未 clamp | clamp + `HAL_CAMERA_VI_3DNR_KB` | hardware |
| 花屏 / 花块但进程在 | DMA 帧不足或格式错 | 走 **C 类** 决策树；核对宽高/stride | debug 内存节 |
| 码流改了不生效 | 改了 `channels/streams` 能力集 | 只动 `app_stream_usage`（framework §2.5.1）| application + framework |

层内补充：[hardware 常见问题](celestia-hardware.md#常见问题)、[application 常见问题](celestia-application.md#常见问题)。

---

## C. 内存 / DMA / OOM（LUNA）

> 数值细节见下方「内存模型」；完整分析可参考 `doc/WEARABLE_MEMORY_FOOTPRINT.md`。

| 现象 / log | 分流 | 解法 |
|---|---|---|
| dmesg `ion`/`dma`/`QBUF`/`VENC` alloc fail，meminfo 尚可 | **DMA 池不足** | A：DTS `RESERVED_MEMORY_SIZE` 升档（board+patch 同步）；B：降 `max_buffers`/分辨率 |
| OOM / malloc 失败 / MemFree 极低，无 ion 报错 | **MemTotal 不足** | C：`CONFIG_FW_MULTIMEDIA_MEDIA_PLAYER=n`、curl mbedTLS；D：调 `cirbuf_size`（勿拿它省 DMA）|
| 预览 OOM 同时编码挂 | 两池此消彼长 | 先分清哪池；加大 DMA 会压缩 MemTotal |

主 skill：本节；改 DTS → [celestia-project](celestia-project.md)；裁库 → [celestia-third-party](celestia-third-party.md)。

### 内存模型（64MB 切两块）

| 预算 | 大小 | 用途 | 调整 | meminfo |
|------|------|------|------|:---:|
| DMA/ION 预留 | **20MB**（`0x1400000`）| VI/VENC/JPEG/显存 | DTS `RESERVED_MEMORY_SIZE` | ❌ |
| Linux MemTotal | **~25MB** | 进程 + cirbuf | config.mk / `fw_video_config.json` | ✅ |

形态：`0x1980000`=25.5M · `0x1A00000`=26M · `0x2200000`=34M · `0x2400000`=36M。
路径（须 **board + patch 双改**）：

- `project/anypi_luna/.../board/PDB_MBD_KM01W_ACTCAM/os/kernel/.../anycloud_km01a.dtsi`
- `project/anypi_luna/.../patch/os/kernel/.../anycloud_km01a.dtsi`

`fw_video_config.json` 在 **board rootfs overlay**（非 `project/*/config/`）。

---

## D. 启动 / 驱动 / Board

| 现象 / log | 根因 | 解法 | skill |
|---|---|---|---|
| `/dev/video0` 等缺失 | insmod 顺序/路径错 | 核对 `app_01_start.sh` / `app.sh` + `lsmod`/`dmesg` | project |
| 开机拉错主进程 | 脚本二进制 ≠ `CONFIG_APP_MAIN_TARGET` | 改启动脚本对齐 `<suffix>_main` | project |
| overlay 配置不生效 | 改了 vendor 树或被 patch 覆盖 | 只改 `board/<BOARD>/`；核对接叠顺序 | project |
| 无串口 / 不启动 | bootloader/kernel/DTB | 查烧录分区、DTB 与 BOARD 是否匹配 | project |
| `unknown-block(31,6)` / 挂不上 root | burn / env / DTS `mtdparts` 三处不一致 | 对齐现表；重编 DTB + 重烧 ENV/DTB | project §分区 |
| 烧录 `Page more than end page`（APP/ROOTFS） | `usr.sqsh4`/`root.sqsh4` > 分区；APP 常见 FFmpeg 膨胀 | `DECODE_ONLY`+`LINK_MODE=decode` 或挪 SD `install/`；见 project | project / third-party |
| SDK 版本与 overlay 目录不一致 | `CONFIG_SOC_SDK_VERSION` 错 | 对齐版本目录 | project |

详见 [project 常见问题](celestia-project.md#常见问题) / [分区调优](celestia-project.md#anyka-burntool-分区调优nova-16mb-spl-fastsys)。

---

## E. App / GUI / SO

| 现象 / log | 根因 | 解法 | skill |
|---|---|---|---|
| `dlopen` 失败 / 入口找不到 | `--gc-sections` 剪 entry | `visibility("default")` + keep-alive | application |
| App 不上桌面 | `installed_apps` 未注册 / path 错 | 注册并对齐运行期 path | application |
| `so_path` 加载失败 | 写了源码路径 | AV100→`/usr/lib`；AV200→`/mnt/afht` | application |
| Luna 图标空白 | GPU draw 失败丢弃、无 SW 回退 | `ret<0` 重新入队回退 SW（id=1）| application / hardware |
| IOTCamera 形态启动即退出 | 无 GUI 时 `main` return | SO `module_start` 阻塞保活 | application |
| 通话无画面 / `VIDEO_DECODER_SW_STUB` | `FFMPEG_LINK_MODE=none`（stub） | Nova：`decode` + `DECODE_ONLY`；`-t`/`-f`/`-p` | framework / third-party |

详见 [application 常见问题](celestia-application.md#常见问题)。

---

## F. 构建 / stale / 配置

| 现象 / log | 根因 | 解法 | skill |
|---|---|---|---|
| 改源板上行为不变 | stale `out/` / 旧 install 头 | `./build.sh -c` 再编对应层；确认 `--product-name` | scripts |
| 改私有头的结构体布局后随机崩溃 / `free` 内 SIGSEGV | **Makefile 无头依赖**：`%.o: %.c` 只依赖 .c，未改的 .c 保留旧 offset，指针字段落到 int 字段上 | Makefile 加 `-MMD -MP` + `-include $(OBJS:.o=.d)`（`hardware/camera` 已加）；删该模块 `out/**/build` 下旧 `.o` 再全编 | scripts + 对应层 |
| 想确认是不是 stale | — | **比时间戳**：`ls -l out/<product>/build/<模块>/**/*.o`，只要有一批明显早于本次改动就是 stale；全批同一时间戳 + 有 `.d` ⇒ **排除** stale，别再往这条路上查 | scripts |
| `Failed to get video hardware module: -1` | stale `hardware.o`（config.h 非依赖）| Makefile 加 config.h 依赖；`-c -h` | hardware + scripts |
| `config.h` 与预期不符 | config.mk 语法 / 产品名错 | 核对 `project/<p>/config.mk` | project |
| 链接缺符号 | third_party 未先编 | `-t` 再编上层；查 CONFIG 裁剪 | scripts / third-party |
| 工具链错误 | 按产品名选链 | 按平台：AV200 riscv64 / AV100_V2 arm | project / scripts |

---

## G. 出流 / 网络 / 云

| 现象 / log | 根因 | 解法 | skill |
|---|---|---|---|
| RTSP 无流 | 未 `stream_reg_client` / `CONFIG_APP_RTSP` 关 / 编码失败 | 开开关；App 注册 main；先 B 类保证编码 | framework + application |
| nebula 连不上 | 配置/开关/网络 | 核 `CONFIG_APP_NEBULA` 与 gateway open | framework |
| 云推流失败 | cloud SO 未加载或网络 | 查 `settings.json`/`load_so_module`；禁依赖 `module_exit` | application |
| WiFi 无 `wlan0`（Luna）| 接口名是 `vnet0` | 用 `vnet0`；查 `wifi_driver.sh` | project |

---

## 排障流程（给 AI）

```
1. 收集：产品名 + 复现步骤 + 关键 log（含前后文）
2. 用「Bug 分类总表」关键词命中 A–G
3. 打开对应解法表；能套已知模式则直接修
4. 否则进主 skill，以本层为主；有证据再下钻
5. 改完：对应 build.sh flag → 烧录 → 串口按 PASS/FAIL 复测
6. 回填 roles 排障表：现象 / 归类 / 证据 / 根因 / 修复 / 验证
```

## 定位到层后

按 [celestia](celestia.md)「本层 → 接口契约 →（有据）下层」收敛；改完回到本 skill 闭环直到 PASS。
