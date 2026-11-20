<p align="right"><a href="conventions.en.md">English</a> | <b>中文</b></p>

# 约定与规范

> **文档定位**：本文是 Celestia **文档与代码的统一约定**：写文档、写代码、改配置时以本文为准。
> 术语以 [术语与命名对照表](../glossary.md) 为唯一解释源；分层与契约见
> [架构总览](../architecture/overview.md)。

---

## 1. 事实优先级

文档与代码冲突时，按下列顺序采信：

```text
tests/  ›  *.c  ›  *.h  ›  Makefile / build.sh / config.mk  ›  docs/
```

改代码后同步改文档；反过来不成立 —— 不允许为了迁就文档改代码。

## 2. 产品差异的唯一入口

| 约定 | 说明 |
|:-----|:-----|
| 产品差异只写在 `project/<product>/config.mk` | 由它生成 `out/<product>/build/project/include/config.h` |
| 源码内禁止 `#ifdef CONFIG_SOC_*` 做产品分支 | 芯片差异收敛到 `hardware/**/hal/<SoC>/` |
| 运行期差异写板级 `settings.json` | 编译期开关决定"能不能"，`settings.json` 决定"这次怎么跑" |

## 3. 跨层访问

| 约定 | 说明 |
|:-----|:-----|
| 只走对方 `include/` 公开头 | 私有实现目录不跨层 include |
| 依赖方向 ① → ② → ③ → ⑤，①②③ → ④ | 任何反向依赖都是设计违规 |
| 自检命令 | 见 [架构总览 §10](../architecture/overview.md#10-架构约束自检) |

## 4. 形态描述

| 约定 | 说明 |
|:-----|:-----|
| 一律用形态 ID 描述产品形态 | `IOTCamera` / `AI_GLASSES` / `TABLET` / `WEARABLE` |
| 形态 ID 全大写 + 反引号 | 作形容词时小写加连字符：tablet-class、wearable-class |
| 不用口语形容词代替形态 | 不写"有屏 / 无屏 / 大屏 / 小屏" |
| 摄像头是必备能力 | 形态差别在画面去向，不在有没有摄像头 |

形态定义与判据见 [产品形态](../architecture/form-factor.md)。

## 5. 命名

| 约定 | 说明 |
|:-----|:-----|
| 产品 / 板型 / SoC 名与代码完全一致 | 会被脚本拼成路径，写错即编译失败 |
| 配置宏 `CONFIG_<域>_<对象>`，取值只写 `y` / `n` | 详见 [glossary §11](../glossary.md#11-标识符命名规范) |
| 路径原样反引号，目录带 `/`；函数带 `()` | 例：`` `application/app_main/main.c` ``、`` `module_start()` `` |

## 6. 提交

| 约定 | 说明 |
|:-----|:-----|
| DCO 签名 | `git commit -s`；见 [CONTRIBUTING.md](../CONTRIBUTING.md) |
| 一个改动涉及多个目录时同分支成对提交 | 避免半边合入导致编译不过 |

## 7. 文档写作

| 约定 | 说明 |
|:-----|:-----|
| 只记录本项目实际所指与代码位置 | 不写词源、字面出处、行业通用定义 |
| 每个术语至少一条代码依据 | 没有代码依据的术语不入 [术语表](../glossary.md) |
| 不写改动过程 | 不出现"早期写法""已弃用""现已统一"这类说明 |
| 新增章节排在文末编号 | 避免打乱既有锚点；小节标题用空格代替冒号 |

---

[← 返回文档首页](../README.md) · [架构总览](../architecture/overview.md) ·
[产品形态](../architecture/form-factor.md) · [术语对照表](../glossary.md)
