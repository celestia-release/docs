---
name: celestia-debug
description: >-
  Celestia on-device debug, closed-loop verification, memory/DMA troubleshooting, and Bug taxonomy
  skill: serial log PASS/FAIL; classify into seven buckets (crash/preview/memory-DMA/boot/App-SO/build-stale/stream-network)
  with typical logs, root causes, fixes, and routes to celestia-application/framework/hardware/project/scripts layer skills.
  Includes Nova partition overflow (Page more than end page / unknown-block) and call stub decode. Use when
  validating on board, pasting logs, locating crash/timeout/OOM, or unsure which layer owns the issue.
---

<p align="right"><b>English</b> | <a href="celestia-debug.md">中文</a></p>

# Celestia on-device debug, verification, and Bug taxonomy skill

> Layer routing: [celestia](celestia.en.md); build: [celestia-scripts](celestia-scripts.en.md).
> Loop: build → flash → serial log → **classify** below → apply fix / layer skill → retest PASS.
> AI worksheet: [roles §5](../quick-start/roles.en.md#5-调试与维护).

## Core principles

1. Compile success ≠ runtime success; behavior changes need on-device logs.
2. **Classify before coding**: match log keywords to Bug class, then solution table.
3. Own layer first; drill down only with return codes/logs/tests.
4. Re-run loop until PASS/FAIL criteria met.

Serial: Luna default `115200` 8N1 (`picocom -b 115200 /dev/ttyUSB0`).

## PASS/FAIL criteria

| Scenario | PASS | FAIL keywords |
|---|---|---|
| Preview | frames ≥20s | `invalid inbuf` / `QBUF fail` / select timeout / sensor power cycling |
| Record | file grows, clean stop | live config reload, write fail |
| Crash | no signal | `signal=11` / `fault_addr=` |
| Boot | main up, devices present | panic, insmod fail, wrong main name |
| Stream | RTSP/nebula ≥20s stable | drop, port fail, encode fail |

---

## Bug taxonomy (check logs here first)

| Class | Typical log / symptom | Primary skill | Secondary |
|---|---|---|---|
| **A Crash** | `signal=11`, `fault_addr=`, segfault, backtrace | layer from stack | [celestia](celestia.en.md) |
| **B Preview/capture/encode** | `invalid inbuf`, `QBUF fail`, `select timeout`, `num_slices`, no frames | [celestia-hardware](celestia-hardware.en.md) / [celestia-framework](celestia-framework.en.md) | — |
| **C Memory/DMA/OOM** | ion/dma alloc fail, `QBUF`+meminfo OK, OOM, corruption | this doc “Memory/DMA” | [celestia-project](celestia-project.en.md), [celestia-third-party](celestia-third-party.en.md) |
| **D Boot/driver/board** | missing `/dev/*`, insmod fail, wrong main, bad overlay, `unknown-block`, burn `Page more than end page` | [celestia-project](celestia-project.en.md) | [celestia-hardware](celestia-hardware.en.md), [celestia-third-party](celestia-third-party.en.md) |
| **E App/GUI/SO** | `dlopen` fail, not on desktop, blank icon, IOTCamera instant exit | [celestia-application](celestia-application.en.md) | — |
| **F Build/stale/config** | source change ignored, `Failed to get … module: -1`, wrong config.h | [celestia-scripts](celestia-scripts.en.md) | [celestia-hardware](celestia-hardware.en.md), [celestia-project](celestia-project.en.md) |
| **G Stream/network/cloud** | no RTSP/nebula, cloud fail, `stream_reg_client` fail | [celestia-framework](celestia-framework.en.md) | [celestia-application](celestia-application.en.md) |

Unsure → [celestia](celestia.en.md) five-layer route, then back here.

---

## A. Crash / hang

| Log / symptom | Root cause | Fix | Skill |
|---|---|---|---|
| AI on → `signal=11 fault_addr=0x90` | `camera_manager` close vs `ai_enable` race | lifecycle mutex; local device ptr on ai path | framework / hardware |
| Any `signal=11` + backtrace | null deref / UAF / OOB | stack → layer → handle lifecycle/locks | [celestia](celestia.en.md) → layer |
| `signal=11` in `free`/`malloc`, `fault_addr` ~ `0xffffffff` | uclibc `free()` reads chunk header at mem-8: small integer passed to `free` — often struct layout stale (class F) or earlier heap corruption | check `r0`, `tid`, `stack[]` | follow stack |
| Hang after capture/record | teardown power-off or live reload clears priv | symmetric teardown; no live `load_config` | framework / hardware |
| watchdog / reset | driver deadlock or kernel oops | `dmesg`; class D ko first | project / hardware |

**Evidence**: full backtrace + 50 lines before + product + steps.

---

## B. Preview / capture / encode

| Log / symptom | Root cause | Fix | Skill |
|---|---|---|---|
| `invalid inbuf cap=524288` dropped | `max_frame_payload` before `codec_mode`, NV12 → 512KB | **parse `codec_mode` before payload** | framework / hardware |
| `QBUF fail` / re-entry fail | teardown broke live path | symmetric teardown; no live reload | framework / hardware |
| CHN0 `select timeout` | `num_slices≥2` but VENC not bound (slices no YUV to userspace) | AV100_V2 bind + `venc_get_stream`; else `num_slices=1` | hardware |
| No sensor / no `/dev/video0` | ko or HAL not open | class D insmod; `hw_get_module` | project → hardware |
| abnormal `vi 3dnr memsize` | 3DNR not clamped | clamp + `HAL_CAMERA_VI_3DNR_KB` | hardware |
| corruption, process alive | DMA/frame/format | class C; check WxH/stride | debug memory |
| stream config ignored | changed `channels/streams` | only `app_stream_usage` (framework §2.5.1) | application + framework |

More: [hardware FAQ](celestia-hardware.en.md#常见问题), [application FAQ](celestia-application.en.md#常见问题).

---

## C. Memory / DMA / OOM (LUNA)

> Numbers below; full write-up `doc/WEARABLE_MEMORY_FOOTPRINT.md`.

| Log / symptom | Branch | Fix |
|---|---|---|
| dmesg ion/dma/QBUF/VENC alloc fail, meminfo OK | **DMA pool low** | A: raise DTS `RESERVED_MEMORY_SIZE` (board+patch); B: lower `max_buffers`/resolution |
| OOM / malloc fail / low MemFree, no ion errors | **MemTotal low** | C: `CONFIG_FW_MULTIMEDIA_MEDIA_PLAYER=n`, curl mbedTLS; D: `cirbuf_size` (not for DMA savings) |
| preview OOM + encode hang | pools trade off | identify pool first | |

Primary: this section; DTS → [celestia-project](celestia-project.en.md); trim libs → [celestia-third-party](celestia-third-party.en.md).

### Memory model (64MB split)

| Budget | Size | Use | Tune | meminfo |
|------|------|------|------|:---:|
| DMA/ION reserved | **20MB** (`0x1400000`) | VI/VENC/JPEG/display | DTS `RESERVED_MEMORY_SIZE` | ❌ |
| Linux MemTotal | **~25MB** | processes + cirbuf | config.mk / `fw_video_config.json` | ✅ |

Sizes: `0x1980000`=25.5M · `0x1A00000`=26M · `0x2200000`=34M · `0x2400000`=36M.
Paths (**board + patch both**):

- `project/anypi_luna/.../board/PDB_MBD_KM01W_ACTCAM/os/kernel/.../anycloud_km01a.dtsi`
- `project/anypi_luna/.../patch/os/kernel/.../anycloud_km01a.dtsi`

`fw_video_config.json` in **board rootfs overlay** (not `project/*/config/`).

---

## D. Boot / driver / board

| Log / symptom | Root cause | Fix | Skill |
|---|---|---|---|
| missing `/dev/video0` etc. | insmod order/path | `app_01_start.sh` / `app.sh` + `lsmod`/`dmesg` | project |
| wrong main on boot | script ≠ `CONFIG_APP_MAIN_TARGET` | align `<suffix>_main` | project |
| overlay config dead | edited vendor tree | only `board/<BOARD>/` | project |
| no serial / no boot | bootloader/kernel/DTB | flash partitions, DTB vs BOARD | project |
| `unknown-block(31,6)` / no root | burn/env/DTS `mtdparts` mismatch | align table; reflash ENV+DTB | project §partitions |
| burn `Page more than end page` (APP/ROOTFS) | image > partition; APP FFmpeg bloat | `DECODE_ONLY`+`LINK_MODE=decode` or SD `install/` | project / third-party |
| SDK vs overlay dir mismatch | wrong `CONFIG_SOC_SDK_VERSION` | align version dir | project |

See [project FAQ](celestia-project.en.md#常见问题) / [partition tuning](celestia-project.en.md#anyka-burntool-分区调优nova-16mb-spl-fastsys).

---

## E. App / GUI / SO

| Log / symptom | Root cause | Fix | Skill |
|---|---|---|---|
| `dlopen` / missing entry | `--gc-sections` | visibility + keep-alive | application |
| not on desktop | `installed_apps` / path | register + runtime path | application |
| `so_path` fail | source path not runtime | AV100→`/usr/lib`; AV200→`/mnt/afht` | application |
| Luna blank icon | GPU draw fail, no SW fallback | re-queue SW (id=1) on `ret<0` | application / hardware |
| IOTCamera exits at start | `main` returns without GUI | SO `module_start` blocks | application |
| call no video / `VIDEO_DECODER_SW_STUB` | `FFMPEG_LINK_MODE=none` | Nova `decode` + `DECODE_ONLY`; `-t`/`-f`/`-p` | framework / third-party |

See [application FAQ](celestia-application.en.md#常见问题).

---

## F. Build / stale / config

| Log / symptom | Root cause | Fix | Skill |
|---|---|---|---|
| source change not on board | stale `out/` / old headers | `./build.sh -c`; `--product-name` | scripts |
| random crash / SIGSEGV in `free` after private header layout change | **no header deps** in `%.o: %.c` | `-MMD -MP` + `-include $(OBJS:.o=.d)`; delete stale `.o` | scripts + layer |
| confirm stale? | — | **timestamps** on `out/<product>/build/**/**/*.o`; batch older than edit ⇒ stale; uniform batch + `.d` ⇒ exclude stale | scripts |
| `Failed to get video hardware module: -1` | stale `hardware.o` | config.h dep; `-c -h` | hardware + scripts |
| `config.h` wrong | config.mk / product | `project/<p>/config.mk` | project |
| link undefined | third_party not built | `-t` then upper layers | scripts / third-party |
| toolchain error | wrong product toolchain | AV200 riscv64 / AV100_V2 arm | project / scripts |

---

## G. Stream / network / cloud

| Log / symptom | Root cause | Fix | Skill |
|---|---|---|---|
| no RTSP | no `stream_reg_client` / RTSP off / encode fail | enable switch; register main; fix class B first | framework + application |
| nebula fail | config/switch/network | `CONFIG_APP_NEBULA`, gateway open | framework |
| cloud stream fail | SO not loaded / network | `settings.json`/`load_so_module`; no `module_exit` | application |
| no `wlan0` on Luna | interface `vnet0` | use `vnet0`; `wifi_driver.sh` | project |

---

## Troubleshooting flow (for AI)

```
1. Collect: product + steps + key log (with context)
2. Hit Bug taxonomy keywords A–G
3. Open solution table; apply known pattern
4. Else primary skill, own layer; evidence → drill down
5. After fix: build.sh flag → flash → serial PASS/FAIL
6. Fill roles debug table: symptom / class / evidence / cause / fix / verify
```

## After locating the layer

Per [celestia](celestia.en.md) “this layer → contract → (evidence) lower”; return here until PASS.