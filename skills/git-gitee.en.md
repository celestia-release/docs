---
name: git-gitee
description: >-
  Commit and sync the Celestia multi-repo workspace on GitHub. Remotes are
  github.com/celestia-release and the default branch is main. Covers status,
  pull, and push for manifest, application, docs, framework, hardware, project,
  scripts, system/core, and third_party. Use a GitHub token or gh auth. Never
  commit the token.
---

<p align="right"><b>English</b> | <a href="git-gitee.md">中文</a></p>

# Git commit and sync (Celestia multi-repo, GitHub)

The workspace root is **not** a git repository. `repo` checks out one repository per directory. In this GitHub tree each remote is `https://github.com/celestia-release/<repo>.git` and the default branch is **`main`**. Feature work happens on a feature branch and merges back to `main`.

The file is still named `git-gitee.md` because the skill index uses that path. Remotes and the default branch here are GitHub and `main`.

Gitee is a later mirror only. The publication source is `https://github.com/celestia-release`. When the mirror exists, push GitHub `main` to it. Do not start commits on Gitee, and do not point the manifest default remote back at Gitee.

## Repositories

| Directory | GitHub repository |
|-----------|-------------------|
| `manifest` | manifest |
| `application` | application |
| `docs` | docs |
| `framework` | framework |
| `hardware` | hardware |
| `project` | project |
| `scripts` | scripts |
| `system/core` | core |
| `third_party` | third_party (large; `git status` is slow) |

`SUBREPOS="manifest application docs framework hardware project scripts system/core third_party"`

## Credentials

Use a GitHub personal access token or `gh auth login`. Keep the token in the local credential store. Do not write it into a repository or a script.

```bash
gh auth login
```

If Git asks, the username is the GitHub user and the password is the personal access token.

On a Windows/Linux mount, Git may report dubious ownership:

```bash
git config --global --add safe.directory '*'
```

## Everyday commands

Unpushed commits in every repo:

```bash
for d in $SUBREPOS; do
  echo "== $d =="
  git -C "$d" status -sb | head -1
  echo "  unpushed: $(git -C "$d" rev-list --count @{u}..HEAD 2>/dev/null)"
done
```

For `third_party`, prefer `git -C third_party for-each-ref --format='%(refname:short) %(upstream:track)' refs/heads`.

Push only repos that are ahead:

```bash
for d in $SUBREPOS; do
  n=$(git -C "$d" rev-list --count @{u}..HEAD 2>/dev/null)
  [ "${n:-0}" -gt 0 ] && { echo ">> push $d ($n)"; git -C "$d" push; }
done
```

Pull (rebase) every repo:

```bash
for d in $SUBREPOS; do echo "== pull $d =="; git -C "$d" pull --rebase; done
```

One repository:

```bash
git -C <dir> add -A
git -C <dir> commit -m "type: short summary"
git -C <dir> push
```

## Merge

- Rebase onto `main` before merging: `git -C <dir> pull --rebase`.
- Merge with `git -C <dir> checkout main && git -C <dir> merge --no-ff <feature-branch>`.
- Resolve conflicts inside the layer that owns the file (see the `celestia` routing skill), then push.
- One commit per repository. Use the same type prefix across repositories that land together (`feat` / `fix` / `refactor` / `chore` / `docs`). No `Co-Authored-By` trailer. Do not use the word "merge" in the summary.

## Commit message

Integration-branch commits are English: one summary line, a blank line, then bullets.

```text
<type>: <short summary of the change set>

- <what changed and why, one bullet per logical change>
- <mention renames / config switches / gating macros explicitly>
- <note any cross-repo dependency, e.g. "follows system/core's rename of ...">
```

The summary is one line. Join several themes with ` + `. Each bullet states cause and effect, not a file list. A renamed header, a macro, or a gating change must show old → new.

## Executable bit

After `repo sync` or a fresh checkout, a `.sh` file can lose its executable bit and `./scripts/build_kernel.sh` fails with `Permission denied`. `git mv` keeps the mode. `cp` without `-p`, or `git add` from NTFS, stores `100644` instead of `100755`.

List shell scripts that are not executable:

```bash
git -C <dir> ls-files -s | grep '\.sh' | grep '100644'
```

Fix the index (chmod on Windows does not change the git mode):

```bash
git -C <dir> update-index --chmod=+x <file>
```

Scripts that are executed or sourced must be `100755`. `defconfig`, `*.json`, `*.ini`, and `*.dts` stay `100644`.

## Chip SDK is not in the manifest

`manifest/default.xml` lists eight code repositories plus `manifest`. It does **not** list `system/AnyCloud39AV100_V2`, `system/AnyCloud39AV200`, or `system/Sky39AV130`.

After a fresh `repo sync` those SDK directories are absent, and the kernel build fails with `No rule to make target '<chip>_mini_defconfig'`.

Unpack the vendor SDK under `system/AnyCloud39AV*/` yourself. Board defconfig and `build_kernel.sh` overlays live under `project/<product>/system/...` and are copied onto the out-of-tree kernel by `build_soc_system.sh`. Check `ls system/AnyCloud39AV*/V*/os/linux.tar.gz` and that `build_kernel.sh` is mode `100755`.

## Do not commit

Do not commit `~/.git-credentials`, a personal access token, or a plaintext password. If push is rejected, pull first or refresh the token.
