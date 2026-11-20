---
name: celestia-scripts
description: >-
  Celestia 构建与脚本层（scripts/、build.sh）：build.sh 分层编译入口（-s system-core / -t third-party /
  -h hardware / -f framework / -a application / -k kernel / -p package / -c clean，--product-name、
  --jobs）与构建依赖顺序。当需要编译、用命令或 burntool 下载到开发板、串口跑板看 log、或修改/排查构建脚本时使用。
---

<p align="right"><a href="celestia-scripts.en.md">English</a> | <b>中文</b></p>

# Scripts / 构建层 skill

> 文档：`docs/quick-start/bsp-and-products.md §4`。
> 上板环境搭建见 [celestia-project](celestia-project.md) 的「Luna 上板」。总览见 [celestia](celestia.md)。

## 运行位置（重要）

编译与打包在 **Linux 构建环境**（WSL 或原生 Linux）内执行，工程路径 `~/celestia`。
**勿在 Windows 路径下（如 PowerShell 内的 `Z:\...`）直接执行 build.sh**。
打包后的镜像也在 `out/<product>/burntool/`，可用 Burn Tool 整片烧录。Nova 日常改应用或用户区时，用设备上的命令写入，不必打开 Burn Tool。见下文「命令下载到开发板」与 [celestia-studio](celestia-studio.md)。

## build.sh —— 分层编译入口

`build.sh`（仓库根，与 `scripts/build.sh` 内容相同）。无层 flag = 全量构建 + package。

| flag | 长选项 | 作用 |
|------|--------|------|
| `-s` | `--system-core` | system/core |
| `-t` | `--third-party [module]` | third_party（可指定单模块，如 `-t lvgl`）|
| `-h` | `--hardware` | hardware HAL |
| `-f` | `--framework` | framework |
| `-a` | `--application` | application |
| `-k` | `--kernel` | SoC platform / kernel（system/<SoC>/<SDK>）|
| `-p` | `--package` | rootfs / 镜像打包 |
| `-c` | `--clean` | 清理选中层（无层 flag 则清全部）|

常用其他选项：`--product-name anypi_nova|anypi_nova_dev|anypi_luna|anypi_luna_dev|anypi_mira_dev`（未指定时交互选择并缓存到 `out/.build_product_name`，下次直接运行）、`--jobs N`、`--log-level DEBUG|INFO|ERROR`、`--build-level DEBUG|RELEASE`、`--release [path]`、`--copy-build-src`（只把最小源码集拷到 `out/<product>/release-src/`，**不编译**）。工具链按平台：nova（AV200）`riscv64-anyka-linux-musl-`，luna 系（AV100_V2）`arm-anycloud-linux-uclibcgnueabi-`，mira（Sky39AV130，aos）由 `project/anypi_mira_dev/setup_toolchain.sh` 提供 `arm-none-eabi-`（`CONFIG_OS_NAME=aos` 时 build.sh 自动 source）。

构建依赖顺序：`① third_party → ② system/core → ③ hardware → ④ framework → ⑤ application → ⑥ package`。`build_framework/build_system_core` 会自检 `liblvgl.a`/`cJSON.h`，缺失自动先建 third_party。可组合多层并按序构建：`./build.sh -h -f -a`。

```bash
cd ~/celestia
./build.sh                                         # 未指定产品则交互选择并缓存，下次直接运行
./build.sh --product-name anypi_luna               # 全量 luna
./build.sh -f -a --jobs 4 --product-name anypi_luna
./build.sh -c -f                                   # clean 后只编 framework
./build.sh --copy-build-src --product-name anypi_luna   # 只拷最小源码集，不编译
```

打包后整片烧录、串口跑板见 [celestia-project](celestia-project.md) 与 [celestia-debug](celestia-debug.md)。只更新应用或 APP 分区时用下面的命令，不经 burntool。

## 命令下载到开发板（不经 burntool）

Nova（`anypi_nova`）在设备已运行、`studio` 听 TCP 7777 时，用同步工具或串口/`EXEC` 下发。存储须是本地挂载：`/mnt/install` 存在。U 盘模式把 SD 导出给电脑后，设备上没有该目录，不能按下面替换应用。

**U 盘上的 tablet 应用**（`setting` 除外）替换目录，不写 flash：

```sh
# 包已在 /tmp/sync_push.tar.gz，内容是 app.json、res/、lib/*.so
DEST=/mnt/install/apps/<id>
rm -rf /tmp/sync_pkg "$DEST.bak"
mkdir -p /tmp/sync_pkg /mnt/install/.app_bak
tar -xzf /tmp/sync_push.tar.gz -C /tmp/sync_pkg
BAK=/mnt/install/.app_bak/<id>
rm -rf "$BAK"
[ -d "$DEST" ] && mv "$DEST" "$BAK"
mv /tmp/sync_pkg "$DEST"
chmod -R a+rX "$DEST"
```

备份放在 `/mnt/install/.app_bak/`，不要放在 `apps/<id>.bak`，否则桌面会多登记一个应用。换完后重启界面：`killall nova_main ipc_main ipc_home`，再 `/usr/sbin/start_ipc_main.sh --bg`。

**用户区**（`/usr`，含 `apps/setting` 与 `/usr/lib/libstudio.so`）是 squashfs。重打 `usr.sqsh4` 后按 `/proc/mtd` 里名为 `"APP"` 的分区写入：

```sh
grep '"APP"' /proc/mtd
# fastsys 实机示例：mtd8  size 0x008ac000
mtd_debug erase /dev/mtd8 0 <分区字节数>
mtd_debug write /dev/mtd8 0 <镜像字节数> /tmp/usr.sqsh4
sync
reboot
```

镜像大于该分区则不要写。上限以 `/proc/mtd` 为准（上述实机为 `0x8AC000`），不要用 burntool 配置里更大的 `0x8AE000`。不要对正在映射的 `/usr/lib/*.so` 做截断覆盖。完整的新建/修改到推送步骤见 [celestia-studio](celestia-studio.md)。

## 最小源码集（对外释放）

`scripts/release_src.sh` 按 `project/<product>/config.mk` 的 `CONFIG_*` 开关，只把启用模块的源码拷到
`out/<product>/release-src/`，剔除未启用模块、其他 SoC 的 `hal/`、其他产品目录、`.git`/构建产物。
等价于 `build.sh --copy-build-src`（后者调它且跳过编译）。

```bash
bash scripts/release_src.sh --product-name anypi_luna                 # 默认不含厂商 BSP
bash scripts/release_src.sh --product-name anypi_luna --with-vendor-bsp  # 连 system/<SoC>/<SDK>/ 一起拷
bash scripts/release_src.sh --product-name anypi_luna,anypi_nova,anypi_mira_dev --dest out/release-open  # 多产品 + 自定义输出目录
bash scripts/release_src.sh --product-name anypi_nova --with-cloud    # 含 application/cloud 厂商 SDK
```

> 厂商 BSP（`system/<SoC>/<SDK>/`）默认不释放；构建前需自行放置或加 `--with-vendor-bsp`。
> rootfs 路径推导已内联进 `application/` 与 `hardware/input/evdev/` 的 Makefile（无独立 `.mk`）；`scripts/` 现仅含 `build.sh` + `release_src.sh` + 许可/说明。

## 排障 / 改脚本时

1. 编译失败先读 build.sh 输出定位是哪一层、哪条命令；区分**源码错误**（回对应层 skill）与**脚本/路径问题**（本层）。
2. 改 `build.sh` 属构建逻辑变更：保持分层 flag 语义与依赖顺序不变；改完用 `--help` 自检并跑一次受影响层编译。
3. 上板 PASS/FAIL 判据见 [celestia-debug](celestia-debug.md) skill（预览 ≥20s 的关键 log）。

## 常见问题（构建）

| 现象 / log | 根因 | 修复 |
|---|---|---|
| 改源板上不变 | stale `out/` / 未指定产品 | `-c` 再编；确认 `--product-name` |
| 在 Windows 路径跑 build.sh | 非 Linux 构建环境 | 到 WSL/`~/celestia` 再编 |
| 链接缺 lvgl/cjson | third_party 未建 | 先 `-t` 或让依赖自检拉起 |
| 只编一层仍链到旧 .o | 未 clean 依赖层 | `-c -h -f -a` 按依赖清编 |
| package 缺文件 | overlay/开关未装进 rootfs | 核 config.mk 与 board rootfs |
| 烧录 APP `Page more than end page` | `usr.sqsh4` > burn APP（Nova=`0x82E000`） | `-p` 后 `stat out/<p>/burntool/usr.sqsh4`；裁 FFmpeg/`install/`；见 [celestia-project](celestia-project.md) §分区 |

运行期归类见 [celestia-debug](celestia-debug.md) **D/F 类**。

## 编码完成 → 编译（必做）

改构建脚本或新写脚本后，**实际跑一次**对应命令验证：

```bash
cd ~/celestia
./build.sh --help                                  # 选项语义自检
./build.sh -a --product-name anypi_luna            # 触发受影响层，确认 0 error
```

确保脚本在构建环境中真实执行通过、产物/打包符合预期后再交付。
