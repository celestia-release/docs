<p align="right"><b>English</b> | <a href="build.md">中文</a></p>

# Build guide

> Prerequisite: [Source download](download.en.md) and chip SDK placement are complete.

## Overall flow

```mermaid
flowchart LR
    S1["① Check config.mk"] --> S2["② build.sh build"]
    S2 --> S3["③ Deploy on board"]
    S3 --> S4["④ Verify"]
    style S1 fill:#fef9c3,stroke:#ca8a04
    style S2 fill:#eff6ff,stroke:#3b82f6
    style S3 fill:#ecfdf5,stroke:#10b981
    style S4 fill:#dcfce7,stroke:#16a34a
```

## Prerequisites

| Requirement | anypi_nova | anypi_nova_dev | anypi_luna | anypi_luna_dev | anypi_mira_dev |
|------|------------|----------------|------------|----------------|------------|
| Cross toolchain | `riscv64-anyka-linux-musl-` | `riscv64-anyka-linux-musl-` | `arm-anycloud-linux-uclibcgnueabi-` | `arm-anycloud-linux-uclibcgnueabi-` | `arm-none-eabi-` (aos, `setup_toolchain.sh`) |
| Chip SDK | `system/AnyCloud39AV200/V1.04/` | `system/AnyCloud39AV200/V1.04/` | `system/AnyCloud39AV100_V2/V2.05/` | `system/AnyCloud39AV100_V2/V2.04/` | `system/Sky39AV130/V1.01/` |
| Main binary | `ipc_main` | `nova_main` | `luna_main` | `luna_dev_main` | `mira_main` |
| Packages | `TABLET=y` | `TABLET=y` | `WEARABLE=y` | `IPCAMERA=y` | `IPCAMERA=y` |

> Toolchains are set by each SoC SDK’s `rules.mk`: AV200 (nova) uses RISC-V/musl (`riscv64-anyka-linux-musl-`); AV100_V2 (luna family) uses ARM/uClibc (`arm-anycloud-linux-uclibcgnueabi-`). At startup, `build.sh` clears inherited `CROSS_COMPILE` / `CROSS_TOOLCHAIN` so `rules.mk` `?=` assignments take effect. Exception: anypi_mira_dev has `CONFIG_OS_NAME=aos`; `project/anypi_mira_dev/setup_toolchain.sh` supplies the `arm-none-eabi-` (alios) toolchain (default `/opt/arm-alios-eabi`, or auto-extract from the SDK tarball).

## Source layout

```text
celestia/
├── build.sh
├── project/
│   ├── anypi_nova/config.mk       # Production Nova (TABLET)
│   ├── anypi_nova_dev/config.mk   # Nova EVB
│   ├── anypi_luna/config.mk       # Luna wearable
│   ├── anypi_luna_dev/config.mk   # Luna IOTCamera
│   └── anypi_mira_dev/config.mk       # Mira (Sky39AV130 / aos, IOTCamera)
├── application/
│   ├── app_main/                  # CONFIG_APP_MAIN_TARGET → <suffix>_main
│   └── packages/{common,tablet,wearable,ipcamera}/
├── framework/
├── hardware/
├── system/core/
└── third_party/
```

## Check config.mk

**anypi_nova** — `project/anypi_nova/config.mk`:

```makefile
CONFIG_SOC_PLATFORM     = AnyCloud39AV200
CONFIG_SOC_SDK_VERSION  = V1.04
CONFIG_BOARD            = PDB_MBD_AK3918AV200_ACTCAM
CONFIG_APP_PACKAGES_TABLET     = y
CONFIG_APP_PACKAGES_WEARABLE   = n
CONFIG_APP_MAIN_TARGET ?= nova_main
```

**anypi_luna** — `project/anypi_luna/config.mk`:

```makefile
CONFIG_SOC_PLATFORM     = AnyCloud39AV100_V2
CONFIG_SOC_SDK_VERSION  = V2.05
CONFIG_BOARD            = PDB_MBD_KM01W_ACTCAM
CONFIG_APP_PACKAGES_TABLET     = n
CONFIG_APP_PACKAGES_WEARABLE   = y
CONFIG_APP_MAIN_TARGET ?= luna_main
```

**anypi_luna_dev** — `project/anypi_luna_dev/config.mk`:

```makefile
CONFIG_SOC_PLATFORM     = AnyCloud39AV100_V2
CONFIG_SOC_SDK_VERSION  = V2.04
CONFIG_BOARD            = EVB_MBD_KM01A
CONFIG_APP_PACKAGES_TABLET     = n
CONFIG_APP_PACKAGES_WEARABLE   = n
CONFIG_APP_PACKAGES_IPCAMERA   = y
CONFIG_APP_MAIN_TARGET ?= luna_dev_main
```

**anypi_mira_dev** — `project/anypi_mira_dev/config.mk` (aos product, includes `setup_toolchain.sh`):

```makefile
CONFIG_SOC_PLATFORM     = Sky39AV130
CONFIG_SOC_SDK_VERSION  = V1.01
CONFIG_OS_NAME          = aos
CONFIG_BOARD            = EVB_CBD_AK3918AV130
CONFIG_APP_PACKAGES_IPCAMERA   = y
CONFIG_APP_MAIN_TARGET  = mira_main
```

> Differences between aos (RTOS) and Linux products (POSIX compat shims, `.so`→`.a` static linking, BSP kernel patches) are in the [AOS port skill](../skills/celestia-aos-port.en.md).

## Build commands

Product selection order: `--product-name` → cache `out/.build_product_name` → **interactive list of `project/*` for choice** (written to cache; next time you can run `./build.sh` directly). Exits with a hint if non-interactive and not specified.

```bash
cd ~/celestia

./build.sh                              # Unspecified: interactive choice and cache
./build.sh --product-name anypi_nova
./build.sh --product-name anypi_nova_dev
./build.sh --product-name anypi_luna
./build.sh --product-name anypi_luna_dev
./build.sh --product-name anypi_mira_dev
./build.sh -a --product-name anypi_nova
./build.sh --product-name anypi_luna -f -a
./build.sh --jobs 8
```

| Flag | Meaning |
|------|------|
| `--product-name <name>` | Use `project/<name>/`; omit for interactive choice / cached value |
| `-s` | system/core |
| `-t` | third_party |
| `-h` | hardware |
| `-f` | framework |
| `-a` | application |
| `-k` | kernel / BSP |
| `-p` | Package images |
| `-c` | clean |
| `--build-module <name>` | Build exact module from config.mk (e.g. `FW_MULTIMEDIA`) |
| `--build-level / --log-level / --release / --copy-build-src` | See `./build.sh --help` |

Output directory: `out/<product-name>/`.

## Common build errors

| Error | Action |
|------|------|
| `config.mk not found` | Confirm `--product-name` / cache / interactive choice matches `project/<name>/` |
| `liblvgl.a not found` | Build third_party first: `./build.sh -t` |
| Toolchain not found | Check PATH and cross-compiler installation |

## Build artifacts

```text
out/install/
├── application/<CONFIG_APP_MAIN_TARGET>   # ipc_main / nova_main / luna_main / luna_dev_main / mira_main
├── framework/lib/*.so
├── hardware/lib/*.so
└── third_party/
```

## Next steps

- [Deploy and on-board verification](deploy.en.md)
- [BSP and product assembly](bsp-and-products.en.md)
- [Port Linux products to AOS (RTOS)](../skills/celestia-aos-port.en.md) (new RTOS product / aos differences)