<p align="right"><b>English</b> | <a href="conventions.md">中文</a></p>

# Conventions and Standards

> **Document role**: This is Celestia’s **unified convention for documentation and code**. When writing docs, code, or changing configuration, follow this document.
> Terminology is defined solely in the [Glossary and naming reference](../glossary.en.md); layering and contracts are in
> [Architecture overview](../architecture/overview.en.md).

---

## 1. Source-of-truth Priority

When documentation conflicts with code, trust in this order:

```text
tests/  ›  *.c  ›  *.h  ›  Makefile / build.sh / config.mk  ›  docs/
```

After code changes, update docs; the reverse does not hold — do not change code to match outdated docs.

## 2. Single Entry Point for Product Differences

| Convention | Description |
|:-----|:-----|
| Product differences live only in `project/<product>/config.mk` | Generates `out/<product>/build/project/include/config.h` |
| No `#ifdef CONFIG_SOC_*` product branches in source | Chip differences belong in `hardware/**/hal/<SoC>/` |
| Runtime differences in board-level `settings.json` | Build-time switches define capability; `settings.json` defines how a given run behaves |

## 3. Cross-layer Access

| Convention | Description |
|:-----|:-----|
| Use only the other layer’s public `include/` headers | Do not include private implementation directories across layers |
| Dependency direction ① → ② → ③ → ⑤; ①②③ → ④ | Any upward dependency is a design violation |
| Self-check command | See [Architecture overview §10](../architecture/overview.en.md#10-architecture-constraint-self-check) |

## 4. Form-factor Descriptions

| Convention | Description |
|:-----|:-----|
| Always describe product form with form-factor IDs | `IOTCamera` / `AI_GLASSES` / `TABLET` / `WEARABLE` |
| Form-factor IDs: ALL CAPS + backticks | As adjectives: lowercase with hyphen: tablet-class, wearable-class |
| Do not use colloquial adjectives for form | Do not write “with screen / headless / large screen / small screen” |
| Camera is a baseline capability | Form differs in where video goes, not whether a camera exists |

Definitions and criteria: [Product form factors](../architecture/form-factor.en.md).

## 5. Naming

| Convention | Description |
|:-----|:-----|
| Product / board / SoC names must match code exactly | Scripts build paths from them; typos break the build |
| Config macros `CONFIG_<domain>_<object>`, values only `y` / `n` | See [glossary §11](../glossary.en.md#11-identifier-naming-conventions) |
| Paths in backticks with trailing `/`; functions with `()` | e.g. `` `application/app_main/main.c` ``, `` `module_start()` `` |

## 6. Commits

| Convention | Description |
|:-----|:-----|
| DCO sign-off | `git commit -s`; see [CONTRIBUTING.md](../CONTRIBUTING.md) |
| One branch, paired commits when a change spans multiple directories | Avoid half-merged states that fail to build |

## 7. Writing Documentation

| Convention | Description |
|:-----|:-----|
| Record only what this project means and where it lives in code | No etymology, literal translations, or generic industry definitions |
| Every term needs at least one code reference | Terms without code basis do not belong in the [glossary](../glossary.en.md) |
| Do not narrate change history | No “early style”, “deprecated”, “now unified”, etc. |
| New sections at the end with numbering | Avoid breaking existing anchors; use spaces instead of colons in subsection titles |

---

[← Back to docs home](../README.en.md) · [Architecture overview](../architecture/overview.en.md) ·
[Product form factors](../architecture/form-factor.en.md) · [Glossary](../glossary.en.md)