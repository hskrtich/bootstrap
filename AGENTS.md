# AGENTS.md

This is a **bootstrap repo**: the shared baseline of repo-level config (`.gitignore`, `.editorconfig`, `.pre-commit-config.yaml`, and similar) applied when creating a new repository or bringing an existing one up to date.

## Templates

- The root `.editorconfig`, `.gitignore`, and `.pre-commit-config.yaml` are templates for *other* repos. Write them generic, so they fit any project that adopts them.
- Repos that already adopted a template will pull later changes, so make changes additive and backwards-compatible.
- `.gitignore` ends with an empty `# Project Files` section: the slot each adopting repo fills with its own entries. Keep it last.

## Checks

The files in this repo are validated with [pre-commit](https://pre-commit.com):

```sh
pre-commit install          # once per clone
pre-commit run --all-files  # run every hook
```

## `bootstrap-repo` skill

[`.claude/skills/bootstrap-repo/`](.claude/skills/bootstrap-repo/SKILL.md) applies the templates to a target repo.

- It finds the templates by path from its own folder, so keep them at the repo root.
- To use it from any repo, symlink it: `ln -s "$PWD/.claude/skills/bootstrap-repo" ~/.claude/skills/bootstrap-repo`.
- New stack support goes in `HOOKS.md` as a signal plus a hook block; hooks every repo needs go in the root `.pre-commit-config.yaml` instead.
