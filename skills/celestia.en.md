---
name: celestia
description: >-
  Celestia embedded layered project overview and routing skill: 5-layer architecture
  (Application → Framework → Hardware HAL → System Core; System Core is horizontally depended on by
  three layers; HAL → Kernel/BSP), allowed/forbidden cross-layer call directions,
  troubleshooting principle of "analyze primarily at this layer, confirm only public
  interfaces of callees, drill down only when lower layer is proven wrong", and build.sh
  layered compile loop. When handling issues in any Celestia layer, deciding which layer
  owns the problem, or unsure which layer skill to use, read this skill first then route
  to the layer skill.
---

<p align="right"><b>English</b> | <a href="celestia.md">中文</a></p>

# Celestia project overview (routing skill)

The repo-root `AGENTS.md` links here and is generated with `build.sh` via manifest linkfile. Models read `AGENTS.md` first, then open `docs/skills/` via the paths below:

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

> Per `docs/architecture/overview.md`. This skill handles **layer placement + cross-layer interface confirmation + routing**;
> per-layer implementation details, directory conventions, and compile commands are in the matching layer skill.

## Layers and layer skills

| Layer | Directory | Layer skill | Concept / API docs |
|----|------|-----------|----------------|
| 🖥️ Application | `application/` | [celestia-application](celestia-application.en.md) | `docs/guide/application.md` |
| ⚙️ Framework | `framework/` | [celestia-framework](celestia-framework.en.md) | `docs/guide/framework.md` · `docs/guide/framework-api.md` |
| 🔌 Hardware HAL | `hardware/` | [celestia-hardware](celestia-hardware.en.md) | `docs/guide/hardware-hal.md` · `docs/guide/hardware-api.md` |
| 🔧 System Core | `system/core/` | [celestia-system-core](celestia-system-core.en.md) | `docs/guide/system-core.md` · `docs/guide/system-core-api.md` |
| 🏗️ Project / BSP | `project/`, `system/<SoC>/` | [celestia-project](celestia-project.en.md) | `docs/quick-start/bsp-and-products.md` |
| 🔁 AOS port | Linux product → Aos (RTOS) | [celestia-aos-port](celestia-aos-port.en.md) | [quick-start/build](../quick-start/build.en.md) (anypi_mira_dev config) |
| 🛠️ Scripts / build | `scripts/`, `build.sh` | [celestia-scripts](celestia-scripts.en.md) | — |
| 📲 Sync / push | `scripts/studio/` | [celestia-studio](celestia-studio.en.md) | `docs/guide/studio_agents.md` |
| 📦 Third-Party | `third_party/` | [celestia-third-party](celestia-third-party.en.md) | `docs/quick-start/bsp-and-products.md §3.5` |
| 🐛 Debug / on-device / Bug taxonomy | Post-deploy observe + PASS/FAIL + A–G fixes + DMA/memory | [celestia-debug](celestia-debug.en.md) | `roles.md` §5 · `doc/WEARABLE_MEMORY_FOOTPRINT.md` |
| 📝 Contributing | Naming/headers/docs rules | [celestia-contributing](celestia-contributing.en.md) | — |
| 🔀 Git / Gitee | Commit and sync conventions | [git-gitee](git-gitee.en.md) | — |

Build commands: [celestia-scripts](celestia-scripts.en.md); post-deploy loop and PASS/FAIL: [celestia-debug](celestia-debug.en.md); wearable bring-up and **16MB NOR partition tuning**: [celestia-project](celestia-project.en.md).

## Call graph (allowed directions)

```
🖥️ Application ──app_main_get_*() / *_open()──► ⚙️ Framework
       │                                              │
       └──#include system_core.h──► 🔧 System Core ◄──┤  (shared horizontally by three layers)
                                          ▲           │
🔌 Hardware HAL ──#include system_core.h──┘           │
       ▲                                              │
       └────────hardware/include/*.h◄────────────────┘
       │
       └──/dev/* ioctl / sysfs──► 🏗️ Kernel/BSP
```

Build dependency order (matches `build.sh`): `third_party → system/core → hardware → framework → application → package`.

## Forbidden directions (AI must refuse)

| Forbidden | Reason |
|------|------|
| `Application → HAL` or `Application → Kernel/BSP` | Must go through Framework; App must not include `hardware/include/*.h` or SoC-private headers |
| `Framework → hardware/*/hal/<SoC>/` (chip-private implementation) | Framework may only include public `hardware/include/*.h` |
| `System Core → Framework / Hardware` | System Core is an independent foundation layer, no upward deps |
| Any layer includes `hal_*_api.h` / `*_hw.h` / BSP-private headers | Cross-layer uses only public `include/` headers |

Layer and call constraints: `docs/architecture/overview.md` §2.

## Core troubleshooting: own layer first, callees = public interface only

For any layer issue, follow this order strictly — **do not jump straight to lower layers or BSP**:

1. **Locate the layer** — map symptoms to the primary layer:

   | Symptom / stack | Layer to check first |
   |---------------|-----------|
   | GUI, launcher, business logic, App start/stop | Application (`packages/tablet`, `packages/wearable`, `app_main`) |
   | Multimedia / network / storage / notification / runtime / gateway protocols | Framework (`services/`, `runtime/`, `gateway/`) |
   | Drivers, sensors, display, audio, ISP/encode behavior | Hardware HAL (**start with `hardware/include/*.h` public contract only**) |
   | Logs, threads, config, timers, error codes, utilities | System Core |
   | Build options, SoC/board, DTS, rootfs, flash | Project / BSP |
   | Build commands, serial bring-up | Scripts |
   | Third-party versions / link errors | Third-Party |

2. **Focus on this layer** — assume callees are correct; analyze this layer’s logic.

3. **For callees, only “confirm public interface”** — do not read implementation:
   - Check `include/*.h` signatures, parameters, return conventions, error codes;
   - Check matching `*_API.md` / concept docs and `tests/`;
   - Verify **how this layer calls** the API (params, ordering, return handling).

4. **Drill down only with evidence** — return codes / logs from lower API, docs or `tests/` show contract mismatch, or this layer’s usage is correct but still fails. Switch to that layer’s skill and repeat steps 2–4 treating it as “current layer”.

5. **After fixes, build-verify by layer** (below).

> One line: **this layer logic → lower interface contract → (with evidence) lower implementation**, converge level by level, avoid skipping layers.

## Layered build (required after coding)

After editing/writing code in any layer skill, **must** run `build.sh` for that layer and fix all compile errors before delivery.
Full commands and options: [celestia-scripts](celestia-scripts.en.md).

```bash
cd ~/celestia
./build.sh -s --product-name <product>    # system/core
./build.sh -t --product-name <product>    # third_party (or -t <module>)
./build.sh -h --product-name <product>    # hardware
./build.sh -f --product-name <product>    # framework
./build.sh -a --product-name <product>    # application
./build.sh -k --product-name <product>    # kernel / BSP
./build.sh    --product-name <product>    # full + package
```

For multi-layer changes, build bottom-up (e.g. `-h -f -a`). On compile failure → decide if this layer misused API or lower contract changed → fix per troubleshooting principles → rebuild until 0 errors.

| Product | SoC | Main binary | config | Notes |
|------|-----|--------|--------|------|
| anypi_nova | AnyCloud39AV200 / V1.04 | `ipc_main` | `project/anypi_nova/config.mk` | Mass-production Nova (`PACKAGES_TABLET=y`) |
| anypi_nova_dev | AnyCloud39AV200 / V1.04 | `nova_main` | `project/anypi_nova_dev/config.mk` | Tablet-class (`TABLET`) desktop EVB |
| anypi_luna | AnyCloud39AV100_V2 / V2.05 | `luna_main` | `project/anypi_luna/config.mk` | Wearable-class (`WEARABLE`) ActionCam |
| anypi_luna_dev | AnyCloud39AV100_V2 / V2.04 | `luna_dev_main` | `project/anypi_luna_dev/config.mk` | IOTCamera |
| anypi_mira_dev | Sky39AV130 / V1.01 | `mira_main` | `project/anypi_mira_dev/config.mk` | IOTCamera (aos, `setup_toolchain.sh`) |

> Note: GUI dirs `application/packages/tablet` (shared nova / nova_dev) and `application/packages/wearable`; IOTCamera uses `packages/ipcamera` (`ipc_cam`).

## Product isolation

Each product selects SoC, board, and feature switches via `project/<product>/config.mk`. **Configs do not affect each other**:

- `config.mk` → generates `config.h`, the only valid cross-product differentiation point;
- Do not branch in source with `#ifdef CONFIG_SOC_*`; use HAL ops / module switches;
- After changing one product’s `config.mk`, **build only that product** — others’ artifacts are unaffected.

## When docs and code disagree

**Code wins** (`.h` sources + `config.mk` + `tests/`), docs are secondary. Call out stale docs (main binary name, directory names) in delivery notes; update `docs/` when needed.