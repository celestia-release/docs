<p align="right"><b>English</b> | <a href="open-source.md">中文</a></p>

# Open-source risks and what this tree already changed

This note describes what `release_github` removed compared with the Gitee release tree, and which **console actions Git history cannot replace**. It does not record passwords, keys, or tokens.

This directory is the tree prepared for GitHub. The remote is `https://github.com/celestia-release`, branch `main`. **The previous `.git` history was not copied**, so the binaries and credentials below are not in a new `main` commit. History already pushed to the old Gitee repositories is unchanged. Accounts that were exposed still have to be rotated.

A Gitee mirror comes later, and it is only a mirror: push GitHub `main` to it. Do not start a second history on Gitee, and do not point the manifest default remote back at Gitee.

## Open-source tiers

| Tier | What | License |
|------|------|---------|
| A | Sources and config in `application`, `framework`, `hardware`, `system/core`, `scripts`, and `project` | Apache-2.0 |
| B | `third_party/` | Each upstream license. See `THIRD_PARTY_NOTICES.md` |
| C | `.ko` `.so` `.a`, models, fonts, and burn images in each `project/` product overlay | Shipped in the repository. **Not** Apache-2.0. Do not relicense them as Apache-2.0 |
| D | The chip SDK at the workspace root, and closed `.a` / `.so` files under `application/cloud` | Not in this tree |

Tier C stays under `project/`. The Chinese page lists all 66 paths. The groups are kernel modules (Wi-Fi, sensor, ISP, LCD, touch), Anyka `libak_nna` / `libak_nne` / `libmpi_svp` / `libmpi_venc` / `libplat_isp_sdk`, `libVolcEngineRTCLite.so` in the nova rootfs, Mira sensor `.a` files, `model_0x0A000027_*.bin`, `ak_font_16.bin`, and `env_*.img` / DDR init binaries.

### Still absent (tier D)

| Kind | Where | Action |
|------|--------|--------|
| Alibaba Cloud static libraries | `application/cloud/aliyun/aliyun_sdk/lib/*.a` | Removed. Headers marked Apache-2.0 stay |
| Volcengine RTC prebuilt library | `application/cloud/volc_engine/lib/*.so` | Removed. The public header is marked MIT and stays. The same `.so` under `project/` is tier C and stays |
| Chip SDK | `system/AnyCloud*`, `system/Sky39AV130` | Never in git |
| Build output and local notes | `out/`, `.repo/`, `tmp/` | Not copied |

`scripts/release_src.sh` keeps tier C files under `project/`. Without `--with-cloud` it still drops `.a` / `.so` under `application/`. It fails if it finds an `enc:v1:` ciphertext or an `AKLT` access-key prefix.

## Credentials

Wi-Fi passphrases are obfuscated in `framework` with a fixed salt, product string, and IV. The prefix is `enc:v1:`. The source says this is not secure storage. Ciphertext in product configs in this tree was cleared. `app_volc_config.ini` is placeholders (`YOUR_*` and `${VOLC_*}`). The test function that contained a real credential was removed from `volc_engine_authorization.c`, and old commented assignments were removed.

Treat the values as exposed if they ever appeared in a Gitee commit, a firmware image, or a log:

1. Rotate the Volcengine AK/SK and RTC app key, revoke ASR/TTS tokens, and check room tokens and billing.
2. Rotate any Alibaba Cloud Bailian app secret and API key that was ever committed.
3. If a `settings.json` PSK belonged to a real network, change that Wi-Fi password.

Editing Git does not do those three steps.

## License boundaries that remain

First-party code is Apache-2.0. `third_party/` keeps each upstream license. The inventory is `third_party/THIRD_PARTY_NOTICES.md`.

- FFmpeg, libiconv, tslib, and AWTK are LGPL (weak copyleft). The FFmpeg configure line does not pass `--enable-gpl` or `--enable-nonfree`, so it stays LGPL, but it does pass `--enable-static`. Publishing this source tree is fine. A firmware image that statically links these libraries still owes the LGPL relink materials to the recipient. This tree does not change that build.
- AWTK is also offered under a commercial license. Confirm that license before shipping a product. The LGPL text alone is not the commercial answer.
- H.264 patents are separate from the open-source license. Commercial encode/decode needs its own review.
- Do not describe the whole tree as "license review complete".

## Before pushing

1. Finish the console rotations above.
2. `git init -b main` in each subdirectory and commit this work tree only. Do not copy `.git` from `s:\release`, or the old commits (binaries and credentials) go to GitHub with it.
3. Search once more for `enc:v1:` followed by encoded text, and for `AKLT` followed by key characters. `.ko` `.so` `.a` under `project/` are tier C and stay.
