# BlueBuild Images - Agent Instructions

## What this repo does

Builds custom Fedora Atomic container images (Silverblue, Kinoite, uCore) using [BlueBuild](https://blue-build.org/). Images are pushed to `ghcr.io/bluebuild/<name>` and signed with cosign.

## Architecture

```
recipes/
  common.yaml              # shared modules (dnf, files, systemd) for all images
  common-nvidia.yaml       # shared nvidia modules
  workstation/
    common.yaml            # workstation shared modules
    common-nvidia.yaml     # workstation nvidia overrides
    silverblue/
      base.yaml            # silverblue module composition
      main.yaml            # ← main entry: composes base.yaml + signing
      main-gts.yaml        # main with GTS kernel
      main-beta.yaml       # main with beta kernel
      nvidia.yaml          # main + common-nvidia.yaml
      nvidia-gts.yaml
      nvidia-beta.yaml
    kinoite/
      base.yaml
      main.yaml
      nvidia.yaml
  server/
    common.yaml            # server shared modules
    common-nvidia.yaml
    ucore/
      base.yaml            # ucore module composition
      main.yaml            # ← main entry: composes base.yaml
      nvidia.yaml
files/
  common/                  # files deployed to / for all images
  nvidia/common/           # nvidia-specific files
  workstation/             # workstation-specific files
  server/                  # server-specific files
  scripts/
modules/                   # (empty - for custom module scripts)
```

Recipe files compose modules via `from-file:` references. The build tool reads the top-level recipe (e.g. `recipes/workstation/silverblue/main.yaml`) and resolves all `from-file:` chains.

## Key commands

**Local build** (requires Fedora/BlueBuild host):
```bash
bluebuild build recipes/workstation/silverblue/main.yaml
```

**CI** runs via `blue-build/github-action@v1.11` in `.github/workflows/build.yml`. Recipes are added to the matrix in that file.

## Recipe structure

Each recipe YAML uses the BlueBuild schema (`# yaml-language-server: $schema=...`). Modules include:
- `type: dnf` — packages, repos, removals
- `type: files` — copy `files/<source>` to destination
- `type: systemd` — enable/disable services
- `type: signing` — cosign signing step
- `type: akmods` — build kernel modules (e.g. v4l2loopback)
- `type: default-flatpaks` — install flatpaks at build time
- `type: script` — run arbitrary shell snippets

## Build artifacts & conventions

- `cosign.pub` — public signing key (committed to repo)
- `cosign.key`, `cosign.private` — gitignored, never commit
- `/.bluebuild-scripts_*` — gitignored, generated at build time
- `/Containerfile` — gitignored, generated at build time
- `latest` tag always points to the latest build; Fedora version is pinned by `recipe.yml`
- Images are **unsigned** on first rebase, then **signed** on second rebase (see README)

## Adding a new image variant

1. Copy an existing recipe YAML from the target family (e.g. `recipes/workstation/silverblue/main.yaml`)
2. Update `name`, `description`, and `base-image`
3. Compose from `base.yaml` or `from-file:` as needed
4. Add the recipe path to the matrix in `.github/workflows/build.yml`

## Adding nvidia support

Include `from-file: common-nvidia.yaml` (or `workstation/common-nvidia.yaml` for workstation images) in the recipe's modules list.

## CI triggers

- Daily at 06:00 UTC
- On push to non-`**.md` files
- On pull requests
- Manual workflow dispatch

`cancel-in-progress: true` is set — concurrent PR builds cancel prior ones.

## Renovate

Uses `config:recommended` in `renovate.json`. No custom rules — relies on Renovate defaults.
