<p align="right"><b>English</b> | <a href="CONTRIBUTING.md">中文</a></p>

# Contributing to Celestia

Thank you for your interest in Celestia! This document explains how to contribute and the related license and process conventions.

## License and Copyright

- Code in this repository is licensed under **Apache License 2.0** (the documentation repo uses CC BY 4.0).
- All contributions you submit will be released under the same license.

## Developer Certificate of Origin (DCO)

This project uses **DCO (Developer Certificate of Origin)** instead of a CLA. You must add a `Signed-off-by` line to each commit, indicating you have the right to submit the contribution and agree to release it under the project license:

```
git commit -s -m "your commit message"
```

This automatically appends at the end of the commit message:

```
Signed-off-by: Your Name <your.email@example.com>
```

Full DCO text: <https://developercertificate.org/>. Commits without `Signed-off-by` cannot be merged.

## Submission Workflow

1. Fork this repository on GitHub and create a feature branch from the latest default branch (`main`);
2. Keep commits focused and messages clear; follow existing code style and naming conventions;
3. Ensure changes build (see build instructions in `README.en.md`);
4. Open a Pull Request describing motivation and changes;
5. Merge after review.

## Code Conventions

- New source files **must** start with an SPDX identifier, followed by a copyright notice:
  - C/C++: `// SPDX-License-Identifier: Apache-2.0`
  - Shell/Makefile: `# SPDX-License-Identifier: Apache-2.0`
- Copyright line format: `Copyright (c) <current year> Guangzhou Anyka Microelectronics Co., Ltd.`
  (use the current year; company name as shown).
- Files must not start with a BOM; the SPDX line must be the absolute first line.
- Match surrounding comment density, naming, and indentation; naming/header rules in `skills/celestia-contributing.en.md`.

## Security and Sensitive Information

**Do not** commit keys, tokens, certificates, real Wi-Fi credentials, or internal hostnames. Cloud credentials must be injected via runtime configuration or environment variables. To report security issues, contact maintainers through private channels; do not disclose in public issues.

## Third-Party Code

Before importing third-party open-source code, confirm its license is compatible with this project and record it in `THIRD_PARTY_NOTICES.md` (or the appropriate NOTICE). Do not commit unauthorized commercial binaries or vendor-proprietary code.