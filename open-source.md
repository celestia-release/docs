<p align="right"><a href="open-source.en.md">English</a> | <b>中文</b></p>

# 开源风险与本树已做的处理

本文描述 `release_github` 相对 `release`（Gitee 发布树）做了哪些删除，以及 **Git 删文件不能代替的控制台操作**。这里不记录任何口令、密钥或 Token。

本目录是准备推到 GitHub 的工作树，远程是 `https://github.com/celestia-release`，分支 `main`。**没有带上原来的 `.git` 历史**。因此下面这些二进制和凭据不会进入新的 `main` 提交。Gitee 上已经推送过的历史不受影响，该轮换的账号仍要轮换。

Gitee 镜像后做，而且只做镜像：从 GitHub 的 `main` 推过去，不在 Gitee 上另起历史，也不把 manifest 的默认 remote 改回 Gitee。

## 开源分级

| 级 | 范围 | 协议 |
|----|------|------|
| A | `application`、`framework`、`hardware`、`system/core`、`scripts`、`project` 里的源码和配置 | Apache-2.0 |
| B | `third_party/` | 各上游协议，见 `THIRD_PARTY_NOTICES.md` |
| C | `project/` 各产品 overlay 里的 `.ko` `.so` `.a`、模型、字库、烧录镜像 | 随仓库分发，**不适用** Apache-2.0，不得再许可成 Apache-2.0 |
| D | 工作区根上的芯片 SDK；`application/cloud` 里的闭源 `.a` / `.so` | 不在本树 |

C 级留在 `project/`，不删除。名单与原来 `NOTICE` 一致，按产品如下。

### C 级：仍在 `project/`（66 个文件）

内核模块 `.ko`：

- `anypi_luna` V2.04 / V2.05：`aic8800_netdrv.ko`、`ak_isp.ko`、`ak_mci.ko`、`cst816x.ko`、`sensor_os05a20.ko`
- `anypi_luna_dev`：`ak_isp.ko`、`sensor_gc4053.ko`
- `anypi_nova`：`ak_venc_adapter.ko`（fast / normal）、`ts_ft5426g.ko`、`ATBM6162_wifi_sdio.ko`、`ak_lcd.ko`、`ak_vi.ko`、`cfg80211.ko`、`sensor_ov8856.ko`、`vcm_fp5510.ko`
- `anypi_nova_dev`：`ts_ft5426g.ko`、`ts_gt911.ko`、`ak_lcd.ko`、`fb.ko`、`ts_ft5426g.ko`（rootfs）

库 `.so` / `.a`：

- `anypi_nova`、`anypi_nova_dev` 的 `patch/platform/lib/`：`libak_nna`、`libak_nne`、`libmpi_svp`、`libplat_isp_sdk`（`.a` 与 `.so`）；nova 另有 `libmpi_venc`
- 两份 `libVolcEngineRTCLite.so`、两份 rootfs `libplat_isp_sdk.so`
- `anypi_mira_dev`：`libdrv_sensor_sc1346.a`、`libdrv_sensor_sc1346_new.a`

模型、字库、烧录镜像：

- `model_0x0A000027_V1.00.00_01.bin`：luna V2.04、luna V2.05、luna_dev
- `ak_font_16.bin`：luna V2.04、luna V2.05、nova、nova_dev
- `env_*.img`：luna V2.05、mira_dev、nova、nova `patch_fastsys`（burntool 与 envtool 各一份）、nova_dev（含 lcd / 32M / fastsys）
- nova：`DDR3_256MB_PHY0_PHY1_900MHZ_U_P.bin`

### 仍不入库（D 级）

| 类别 | 位置 | 处理 |
|------|------|------|
| 阿里云百炼静态库 | `application/cloud/aliyun/aliyun_sdk/lib/*.a` | 删除。头文件标明 Apache-2.0，予以保留 |
| 火山 RTC 预编译库 | `application/cloud/volc_engine/lib/*.so` | 删除。公开头标明 MIT，予以保留。`project/` 里的同名 `libVolcEngineRTCLite.so` 按 C 级保留 |
| 芯片 SDK | `system/AnyCloud*`、`system/Sky39AV130` | 本来就不在 git 里 |
| 构建产物与本地笔记 | `out/`、`.repo/`、`tmp/` | 没有拷进本目录 |

`scripts/release_src.sh` 导出时保留 `project/` 的 C 级文件。未加 `--with-cloud` 时仍去掉 `application/` 里的 `.a` / `.so`。扫到 `enc:v1:` 密文或 `AKLT` 访问密钥前缀则失败。

## 凭据

Wi-Fi 口令在 `framework` 里用固定 salt、product 和 IV 做 AES 混淆，前缀是 `enc:v1:`。源码写明这不是安全存储。本树里产品配置中的 `enc:v1:` 密文已清空，`app_volc_config.ini` 已改成 `YOUR_*` 和 `${VOLC_*}` 占位。带真实口令的测试函数已从 `volc_engine_authorization.c` 去掉，源码注释里的旧赋值也已去掉。

这些值如果曾经出现在 Gitee 提交、固件或日志里，视为已经泄露：

1. 在火山引擎轮换 AK/SK、RTC App Key，作废 ASR/TTS Token，并检查进房 Token 和账单。
2. 在阿里云百炼轮换曾经提交过的 App Secret 和 API Key。
3. 若 `settings.json` 里的 PSK 对应真实网络，轮换那些 Wi-Fi 密码。

改 Git 不能完成上面三步。

## 仍然存在的许可边界

自研代码是 Apache-2.0。`third_party/` 保留各上游协议，清单见 `third_party/THIRD_PARTY_NOTICES.md`。

- FFmpeg、libiconv、tslib、AWTK 是 LGPL（弱 copyleft）。当前 FFmpeg 配置未打开 `--enable-gpl` / `--enable-nonfree`，停在 LGPL，但是 `--enable-static`。只分发源码仓库没有问题；若固件静态链了这些库，向最终用户提供可重链接材料是 LGPL 的义务，本树没有改构建方式。
- AWTK 另有商业授权。商用出货前要确认授权，而不是只看 LGPL 文本。
- H.264 专利与开源许可是两件事，商用编解码要单独评估。
- 不能把整棵树说成「许可证已审核完成」。

## 推送之前

1. 完成上面的控制台轮换。
2. 每个子目录单独 `git init -b main`，只提交这一份工作树。不要从 `s:\release` 拷贝 `.git`，否则旧提交里的二进制和凭据会一起进 GitHub。
3. 推送前再搜一遍 `enc:v1:` 后面紧跟编码、以及 `AKLT` 后紧跟密钥字符。`project/` 里的 `.ko` `.so` `.a` 是 C 级，要留着。
