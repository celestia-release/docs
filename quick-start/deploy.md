<p align="right"><a href="deploy.en.md">English</a> | <b>中文</b></p>

# 部署与上板验证

> 前置：已完成 [构建](build.md)，产物在 `out/<product>/install/`，打包后的镜像/烧录工具在 `out/<product>/burntool/`。

## 烧录上板（推荐）

构建打包后，烧录产物位于 `out/<product>/burntool/`（镜像 + 烧录工具）。

1. 进入烧录工具目录，按板型启动烧录工具（具体工具与参数以板级 `burntool/` 目录为准）：
   ```bash
   ls out/<product>/burntool/        # 查看镜像与烧录工具
   ```
2. 板子进入烧录模式（USB 烧录，板级按键/复位方式见板型说明），执行烧录。
3. 烧录完成，复位上电。

> 镜像清单（`uImage` / `fastsys_uImage` / `initfs.img` / `*.dtb` / `env_*.img` / `dss_*.bin`）见 [BSP 与产品组装](bsp-and-products.md) §6.4。

## SCP 覆盖部署（增量调试）

仅替换主程序与库，不重新烧录整片：

```bash
scp out/<product>/install/application/<main> root@<board_ip>:/usr/sbin/
scp out/<product>/install/framework/lib/*.so   root@<board_ip>:/usr/lib/
scp out/<product>/install/hardware/lib/*.so    root@<board_ip>:/usr/lib/
```

`<main>` = `CONFIG_APP_MAIN_TARGET`（`ipc_main` / `nova_main` / `luna_main` / `luna_dev_main` / `mira_main`）。

## 板上启动

### 1. 加载驱动

```bash
sh /etc/config/app_01_start.sh
# UI 模式下可能还需
sh /etc/config/app_02_load.sh
```

### 2. 运行时配置（`/etc/config/`）

| 文件 | 说明 |
|------|------|
| `settings.json` | `launch_mode`、`modules`（tablet / wearable / cloud SO） |
| `app_gui_config.ini` | LVGL 显示与输入 |
| `fw_wlan_config.ini` | WiFi |
| `hal_camera_config.ini` | 摄像头 |
| `hal_audio_config.ini` | 音频 |

**luna UI**（源文件在板级 overlay `project/anypi_luna/system/AnyCloud39AV100_V2/<SDK>/board/<BOARD>/rootfs/utils/etc/config/settings.json`）：

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

**nova / tablet UI**：

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

> 桌面 App 清单同样内嵌在 `settings.json` 的 `installed_apps` 对象中（旧独立 `installed_apps.json` 仅作兼容回退）。

### 3. 启动主进程

```bash
/usr/sbin/ipc_main                                  # anypi_nova
/usr/sbin/nova_main                                 # anypi_nova_dev
/usr/sbin/luna_main                                 # anypi_luna
/usr/sbin/luna_dev_main                             # anypi_luna_dev（`IOTCamera`）
/usr/sbin/mira_main                                 # anypi_mira_dev（`IOTCamera`，aos）
/usr/sbin/ipc_main --config /etc/config/settings.json
```

## 串口观察

USB 转串口连接板子调试串口（luna 默认 `115200` 8N1），观察 boot log 与应用日志：

```bash
# 串口终端（设备名按实际，如 /dev/ttyUSB0 或 Windows COM3）
picocom -b 115200 /dev/ttyUSB0
```

## 启动流程（代码路径）

```text
app_main_platform_startup()
  → runtime_bootstrap + app_main_load_config()
  → app_main_run()                    # 加载 SO 模块
  → app_main_start_framework_modules()
  → tablet_run() / app_gui_run()
```

## 验证

正常日志：

```text
[APP MAIN] framework modules started
[MULTIMEDIA] open ok
[NETWORK] open ok
```

| 功能 | 验证 |
|------|------|
| RTSP | VLC `rtsp://<ip>/live` |
| LCD | 观察 tablet/wearable 桌面 |
| WiFi | `ifconfig wlan0` |

## 常见故障

| 现象 | 排查 |
|------|------|
| 启动即退出 | `ldd /usr/sbin/<CONFIG_APP_MAIN_TARGET>` |
| 无画面 | `/dev/video*`、`app_02_load.sh` |
| LCD 黑屏 | `/dev/fb0`、`fb.ko` |
| multimedia 失败 | HAL SO 与驱动是否就绪 |

## 延伸阅读

- [guide/application.md](../guide/application.md) — launch_mode 与模块
- [celestia-project skill](../skills/celestia-project.md) — 板级 overlay 与串口
