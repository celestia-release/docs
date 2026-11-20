<p align="right"><a href="studio_agents.en.md">English</a> | <b>中文</b></p>

# 同步代理操作

设备上的同步代理和电脑上的同步工具各只有一份，所有产品共用。产品之间不同的部分（装到哪里、哪些应用进用户区、桌面进程怎么重启）写在该产品自己的配置里，不复制代理源码。

当前实现：

| 角色 | 位置 | 说明 |
|:-----|:-----|:-----|
| 设备代理 | `application/services/studio/` | 唯一公开头 `include/studio.h`。编出 `libstudio.so`，TCP 7777 |
| 电脑工具 | `scripts/studio/` | 页面 `http://localhost:8081`。Windows / Linux / macOS 都能开页面 |
| 产品差异 | `project/<product>/` 与 `scripts/studio/products/<product>.json` | 编译开关、加载项、安装目录、拉起命令、进程显示名 |

`anypi_nova` 是现在接上这套代理的产品。下面第 2 节是它单独的部分。

## 1. 共用代理做什么

代理随主程序以 `.so` 加载，不单独起进程。电脑连上之后可以看画面和指标、在设备上执行命令、把编译好的应用推上去。

帧格式是 `[4B type LE][4B len LE][payload]`，类型定义在 `application/services/studio/src/studio_internal.h`。默认端口 7777。

电脑上的编译步骤由 `scripts/studio/wsl_build.sh` 完成。Windows 经 WSL 调 bash，Linux 和 macOS 直接调本机 bash。固件工具链仍在 Linux / WSL 里，Mac 本机编不了固件。

打开页面：

| 系统 | 命令 |
|:-----|:-----|
| Windows | `scripts\studio\start-windows.cmd`（或 `start.cmd`） |
| Linux | `sh scripts/studio/start-linux.sh` |
| macOS | `sh scripts/studio/start-mac.command` |

端口已经是 8081 时，启动脚本只打开浏览器，不重启已经在跑的服务。设备 IP 在页面顶部填写并点「连接」，不写进启动脚本。

改 `server.mjs` 之后要重新启动这个 Node 进程才生效。改 `wsl_build.sh` 或页面 HTML 不用重启。

## 2. Nova（anypi_nova）单独的部分

这些只属于 `anypi_nova`，换产品时改产品配置，不改代理源码。

| 项 | Nova 取值 |
|:---|:----------|
| 编译开关 | `project/anypi_nova/config.mk` 中 `CONFIG_APP_STUDIO = y` |
| 加载项 | 板级 `settings.json` 的 `modules.service_studio`，`so_path` 为 `/usr/lib/libstudio.so` |
| 主进程 | `ipc_main`。页面把这个名字显示成 `nova_main` |
| 用户区 | `setting` 和 `libstudio.so` 打进只读 `usr.sqsh4`，写入分区名 `APP`（常见为 `/dev/mtd8`），写完重启 |
| U 盘应用 | 其它 tablet 应用替换 `/mnt/install/apps/<id>`。旧目录备份到 `/mnt/install/.app_bak/<id>` |
| 推送后拉起桌面 | `killall nova_main ipc_main ipc_home`，再执行 `/usr/sbin/start_ipc_main.sh --bg` |
| 不从页面删除的内置应用 | `studio`、`ipc_camera`、`camera`、`audio_algo`、`setting`、`volc_engine` |

用户区用设备上的 `mtd_debug erase` / `mtd_debug write` 写入，不用 Burn Tool，也不往已挂载的 `.so` 上做覆盖写。`/mnt/install` 必须已挂载且可写；存储处在 U 盘模式、目录不在设备上时，目录替换会失败。

`application/Makefile` 在 `CONFIG_APP_STUDIO=y` 时把 `libstudio.so` 装进 rootfs 的 `/usr/lib`。模块编译名是 `APP_STUDIO`。

## 3. 一次推送落在哪

页面上的「修改 APP」「新增 APP」会自己编译再推送。

| 内容 | 编译模块 | 下载到 |
|:-----|:---------|:-------|
| 同步代理 | `APP_STUDIO` | 用户区 `usr.sqsh4` → `APP` 分区的 `/usr/lib/libstudio.so`，然后重启 |
| `setting` | `APP_TABLET_SETTING` | 同一张用户区镜像，`/usr/apps/setting` |
| 其它 tablet 应用（calculator、snake 等） | `APP_TABLET_<ID>` | `/mnt/install/apps/<id>`，不烧分区 |
| rootfs 脚本（`start_ipc_main.sh`、挂载脚本） | 不在应用推送里 | 要进系统需重做对应 rootfs / 用户区镜像，应用推送不会带上它们 |

编译命令形如：

```bash
./build.sh --product-name anypi_nova -a --build-module APP_STUDIO
```

在仓库所在的 Linux 或 WSL 里执行。不要在 Windows 盘符上直接跑 `build.sh`。

## 4. 换一块产品时改哪里

1. 在该产品的 `config.mk` 打开 `CONFIG_APP_STUDIO`，让 `application/Makefile` 编进并安装 `libstudio.so`。
2. 在该板 `settings.json` 的 `modules` 里加一条 `type: so`、`enable: true`，`so_path` 指向 `/usr/lib/libstudio.so`。加载器按对象遍历，键名只是这一条的名字。
3. 在 `scripts/studio/products/<product>.json` 写安装目录、哪些应用在只读用户区、桌面进程显示名和拉起命令。`anypi_nova` 的这份配置包含 `ipc_main` 显示成 `nova_main`，以及 `start_ipc_main.sh`。
4. 不要为新产品再复制一份 `application/services/studio/`。

页面上选模型和密钥的说明见 `scripts/studio/README.md`。`llm.json` 和 `device.json` 留在本机，不提交。

## 5. 相关文档

| 文档 | 内容 |
|:-----|:-----|
| [celestia-studio](../skills/celestia-studio.md) | 给编码助手的步骤：新建、修改、删除 tablet 应用并推送 |
| `scripts/studio/README.md` | 启动参数、界面和协议边界 |
| `docs/design/nova/anypi_nova_pc_studio.md` | Nova 同步工具设计 |
