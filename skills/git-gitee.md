---
name: git-gitee
description: >-
  Celestia 多仓（repo 管理）在 GitHub 上的提交与同步：远程为 github.com/celestia-release，
  默认分支 main。覆盖 manifest / application / docs / framework / hardware / project /
  scripts / system/core / third_party 的 status、pull、push。凭据用 GitHub 个人访问令牌
  或 gh auth，不要把令牌写进仓库。
---

# Git 提交与同步（Celestia 多仓，GitHub）

<p align="right"><a href="git-gitee.en.md">English</a> | <b>中文</b></p>

Celestia 是 `.repo` 管理的多仓项目，工作区根目录 **不是** git 仓库。本 GitHub 发布树里，各子仓远程为
`https://github.com/celestia-release/<repo>.git`，默认分支是 **`main`**。特性开发在特性分支上进行，
完成后合并回 `main`。

文件名仍是 `git-gitee.md`，因为技能索引沿用这个路径。这里的远程和分支以 GitHub / `main` 为准。

Gitee 只做后续镜像。发布源是 `https://github.com/celestia-release`。镜像建好之后，把 GitHub 的 `main` 推到 Gitee，不要在 Gitee 上另起提交，也不要把 manifest 的默认 remote 改回 Gitee。

## 子仓清单

| 目录 | GitHub 仓库 |
|------|-------------|
| `manifest` | manifest |
| `application` | application |
| `docs` | docs |
| `framework` | framework |
| `hardware` | hardware |
| `project` | project |
| `scripts` | scripts |
| `system/core` | core |
| `third_party` | third_party（大仓，git log/status 慢） |

`SUBREPOS="manifest application docs framework hardware project scripts system/core third_party"`

## 账号与凭据

用 GitHub 个人访问令牌或 `gh auth login`。令牌只放在本机凭据存储里，**不要写进仓库、不要写进脚本**。

```bash
gh auth login
# 或：git 询问凭据时，用户名填 GitHub 用户名，密码填 personal access token
```

> 在 Windows/Linux 跨文件系统挂载环境下，git 可能报 *dubious ownership*，可用：
> ```bash
> git config --global --add safe.directory '*'
> ```

## 常用操作

### 查看所有仓未推送状态
```bash
for d in $SUBREPOS; do
  echo "== $d =="
  git -C "$d" status -sb | head -1
  echo "  unpushed: $(git -C "$d" rev-list --count @{u}..HEAD 2>/dev/null)"
done
```
> third_party 慢，单独用 `git -C third_party for-each-ref \
> --format='%(refname:short) %(upstream:track)' refs/heads` 判断是否 ahead。

### 一键 push 所有有未推送提交的仓
```bash
for d in $SUBREPOS; do
  n=$(git -C "$d" rev-list --count @{u}..HEAD 2>/dev/null)
  [ "${n:-0}" -gt 0 ] && { echo ">> push $d ($n)"; git -C "$d" push; }
done
```

### 一键 pull（rebase）所有仓
```bash
for d in $SUBREPOS; do echo "== pull $d =="; git -C "$d" pull --rebase; done
```

### 单仓提交并推送
```bash
git -C <dir> add -A
git -C <dir> commit -m "type: short summary"   # see 提交信息格式 below
git -C <dir> push
```

## Merge 约定

- 特性开发在特性分支上完成，合并到 `main` 前先 `git -C <dir> pull --rebase`。
- 合并：`git -C <dir> checkout main && git -C <dir> merge --no-ff <feature-branch>`，
  冲突按层归属（见 `celestia` 路由 skill）解决后再 push。
- 跨仓改动需保持各仓提交原子、消息前缀一致（feat/fix/refactor/chore/docs…），并遵循
  上面的提交信息格式（英文 summary + bullet 详情，一仓一提交）。

## 提交信息格式（Commit Message Format）

每个子仓的集成分支提交使用**英文**，单行 summary + 空行 + bullet 详情。summary 用
`<type>: <概述>` 前缀（feat / fix / refactor / chore / docs …），概述列出该提交涉及的
主要改动点；bullet 逐项说明每类改动的来龙去脉（为什么改、改了什么、影响范围）。
**一个子仓一个提交**（squash 合并），不要带 `Co-Authored-By` 尾注，不要写 "merge"。

模板：

```
<type>: <short summary of the change set>

- <what changed and why, one bullet per logical change>
- <mention renames / config switches / gating macros explicitly>
- <note any cross-repo dependency, e.g. "follows system/core's rename of ...">
```

要点：
- summary 一行，概括该仓本次提交的全部改动；多类改动用 ` + ` 串联。
- bullet 用 `- `，每条聚焦一个逻辑改动，说清因果（避免只罗列文件名）。
- 涉及头文件重命名 / 宏开关 / gating 变更时必须显式写出旧→新。
- 跨仓改动在 bullet 中点名依赖（如 "follows system/core's rename"）。

## 文件权限（可执行位）陷阱

`repo sync` / fresh checkout 后，`.sh` 脚本丢可执行位会导致 `Permission denied`（典型：
`./scripts/build_kernel.sh: Permission denied`）。常见原因：**重命名/拷贝提交时丢了
+x**——`git mv` 保留模式，但先 `cp`（未 `-p`）/Windows 文件系统再 `git add` 会把
`100755` 降成 `100644`（Windows NTFS 无 unix 模式，git 取 index 记录的位）。

排查（列出所有应可执行却是 `100644` 的 .sh）：
```bash
git -C <dir> ls-files -s | grep '\.sh' | grep '100644'
```

修复（直接改 git index，文件系统 chmod 在 Windows 不生效）：
```bash
git -C <dir> update-index --chmod=+x <file>   # 逐个
# 或批量：
git -C <dir> ls-files -s | grep '\.sh' | grep '100644' | awk '{print $4}' \
  | while read f; do git -C <dir> update-index --chmod=+x "$f"; done
git -C <dir> commit -m "fix: restore exec mode on scripts"
```

要点：
- 凡是用 `cp` / 跨目录搬运 / 跨分支捡文件方式纳入 git 的脚本，提交前都务必
  `git ls-files -s | grep '\.sh' | grep 100644` 核对可执行位，发现丢失用
  `git update-index --chmod=+x` 补回（`git mv` 保留模式，无需此举）。
- 判定规则只看"该文件是否会被执行"：会被 `source`/`sh`/开机拉起的脚本必须 `100755`；
  `defconfig`、`*.json`、`*.ini`、`*.dts` 等数据/配置文件保持 `100644`。

## SDK（system/AnyCloud39AV\*）不在 manifest

`manifest/default.xml` 只管 8 个子仓（application/docs/framework/hardware/project/
scripts/system/core/third_party），**不含 `system/AnyCloud39AV100_V2`、
`system/AnyCloud39AV200` 等 vendor SDK**（含 `os/linux.tar.gz`、`platform/`、`scripts/`、
`rootfs/`）。

因此 fresh `repo init && repo sync` 后，`system/AnyCloud39AV*` 是空的，内核构建会报
`No rule to make target '<chip>_mini_defconfig'`（kernel 源码树缺失）。

处理：
- SDK 需单独获取（解压 vendor 包到 `system/AnyCloud39AV*/`），`repo sync` 不会拉取。
- 板级 defconfig / build_kernel.sh 等在 `project/<product>/system/AnyCloud39AV*/V*/`
  下（board overlay / patch），由 `build_soc_system.sh` 的 `cp -rf` 叠加到 out/ 内核树。
- 确认 SDK 完整：`ls system/AnyCloud39AV*/V*/os/linux.tar.gz`、
  `.../scripts/build_kernel.sh`（须有 +x，见上节）。

## 注意

- 不要把 `~/.git-credentials` 或明文密码写进任意子仓的文件。
- push 失败优先看是否被远程 reject（need pull）或凭据失效（重写凭据行）。
