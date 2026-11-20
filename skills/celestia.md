---
name: celestia
description: >-
  Celestia 嵌入式分层项目总览与路由 skill：5 层架构（Application → Framework →
  Hardware HAL → System Core，System Core 被三层横向依赖；HAL → Kernel/BSP）、
  允许/禁止的跨层调用方向、"以本层为主分析、对被调层只确认对外接口、确认下层有问题再下钻"
  的排障原则，以及 build.sh 分层编译闭环。当处理 celestia 任意层的问题、需要判断问题落在
  哪一层、或不确定该用哪个分层 skill 时，先读本 skill 再路由到对应层 skill。
---

<p align="right"><a href="celestia.en.md">English</a> | <b>中文</b></p>

# Celestia 项目总览（路由 skill）

工程根目录的 `AGENTS.md` 链到本文件，和 `build.sh` 一样由 manifest 的 linkfile 生成。各模型先读 `AGENTS.md`，再按下面的路径打开 `docs/skills/`：

- `docs/skills/README.md`
- `docs/skills/celestia.md`
- `docs/skills/celestia-application.md`
- `docs/skills/celestia-framework.md`
- `docs/skills/celestia-hardware.md`
- `docs/skills/celestia-system-core.md`
- `docs/skills/celestia-project.md`
- `docs/skills/celestia-aos-port.md`
- `docs/skills/celestia-scripts.md`
- `docs/skills/celestia-studio.md`
- `docs/skills/celestia-third-party.md`
- `docs/skills/celestia-debug.md`
- `docs/skills/celestia-contributing.md`
- `docs/skills/git-gitee.md`

> 依据 `docs/architecture/overview.md`。本 skill 负责**定位层级 + 跨层接口确认 + 路由**；
> 具体某层的实现细节、目录约定与编译命令见对应分层 skill。

## 分层与各层 skill

| 层 | 目录 | 分层 skill | 概念 / API 文档 |
|----|------|-----------|----------------|
| 🖥️ Application | `application/` | [celestia-application](celestia-application.md) | `docs/guide/application.md` |
| ⚙️ Framework | `framework/` | [celestia-framework](celestia-framework.md) | `docs/guide/framework.md` · `docs/guide/framework-api.md` |
| 🔌 Hardware HAL | `hardware/` | [celestia-hardware](celestia-hardware.md) | `docs/guide/hardware-hal.md` · `docs/guide/hardware-api.md` |
| 🔧 System Core | `system/core/` | [celestia-system-core](celestia-system-core.md) | `docs/guide/system-core.md` · `docs/guide/system-core-api.md` |
| 🏗️ Project / BSP | `project/`, `system/<SoC>/` | [celestia-project](celestia-project.md) | `docs/quick-start/bsp-and-products.md` |
| 🔁 AOS 移植 | Linux 产品 → Aos（RTOS） | [celestia-aos-port](celestia-aos-port.md) | [quick-start/build](../quick-start/build.md)（anypi_mira_dev 配置） |
| 🛠️ Scripts / 构建 | `scripts/`, `build.sh` | [celestia-scripts](celestia-scripts.md) | — |
| 📲 同步推送 | `scripts/studio/` | [celestia-studio](celestia-studio.md) | `docs/guide/studio_agents.md` |
| 📦 Third-Party | `third_party/` | [celestia-third-party](celestia-third-party.md) | `docs/quick-start/bsp-and-products.md §3.5` |
| 🐛 Debug / 上板 / Bug 分类 | 部署后观察+PASS/FAIL+A–G 类解法+DMA/内存 | [celestia-debug](celestia-debug.md) | `roles.md` §5 · `doc/WEARABLE_MEMORY_FOOTPRINT.md` |
| 📝 Contributing | 命名/文件头/文档规范 | [celestia-contributing](celestia-contributing.md) | — |
| 🔀 Git / Gitee | 提交与同步规范 | [git-gitee](git-gitee.md) | — |

编译命令见 [celestia-scripts](celestia-scripts.md)；部署后跑闭环判 PASS/FAIL 见 [celestia-debug](celestia-debug.md)；wearable 上板与 **16MB NOR 分区调优**见 [celestia-project](celestia-project.md)。

## 调用关系（允许的方向）

```
🖥️ Application ──app_main_get_*() / *_open()──► ⚙️ Framework
       │                                              │
       └──#include system_core.h──► 🔧 System Core ◄──┤  (横向被三层依赖)
                                          ▲           │
🔌 Hardware HAL ──#include system_core.h──┘           │
       ▲                                              │
       └────────hardware/include/*.h◄────────────────┘
       │
       └──/dev/* ioctl / sysfs──► 🏗️ Kernel/BSP
```

构建依赖顺序（与 `build.sh` 一致）：`third_party → system/core → hardware → framework → application → package`。

## 禁止的方向（AI 必拒）

| 禁止 | 原因 |
|------|------|
| `Application → HAL` 或 `Application → Kernel/BSP` | 必须经 Framework；App 不得 include `hardware/include/*.h` 或 SoC 私有头 |
| `Framework → hardware/*/hal/<SoC>/`（芯片私有实现） | Framework 只能 include `hardware/include/*.h` 公开头 |
| `System Core → Framework / Hardware` | System Core 是独立基础层，不上向依赖 |
| 任意层 include `hal_*_api.h` / `*_hw.h` / BSP 私有头 | 跨层只走对外 `include/` 公开头 |

分层与调用约束见 `docs/architecture/overview.md` §2。

## 核心排障原则：以本层为主，对下层只确认接口

处理任意层的问题时，严格按下面顺序，**不要一上来就改下层或 BSP**：

1. **定位层级**——按现象映射到主排查层：

   | 现象 / 调用栈 | 优先排查层 |
   |---------------|-----------|
   | GUI、launcher、业务逻辑、App 启停 | Application（`packages/tablet`、`packages/wearable`、`app_main`） |
   | 多媒体 / 网络 / 存储 / 通知 / runtime / 网关协议 | Framework（`services/`、`runtime/`、`gateway/`） |
   | 驱动、传感器、显示、音频、ISP/编码行为 | Hardware HAL（**先只看 `hardware/include/*.h` 对外契约**） |
   | 日志、线程、配置、定时器、错误码、通用工具 | System Core |
   | 编译选项、SoC/板型、DTS、rootfs、烧录 | Project / BSP |
   | 编译命令、串口跑板 | Scripts |
   | 第三方库版本 / 链接错误 | Third-Party |

2. **主攻本层代码**——先假设被调用的下层是正确的，集中分析本层逻辑。

3. **对被调层只"确认对外接口"**，不读其实现：
   - 看该层 `include/*.h` 的函数签名、参数、返回值约定、错误码；
   - 看对应 `*_API.md` / 概念文档与 `tests/`；
   - 检查本层对接口的**调用方式**（参数、调用时序、返回值处理）是否正确。

4. **仅当有证据表明下层有问题时才下钻**——证据指：下层接口返回错误码 / 日志、文档或 `tests/` 表明契约不符、本层调用完全正确仍失败。此时切换到下层的分层 skill，把该下层当作"当前层"重复步骤 2–4。

5. **改完即按层编译验证**（见下）。

> 一句话：**本层逻辑 → 下层接口契约 → （有据再）下层实现**，逐级收敛，避免越级猜测。

## 分层编译（编码完成必做）

每层 skill 在改完/新写代码后，**必须**调用 `build.sh` 编译对应层并解决全部编译错误，再交付。
完整编译命令与选项详见 [celestia-scripts](celestia-scripts.md)。

```bash
cd ~/celestia
./build.sh -s --product-name <product>    # system/core
./build.sh -t --product-name <product>    # third_party（或 -t <module>）
./build.sh -h --product-name <product>    # hardware
./build.sh -f --product-name <product>    # framework
./build.sh -a --product-name <product>    # application
./build.sh -k --product-name <product>    # kernel / BSP
./build.sh    --product-name <product>    # 全量 + package
```

改了跨多层时按依赖顺序从下往上编（如 `-h -f -a`）。编译失败→定位是本层语法/接口误用还是下层契约变化→回到排障原则修正→重编，直到 0 error。

| 产品 | SoC | 主程序 | config | 说明 |
|------|-----|--------|--------|------|
| anypi_nova | AnyCloud39AV200 / V1.04 | `ipc_main` | `project/anypi_nova/config.mk` | 量产 Nova（`PACKAGES_TABLET=y`）|
| anypi_nova_dev | AnyCloud39AV200 / V1.04 | `nova_main` | `project/anypi_nova_dev/config.mk` | 平板级（`TABLET`）桌面 EVB |
| anypi_luna | AnyCloud39AV100_V2 / V2.05 | `luna_main` | `project/anypi_luna/config.mk` | 穿戴级（`WEARABLE`） ActionCam |
| anypi_luna_dev | AnyCloud39AV100_V2 / V2.04 | `luna_dev_main` | `project/anypi_luna_dev/config.mk` | IOTCamera |
| anypi_mira_dev | Sky39AV130 / V1.01 | `mira_main` | `project/anypi_mira_dev/config.mk` | IOTCamera（aos，`setup_toolchain.sh`）|

> 注：GUI 目录 `application/packages/tablet`（nova / nova_dev 共用）与 `application/packages/wearable`；IOTCamera 形态用 `packages/ipcamera`（`ipc_cam`）。

## 产品之间的隔离原则

各产品通过独立 `project/<product>/config.mk` 选择 SoC、板型、功能开关。**不同产品的配置互不影响**：

- `config.mk` → 生成 `config.h`，是唯一合法的跨产品差异化点；
- 不得在源码里用 `#ifdef CONFIG_SOC_*` 直接做产品分支；应通过 HAL ops / 模块开关实现；
- 修改一个产品的 config.mk 后**只编译该产品**，不会影响其他产品的构建产物。

## 文档与代码不一致时

以**代码（`.h` 源文件 + `config.mk` + `tests/`）为准**，文档为辅。发现文档过期（如主程序名、目录名）时在交付说明里指出，必要时更新 `docs/` 中对应文档。
