---
name: celestia-studio
description: >-
  Celestia sync tool (scripts/studio) for creating and editing tablet APPs through push to
  AnyCloud39AV200 devices: generate or modify code, build, self-check, push. USB apps use
  directory replace; user area uses mtd_debug on APP partition, not burntool. Use when changing
  Nova desktop apps or pushing to a dev board.
---

<p align="right"><b>English</b> | <a href="celestia-studio.md">中文</a></p>

# Sync tool: create / edit APP through push

Tool lives in `scripts/studio/`. Windows: `start-windows.cmd`; Linux: `sh start-linux.sh`; macOS: `sh start-mac.command`. UI: `http://localhost:8081`. Enter device IP and Connect (studio, TCP 7777). Progress appears under “Execution status”.

Default product `anypi_nova`. Build in WSL/Linux: `./build.sh --product-name anypi_nova -a` — do not run `build.sh` from a Windows drive letter.

Desktop apps fall in two classes:

| App | Location | Push method |
|------|------|----------|
| `setting`, `libstudio.so` | Read-only `/usr` (squashfs, partition name `APP`) | Rebuild `usr.sqsh4`, `mtd_debug` write partition, reboot |
| Other tablet APPs (e.g. calculator) | `/mnt/install/apps/<id>` | Replace directory, no partition burn |

`/mnt/install` must be mounted writable. In USB storage mode the SD is handed to the PC — no mount on device; directory replace fails.

## New APP

Page: “New APP”, enter id (Chinese or English) and feature description, Execute. This path builds and pushes automatically (no separate build button). Apps not yet pushed stay in the new list as “in progress” and can be deleted. Resuming in-progress skips name confirm and “continue vs delete and restart”. After successful push, the app appears under “Edit APP”.

1. **Generate code**. Chinese names are translated to an English directory id (lowercase letters, digits, underscore). The model writes only `application/packages/tablet/<id>/tablet_<id>.c`, implementing `lv_result_t tablet_<id>_create(lv_obj_t *main)` per integration rules: UI in the current screen window, `tablet_create_top` for swipe-to-exit bar, exit restores desktop and deletes window, text via `tablet_font_get_freetype_font()`. `app.json`, `app_entry_<id>.c`, `tablet_<id>.h` are templated by the tool. Launcher draws icons as gradient rounded squares + white glyph (same style as camera, screen cam, audio algo, settings, etc.). Generated `app.json` sets `icon_kind` (`glyph`/`camera`/`screen_cam`/`chat`/`gamepad`/`mic`), `icon_glyph` (for `glyph`, e.g. `audio`, `image`, `bell`, `list`), `icon_color`/`icon_color2` (`#RRGGBB` gradient). Studio picks from launcher-supported glyphs/colors and avoids collisions; if fields missing or user asks for “icon”, re-pick. No placeholder icon files. Chinese input → English name + suggested id in confirm box (editable). Tool writes `CONFIG_APP_TABLET_<ID> = y` to `project/<product>/config.mk`. Tablet Makefile discovers apps via `app.json` and switches; `.so` name from `entry_so`; per-app flags in `app.mk` — no shared Makefile edit. `app_id` in `app.json` is the English id. Field rules: [celestia-application](celestia-application.en.md) “Add tablet App”.
2. **Check**. Fixed `app.json` fields and icon metadata are rule-filled, not model-filled. Verify root `.c`, `app_entry_<id>.c`, integration: no extra screen (`lv_obj_create(NULL)`, `lv_scr_load`), exit bar present, exit restores desktop, FreeType fonts, no `lv_font_montserrat_*`, no weak empty stubs. Fail → one model retry. Still fail → no build, no push.
3. **Build until pass**. Max 4 rounds. App source errors (`file:line: error`, missing/duplicate symbols) → model fix with sources. Kernel/U-Boot/OpenSBI races → rebuild only, no app source change. Makefile glued lines → tool splits. Missing install dir and no root `.c` → model moves sources to root, rebuild.
4. **Push and show**. Except `setting`, tar and replace `/mnt/install/apps/<id>` (commands below). On success background:

```sh
(sleep 1; killall nova_main ipc_main ipc_home; sleep 1; /usr/sbin/start_ipc_main.sh --bg) >/tmp/studio_relaunch.log 2>&1 &
```

Desktop reconnects and scans new directory. Launcher registers by directory name; duplicate `app_id` is not added again.

## Edit APP

Page: “Edit APP”. List only completed apps (built-in + pushed extras). In-progress not listed. Describe behavior change, Execute — builds and pushes automatically.

## Delete APP

Page: “Delete APP”. Only user-added apps; cannot delete built-in `ipc_camera`, `camera`, `audio_algo`, `setting`, `volc_engine`, `studio`. Removes source dir and product switch (Makefile auto-discovers, no Makefile revert), deletes `/mnt/install/apps/<id>` and registry entry, then relaunches desktop.

1. **Edit code** (not “generate”). Only existing files under `application/packages/tablet/<id>/`. No new app dir, no duplicate `app_id`, no extra `installed_apps` row.
2. **Check** and **build until pass** same as create. Edit does not force entry file rename; fills icon metadata if missing. Module name stays `APP_TABLET_<ID>`.
3. **Push** still to `/mnt/install/apps/<id>` (`setting` still user-area burn). Old dir moved to `/mnt/install/.app_bak/<id>`. Dot-prefixed backup — launcher ignores. Do not use `/mnt/install/apps/<id>.bak` (second desktop app).

## Push commands

Sync tool uses device studio proxy `FILE_PUT` + `EXEC`. Manual debug uses the same.

### USB app: replace directory

`/mnt/install` mounted. Package at `/tmp/sync_push.tar.gz`.

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

On move failure, restore `$BAK` → `$DEST`.

### User area: command-write APP partition

No PC Burn Tool. `/usr` is squashfs — cannot truncate-write `/usr/lib/libstudio.so` live.

1. Build xz `usr.sqsh4` from image tree `rootfs/usr` with `mksquashfs`.
2. On device, partition name must be `"APP"`:

```sh
grep '"APP"' /proc/mtd
```

fastsys production boards: `/dev/mtd8`, size `0x008ac000` (8880K). Trust `/proc/mtd`. Burntool text `0x8AE000` is larger than real partition — not write limit.

3. Stop if image > partition. Pad with `0xFF` to erase alignment if needed, then:

```sh
mtd_debug erase /dev/mtd8 0 <partition bytes>
mtd_debug write /dev/mtd8 0 <image bytes> /tmp/usr.sqsh4
sync
reboot
```

On-device tool is `/bin/mtd_debug`, not `flashcp`.