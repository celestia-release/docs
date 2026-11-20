---
name: celestia-aos-port
description: >-
  Celestia Linux product port to AOS (RTOS) skill: port product lines originally
  on Linux (.so dynamic link, multi-process assumptions) to Aos user space (single
  static ELF). Covers Linux↔Aos differences and mitigations, three isolation
  principles, AOS POSIX support inventory (usable/linkable-but-treat-as-unavailable/
  unimplemented/header-with-stub-impl four categories + behavioral notes), norms for
  new POSIX compat APIs (compat shim three cases, __wrap stub replacement, SDK
  upgrade migration rules), .so→.a static link refactor, static dlopen table, HAL
  vs platform/sample, BSP kernel patches, RomFS+config.lfs, final link aos_user.ld,
  boot chain (app_entry→app_main→module_start), Linux regression and paired-merge
  discipline, Aos troubleshooting. Use for new AOS/RTOS product ports, aos compat
  layer additions, Aos link/on-board debug.
---

<p align="right"><b>English</b> | <a href="celestia-aos-port.md">中文</a></p>

# Linux product port to AOS (RTOS) skill

## When to use

- New chip / new SDK only provides Aos user space (outputs `.a` only, final single static ELF), while product code was native Linux.
- Existing SoC SDK switches from Linux to Aos version.

**Target shape**: Application and framework sources **same** as Linux products; Aos differences only in project overlay POSIX compat + chip HAL; Linux product build behavior unchanged.

---

## 1. Linux ↔ Aos differences and mitigations

Build a difference map before porting; each step implements one row’s “mitigation” column. Table verified on Sky39AV130 V1.01; re-check each row when SDK changes.

| Dimension | Linux product assumption | Aos notes and mitigations |
|------|---------------|----------------|
| Link form | Multiple `.so` dynamic link, `dlopen` at runtime | All static **single ELF**: each layer Makefile outputs `.a` only; `dlopen` replaced by static table (step 4) |
| Process model | `fork` / `exec` / `system` / `popen` available | Never use: workers → `thread_create`; shell calls → real syscalls (e.g. `system("mount …")` → `mount()` + format API) |
| Thread stack | glibc pthread, default 8MB stack | User task stacks **~64KB**: audit large locals, deep recursion; use static or heap |
| Memory | Process isolation, per-process reclaim | Single address space: buffers via static or chip DMA HAL (`ak_mem_*` class), no auto reclaim |
| Filesystem | ext4/fat RW, `truncate` works | RomFS **read-only** + config.lfs (LittleFS) + akfat (TF); `ftruncate` shrink → **failure path** (`EOPNOTSUPP` is contract, not bug) |
| Debug | gdb, strace, shell | Serial log + ucontext crash stack: log facts only (sensor_id, path, ret), `dladdr` for crash stack placeholder |
| C++ | glibc + libstdc++.so | CLIB static libstdc++.a / libgcc.a **exceptions supported**: keep exception semantics on Aos branch, do not add `-fno-exceptions` (`-fno-rtti` OK) |
| Boot | init → boot script launches `<x>_main` | Kernel loads RomFS `usr/user.strip.elf`, entry `app_entry`: BSP packaging step 6 |
| Compile ABI | per-target config possible | **All ELF flags identical** (enum ABI): all `.o`, SDK `.a`, libgcc `-mcpu/-mfloat-abi` must match product `config.mk` / SDK `rules.mk`; Aos branch in `hardware/Makefile` use `CFLAGS +=` (`:=` wipes `-mcpu`) |

---

## 2. Three principles (violation → rework)

| Principle | Explanation |
|------|------|
| **Minimal shared code change** | `ipc_cam`, framework services stay OS/product neutral; Aos diff in `hardware/**/hal/<SoC>/` and project overlay POSIX additions |
| **No OS fork in application** | Forbidden `#if aos` or OS includes in `application` / `framework`; POSIX gaps caught by compat shim headers on **include path** |
| **Isolation concentrated** | POSIX additions only `project/<p>/system/<SoC>/<V>/platform/{include/alios/compat,src/aos}`; switches only `project/<p>/config.mk`; `system/core` zero diff |

---

## 3. AOS POSIX support inventory

Implementation in SDK closed prebuilt libs (`platform/lib/alios/usyscall.a` / `uspace_posix.a` / `uspace_libc.a` / `uspace_aos.a`, symbol `nm` verified). Before port, classify each POSIX API used by product code, then handle per category.

### ✅ Direct use

| Category | APIs |
|------|------|
| Thread / sync | `pthread_create` / `join` / mutex / cond / rwlock |
| File / dir | `open/read/write/stat/opendir/readdir`, `statvfs/statfs` (verify struct layout before use) |
| Network | full socket (lwip), `poll`, `epoll_create1/ctl/wait`, `getaddrinfo` |
| Time | `usleep`, `nanosleep`, `CLOCK_REALTIME`, `CLOCK_MONOTONIC` |
| Other | `prctl(PR_SET_NAME)`, `mount/umount`, `reboot`, VFS gateway (`vfs_*`) |

**Keep source as-is**—standard headers like `#include <pthread.h>` unchanged; missing decls/macros from compat shims.

### ⚠️ Linkable but treat as unavailable

`alarm`, `setsid`, `umask`, `syscall`, `getgrnam`, `getpwnam`, `syslog`, `_isatty`, `getuid`, `readlink`, `shmget/shmat/shmdt/shmctl`, `get_nprocs*`, `get_phys_pages`, `get_avphys_pages`, `pthread_cancel`, `pthread_kill`, `pthread_atfork`, `timer_getoverrun`.

Mitigation: equivalent impl—`syslog` → `log_*`; `shmget` shared mem → in-process global/static; `pthread_cancel` → cooperative exit flag.

### ❌ Not implemented

`fork`, `exec`, `system`, `popen`.

Mitigation: **eliminate in product code**—`system("mount/mkfs …")` → real syscalls (Mira e.g.: Aos branch in `driver/blkdev/mmc.c` uses `mount/umount` + `Driver_QuickFormat`); `fork` workers → `thread_create`.

### ⚠️ Header present, stub implementation

`getifaddrs` / `freeifaddrs` / `gai_strerror` (lwIP provides `ifaddrs.h`, stub is printf).

Mitigation (priority): ① local IPv4 via chip network HAL `ioctl(SIOCGIFADDR)`; ② fallback compat `__wrap` (see norm below).

### Behavioral notes (compiles, follow contract)

| Item | Contract and mitigation |
|------|-----------|
| `mmap` prot | Impl matches **single** prot; `PROT_READ\|PROT_WRITE` combo may map write-only: do not use mmap for RW; display/regs via chip ioctl |
| `ftruncate` | Grow OK (`lseek`+write 0); shrink returns `EOPNOTSUPP`: callers accept failure, do not rely on shrink |
| `strncpy` remap | compat `string.h` maps `strncpy` → `string_strncpy` (truncate + NUL): new code needing termination uses it; do not add code relying on POSIX non-NUL guarantee |
| lwIP err codes | magic like `e == 119` from lwIP: macro before compare |
| Thread stack | ~64KB (kernel ustack patch): large buffers in `main.c` → static; new threads set stack size as needed |

---

## 4. Norm for new POSIX compat APIs

Fixed placement (source **never in SDK tree**):

```text
project/<p>/system/<SoC>/<V>/platform/
├── include/alios/compat/   ← shim headers (self-contained + include_next)
├── src/aos/                ← implementation (*.c, Makefile wildcard → libaos_posix.a)
└── lib/alios/              ← SDK closed libs (read-only, nm only)
```

- Include chain: **compat → SDK `alios/external` → toolchain libc** (shim `#include_next`).
- compat must be **first `-I`**: product `config.mk` (`CORE_COMPAT_INC`) and project `rules.mk` (`AOS_COMPAT_INC`) double guarantee.
- New headers/sources **zero build churn** (compat via first `-I`; src via wildcard)—only `__wrap` needs one line in product Makefile `USER_LINK_FLAGS`.
- SDK-native headers (e.g. `sys/{prctl,ioctl,socket}.h`) **no shim**, use directly; shim shadowing SDK headers is upgrade hazard.

### Three cases (decide → act)

| Case | How to decide | Header (compat/) | Impl (src/aos/) |
|------|----------|---------------|------------------|
| ① SDK has impl, missing decl/macro | `nm` finds symbol | same-name shim `include_next` + fill gaps (e.g. `sys/reboot.h` decl, `sys/mount.h` `MS_*`) | **none** |
| ② SDK stub, need replacement | disasm confirms printf stub | shim decl | **`__wrap_<name>`** + product Makefile `-Wl,--wrap=<name>`; **no same-name strong def** (collides with `lwip_usyscall.o`) |
| ③ SDK has nothing | `nm platform/lib/alios/*.a` none | self-contained header (no `include_next`) | define original name (`dlopen` table, `realpath`, `ftruncate`) |

### Workflow (add one API)

1. `nm` → three cases;
2. **Header**: case ①② same as standard (`include_next`), ③ self-contained; decls in header, not bare in `.c`;
3. **Impl** in `platform/src/aos/`: case ② `__wrap_<name>` + `-Wl,--wrap=<name>`; case ① no impl;
4. **Ledger**: sync `include/alios/compat/README.md` and `src/aos/README.md` rules / file list (two READMEs = API ledger);
5. **Verify**: Aos product full build + `nm -u user.elf` closed + Linux product full regression (compat headers off `KERNEL_AOS` must hit system headers, Linux unchanged).

### Prohibited

- Do not change `application` / `framework` includes or add OS macros there;
- No semantic rewrite of standard functions (`strncpy` remap is the only recorded legacy exception—do not spread);
- No per-caller “dedicated headers”—compat only fills standard gaps, product diff in HAL or product layer;
- Impl must not have global side effects (e.g. changing global log buffers for the whole product).

### SDK upgrade migration rules (e.g. V1.01 → V1.02)

Before moving `include/alios/compat/` and `src/aos/` into new overlay, **re-review each file**:

- Remove from compat headers natively provided by new SDK (shim would shadow native headers);
- For case ② stub replacement, disassemble again to confirm new lib is still stub;
- Re-`nm` symbol↔library mapping (stub ownership may change with lib version).

---

## 5. Port steps

### Step 0 Preparation

| Item | Action | Done when |
|----|------|----------|
| Toolchain | SDK bundled (e.g. `tools/toolchain/arm-alios-eabi-*.tar.gz`); product `setup_toolchain.sh` prefers `/opt/...`, else extract to `out/<p>/toolchain/`; `build.sh` sources only if `CONFIG_OS_NAME=aos` and script exists | `arm-none-eabi-gcc -dumpmachine` runs |
| SDK tree | `system/<SoC>/<V>/` read-only; confirm `platform/lib/*.a`, `platform/dep_lib/*.a`, `platform/lib/alios/uspace_posix.a`, `aos_user.ld`, `platform/sample/` exist | paths complete |
| Vendor reference | build vendor reference image with `auto_build.sh` (manual, not automated here) | `image/` has kernel, rootfs, config partitions |
| POSIX subset verify | per section 3 inventory, `nm` / disasm, produce product “usable / unavailable / none” lists | documented in project |

### Step 1 Project skeleton

Copy layout from nearest existing Aos product, replace SoC/board item by item; **do not change `application/app_main/` or vendor tree**.

| Location | Contents |
|------|------|
| `project/<p>/config.mk` | `CONFIG_OS_NAME=aos`, `CONFIG_SOC_PLATFORM/SOC_SDK_VERSION/BOARD`, `CONFIG_APP_MAIN_TARGET=<x>_main`, `CONFIG_APP_MAIN_DYNAMIC_CORE_SO=n`, product `CONFIG_APP_PACKAGES_IPCAMERA=y` + `CONFIG_APP_RTSP=y`, GUI/cloud/player etc. `n`; in `ifeq ($(CONFIG_OS_NAME),aos)` add `CFLAGS += -DKERNEL_AOS` and `CORE_COMPAT_INC` (first `-I`) |
| `project/<p>/system/<SoC>/<V>/rules.mk` | after vendor `rules.mk`: `CROSS_COMPILE`, `BUILD_SHARED=n`, `AOS_COMPAT_INC` (first `-I`), `-DKERNEL_AOS`, chip macros (e.g. `-D__CHIP_AK3918AV130_SERIES`). `build.sh` prefers project rules.mk when present |
| `project/<p>/setup_toolchain.sh` | toolchain install for this product only |
| Board overlay `board/<Board>/` | DTB, defconfig, `resource/etc/config/` (settings.json / fw_video_config.json / ISP conf), flash layout (NOR: `root.romfs` + `config.lfs`) |

> ⚠️ Write `CONFIG_OS_NAME` as `aos` consistently (product and board level); if the board needs a separate SDK-side OS identifier, use a distinct variable name—do not collide with product-layer `CONFIG_OS_NAME`.

### Step 2 Static link refactor (.so → .a)

`CONFIG_OS_NAME` is the Makefile gate. Linux products stay `linux` and build `.so`; Aos only `$(AR) rcs` `.a`, no `-shared/-fPIC/-rpath/-soname`, install copies `.a` and headers only.

- Each layer Makefile with `CONFIG_OS_NAME` branch—verify; fill gaps (e.g. `rtsp_server`) with Aos branch;
- Only application Makefile to change: `packages/ipcamera/ipc_cam/Makefile` (Aos outputs `libipc_cam_app.a`; when `BUILD_SHARED=n` drop `-fPIC`, add `-DIPC_CAM_NEBULA_STATIC=1`);
- third_party: only required libs (cjson / iniparser / mbedtls), `CONFIG_THIRD_PARTY_*_SHARED=n`; ffmpeg / curl / openssl / GUI libs off;
- C++ libs (zxing): Aos branch keeps exceptions, do not add `-fno-exceptions`.

### Step 3 Product code POSIX cleanup

For `ipc_cam` and enabled framework modules (multimedia / network / rtsp / storage / device / notification / runtime), walk files per section 3 inventory:

| Inventory category | Action |
|----------|------|
| ✅ Direct use | source unchanged, standard headers as-is |
| ❌ fork / exec / system / popen | eliminate: real syscall or `thread_create` |
| ⚠️ stub | address via chip HAL (`SIOCGIFADDR`); fallback `__wrap` |
| ⚠️ linkable unavailable | swap: `syslog` → `log_*`, shm → static |
| Behavioral notes | large-stack audit, eliminate combined mmap prot |

Final bar: `ipc_cam` and enabled framework units do not include un-audited headers beyond `pthread.h / unistd.h / dlfcn.h / sys/socket.h / sys/ioctl.h / sys/epoll.h / sys/mman.h`, no forbidden symbols; headers only from `system/core` public API or compat shims. Verify with `nm -u` + source grep.

### Step 4 Replace dynamic load (zero change to app_main entry)

Linux products rely on `.so` plugins (`dlopen`/`dlsym` load modules from config); Aos has no dynamic linker—for **each `dlopen` call site** pick a static replacement:

| Approach | When | Action |
|------|------|------|
| Direct entry | single app, fixed load graph | link-time call exported functions, remove dlopen path |
| **Static registry** (recommended, minimal diff) | keep “load by config” structure | same-name `dlopen/dlsym/dlclose/dlerror`: `so_path` in config is lookup key only, hit returns pre-registered function table |
| Link stub | optional/off modules | feature `n` + stub symbols, callers unchanged |

With static registry (Celestia: `platform/src/aos/aos_dl.c` registers `module_init` / `module_start` → single app `ipc_cam`), three points universal in static link:

1. **Guard `--gc-sections` dropping symbols**: linker sees no edge to libs loaded via `dlopen`—link those libs with `--whole-archive`, mark export objects `used` (e.g. `s_keep_ptrs`). Loss only shows on board (`dlsym` null), no link warning;
2. **Config fields unchanged structurally**: keep `settings.json` `type=so`, `so_path`; path string demoted to lookup key, upper parser zero change;
3. **Error paths like Linux debug**: `dlerror` returns explanatory string when registry miss, preserve existing error handling.

Main program (`application/app_main/`) **zero change**—`dlopen` from compat layer, boot order same as Linux: `main` → open framework → `dlopen` + `module_init` → `module_start` blocking keepalive.

### Step 5 HAL (vs platform/sample)

- Under `hardware/**/hal/<SoC>/` (flat drivers without `hal/<SoC>/` use Aos branch in source); public `hardware/include/*.h` unchanged;
- Control flow from matching `platform/sample/*` (open → set attr → get frame/stream → release/close); do not link sample `main` / arg parsing;
- Maturity: **L0** link symbols only → **L1** partial MPI → **L2** sample order aligned → **L3** on-board verified; “links” ≠ HAL done;
- No empty stubs as HAL: framework feature-off link stubs (`*_stub.c`) are not HAL capability;
- Reuse same MPI impl when Linux SoC name matches (`ak_vi/ak_venc/ak_svp2/ak_ai/ak_aenc/ak_ao`); chip macros (`__CHIP_*_SERIES`) for struct layout; branches without headers only compile same-family API branch.

### Step 6 BSP and images

Vendor tar unchanged; diff only on staged copy under `out/` (`build_soc_system.sh` staging patches). Four kernel patch types (copy pattern for new SoC):

| Patch | Role |
|------|------|
| `*_patch_kernel_out_bin_check` | kernel `kernel.bin` under `$BUILD_SRC_DIR/alios_kernel@.../binary/` but SDK only checks `kernel/out/`: python3 block replace + copy if needed; **must `env BUILD_DIR="$SDK_BUILD"` redirect** or output lands under `build/` root |
| `*_patch_kernel_uspace_trees` | RHINO `APP_ENTRY_TASK_NAME` macro + 64KB ustack: scan all uspace copies under out, hook staging `build_kernel.sh` |
| `*_patch_kernel_host_python` | host python2 scripts → python3 |
| `*_patch_kernel_build_scripts` | remove `distclean` that wipes config |

Image rules: user ELF in RomFS `usr/user.strip.elf`; JSON / ISP conf in `config.lfs` (LittleFS); strip `etc/config` before RomFS pack; drivers built into kernel image, no rootfs `insmod`; `out/<p>/` root only `build/ install/ burntool/`, flashable files in `out/<p>/build/system/<SoC>/<V>/image/`.

### Step 7 Boot chain and on-board acceptance

Aos boot chain (unlike Linux init scripts—debug on board along this path):

```text
Kernel (RHINO) → load RomFS usr/user.strip.elf → app_entry
  → app_main main() (framework handles ready)
  → dlopen(so_path) (static table hit) → module_init → module_start (blocking keepalive: capture/record/RTSP)
```

| Level | Criteria |
|------|------|
| Full build | `./build.sh --product-name <p> --jobs N` (**not bare `-j`**; U-Boot `-j1`) zero errors |
| User ELF | `readelf -h` ELF32 / arch; `readelf -d` no `NEEDED`; `nm -u` no undefined `dlopen/getifaddrs/fork/shmget`; `module_start` type `T` |
| Linux regression | **after every shared-tree change run full Linux product build** (luna or nova)—all macro guards and compat header changes must pass this |
| On-board | 7.1 boot reaches `module_start` → 7.2/7.3 frames / audio → 7.4 RTSP → 7.5 config partition persist → 7.7 TF / 7.8 WiFi |

---

## 6. Merge discipline

1. **application and framework must merge as a pair**: `ipc_cam_rtsp.c` calls `rtsp_protocol_log_suggested_urls()` (cross-repo dep)—single-repo merge breaks all products’ link.
2. framework revert must amend / squash into original commit before merge (working-tree revert does not count); cross-layer deps get separate commits noting Linux impact.
3. Before merge: one full Linux product build + one full Aos product build each.
4. `system/core` stays zero diff; rejected changes do not land in commits.

---

## 7. Troubleshooting: common AOS issues

| Symptom | Action |
|------|------|
| kernel build `fixdep: error opening depfile` | dirty kernel increment from interrupted loop (compile may succeed): `build.sh -c -k` clean soc_platform rebuild—not a code bug |
| link `getifaddrs` duplicate definition | SDK stub in `lwip_usyscall.o`, strong symbol collides: remove strong def, use `__wrap` + `-Wl,--wrap=` |
| third_party odd behavior (e.g. mbedtls RNG) | inconsistent OS id vars: unify `CONFIG_OS_NAME` as `aos` everywhere, board must not define colliding name; rebuild |
| kernel `kernel build failed` but bin exists | SDK only checks `kernel/out/`: confirm out bin check patch + `BUILD_DIR` redirect (step 6) |
| `ftruncate` shrink fails | VFS no truncate, `EOPNOTSUPP` is contract: caller failure path |
| Aos link drops `module_init` / `module_start` | `--whole-archive` around `libipc_cam_app.a` + `s_keep_ptrs` marked `used` |
| final link relocation / ABI wrong | inconsistent flags (enum ABI): all `.o` and libgcc same `-mcpu/-mfloat-abi`; check Aos CFLAGS not wiped by `:=` |
| C++ lib missing exception symbols | Aos branch added `-fno-exceptions`: CLIB libstdc++ supports exceptions—remove it |
| parallel full build fails | use `--jobs N`, not bare `-j` (U-Boot fixed `-j1`) |

---

## 8. Quick checklist (new port product)

- [ ] Step 0: toolchain + SDK libs/sample complete + vendor reference image + POSIX three-list verify
- [ ] Step 1: config.mk / rules.mk / setup_toolchain.sh / board overlay; `CONFIG_OS_NAME` consistent everywhere
- [ ] Step 2: Aos chain outputs `.a` only; no Celestia `.so` in `install/`; C++ libs keep exceptions
- [ ] Step 3: forbidden symbols (fork / exec / system / popen / unavailable list) zeroed; stubs have replacement path
- [ ] Step 4: `nm user.elf` `module_start` type `T`; app_main no functional change
- [ ] Step 5: HAL ≥ L2 (sample order), on-board L3 tied to 7.x
- [ ] Step 6: four kernel patches; `root.romfs` + `config.lfs`; flashables in `build/system/<SoC>/<V>/image/`
- [ ] Step 7: Linux product full regression + Aos full build + `nm -u` bar
- [ ] compat/README and src/aos/README updated with new APIs
</think>
<｜tool▁calls▁begin｜><｜tool▁call▁begin｜>
Grep
</think>


<｜tool▁calls▁begin｜><｜tool▁call▁begin｜>
Read