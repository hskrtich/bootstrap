# Hook catalog

Stack-specific hooks layered on top of the base template `.pre-commit-config.yaml`. Detect against `git ls-files --cached --others --exclude-standard`, so ignored trees (`node_modules`, `.terraform`) never count. Add a block for **every** row whose signal matches. The `rev`s here are seeds; `pre-commit autoupdate` sets the real latest.

**Adopt, don't impose**: linters go in whenever their signal matches; a formatter goes in only when the row says so, since it rewrites code the team already formats some other way.

## Shell

Signal: `*.sh` / `*.bash` files, or tracked files with a `sh`/`bash`/`zsh` shebang.

```yaml
  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: v0.11.0.1-1
    hooks:
      - id: shellcheck
```

Add `scop/pre-commit-shfmt` (id `shfmt`) only when the repo already runs `shfmt` (Makefile, CI, scripts). It reads `.editorconfig` for indent style.

## OpenTofu

Signal: `*.tf` / `*.tofu` files **and** any of: a `*.tofu` file, `registry.opentofu.org` in `.terraform.lock.hcl`, a `.opentofu-version`/`.tofu-version` file, or `tofu` invoked in CI, Makefile, or scripts.

```yaml
  - repo: https://github.com/tofuutils/pre-commit-opentofu
    rev: v2.4.2 # Get the latest from: https://github.com/tofuutils/pre-commit-opentofu/releases
    hooks:
      - id: tofu_fmt
      - id: tofu_tflint
      - id: tofu_checkov
        name: Terraform security with Checkov
        args:
          - --env-vars=CKV_IGNORE_HIDDEN_DIRECTORIES=false
          - --args=--quiet
          - --args=--framework=terraform
          - --args=--skip-check CKV_TF_1
          - --args=--download-external-modules true
          - --args=--external-modules-download-path .terraform/checkov
```

## Terraform

Signal: `*.tf` files **and** any of: `registry.terraform.io` in `.terraform.lock.hcl`, a `.terraform-version` file, or `terraform` invoked in CI, Makefile, or scripts. When the signals for OpenTofu and Terraform tie or conflict, ask the user which one the repo runs.

```yaml
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.109.5
    hooks:
      - id: terraform_fmt
      - id: terraform_tflint
      - id: terraform_checkov
        name: Terraform security with Checkov
        args:
          - --env-vars=CKV_IGNORE_HIDDEN_DIRECTORIES=false
          - --args=--quiet
          - --args=--framework=terraform
          - --args=--skip-check CKV_TF_1
          - --args=--download-external-modules true
          - --args=--external-modules-download-path .terraform/checkov
```

Both the OpenTofu and Terraform hook repos shell out to `tofu`/`terraform`, `tflint`, and `checkov` on `PATH`; pre-commit does not install them.

## Python

Signal: `*.py` files or `pyproject.toml`.

```yaml
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.17.5
    hooks:
      - id: ruff-check
        args: [--fix]
```

Add `ruff-format` to the same entry only when the repo already formats with ruff (a `[tool.ruff.format]` table, or `ruff format` in CI, Makefile, or scripts).

## Go

Signal: `go.mod`.

```yaml
  - repo: https://github.com/golangci/golangci-lint
    rev: v2.15.0
    hooks:
      - id: golangci-lint
      - id: golangci-lint-fmt
```

## JavaScript / TypeScript

Signal: `package.json`. Wire up only the tooling the project already configures:

- `biome.json` / `biome.jsonc` → `biomejs/pre-commit` (rev `v2.6.7`), id `biome-check`.
- ESLint or Prettier config → a `repo: local` hook per tool with `language: system` and `entry: npx --no-install eslint` / `npx --no-install prettier --check`, scoped with `types_or: [javascript, jsx, ts, tsx]`. These use the project's own `node_modules`, so plugins resolve.

No JS tooling configured → add nothing, and name that gap in the report.

## Dockerfiles

Signal: `Dockerfile*`, `*.dockerfile`, or `Containerfile`.

```yaml
  - repo: https://github.com/AleksaC/hadolint-py
    rev: v2.15.1
    hooks:
      - id: hadolint
```

## GitHub Actions

Signal: `.github/workflows/*.yml` or `*.yaml`.

```yaml
  - repo: https://github.com/rhysd/actionlint
    rev: v1.7.12
    hooks:
      - id: actionlint
```

## Data files

Extra ids for the existing `pre-commit/pre-commit-hooks` entry:

- `*.json` → `check-json`. It rejects JSON-with-comments, so add `exclude` for `tsconfig*.json`, `.vscode/`, `.devcontainer/`, and any other JSONC the repo tracks.
- `*.toml` → `check-toml`.

`check-yaml` false-fails on a few common YAML shapes. Tune it rather than drop it:

- Multi-document files (Kubernetes manifests) → `args: [--allow-multiple-documents]`.
- CloudFormation intrinsics (`!Ref`, `!Sub`) → `args: [--unsafe]`.
- Helm chart `templates/` are Go templates, not YAML → `exclude: ^charts/.*/templates/` (match the repo's chart path).

## Suggested

Noisy on an existing codebase, so by default these go in the report as suggestions when their signal matches:

- Markdown → `DavidAnson/markdownlint-cli2` (id `markdownlint-cli2`).
- YAML style → `adrienverge/yamllint` (id `yamllint`).

Add either directly when the repo already has a `.markdownlint*` or `.yamllint*` config.
