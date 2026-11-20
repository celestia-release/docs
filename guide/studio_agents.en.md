<p align="right"><b>English</b> | <a href="studio_agents.md">中文</a></p>

# Sync Agent Operations

There is one on-device sync agent and one desktop sync tool, shared across all products. Per-product differences (install location, which apps go in the user partition, how the desktop process restarts) live in each product’s configuration, not duplicated agent source.

Current implementation:

| Role | Location | Description |
|:-----|:-----|:-----|
| Device agent | `application/services/studio/` | Single public header `include/studio.h`. Builds `libstudio.so`, TCP port 7777 |
| Desktop tool | `scripts/studio/` | UI at `http://localhost:8081`. Windows / Linux / macOS |
| Product differences | `project/<product>/` and `scripts/studio/products/<product>.json` | Build switches, load entries, install dirs, launch commands, process display names |

`anypi_nova` is the product currently wired to this agent. Section 2 below is Nova-specific.

## 1. What the Shared Agent Does

The agent loads as a `.so` with the main program, not a separate process. After the PC connects you can view video and metrics, run commands on the device, and push built applications.

Frame format: `[4B type LE][4B len LE][payload]`; types are defined in `application/services/studio/src/studio_internal.h`. Default port 7777.

Build steps on the PC are handled by `scripts/studio/wsl_build.sh`. Windows invokes bash via WSL; Linux and macOS use native bash. Firmware toolchain still runs on Linux / WSL; firmware cannot be built natively on Mac.

Open the UI:

| OS | Command |
|:-----|:-----|
| Windows | `scripts\studio\start-windows.cmd` (or `start.cmd`) |
| Linux | `sh scripts/studio/start-linux.sh` |
| macOS | `sh scripts/studio/start-mac.command` |

If port 8081 is already in use, the start script only opens the browser and does not restart a running service. Device IP is entered at the top of the page and you click connect; it is not baked into the start script.

After changing `server.mjs`, restart the Node process. Changes to `wsl_build.sh` or page HTML do not require a restart.

## 2. Nova (anypi_nova) Specifics

These apply only to `anypi_nova`; other products change product config, not agent source.

| Item | Nova value |
|:---|:----------|
| Build switch | `CONFIG_APP_STUDIO = y` in `project/anypi_nova/config.mk` |
| Load entry | Board `settings.json` → `modules.service_studio`, `so_path` `/usr/lib/libstudio.so` |
| Main process | `ipc_main`. UI shows this name as `nova_main` |
| User partition | `setting` and `libstudio.so` packed into read-only `usr.sqsh4`, written to partition `APP` (often `/dev/mtd8`), reboot after write |
| USB apps | Other tablet apps replace `/mnt/install/apps/<id>`. Old dir backed up to `/mnt/install/.app_bak/<id>` |
| Restart desktop after push | `killall nova_main ipc_main ipc_home`, then `/usr/sbin/start_ipc_main.sh --bg` |
| Built-in apps not removable from UI | `studio`, `ipc_camera`, `camera`, `audio_algo`, `setting`, `volc_engine` |

User partition is written with on-device `mtd_debug erase` / `mtd_debug write`, not Burn Tool, and no overwrite on a mounted `.so`. `/mnt/install` must be mounted writable; when storage is in USB mode and paths are not on the device, directory replace fails.

`application/Makefile` installs `libstudio.so` into rootfs `/usr/lib` when `CONFIG_APP_STUDIO=y`. Module build name: `APP_STUDIO`.

## 3. Where a Push Lands

“Modify APP” and “Add APP” in the UI build then push automatically.

| Content | Build module | Download target |
|:-----|:---------|:-------|
| Sync agent | `APP_STUDIO` | User `usr.sqsh4` → `APP` partition `/usr/lib/libstudio.so`, then reboot |
| `setting` | `APP_TABLET_SETTING` | Same user image, `/usr/apps/setting` |
| Other tablet apps (calculator, snake, etc.) | `APP_TABLET_<ID>` | `/mnt/install/apps/<id>`, no partition flash |
| Rootfs scripts (`start_ipc_main.sh`, mount scripts) | Not in app push | Need corresponding rootfs / user image rebuild; app push does not include them |

Example build command:

```bash
./build.sh --product-name anypi_nova -a --build-module APP_STUDIO
```

Run in Linux or WSL at the repo root. Do not run `build.sh` directly on a Windows drive letter path.

## 4. Enabling Another Product

1. Set `CONFIG_APP_STUDIO` in that product’s `config.mk` so `application/Makefile` builds and installs `libstudio.so`.
2. Add a `modules` entry in board `settings.json`: `type: so`, `enable: true`, `so_path` pointing to `/usr/lib/libstudio.so`. The loader walks by object; the key name is only this entry’s label.
3. Author `scripts/studio/products/<product>.json` with install dirs, read-only user apps, desktop display name, and launch command. `anypi_nova` maps `ipc_main` to `nova_main` and includes `start_ipc_main.sh`.
4. Do not copy `application/services/studio/` for a new product.

Model and API key setup: `scripts/studio/README.md`. `llm.json` and `device.json` stay local; do not commit.

## 5. Related Docs

| Doc | Content |
|:-----|:-----|
| [celestia-studio](../skills/celestia-studio.en.md) | Steps for coding assistants: add, change, remove tablet apps and push |
| `scripts/studio/README.md` | Startup args, UI, protocol boundaries |
| `docs/design/nova/anypi_nova_pc_studio.md` | Nova sync tool design |