---
name: bootstrap-repo
description: Add or update a repo's .editorconfig, .gitignore, and .pre-commit-config.yaml from the bootstrap templates, merging in its existing settings, and install pre-commit as its git hook.
disable-model-invocation: true
argument-hint: "[path to target repo, default: current directory]"
---

Bring a **target repo** up to the bootstrap baseline. Target = `$ARGUMENTS` if given, else the current directory.

The **templates** are the three files at the bootstrap repo root, three levels above this skill's folder (`.claude/skills/bootstrap-repo/`). Resolve that physically, so a symlinked install still finds them:

```sh
BOOTSTRAP="$(cd -P "<this skill's base directory>/../../.." && pwd)"
```

Every merge below has the same shape: the result is the template's layout, order, and comments, with the target's existing settings carried into it so that they **still take effect** wherever the two disagree. Record every such disagreement for the report.

## 1. Preflight

Confirm the target is a git work tree (`git -C <target> rev-parse --show-toplevel`) and that `$BOOTSTRAP/.editorconfig` exists. Note uncommitted changes in the target, so your edits stay separable from them. If the target is not a git repo, ask whether to `git init` it.

Done when: both paths resolve and you know the target's starting `git status`.

## 2. `.editorconfig`

Absent → copy the template verbatim.

Present → the template in full, then every target section after it, verbatim and in original order, under a final divider styled like the template's:

```
# -----------------------------------------------------------------------------
# PROJECT-SPECIFIC SETTINGS
```

EditorConfig applies sections top to bottom with later ones winning, so this placement keeps each target key in effect for exactly the files it covered before. Keep the target's comments with their keys. The target's preamble (`root` and anything above its first section) folds into the template's preamble; a target with no `root = true` gains one, which stops parent-directory `.editorconfig` files from applying, so list it in the report.

Done when: every target section sits below the divider unchanged, and each file type the target contains has its settings that the template now changes (the keys the target left unset) recorded.

## 3. `.gitignore`

Absent → copy the template verbatim.

Present → start from the template, then put every target line not already covered by it under the template's `# Project Files` section, last in the file. Treat `foo` and `foo/` as covered when the template has `foo`. Keep the target's comment headers, blank-line grouping, and order: order matters for `!` negations. Drop headers left with no patterns.

Then list tracked files the new patterns now ignore:

```sh
git -C <target> ls-files --cached --ignored --exclude-standard
```

Tracked files stay tracked, but new siblings will be silently ignored (common hits: `.vscode/`, Go `vendor/`, `*.patch`, `*.zip` test fixtures). Show the list and ask the user, per path, whether to re-include it. A negation cannot re-include a file under an excluded directory, so re-include the directory itself (`!.vscode/`).

Done when: every original pattern is either in the template or under `# Project Files`, and every newly ignored tracked path has a user decision.

## 4. `.pre-commit-config.yaml`

Detect the target's stacks and pick hooks from [HOOKS.md](HOOKS.md). Read it in full; it holds the detection signals and the hook blocks.

Build the config:

- Start from the template's repos, then append each detected stack's block.
- Merge the target's existing config in, keeping every target entry as it is:
  - Template repo URL the target lists once → add the template hook ids that entry lacks.
  - Listed more than once (deliberate: same hook, different args or files) → add the missing ids to the first entry only.
  - Same hook id in both → keep the target's hook. Add a template arg only when the target sets no value for that flag (compare the part before `=`), so `--fix=auto` stays and never gains a second `--fix=lf`.
  - Keep `repo: local`, `repo: meta`, and top-level keys (`exclude`, `default_language_version`, `ci`, etc.) exactly as the target has them.

Then bring every `rev`, the target's own repos included, to latest and validate:

```sh
cd <target> && pre-commit autoupdate && pre-commit validate-config
```

If `pre-commit` is missing, use `uvx pre-commit` in its place. A repo that `autoupdate` cannot update goes in the report with its error.

Done when: every matched HOOKS.md row has its block, every original hook survives, `autoupdate` ran, and `validate-config` passes.

## 5. Install and check

```sh
cd <target> && pre-commit install
```

If it refuses because `core.hooksPath` is set (husky, lefthook, or similar), stop and ask the user how to integrate. Any pre-existing `.git/hooks/pre-commit` is moved to `pre-commit.legacy` and still runs; mention it.

Then run every hook once in a **throwaway worktree**, so fixers rewrite only the copy and the target's files stay untouched. Switching the copy to an unborn orphan branch makes every file a new staged addition, so staged-only hooks (`gitleaks`, `check-added-large-files`) scan the whole tree and the new config files are checked too:

```sh
SCRATCH="$(mktemp -d)/bootstrap-check"
git -C <target> worktree add --detach "$SCRATCH" HEAD
cp <target>/.editorconfig <target>/.gitignore <target>/.pre-commit-config.yaml "$SCRATCH"/
cd "$SCRATCH" && git checkout -q --orphan bootstrap-check && git add -A
pre-commit run --all-files > "$SCRATCH.log" 2>&1
git -C "$SCRATCH" diff --stat           # files the fixers would rewrite
git -C <target> worktree remove --force "$SCRATCH"
```

The check reflects the target's `HEAD` plus the three new files; say so when the target had uncommitted changes. A target with no commits yet has no `HEAD` to check out: run the same sequence in a scratch `git init` repo holding a copy of the target's files (everything but `.git`).

Done when: the hook is installed, the per-hook outcome and the would-rewrite file list are captured, and the worktree is removed (`git -C <target> worktree list` shows only the target).

## 6. Report

Leave everything uncommitted. Report:

- Each of the three files: created or merged, with every disagreement where the target's value was kept and every template setting the target now picks up.
- Stacks detected → hooks added, plus suggested hooks from HOOKS.md that matched but were left out.
- `rev` changes from `autoupdate`.
- Files the fixers would rewrite, from the worktree diff. The hook fixes each one the next time it is committed; `pre-commit run --all-files` in the target fixes them all at once.
- Hooks that failed, split into "missing tool on PATH" (name the install command) and "real findings" (count and a sample).
