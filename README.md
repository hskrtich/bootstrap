# bootstrap

Baseline repo config for starting a new repository or bringing an existing one up to the same standard.

## What's included

| File | Purpose |
| --- | --- |
| [`.editorconfig`](.editorconfig) | UTF-8, LF line endings, final newline, and per-language indentation. |
| [`.gitignore`](.gitignore) | OS, editor, archive, and common language/tool artifacts. Repo-specific entries go in the `# Project Files` section at the end. |
| [`.pre-commit-config.yaml`](.pre-commit-config.yaml) | General checks via [pre-commit](https://pre-commit.com): whitespace, end-of-file, line endings, YAML, merge conflicts, large files, private keys, and secrets (gitleaks). |

## Applying to a repo

### With Claude Code

The [`bootstrap-repo`](.claude/skills/bootstrap-repo/SKILL.md) skill applies the templates for you. It:

- Creates any of the three files that are missing.
- Merges existing `.editorconfig` and `.gitignore` files into the template layout, keeping the repo's own settings.
- Builds `.pre-commit-config.yaml` from the template plus hooks for the stacks it detects (Shell, OpenTofu/Terraform, Python, Go, JS/TS, Dockerfiles, GitHub Actions), then updates every hook to its latest version.
- Installs pre-commit as the repo's git hook and reports what the hooks would fix or flag, without changing any files.
- Leaves everything uncommitted for you to review.

To use it from any repo, link it into your personal skills folder once:

```sh
ln -s ~/repos/bootstrap/.claude/skills/bootstrap-repo ~/.claude/skills/bootstrap-repo
```

Then, in the target repo, run `/bootstrap-repo`. You can also pass a path: `/bootstrap-repo ~/repos/other-project`.

### By hand

Copy the three files into the repo, move any existing `.gitignore` entries under `# Project Files`, then:

```sh
pre-commit install
pre-commit run --all-files
```

## Working on this repo

The hooks run on this repo too:

```sh
pre-commit install
pre-commit run --all-files
```

Stack-specific hooks live in [`HOOKS.md`](.claude/skills/bootstrap-repo/HOOKS.md). Add a new stack there, with how to detect it and its hook block. Hooks every repo should get go in the root `.pre-commit-config.yaml`.
