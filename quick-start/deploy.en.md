<p align="right"><b>English</b> | <a href="deploy.md">中文</a></p>

# Deploy and on-board verification

> Prerequisite: [Build](build.en.md) is done; artifacts are under `out/<product>/install/`, packaged images and burn tools under `out/<product>/burntool/`.

## Flash to board (recommended)

After build and package, flash assets are under `out/<product>/burntool/` (images + burn tool).

1. Enter the burn tool directory and start the burn tool for your board (exact tool and parameters per the board’s `burntool/` directory):
   ```bash
   ls out/<product>/burntool/        # List images and burn tool
   ```
2. Put the board in flash mode (USB flash; key/reset procedure per board docs), then run the flash.
3. When flash completes, reset and power on.

> Image list (`uImage` / `fastsys_uImage` / `initfs.img` / `*.dtb` / `env_*.img` / `dss_*.bin`) is in [BSP and product assembly](bsp-and-products.en.md) Section 6.4.

## SCP overlay deploy (incremental debug)

Replace only the main binary and libraries without reflashing the whole device:

```bash
scp out/<product>/install/application/<main> root@<board_ip>:/usr/sbin/
scp out/<product>/install/framework/lib/*.so   root@<board_ip>:/usr/lib/
scp out/<product>/install/hardware/lib/*.so    root@<board_ip>:/usr/lib/
```

`<main>` = `CONFIG_APP_MAIN_TARGET` (`ipc_main` / `nova_main` / `luna_main` / `luna_dev_main` / `mira_main`).

## On-board startup

### 1. Load drivers

```bash
sh /etc/config/app_01_start.sh
# In UI mode you may also need
sh /etc/config/app_02_load.sh
```

### 2. Runtime config (`/etc/config/`)

| File | Purpose |
|------|------|
| `settings.json` | `launch_mode`, `modules` (tablet / wearable / cloud SO) |
| `app_gui_config.ini` | LVGL display and input |
| `fw_wlan_config.ini` | WiFi |
| `hal_camera_config.ini` | Camera |
| `hal_audio_config.ini` | Audio |

**luna UI** (source on board overlay `project/anypi_luna/system/AnyCloud39AV100_V2/<SDK>/board/<BOARD>/rootfs/utils/etc/config/settings.json`):

```json
{
  "program": "",
  "launch_mode": "ui",
  "modules": {
    "gui_luna": {
      "enable": true,
      "type": "static",
      "config_path": "/etc/config/settings.json"
    }
  }
}
```

**nova / tablet UI**:

```json
{
  "launch_mode": "ui",
  "modules": {
    "gui_tablet": {
      "enable": true,
      "type": "static",
      "config_path": "/etc/config/settings.json"
    }
  }
}
```

> Desktop app list is also embedded in `settings.json` under `installed_apps` (legacy standalone `installed_apps.json` is fallback only).

### 3. Start main process

```bash
/usr/sbin/ipc_main                                  # anypi_nova
/usr/sbin/nova_main                                 # anypi_nova_dev
/usr/sbin/luna_main                                 # anypi_luna
/usr/sbin/luna_dev_main                             # anypi_luna_dev (`IOTCamera`)
/usr/sbin/mira_main                                 # anypi_mira_dev (`IOTCamera`, aos)
/usr/sbin/ipc_main --config /etc/config/settings.json
```

## Serial console

USB–serial to the board debug port (luna default `115200` 8N1); watch boot log and app logs:

```bash
# Serial terminal (device name as applicable, e.g. /dev/ttyUSB0 or Windows COM3)
picocom -b 115200 /dev/ttyUSB0
```

## Startup flow (code path)

```text
app_main_platform_startup()
  → runtime_bootstrap + app_main_load_config()
  → app_main_run()                    # Load SO modules
  → app_main_start_framework_modules()
  → tablet_run() / app_gui_run()
```

## Verification

Normal logs:

```text
[APP MAIN] framework modules started
[MULTIMEDIA] open ok
[NETWORK] open ok
```

| Feature | Check |
|------|------|
| RTSP | VLC `rtsp://<ip>/live` |
| LCD | Tablet/wearable desktop |
| WiFi | `ifconfig wlan0` |

## Common failures

| Symptom | Investigation |
|------|------|
| Exits immediately on start | `ldd /usr/sbin/<CONFIG_APP_MAIN_TARGET>` |
| No video | `/dev/video*`, `app_02_load.sh` |
| LCD black | `/dev/fb0`, `fb.ko` |
| multimedia fails | HAL SO and drivers ready? |

## Further reading

- [guide/application.en.md](../guide/application.en.md) — launch_mode and modules
- [celestia-project skill](../skills/celestia-project.en.md) — board overlay and serial