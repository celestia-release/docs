---
name: celestia-studio
description: >-
  Celestia 同步工具（scripts/studio）上的新建 tablet APP 与修改已有 APP，
  从生成或修改代码、编译、自验到推送到 AnyCloud39AV200 设备。U 盘应用用目录替换，
  用户区用 mtd_debug 写 APP 分区，不经 burntool。改 Nova 桌面应用或推送到开发板时使用。
---

<p align="right"><a href="celestia-studio.en.md">English</a> | <b>中文</b></p>

# 同步工具：新建 / 修改 APP 直到推送

工具在 `scripts/studio/`。Windows 用 `start-windows.cmd`，Linux 用 `sh start-linux.sh`，macOS 用 `sh start-mac.command`。页面是 `http://localhost:8081`。设备 IP 在页面上填写并点「连接」（studio，TCP 7777）。执行过程写在「执行状态」。

产品默认 `anypi_nova`。编译在 WSL 或 Linux 的仓库里跑 `./build.sh --product-name anypi_nova -a`，不要在 Windows 盘符上直接跑 `build.sh`。

桌面应用分两类：

| 应用 | 落点 | 推送方式 |
|------|------|----------|
| `setting`、`libstudio.so` | 只读 `/usr`（squashfs，分区名 `APP`） | 重打 `usr.sqsh4`，`mtd_debug` 写分区后重启 |
| 其它 tablet APP（如 calculator） | `/mnt/install/apps/<id>` | 替换该目录，不烧分区 |

`/mnt/install` 必须已挂载且可写。存储处在 U 盘模式时 SD 交给电脑，设备上没有这块目录，命令替换会失败。

## 新增 APP

页面选「新增 APP」，填写 id（中文或英文）和功能说明，点「执行」。这一路会自己编译并推送，没有单独的编译按钮。还没推送成功的应用留在新增列表里，标为「进行中」，可以删除。从进行中接着做时，不再确认名称，也不再问继续还是删除后重来。推送成功后才出现在「修改 APP」。

1. **生成代码**。中文名先译成英文目录 id（小写字母、数字、下划线）。大模型只写 `application/packages/tablet/<id>/tablet_<id>.c`，按接入约定实现 `lv_result_t tablet_<id>_create(lv_obj_t *main)`：界面建在当前屏幕的 window 里，调用 `tablet_create_top` 得到上滑退出条，退出回调恢复桌面并删除 window，文字用 `tablet_font_get_freetype_font()` 的字体。`app.json`、`app_entry_<id>.c`、`tablet_<id>.h` 由工具按模板写入。桌面图标由启动器（`launcher/tablet_home.c`）在代码里画成渐变圆角方块加白色图形，和相机、带屏摄像机、音频算法、设置、小凯同一风格。生成的应用在 `app.json` 里写 `icon_kind`（`glyph`/`camera`/`screen_cam`/`chat`/`gamepad`/`mic`）、`icon_glyph`（`glyph` 时的图形名，如 `audio`、`image`、`bell`、`list`）、`icon_color`/`icon_color2`（渐变上/下色 `#RRGGBB`）。这几个字段由 Studio 让模型按应用名和需求从启动器支持的图形里选，并避开已被占用的图形和颜色；模型没给出合格结果就按关键词选。`app.json` 还没有这几个字段，或需求里提到“图标”时会重新选。不生成任何占位图标文件。新增时写中文名，工具先翻译出英文名和建议的目录 id 填进确认框，用户可改。工具再把 `CONFIG_APP_TABLET_<ID> = y` 写入 `project/<product>/config.mk`。tablet 的 Makefile 不含任何应用信息：它扫描带 `app.json` 的目录发现应用、按开关启用，`.so` 名取自各应用 `app.json` 的 `entry_so`，应用专属编译/链接选项放在应用自己的 `app.mk`，所以新应用不需要改公共 Makefile。`app.json` 的 `app_id` 用英文 id。字段约定见 [celestia-application](celestia-application.md)「新增 tablet App」。
2. **检查**。`app.json` 的固定字段（`app_id`、`name`、`action`、`entry_so`、`entry_symbol`、`version`）和桌面图标描述（`icon_kind`/`icon_glyph`/`icon_color`/`icon_color2`）由工具按规则写好，不交给模型。再核对根目录 `.c`、`app_entry_<id>.c` 和接入约定：没有另起一屏（`lv_obj_create(NULL)`、`lv_scr_load`）、有退出条、退出时恢复桌面、文字用 FreeType 字体、不直接用 `lv_font_montserrat_*`、没有 weak 空实现。不过则把说明和源码交回模型改一次。仍不过则不编译、不推送。
3. **编译直到通过**。最多 4 次。应用源码错误（`file:line: error`、缺少符号、重复定义）定位到文件后，连同该应用源码交回模型再改。内核 / U-Boot / OpenSBI 的构建竞争不改应用源码，只重编。Makefile 粘行由工具拆开。没有安装目录且根目录缺少 `.c` 时，让模型把源码挪到根目录再编一次。
4. **推送并显示**。除 `setting` 外打成 tar，在设备上替换 `/mnt/install/apps/<id>`（命令见下）。成功后后台执行：

```sh
(sleep 1; killall nova_main ipc_main ipc_home; sleep 1; /usr/sbin/start_ipc_main.sh --bg) >/tmp/studio_relaunch.log 2>&1 &
```

桌面会断开再连上，并扫描到新目录。启动器按目录名登记；已有同名 `app_id` 不会再加一条。

## 修改 APP

页面选「修改 APP」。列表只含已经生成完成的应用（系统自带和推送成功的后加应用，如贪吃蛇）。进行中的不在这里。写要改的行为，点「执行」。这一路同样会自己编译并推送。

## 删除 APP

页面选「删除 APP」。只能删后加应用，不能删系统自带的 `ipc_camera`、`camera`、`audio_algo`、`setting`、`volc_engine`、`studio`。删除会去掉应用源码目录和产品开关里的对应项（Makefile 按目录自动发现应用，无需回改），并在设备上删掉 `/mnt/install/apps/<id>` 和登记后再打开桌面。

1. **修改代码**（不是「生成代码」）。只改 `application/packages/tablet/<id>/` 里已有文件。不新建应用目录，不复制成另一个 `app_id`，不改 `app.json` 的 `app_id`，不往 `installed_apps` 再加一条。
2. **检查**与**编译直到通过**与新建相同。修改不强制补入口文件名；`app.json` 没有图标描述时会补上。模块名仍是原来的 `APP_TABLET_<ID>`。
3. **推送**仍写到原来的 `/mnt/install/apps/<id>`（`setting` 仍走用户区烧录）。旧目录挪到 `/mnt/install/.app_bak/<id>`。这是点开头的备份目录，启动器不扫描，不会在桌面上多出一个应用。不要把备份留在 `/mnt/install/apps/<id>.bak`，否则会被当成第二个应用写进 `settings.json`。

## 推送命令

同步工具通过设备端 studio 代理的 `FILE_PUT` + `EXEC` 在设备上执行。手工排障时用同样的命令。

### U 盘应用：替换目录

`/mnt/install` 已挂载。包在 `/tmp/sync_push.tar.gz`。

```sh
DEST=/mnt/install/apps/<id>
rm -rf /tmp/sync_pkg "$DEST.bak"
mkdir -p /tmp/sync_pkg /mnt/install/.app_bak
tar -xzf /tmp/sync_push.tar.gz -C /tmp/sync_pkg
BAK=/mnt/install/.app_bak/<id>
rm -rf "$BAK"
if [ -d "$DEST" ]; then mv "$DEST" "$BAK"; fi
mv /tmp/sync_pkg "$DEST"
chmod -R a+rX "$DEST"
```

移动失败时把 `$BAK` 移回 `$DEST`。

### 用户区：命令写 APP 分区

不使用 PC 上的 Burn Tool。`/usr` 是 squashfs，不能在运行中把 `.so` 截断写到 `/usr/lib/libstudio.so`。

1. 用镜像树里的 `mksquashfs` 把 `rootfs/usr` 打成 xz 的 `usr.sqsh4`。
2. 在设备上读分区，名字必须是 `"APP"`：

```sh
grep '"APP"' /proc/mtd
```

fastsys 量产板实测为 `/dev/mtd8`，大小 `0x008ac000`（8880K）。以 `/proc/mtd` 为准。burntool 文本里的 `0x8AE000` 比这块分区大，不能当作写入上限。

3. 镜像大于分区则停止。需要时用 `0xFF` 补到擦除块对齐，再：

```sh
mtd_debug erase /dev/mtd8 0 <分区字节数>
mtd_debug write /dev/mtd8 0 <镜像字节数> /tmp/usr.sqsh4
sync
reboot
```

设备上的工具是 `/bin/mtd_debug`，不是 `flashcp`。
