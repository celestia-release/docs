<p align="right"><a href="README.en.md">English</a> | <b>中文</b></p>

# Celestia Skills

`docs/skills/` 是 Celestia 分层 AI 编程 skill 的**唯一源**，随 docs 仓库版本管理。每个 skill
是 `docs/skills/` 下一份扁平的 `<name>.md`，封装该层的**目录约定、对外接口边界、排障原则、build.sh 编译命令**（含"新增 X"清单）。

## 工程建立说明（重要）

- **skill 源在 `docs/skills/`**（扁平 `<name>.md` 文件）。本文件是所有 skill 的索引。
- `.claude/skills/` 与 `.cursor/skills/` **只放一份本 README 的副本**作为索引，**不再各存一份 skill 文件**
  ——避免多份维护、避免内容漂移。这两个目录下的旧 skill 目录内容已全部移除。
- 需要某层 skill 的内容时，**直接读 `docs/skills/<name>.md`**（Agent 按本索引定位）。
- 若要让 Claude Code / Cursor 的 **skill 自动加载**重新生效，需把 `docs/skills/<name>.md`
  复制（或软链）到 `.claude/skills/<name>/SKILL.md` / `.cursor/skills/<name>/SKILL.md`（加载器要求 `<name>/SKILL.md` 结构）：
  - WSL/Linux 原生 ext4：可软链，如 `mkdir -p .claude/skills/<name> && ln -s ../../docs/skills/<name>.md .claude/skills/<name>/SKILL.md`；
  - Windows 映射盘（如 `Z:`，非 NTFS）：软链/联接不可用，请用 `cp`，如 `mkdir -p .claude/skills/<name> && cp docs/skills/<name>.md .claude/skills/<name>/SKILL.md`。
- 修改 skill：**只改 `docs/skills/` 下的源**；如需同步索引，重新把本 README 复制到两个 skills 目录即可。

## skill 索引

| Skill | 层 / 范围 | 源文件 |
|-------|-----------|--------|
| [celestia](celestia.md) | **总览 + 路由**：5 层调用关系、允许/禁止方向、排障原则、分层编译 | `docs/skills/celestia.md` |
| [celestia-application](celestia-application.md) | `application/`（含"新增 tablet App""新增 cloud 适配"） | `docs/skills/celestia-application.md` |
| [celestia-studio](celestia-studio.md) | 同步工具：新建/修改 tablet APP 直到推送；命令写 U 盘目录或 APP 分区 | `docs/skills/celestia-studio.md` |
| [celestia-framework](celestia-framework.md) | `framework/`（含"新增服务"） | `docs/skills/celestia-framework.md` |
| [celestia-hardware](celestia-hardware.md) | `hardware/`（含"新增 SoC"） | `docs/skills/celestia-hardware.md` |
| [celestia-system-core](celestia-system-core.md) | `system/core/` | `docs/skills/celestia-system-core.md` |
| [celestia-project](celestia-project.md) | `project/`, `system/<SoC>/`（含 wearable 上板、16MB NOR 分区/`0x82E000`、hybrid install） | `docs/skills/celestia-project.md` |
| [celestia-aos-port](celestia-aos-port.md) | **Linux 产品移植 AOS（RTOS）**：以 anypi_mira_dev 为蓝本（POSIX compat 垫片、.so→.a、静态 dlopen、BSP 内核补丁、合入纪律） | `docs/skills/celestia-aos-port.md` |
| [celestia-scripts](celestia-scripts.md) | `scripts/`, `build.sh` | `docs/skills/celestia-scripts.md` |
| [celestia-third-party](celestia-third-party.md) | `third_party/`（含 FFmpeg DECODE_ONLY） | `docs/skills/celestia-third-party.md` |
| [celestia-debug](celestia-debug.md) | **上板调试 + Bug 分类 A–G**（崩溃/预览/内存/启动/App/构建/出流）与解法 | `docs/skills/celestia-debug.md` |
| [celestia-contributing](celestia-contributing.md) | 命名规范、SPDX/文件头、错误码、文档编写规则 | `docs/skills/celestia-contributing.md` |
| [git-gitee](git-gitee.md) | Gitee 提交/同步规范（默认分支 release） | `docs/skills/git-gitee.md` |

## 五个产品

| 产品 | SoC / SDK | Board | 主程序 | 说明 |
|------|-----------|-------|--------|------|
| anypi_nova | AnyCloud39AV200 / V1.04 | PDB_MBD_AK3918AV200_ACTCAM | `ipc_main` | `TABLET`（平板级，800×480）|
| anypi_nova_dev | AnyCloud39AV200 / V1.04 | EVB_MBD_AK3918AV200 | `nova_main` | `TABLET`（平板级，1024×600）|
| anypi_luna | AnyCloud39AV100_V2 / V2.05 | PDB_MBD_KM01W_ACTCAM | `luna_main` | `WEARABLE`（穿戴级，240×240）|
| anypi_luna_dev | AnyCloud39AV100_V2 / V2.04 | EVB_MBD_KM01A | `luna_dev_main` | `IOTCamera` |
| anypi_mira_dev | Sky39AV130 / V1.01 | EVB_CBD_AK3918AV130 | `mira_main` | `IOTCamera`（`CONFIG_OS_NAME=aos`）|

> **形态判据**：带显示 = `CONFIG_BOARD_SUPPORT_LCD=y` + `PACKAGES_TABLET|WEARABLE=y` + `launch_mode=ui`；
> `IOTCamera` 形态 = LCD 关且 tablet/wearable 皆 `n`，靠业务 SO `module_start()` 保活。完整定义见
> [术语与命名对照表 §2](../glossary.md#2-产品形态-iotcamera-ai_glasses-tablet-wearable重点)。
>
> 工具链：AV200（nova）= `riscv64-anyka-linux-musl-`；AV100_V2（luna 系）= `arm-anycloud-linux-uclibcgnueabi-`；Sky39AV130（mira，aos）= `arm-none-eabi-`（`project/anypi_mira_dev/setup_toolchain.sh`）。

## 共同约定

1. **以本层为主分析**：处理某层问题先攻本层代码，对被调下层**只确认对外 `include/*.h` 接口契约**，仅当证据（返回码/日志/tests）表明下层有问题才下钻。
2. **跨层只走对外公开头**，禁止 `Application→HAL/BSP`、`Framework→SoC 私有实现`、`System Core→上层`。
3. **编码完成必编译**：改完用 `build.sh` 对应 flag（`-s/-t/-h/-f/-a/-k/-p`）+ `--product-name <product>` 编译并解决全部错误；改公用代码评估各产品影响。
4. **改动有运行期表现的必上板验证**：见 `docs/skills/celestia-debug.md`（含 Bug 分类 A–G 与解法）；贴 log 先归类再下钻层 skill。

## 事实来源

文档与代码不一致时以**代码为准**（`tests/` > `*.c` > `*.h` > `Makefile/build.sh/config.mk` > `docs/`）。
