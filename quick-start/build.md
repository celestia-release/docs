<p align="right"><a href="build.en.md">English</a> | <b>中文</b></p>

# 构建指南

> 前置：已完成 [源码下载](download.md) 与芯片 SDK 放置。

## 整体流程

```mermaid
flowchart LR
    S1["① 检查 config.mk"] --> S2["② build.sh 编译"]
    S2 --> S3["③ 部署上板"]
    S3 --> S4["④ 验证"]
    style S1 fill:#fef9c3,stroke:#ca8a04
    style S2 fill:#eff6ff,stroke:#3b82f6
    style S3 fill:#ecfdf5,stroke:#10b981
    style S4 fill:#dcfce7,stroke:#16a34a
```

## 前置条件

| 条件 | anypi_nova | anypi_nova_dev | anypi_luna | anypi_luna_dev | anypi_mira_dev |
|------|------------|----------------|------------|----------------|------------|
| 交叉工具链 | `riscv64-anyka-linux-musl-` | `riscv64-anyka-linux-musl-` | `arm-anycloud-linux-uclibcgnueabi-` | `arm-anycloud-linux-uclibcgnueabi-` | `arm-none-eabi-`（aos，`setup_toolchain.sh`） |
| 芯片 SDK | `system/AnyCloud39AV200/V1.04/` | `system/AnyCloud39AV200/V1.04/` | `system/AnyCloud39AV100_V2/V2.05/` | `system/AnyCloud39AV100_V2/V2.04/` | `system/Sky39AV130/V1.01/` |
| 主程序 | `ipc_main` | `nova_main` | `luna_main` | `luna_dev_main` | `mira_main` |
| Packages | `TABLET=y` | `TABLET=y` | `WEARABLE=y` | `IPCAMERA=y` | `IPCAMERA=y` |

> 工具链由各 SoC SDK 的 `rules.mk` 决定：AV200（nova）为 RISC-V/musl（`riscv64-anyka-linux-musl-`），AV100_V2（luna 系）为 ARM/uClibc（`arm-anycloud-linux-uclibcgnueabi-`）。`build.sh` 启动时会清空继承的 `CROSS_COMPILE` / `CROSS_TOOLCHAIN`，让 `rules.mk` 的 `?=` 赋值生效。例外：anypi_mira_dev 为 `CONFIG_OS_NAME=aos`，由 `project/anypi_mira_dev/setup_toolchain.sh` 提供 `arm-none-eabi-`（alios）工具链（默认 `/opt/arm-alios-eabi`，或自动解压 SDK 内 tarball）。

## 源码结构

```text
celestia/
├── build.sh
├── project/
│   ├── anypi_nova/config.mk       # 量产 Nova（TABLET）
│   ├── anypi_nova_dev/config.mk   # Nova EVB
│   ├── anypi_luna/config.mk       # Luna 穿戴级
│   ├── anypi_luna_dev/config.mk   # Luna IOTCamera
│   └── anypi_mira_dev/config.mk       # Mira（Sky39AV130 / aos，IOTCamera）
├── application/
│   ├── app_main/                  # CONFIG_APP_MAIN_TARGET → <suffix>_main
│   └── packages/{common,tablet,wearable,ipcamera}/
├── framework/
├── hardware/
├── system/core/
└── third_party/
```

## 检查 config.mk

**anypi_nova** — `project/anypi_nova/config.mk`：

```makefile
CONFIG_SOC_PLATFORM     = AnyCloud39AV200
CONFIG_SOC_SDK_VERSION  = V1.04
CONFIG_BOARD            = PDB_MBD_AK3918AV200_ACTCAM
CONFIG_APP_PACKAGES_TABLET     = y
CONFIG_APP_PACKAGES_WEARABLE   = n
CONFIG_APP_MAIN_TARGET ?= nova_main
```

**anypi_luna** — `project/anypi_luna/config.mk`：

```makefile
CONFIG_SOC_PLATFORM     = AnyCloud39AV100_V2
CONFIG_SOC_SDK_VERSION  = V2.05
CONFIG_BOARD            = PDB_MBD_KM01W_ACTCAM
CONFIG_APP_PACKAGES_TABLET     = n
CONFIG_APP_PACKAGES_WEARABLE   = y
CONFIG_APP_MAIN_TARGET ?= luna_main
```

**anypi_luna_dev** — `project/anypi_luna_dev/config.mk`：

```makefile
CONFIG_SOC_PLATFORM     = AnyCloud39AV100_V2
CONFIG_SOC_SDK_VERSION  = V2.04
CONFIG_BOARD            = EVB_MBD_KM01A
CONFIG_APP_PACKAGES_TABLET     = n
CONFIG_APP_PACKAGES_WEARABLE   = n
CONFIG_APP_PACKAGES_IPCAMERA   = y
CONFIG_APP_MAIN_TARGET ?= luna_dev_main
```

**anypi_mira_dev** — `project/anypi_mira_dev/config.mk`（aos 产品，含 `setup_toolchain.sh`）：

```makefile
CONFIG_SOC_PLATFORM     = Sky39AV130
CONFIG_SOC_SDK_VERSION  = V1.01
CONFIG_OS_NAME          = aos
CONFIG_BOARD            = EVB_CBD_AK3918AV130
CONFIG_APP_PACKAGES_IPCAMERA   = y
CONFIG_APP_MAIN_TARGET  = mira_main
```

> aos 产品（RTOS）与 Linux 产品的差异（POSIX compat 垫片、`.so`→`.a` 静态链接、BSP 内核补丁）见 [AOS 移植 skill](../skills/celestia-aos-port.md)。

## 编译命令

产品选择顺序：`--product-name` → 缓存 `out/.build_product_name` → **交互列出 `project/*` 供选择**（写入缓存，下次可直接 `./build.sh`）。非交互且未指定时退出并提示可选产品。

```bash
cd ~/celestia

./build.sh                              # 未指定：交互选择并缓存
./build.sh --product-name anypi_nova
./build.sh --product-name anypi_nova_dev
./build.sh --product-name anypi_luna
./build.sh --product-name anypi_luna_dev
./build.sh --product-name anypi_mira_dev
./build.sh -a --product-name anypi_nova
./build.sh --product-name anypi_luna -f -a
./build.sh --jobs 8
```

| 参数 | 说明 |
|------|------|
| `--product-name <name>` | 指定 `project/<name>/`；省略则交互选择 / 用缓存 |
| `-s` | system/core |
| `-t` | third_party |
| `-h` | hardware |
| `-f` | framework |
| `-a` | application |
| `-k` | kernel / BSP |
| `-p` | 打包镜像 |
| `-c` | clean |
| `--build-module <name>` | 按 config.mk 模块名精确构建（如 `FW_MULTIMEDIA`）|
| `--build-level / --log-level / --release / --copy-build-src` | 见 `./build.sh --help` |

产物目录：`out/<product-name>/`。

## 常见构建错误

| 错误 | 处理 |
|------|------|
| `config.mk not found` | 确认 `--product-name` / 缓存 / 交互所选与 `project/<name>/` 对应 |
| `liblvgl.a not found` | `./build.sh -t` 先编 third_party |
| 工具链 not found | 检查 PATH 与交叉编译器安装 |

## 产物说明

```text
out/install/
├── application/<CONFIG_APP_MAIN_TARGET>   # ipc_main / nova_main / luna_main / luna_dev_main / mira_main
├── framework/lib/*.so
├── hardware/lib/*.so
└── third_party/
```

## 下一步

- [部署与上板验证](deploy.md)
- [BSP 与产品组装](bsp-and-products.md)
- [Linux 产品移植 AOS（RTOS）](../skills/celestia-aos-port.md)（新 RTOS 产品 / aos 差异）
