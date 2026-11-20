<p align="right"><a href="download.en.md">English</a> | <b>中文</b></p>

# 源码下载

本文说明如何用 **repo + manifest** 从 GitHub 取得 Celestia 源码树。组织是 [celestia-release](https://github.com/celestia-release)，默认分支 **`main`**。Gitee 镜像会在这些仓库推上去之后再做，克隆以 GitHub 为准。

## 前置条件

| 项目 | 要求 |
|------|------|
| 操作系统 | Linux（推荐 Ubuntu 20.04 / 22.04）；Windows 建议 WSL2 |
| Git | 2.x |
| Python | 3.6+ |
| 磁盘空间 | 建议 ≥ 30 GB |

## 安装 repo

使用上游 [git-repo](https://gerrit.googlesource.com/git-repo)：

```bash
mkdir -p ~/bin
curl -fsSL https://storage.googleapis.com/git-repo-downloads/repo -o ~/bin/repo
chmod a+x ~/bin/repo
export PATH="${HOME}/bin:${PATH}"
```

## 初始化与同步

九个仓库需要先在 GitHub 上建好并推送（见工作区根目录 `README.md`）。然后：

```bash
mkdir -p ~/celestia && cd ~/celestia

repo init \
  -u https://github.com/celestia-release/manifest.git \
  -b main \
  -m default.xml

repo sync -c -j4
```

`repo sync` 按 manifest 拉齐全部源码目录。`manifest` 里的 `linkfile` 会在工作区根生成 `build.sh` 和 `AGENTS.md`。

## 放置芯片 SDK

以下目录不在 Git 中，需自行向安凯微取得 SDK 后解压：

```text
celestia/system/
├── core/                     ← repo 已同步
├── AnyCloud39AV200/V1.04/    ← Nova
├── AnyCloud39AV100_V2/V2.05/ ← Luna 量产（V2.04 为 luna_dev）
└── Sky39AV130/V1.01/         ← Mira（AOS）
```

厂商 `.ko`、`.so`、`.a`、模型、字库和烧录镜像也不在树里。说明见 [开源风险](../open-source.md)。

## 常用命令

```bash
repo status
repo sync -c -j4
repo forall -c 'git pull'
```

## 下一步

- [构建指南](build.md)
- [部署上板](deploy.md)
