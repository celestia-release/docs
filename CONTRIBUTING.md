<p align="right"><a href="CONTRIBUTING.en.md">English</a> | <b>中文</b></p>

# 贡献指南（Contributing to Celestia）

感谢你对 Celestia 项目的关注！本文档说明如何参与贡献及相关的许可与流程约定。

## 许可与版权

- 本仓库代码以 **Apache License 2.0** 授权（文档仓库为 CC BY 4.0）。
- 你提交的所有贡献将在相同许可下发布。

## 开发者原产地证明（DCO）

本项目采用 **DCO（Developer Certificate of Origin）** 而非 CLA。你需要为每个提交添加 `Signed-off-by` 声明，表示你有权提交该贡献并同意以项目许可发布：

```
git commit -s -m "你的提交说明"
```

这会在提交信息末尾自动追加：

```
Signed-off-by: Your Name <your.email@example.com>
```

DCO 全文见 <https://developercertificate.org/>。未带 `Signed-off-by` 的提交将无法被合并。

## 提交流程

1. Fork 本仓库并基于最新的默认分支创建特性分支；
2. 保持提交聚焦、信息清晰，遵循仓库既有的代码风格与命名约定；
3. 确保改动可通过构建（见 `README.md` 的构建说明）；
4. 提交 Pull Request / Merge Request，说明动机与改动点；
5. 通过评审后合并。

## 代码规范

- 新增源文件**必须**在文件首行加 SPDX 标识，其后接版权声明：
  - C/C++：`// SPDX-License-Identifier: Apache-2.0`
  - Shell/Makefile：`# SPDX-License-Identifier: Apache-2.0`
- 版权行格式：`Copyright (c) <当前年份> Guangzhou Anyka Microelectronics Co., Ltd.`
  （年份填当前年份，公司名称以此为准）。
- 文件不得以 BOM 起始；SPDX 行必须是文件绝对首行。
- 匹配周边代码的注释密度、命名与缩进风格；命名/头文件规范详见 `skills/celestia-contributing.md`。

## 安全与敏感信息

**严禁**提交任何密钥、令牌、证书、真实 Wi-Fi 凭据或内部主机地址。云服务凭据须通过运行时配置或环境变量注入。如发现安全问题，请通过私有渠道联系维护者，勿在公开 Issue 中披露。

## 第三方代码

引入第三方开源代码前，请确认其许可协议与本项目兼容，并在 `THIRD_PARTY_NOTICES.md`（或相应 NOTICE）中登记。请勿提交未获授权的商业二进制或厂商专有代码。
