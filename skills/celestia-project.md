---
name: celestia-project
description: >-
  Celestia Project / BSP 层（project/、system/<SoC>/）开发与排障：project/<proj>/config.mk 是唯一
  配置源（CONFIG_SOC_PLATFORM/CONFIG_BOARD/CONFIG_APP_*/CONFIG_FW_*/CONFIG_HW_* → 生成 config.h），
  组装 anypi_nova（AnyCloud39AV200/ipc_main）、anypi_nova_dev（AnyCloud39AV200/nova_main）、
  anypi_luna（AnyCloud39AV100_V2/240x240/luna_main）、anypi_luna_dev（AV100_V2 V2.04/luna_dev_main）
  与 anypi_mira_dev（Sky39AV130 V1.01/aos/mira_main）；
  产品选择：--product-name → 缓存 → 交互选 project/*。
  BSP 含 kernel/DTS/rootfs overlay/insmod 启动脚本/burntool。分区调优重点：两份 env_*.cfg
  （mtdparts/kernel_offset/kernel_size/boot_normal）→ 重生 env_*.img → burn download_to_nand
  与 DTS bootargs 三处对齐；APP burn=`0x82E000`；hybrid /usr+SD install。
  vendor 树默认只读。改 config.mk 后用 build.sh -k 或全量编译。排查编译选项、SoC/板型、DTS、
  驱动加载、镜像打包、烧录分区溢出、上板调试时使用。
---

<p align="right"><a href="celestia-project.en.md">English</a> | <b>中文</b></p>

# Project / BSP 层 skill

> 文档：`docs/quick-start/bsp-and-products.md`。总览见 [celestia](celestia.md)。
> 上板烧录与串口观察见 [celestia-scripts](celestia-scripts.md) 与 [celestia-debug](celestia-debug.md)（本文 §「wearable 上板」为板级要点）。

## 组装模型

同一份 Celestia 源码通过不同 `project/<proj>/` 组装出不同 SoC/板型/形态的镜像。Project 三大职责：**选平台**（`CONFIG_SOC_PLATFORM`）、**选板型**（`CONFIG_BOARD`）、**装板级资源**（DTS/rootfs/启动脚本）。`Application/Framework/System Core/hardware/include` 跨 SoC 共用，换 SoC 成本只落在 HAL 实现 + BSP。

```
project/
├── anypi_nova/      config.mk + include/config.h(自动生成)
│                    + system/AnyCloud39AV200/V1.04/board/PDB_MBD_AK3918AV200_ACTCAM/...
├── anypi_nova_dev/  config.mk + ... + system/AnyCloud39AV200/V1.04/board/EVB_MBD_AK3918AV200/...
├── anypi_luna/      config.mk + ... + system/AnyCloud39AV100_V2/V2.05/board/PDB_MBD_KM01W_ACTCAM/...（另保留 V2.04 旧 overlay）
├── anypi_luna_dev/  config.mk + ... + system/AnyCloud39AV100_V2/V2.04/board/EVB_MBD_KM01A/...
└── anypi_mira_dev/      config.mk + setup_toolchain.sh + system/Sky39AV130/V1.01/board/EVB_CBD_AK3918AV130/...
system/
├── AnyCloud39AV200/      # vendor BSP（可含 V1.03/V1.04 PDK）—默认只读，无 board/ overlay
├── AnyCloud39AV100_V2/   # 同上（V2.04 / V2.05）
├── Sky39AV130/           # 同上（V1.01，Mira）
└── core/                 # 见 celestia-system-core（不属于 BSP）
```

⚠️ **板级 overlay 只存在于 `project/<proj>/system/<SoC>/<SDK>/board/<Board>/`**；顶层 `system/<SoC>/<SDK>/` 下**没有** `board/`（那是 vendor PDK 原厂 payload）。上面的装配图即按此结构。

## config.mk —— 唯一配置源（SSOT）

`build.sh` 解析 `project/<proj>/config.mk` 的所有 `CONFIG_*` 并生成 `project/<proj>/include/config.h`。五个产品的核心变量：

| 变量 | anypi_nova | anypi_nova_dev | anypi_luna | anypi_luna_dev | anypi_mira_dev |
|------|------------|----------------|------------|----------------|------------|
| `CONFIG_SOC_PLATFORM` | AnyCloud39AV200 | AnyCloud39AV200 | AnyCloud39AV100_V2 | AnyCloud39AV100_V2 | Sky39AV130 |
| `CONFIG_SOC_SDK_VERSION` | V1.04 | V1.04 | V2.05 | V2.04 | V1.01 |
| `CONFIG_OS_NAME` | linux | linux | linux | linux | aos |
| `CONFIG_BOARD` | PDB_MBD_AK3918AV200_ACTCAM | EVB_MBD_AK3918AV200 | PDB_MBD_KM01W_ACTCAM | EVB_MBD_KM01A | EVB_CBD_AK3918AV130 |
| `CONFIG_APP_MAIN_TARGET` | ipc_main | nova_main | luna_main | luna_dev_main | mira_main |
| `CONFIG_APP_PACKAGES_*` | TABLET=y | TABLET=y | WEARABLE=y | IPCAMERA=y | IPCAMERA=y |
| `CONFIG_WIFI_TYPE` | atbm6162 | —（板级） | aic8800 | atbm6032 | — |

> anypi_mira_dev 为 `CONFIG_OS_NAME=aos`：`build.sh` 自动 source `project/anypi_mira_dev/setup_toolchain.sh`（alios `arm-none-eabi-` 工具链）。Linux 产品移植 AOS 的完整流程（POSIX compat、静态链接、内核补丁）见 [celestia-aos-port](celestia-aos-port.md)。

还含 `CONFIG_APP_*`（App 开关）、`CONFIG_FW_*`（服务开关）、`CONFIG_HW_*`（HAL 开关）、`CONFIG_THIRD_PARTY_*`、`CONFIG_WIFI_TYPE`/`CONFIG_SENSOR_NAME`（板级）。改开关务必**同时检查其他产品的 config.mk**避免互相影响。

> 注：luna 板级 overlay 的 `board/<Board>/config.mk` 定义 `CONFIG_DTB_NAME`（如 `EVB_CBDM_KM01A_V1.0.0`）；产品顶层 `config.mk` 不再定义它。DTB 选择见下节。

## BSP 要点

- 工具链按平台（≠按产品名）：nova（AV200）用 `riscv64-anyka-linux-musl-`（RISC-V/musl）；luna 系（AV100_V2）用 `arm-anycloud-linux-uclibcgnueabi-`；mira（Sky39AV130，aos）由 `project/anypi_mira_dev/setup_toolchain.sh` 提供 `arm-none-eabi-`。
- DTB 选择：nova 系由 `package.sh` 按 FASTSYS/LCD/DMIC 自动选；luna 由板级 overlay/打包脚本确定（不再靠 `CONFIG_DTB_NAME`）。见 `docs/quick-start/bsp-and-products.md §6.5`。
- rootfs 叠加顺序：SOC 基础 → `board/<Board>/rootfs/` → `patch/rootfs/` →（FASTSYS）`patch_fastsys/rootfs/`，后者覆盖前者。
- 启动脚本因产品而异：**tablet 系用 `app_01_start.sh`**（`insmod` 驱动清单 + 拉主进程）；**wearable 用 `app.sh` + `app_02_load.sh`**（不是 `app_01_start.sh`），见「wearable 上板」。

## 禁止 / 注意

- ⚠️ `system/<SoC>/<SDK>/`（kernel、rootfs、platform SDK）中的 **vendor BSP 默认只读**；调试不动芯片原厂树，问题优先在 HAL 或上层 workaround。确需改驱动加载/DTS 时**只改 `project/<proj>/system/.../board/` overlay 与启动脚本的 insmod 列表**。
- 改 `config.mk` 属于配置变更，不应替代代码层修复。

## 排障：Project 层

1. 现象为“编译选项/平台/板型/驱动加载/打包”才在此层：核对 `config.mk` 变量、`config.h` 是否正确生成、DTB 是否匹配板型、启动脚本（nova `app_01_start.sh` / luna `app.sh`）的 `insmod` 列表与 `.ko` 是否齐全。
2. 驱动设备缺失（`/dev/video0` 等）→ 确认 insmod 顺序与 ko 路径，再回 [celestia-hardware](celestia-hardware.md) 看 HAL 是否正确打开设备。
3. 新增 SoC/板型走 `docs/quick-start/bsp-and-products.md §9` 八步流程 + [celestia-hardware](celestia-hardware.md) 的「新增 SoC」。

## wearable 上板（板级要点）

面向 **PDB_MBD_KM01W_ACTCAM**（AnyCloud39AV100_V2）。编译打包后镜像与烧录工具位于 `out/anypi_luna/burntool/`，烧录上板后用串口观察。此处只列 luna 板级必知项：

- **串口控制台**：USB 转串口，`115200` 8N1；登录提示符 `[root@anyka ~]#`，可看 boot log 与 `app.sh` 执行情况。
- **网络**：WiFi 接口为 `vnet0`（非 `wlan0`）。KM01W 的 WiFi 与 TF 共用 SDIO，`ak_mci.ko` 由 `wifi_driver.sh` 加载。
- **启动脚本**：`/etc/config/app.sh` 开机跑驱动/WiFi（**不自动启应用**）；`app_02_load.sh` 加载 ST7789 + 背光，由 `luna_main` 开 UI 前调用。调试时手动 `killall luna_main; /usr/sbin/luna_main`，无需重跑 `app.sh`。
- **显示**：KM01W 无 GPU，走软件刷 fb（PARTIAL 模式）；`/dev/fb0` 期望 240×240，`lv_conf.h` 用 `LV_COLOR_DEPTH 16`、`LV_MEM_SIZE` 512KB 量级。相关改动集中在 `project/anypi_luna/{include/lv_conf.h,config.mk,.../board/etc/config}`。

## Board overlay 内容

路径：`project/<proj>/system/<SoC>/<SDK>/board/<Board>/`（`CONFIG_BOARD` = 目录名）。

| 子目录 / 文件 | 作用 |
|---|---|
| `os/kernel/.../dts/*.dtsi` | 板级 DTS（DMA 预留、pinmux、IO、LCD/sensor）|
| `rootfs/` | 板级 overlay：`utils/etc/config/` 下 `fw_*/hal_*/app_*` + `app.sh` / `app_01_start.sh` |
| `defconfig` | kernel defconfig |
| `burntool/` | 烧录工具与分区配置 |
| `config.mk`（板级）| DTB 名、分区、flash 布局；被顶层 `config.mk` include |

⚠️ board 与 patch 两份 DTS 必须同步改（patch 覆盖 board）；顶层 vendor 树只读。

## 新增产品（指定 BOARD + UI）

开新产品时**必须显式给出**：

1. `CONFIG_BOARD=<BoardName>`（= overlay 目录名）
2. **UI 形态**：ipcamera / wearable / tablet（决定 `CONFIG_APP_MAIN_TARGET` 与 `CONFIG_APP_PACKAGES_*`）
3. 单应用落点（`IOTCamera` 形态 → `packages/ipcamera/`；带显示屏→`packages/tablet` / `packages/wearable`）

命名建议：`<形态>_<载板>_<芯片或模组>`，如 `PDB_MBD_KM01W_ACTCAM`。

| UI | MAIN_TARGET | GUI | 单 App 目录 | 挂载 |
|---|---|---|---|---|
| IOTCamera | `luna_dev_main`（或 `CONFIG_APP_MAIN_TARGET`）| `IPCAMERA=y`；tablet/wearable=`n` | `packages/ipcamera/<id>/` SO | `settings.json`：`launch_mode=ipcamera`（亦接受 `headless`） + `module_start` 保活 |
| wearable | `luna_main` | `WEARABLE=y` | `packages/wearable/<id>/` | `installed_apps` |
| tablet | `nova_main`（nova_dev 亦为 `nova_main`）| `TABLET=y` | `packages/tablet/<id>/` | `installed_apps` |

步骤：

1. `cp -r` 基线 `project/<base>/` → `project/<new>/`
2. 顶层 `config.mk`：`CONFIG_SOC_*` + **`CONFIG_BOARD`** + **MAIN/GUI** + 能力开关
3. 新板则建 `board/<BoardName>/` 并完成 Board 处理清单；复用则核对 SDK 目录
4. 按形态选型落地单应用（IOTCamera 走 application skill「IOTCamera 单应用」）
5. **config.mk 精简** → `-k/-f/-a/-p` → 上板

Board 处理文档（必过清单，细节见 [bsp-and-products](../quick-start/bsp-and-products.md) §1/§6）：

| 项 | 路径 / 动作 |
|---|---|
| DTS（board+patch 同步）| `board/<Board>/os/.../dts/` + `patch/os/.../dts/` |
| rootfs 配置 | `board/<Board>/rootfs/utils/etc/config/`（`fw_*`/`hal_*`/`app_*`）|
| 启动脚本 | nova：`app_01_start.sh`；luna 系：`app.sh` 等；**拉起的二进制 = MAIN_TARGET** |
| 板级 `config.mk` | DTB 名、分区、flash；被顶层 include |
| defconfig / burntool | kernel 选项、烧录分区 |

AI 工作表：[roles.md §2](../quick-start/roles.md#2-新产品移植开发者)（含表 C IOTCamera 单应用）。

## config.mk 精简（移植完成后必做）

目标：镜像只编进本产品用到的能力。对照 `installed_apps.json` / `settings.json` / 实际 App 调用，把**未使用**的开关改为 `n`（勿删行，便于回开）。

| 类别 | 精简原则 |
|---|---|
| `CONFIG_APP_PACKAGES_*` | tablet/wearable 互斥；`IOTCamera` 形态两者都 `n`，主程序用 `luna_dev_main` |
| `CONFIG_APP_*` App/云/协议 | 未装进 `installed_apps` / 未接线的 App、cloud、`RTSP`/`NEBULA`/`UVC` → `n` |
| `CONFIG_FW_*` | 无对应 App/网关消费的服务（如 media_player）→ `n` |
| `CONFIG_HW_*` | 板上无此外设或 App 不调用 → `n`（保留 camera 等刚需）|
| `CONFIG_THIRD_PARTY_*` | 无依赖则关；体积敏感时优先关 ffmpeg 静态链、换 mbedTLS 等 |

精简后必须：生成 `config.h` 正确 → 受影响层重编 → 上板确认功能无回归。改开关时**不要**同步改其他产品的 `config.mk`。

## 新增 Board（同产品换板）

1. `cp -r` 最近板型 → `board/<NewBoard>/`
2. 顶层 `config.mk` 设 `CONFIG_BOARD=<NewBoard>`
3. 按上表「Board 处理文档」改 DTS / rootfs / 启动脚本 / 板级 config / defconfig
4. `./build.sh -k -p --product-name <p>` → `/celestia-debug` 上板

AI 工作表见 [roles.md §新产品](../quick-start/roles.md#2-新产品移植开发者)。

## Anyka burntool 分区调优（Nova 16MB SPL fastsys）

16MB NOR 上优先 **最大化 APP（`usr.sqsh4`）**：先量真实镜像再改表。CONFIG 只需装下 `usr.jffs2`（+余量）。

**三处必须一致**（只改 burn 不改 env/DTS → `unknown-block` / 挂错根；只改 env 不重生 `.img` → 烧进去仍是旧表）：

1. **burn** `config_*.txt` → `download_to_nand*` 的 offset/size
2. **env** `env_*.cfg` → `mtdparts` + `kernel_offset`/`kernel_size` + `boot_normal` 的 `mtdblockN`，再生成 `.img`
3. **DTS** `chosen/bootargs` → 同一条 `mtdparts`（旧 DTB 把 ROOTFS 写成 `4096K` → panic）

### Nova 必改文件（SSOT 在 project overlay）

根目录：`project/anypi_nova/system/AnyCloud39AV200/V1.04/`

| 角色 | 文件 | 改什么 |
|------|------|--------|
| env（短名，与长名保持同内容） | `patch_fastsys/tools/envtool/env_av200_256MB_action_fastsys.cfg` | 见下表「env 键」 |
| env（烧录名对应） | `patch_fastsys/tools/envtool/env_av200_256MB_action_camera_spinor_spl_fastsys.cfg` | **与短名 cfg 内容一致**（当前同 hash） |
| env 镜像 | `patch_fastsys/tools/{envtool,burntool}/env_av200_256MB_action_camera_spinor_spl_fastsys.img` | 由 cfg **重新生成**后拷贝 |
| burn | `patch_fastsys/tools/burntool/config_av200_256MB_action_camera_spinor_spl_fastsys.txt` | `download_to_nand6..9`；ENV/ENVBK 文件名必须对上 `.img` |
| burn 副本 | `patch_fastsys/tools/envtool/config_av200_256MB_action_camera_spinor_spl_fastsys.txt` | 与 burntool 同内容（UTF-16） |
| DTS | `board/PDB_MBD_AK3918AV200_ACTCAM/os/kernel/.../EVB_CBDM_KM02X_ACTION_FASTSYS_V1.0.0.dts` | `bootargs` 里 `mtdparts=...` + `root=/dev/mtdblock6` |
| hybrid 打包 | `patch/scripts/build_rootfs.sh`（同步 `system/.../scripts/`） | `/usr` vs SD `install/` 落点 |

改完后：`out/anypi_nova/burntool/` 须带上新 `.img` / burn config / `.dtb`（`-k`/`-p` 或手工拷贝）。vendor `system/.../tools/` 若仍被 package 引用，一并同步。

> 非 fastsys 的 `EVB_CBDM_KM02X_ACTION_V1.0.1.dts`（UBOOT 布局、`mtdblock5`）**不要**与 fastsys 表混用。

### env 文件改哪些键

`env_*.cfg`（两份一起改）里与分区相关的键：

| 键 | 当前值 | 说明 |
|----|--------|------|
| `kernel_offset` | `0x52000` | = KERNEL 起点 = DTB 尾 |
| `kernel_size` | `0x380000` | = burn KERNEL size = mtdparts `3584K` |
| `dtb_offset` / `dtb_size` | `0x42000` / `0x10000` | 固定头，一般不动 |
| `mtdparts` | 见下方完整串 | **整串替换**；与 DTS `bootargs` 内嵌串一致 |
| `boot_normal` | `root=/dev/mtdblock6` … `${mtdparts}` | fastsys：ROOTFS = **mtdblock6**（SPL=0…APP=8） |
| `read_kernel` | 用 `${kernel_offset}` `${kernel_size}` | 改 size 后靠变量自动跟上 |

当前完整 `mtdparts`（env 里写成 `mtdparts mtdparts=spi0.0:...`）：

```
mtdparts=spi0.0:128K@0x0(SPL),128K@0x20000(OPENSBI),4K@0x40000(ENV),4K@0x41000(ENVBK),64K@0x42000(DTB),3584K@0x52000(KERNEL),3584K@0x3D2000(ROOTFS),512K@0x752000(CONFIG),8368K@0x7D2000(APP)
```

### 改完 env 必须重生 `.img`

```bash
ENVDIR=project/anypi_nova/system/AnyCloud39AV200/V1.04/patch_fastsys/tools/envtool
cd "$ENVDIR"
# 两份 cfg 保持一致后：
CFG=env_av200_256MB_action_camera_spinor_spl_fastsys.cfg
IMG=env_av200_256MB_action_camera_spinor_spl_fastsys.img
sed -i 's/\r$//' "$CFG" fw_env.config
tr '\000' '\377' < /dev/zero | dd of=./env.img bs=1024 count=4 status=none
./fw_setenv -s "$CFG"
./fw_printenv | grep -E 'mtdparts|kernel_offset|kernel_size'
cp -f env.img "$IMG"
cp -f "$IMG" ../burntool/"$IMG"
# 若 package 还读 system/ 下 tools，同样拷过去
```

burn 里 ENV/ENVBK 必须引用**同名**：

```
download_to_nand3 = env_av200_256MB_action_camera_spinor_spl_fastsys.img, ENV, 0x40000, 0x1000, 0
download_to_nand4 = env_av200_256MB_action_camera_spinor_spl_fastsys.img, ENVBK, 0x41000, 0x1000, 0
```

### burn `download_to_nand`（与 env 对齐）

| 分区 | offset | size（burn） | mtdparts 标注 |
|------|--------|--------------|---------------|
| KERNEL | `0x52000` | `0x380000` (3584K) | `3584K@0x52000` |
| ROOTFS | `0x3D2000` | `0x380000` (3584K) | `3584K@0x3D2000` |
| CONFIG | `0x752000` | `0x80000` (512K) | `512K@0x752000` |
| APP | `0x7D2000` | `0x82E000` (**8577024**) | `8368K@0x7D2000` |

```
download_to_nand6 = uImage, KERNEL, 0x52000, 0x380000, 0
download_to_nand7 = root.sqsh4, ROOTFS, 0x3D2000, 0x380000, 0
download_to_nand8 = usr.jffs2, CONFIG, 0x752000, 0x80000, 0
download_to_nand9 = usr.sqsh4, APP, 0x7D2000, 0x82E000, 0
```

> burn APP size `0x82E000` = `0x1000000 − 0x7D2000`（占满 16MB 尾）。mtdparts 标 `8368K`（略小 8KB）与 burn 并存；**offset 三处必须同一套**，且 `usr.sqsh4` ≤ burn size。

**UTF-16**：上述 `config_*.txt` 为 **UTF-16 LE + BOM**。写成 UTF-8 → 烧录工具解析失败（回退 `DDR3_PHY0_720MHZ.bin`）。改完用 `file` 确认仍是 UTF-16 LE。烧录请拷到本机 NTFS，勿直接在 WSL/`Z:`/Samba 跑 Burn Tool。

### 调整流程（再改分区时按此顺序）

1. 量 `out/anypi_nova/burntool/` 的 `uImage` / `root.sqsh4` / `usr.jffs2` / `usr.sqsh4`，各留 ≥88–256KB；超限会报 `Page:N more than end page`。
2. 固定头（SPL/OPENSBI/ENV/ENVBK/DTB）勿乱缩；溢出时从余量大的分区挪（常是 CONFIG）；**不要**把 APP burn size 压到小于 `usr.sqsh4`。
3. 改 **两份** `env_*.cfg` 的 `mtdparts` / `kernel_*` / `boot_normal` → 重生 `.img` → 拷到 burntool。
4. 改 burn `download_to_nand*`（及 envtool 下同名 txt），保持 UTF-16。
5. 改 DTS `bootargs` 的 `mtdparts` + `root=` → `-k` 出新 `.dtb`。
6. `-p` 或拷贝到 `out/.../burntool/`，烧录前核对：连续布局、APP 尾=16MB、burn≡env≡DTS、ENV 文件名、四类镜像 fit。

### hybrid：`/usr`（flash APP）vs `/mnt/install`（SD）

打包 SSOT：`prepare_install_pack_and_exclude_list`（`patch/scripts/build_rootfs.sh`）。

| 落点 | 内容 |
|------|------|
| flash `/usr` → `usr.sqsh4` | `lib/`、`gui/`、`ipc_main`、`apps/setting`、`sbin/`、`tips/` |
| SD `install/` | 其它 tablet apps、`model/`、可外置 `.so`；**不进** `download_to_nand` |

APP 再涨：优先裁 `/usr/lib` 或挪更多到 `install/`，而不是挤 ROOTFS。Nova 通话 SW 解码若撑大 `libmultimedia.so`，用 `LINK_MODE=decode` + `DECODE_ONLY`（见 [celestia-third-party](celestia-third-party.md)），`-p` 后确认 `usr.sqsh4` ≤ `0x82E000`。

增量下到已启动的 Nova 板时不必打开 Burn Tool。U 盘应用替换 `/mnt/install/apps/<id>`（备份放 `/mnt/install/.app_bak/<id>`）。用户区把新的 `usr.sqsh4` 用 `/bin/mtd_debug erase` + `mtd_debug write` 写到 `/proc/mtd` 中名为 `"APP"` 的分区后 `reboot`。写入长度不得超过该分区；fastsys 实机 APP 为 `0x8AC000`，大于 burntool 文本里的长度时以 `/proc/mtd` 为准。命令与新建/修改应用的完整步骤见 [celestia-studio](celestia-studio.md) 与 [celestia-scripts](celestia-scripts.md)「命令下载到开发板」。

## 常见问题

| 现象 | 根因 | 修复 |
|---|---|---|
| 改 config.mk 影响另一产品 | `CONFIG_*` 互相干扰 | 同时检查其他产品 config.mk |
| overlay 没生效 | 改了顶层 vendor 树 | 只改 `project/<p>/system/.../board/<Board>/` |
| rootfs 叠加被覆盖 | `patch/rootfs` 覆盖 `board/rootfs` | 按叠加顺序调整文件位置 |
| DTB 不匹配板型 | DTB 选择逻辑错 | nova: `package.sh` 按 FASTSYS/LCD/DMIC；luna: overlay；勿用废弃 `CONFIG_DTB_NAME` |
| 启动 `unknown-block(31,6)` / 挂不上 root | DTS/env `mtdparts` 与 burn 不一致，或改了 cfg **未重生** `.img` | 对齐两份 `env_*.cfg` → `fw_setenv` 出 `.img` → 同步 burn/DTS；重烧 ENV+DTB |
| 启动拉错主进程 | 脚本二进制 ≠ `CONFIG_APP_MAIN_TARGET` | 改启动脚本与 MAIN_TARGET 一致（nova→`ipc_main`，nova_dev→`nova_main`，luna→`luna_main`，luna_dev→`luna_dev_main`，mira→`mira_main`）|
| SDK 版本对不上 overlay | `CONFIG_SOC_SDK_VERSION` 与 overlay 目录不一致 | 对齐版本号 |
| 工具链报错 | 按产品名而非平台选工具链 | AV200 `riscv64-…` / AV100_V2 `arm-…` |
| `config.h` 生成不正确 | config.mk 语法错 / 产品名不存在 | 核对 config.mk + `project/<p>/` |
| 精简后功能缺失 | 误关仍在用的 `CONFIG_*` | 对照 settings.json/installed_apps 与串口 log 回开 |
| 烧录 `Page more than end page`（ROOTFS） | `root.sqsh4` > ROOTFS burn size | 扩 ROOTFS（同步改 env `mtdparts`+burn+DTS）或缩 rootfs |
| 烧录 `Page more than end page`（APP，part=`0x82E000`） | `usr.sqsh4` > APP；或 burn size 未占满尾 | 确认 APP=`0x82E000`；裁 `/usr` 或 `DECODE_ONLY`；见上节 |
| 烧录找 `DDR3_PHY0_720MHZ.bin` | burn config 被改成 UTF-8 / 在网络盘跑 | 恢复 UTF-16 LE；拷到本机 NTFS 再烧 |
| ENV 烧进去仍是旧分区 | 只改了 `.cfg` / 短名 cfg，未更新 burn 引用的 `.img` | 重生 `env_av200_256MB_action_camera_spinor_spl_fastsys.img` 并拷到 burntool |
| 通话无画面 / `VIDEO_DECODER_SW_STUB` | `FFMPEG_LINK_MODE=none` | Nova 用 `decode` + DECODE_ONLY；见 [celestia-third-party](celestia-third-party.md) |

## 编码完成 → 编译（必做）

```bash
cd ~/celestia
./build.sh -k --product-name anypi_nova   # 只构建 kernel / BSP（system/<SoC>/<SDK>）
./build.sh -p --product-name anypi_nova   # 只打包（rootfs/镜像，跳过编译）
./build.sh    --product-name anypi_luna        # 改 config.mk 后建议全量 + package 验证
```

改 `config.mk` 后至少跑受影响层（如改 `CONFIG_FW_*` 跑 `-f`，改 `CONFIG_APP_*` 跑 `-a`），或全量构建确认 `config.h` 生成正确、镜像可打包。`config.mk not found` → 确认 `--product-name` / 缓存 / 交互所选指向存在的 `project/<proj>/`。
