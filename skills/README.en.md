<p align="right"><b>English</b> | <a href="README.md">中文</a></p>

# Celestia Skills

`docs/skills/` is the **single source** of Celestia layered AI programming skills, versioned with the docs repo. Each skill is a flat `<name>.md` under `docs/skills/`, packaging that layer’s **directory conventions, public interface boundaries, troubleshooting principles, and build.sh compile commands** (including “add X” checklists).

## Project setup (important)

- **Skill sources live in `docs/skills/`** (flat `<name>.md` files). This file is the index for all skills.
- `.claude/skills/` and `.cursor/skills/` hold **only a copy of this README** as the index — **no per-skill copies** there anymore, to avoid duplicate maintenance and content drift. Legacy skill directory content under those paths has been removed.
- When you need a layer skill, **read `docs/skills/<name>.md` directly** (agents locate via this index).
- To restore **automatic skill loading** in Claude Code / Cursor, copy (or symlink) `docs/skills/<name>.md` to `.claude/skills/<name>/SKILL.md` / `.cursor/skills/<name>/SKILL.md` (loaders expect a `<name>/SKILL.md` layout):
  - WSL/Linux native ext4: symlinks work, e.g. `mkdir -p .claude/skills/<name> && ln -s ../../docs/skills/<name>.md .claude/skills/<name>/SKILL.md`;
  - Windows mapped drives (e.g. `Z:`, non-NTFS): symlinks/junctions are unavailable; use `cp`, e.g. `mkdir -p .claude/skills/<name> && cp docs/skills/<name>.md .claude/skills/<name>/SKILL.md`.
- To edit a skill: **change only the source under `docs/skills/`**; to refresh indexes, recopy this README into both skills directories.

## Skill index

| Skill | Layer / scope | Source file |
|-------|-------------|-------------|
| [celestia](celestia.en.md) | **Overview + routing**: 5-layer call graph, allowed/forbidden directions, troubleshooting principles, layered build | `docs/skills/celestia.md` |
| [celestia-application](celestia-application.en.md) | `application/` (incl. “add tablet App”, “add cloud adapter”) | `docs/skills/celestia-application.md` |
| [celestia-studio](celestia-studio.en.md) | Sync tool: create/edit tablet APP through push; commands target USB dir or APP partition | `docs/skills/celestia-studio.md` |
| [celestia-framework](celestia-framework.en.md) | `framework/` (incl. “add service”) | `docs/skills/celestia-framework.md` |
| [celestia-hardware](celestia-hardware.en.md) | `hardware/` (incl. “add SoC”) | `docs/skills/celestia-hardware.md` |
| [celestia-system-core](celestia-system-core.en.md) | `system/core/` | `docs/skills/celestia-system-core.md` |
| [celestia-project](celestia-project.en.md) | `project/`, `system/<SoC>/` (wearable bring-up, 16MB NOR partitions / `0x82E000`, hybrid install) | `docs/skills/celestia-project.md` |
| [celestia-aos-port](celestia-aos-port.en.md) | **Linux product → AOS (RTOS)**: anypi_mira_dev blueprint (POSIX compat shims, .so→.a, static dlopen, BSP kernel patches, merge discipline) | `docs/skills/celestia-aos-port.md` |
| [celestia-scripts](celestia-scripts.en.md) | `scripts/`, `build.sh` | `docs/skills/celestia-scripts.md` |
| [celestia-third-party](celestia-third-party.en.md) | `third_party/` (incl. FFmpeg DECODE_ONLY) | `docs/skills/celestia-third-party.md` |
| [celestia-debug](celestia-debug.en.md) | **On-device debug + Bug classes A–G** (crash/preview/memory/boot/App/build/stream) and fixes | `docs/skills/celestia-debug.md` |
| [celestia-contributing](celestia-contributing.en.md) | Naming, SPDX/headers, error codes, documentation rules | `docs/skills/celestia-contributing.md` |
| [git-gitee](git-gitee.en.md) | Git commit/sync (default branch `main`, GitHub) | `docs/skills/git-gitee.md` |

## Five products

| Product | SoC / SDK | Board | Main binary | Notes |
|---------|-----------|-------|-------------|-------|
| anypi_nova | AnyCloud39AV200 / V1.04 | PDB_MBD_AK3918AV200_ACTCAM | `ipc_main` | `TABLET` (tablet-class, 800×480) |
| anypi_nova_dev | AnyCloud39AV200 / V1.04 | EVB_MBD_AK3918AV200 | `nova_main` | `TABLET` (tablet-class, 1024×600) |
| anypi_luna | AnyCloud39AV100_V2 / V2.05 | PDB_MBD_KM01W_ACTCAM | `luna_main` | `WEARABLE` (wearable-class, 240×240) |
| anypi_luna_dev | AnyCloud39AV100_V2 / V2.04 | EVB_MBD_KM01A | `luna_dev_main` | `IOTCamera` |
| anypi_mira_dev | Sky39AV130 / V1.01 | EVB_CBD_AK3918AV130 | `mira_main` | `IOTCamera` (`CONFIG_OS_NAME=aos`) |

> **Form factor rules**: Display products = `CONFIG_BOARD_SUPPORT_LCD=y` + `PACKAGES_TABLET|WEARABLE=y` + `launch_mode=ui`;
> `IOTCamera` = LCD off and both tablet/wearable `n`, business SO `module_start()` keeps the process alive. Full definition:
> [Glossary §2](../glossary.en.md#2-产品形态-iotcamera-ai_glasses-tablet-wearable重点).
>
> Toolchains: AV200 (nova) = `riscv64-anyka-linux-musl-`; AV100_V2 (luna family) = `arm-anycloud-linux-uclibcgnueabi-`; Sky39AV130 (mira, aos) = `arm-none-eabi-` (`project/anypi_mira_dev/setup_toolchain.sh`).

## Shared conventions

1. **Analyze at your layer first**: For a layer issue, fix that layer’s code; for callees **only confirm public `include/*.h` contracts**; drill down only when evidence (return codes / logs / tests) shows a lower-layer bug.
2. **Cross-layer calls use public headers only** — no `Application→HAL/BSP`, no `Framework→SoC-private HAL`, no `System Core→upper layers`.
3. **Always build after code changes**: Use `build.sh` flags (`-s/-t/-h/-f/-a/-k/-p`) + `--product-name <product>` and fix all errors; for shared code, assess impact on every product.
4. **Runtime behavior needs on-device verification**: See `docs/skills/celestia-debug.en.md` (Bug classes A–G and fixes); classify logs before diving into a layer skill.

## Source of truth

When docs and code disagree, **code wins** (`tests/` > `*.c` > `*.h` > `Makefile/build.sh/config.mk` > `docs/`).