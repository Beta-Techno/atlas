# Mani Manifest — Infrastructure Workspace

This repository owns `mani.yaml`, the manifest Anvil uses to clone and orchestrate every infrastructure repo (bootstrap → anvil → lockbox → harbor). Keep this repo checked out at `code/infra/mani`; when `mani sync` runs it creates sibling directories such as `code/infra/anvil`, `code/infra/harbor`, etc.

```
code/
  infra/
    anvil/       # provisioning playbooks & roles
    bootstrap/   # Ubuntu autoinstall shim that calls anvil
    lockbox/     # sops-encrypted runtime config + lockctl
    harbor/      # stack metadata + harborctl compose deployer
    key/         # bootstrap secrets bundles
    bricks/      # shared container images
    ops/         # fleet tooling / runbooks
    flakes/      # future nix-flake payloads
    dotfiles/    # chezmoi repo consumed by anvil
    mani/        # this repository
```

> Skip personal/dev or org-specific forks for now. Add them later under the same structure once requirements are clear.

## Getting started

1. Install [mani](https://github.com/alajmo/mani) (>=0.25). `curl -sfL https://raw.githubusercontent.com/alajmo/mani/main/install.sh | sh` works on Ubuntu 24.04.
2. Clone this repo into `code/infra/mani`.
3. From that directory run:
   ```bash
   mani list projects          # sanity-check manifest
   mani sync --tags infra      # clone/update all infra repos
   mani run status --tags infra
   ```

`mani sync` handles cloning/missing repos; commands defined inside `mani.yaml` perform per-repo tasks afterwards. Tag filters (`--tags infra,server` or `--tags secrets`) let Anvil/you target subsets.

## Commands

| Command | Description | Notes |
| --- | --- | --- |
| `status` | `git status -sb` in each repo | quick diff check before commits |
| `sync` | `git fetch --all --prune && git pull --ff-only` | safe fast-forward helper after `mani sync` |
| `bootstrap` | Runs `./bootstrap.sh`, `./run.sh`, or `npm install` if present | ensures repo-specific prerequisites finish |
| `lockbox-render` | Runs `lockctl render` (tags: `lockbox`) | requires `LOCKCTL_BIN`, `LOCKBOX_ENV`, `LOCKBOX_STACK`, `LOCKBOX_OUT`, and `SOPS_AGE_KEY_FILE` env vars (`LOCKCTL_BIN` defaults to `lockctl`; others have sensible defaults noted in `mani.yaml`) |
| `harbor-validate` | `go run ./cmd/harborctl catalog validate --env ${HARBOR_ENV:-dev}` | expects `SOPS_AGE_KEY_FILE` to point at the age key from the bootstrap bundle |
| `bootstrap-all` | Custom pipeline that calls `sync` + `bootstrap` across every infra repo | run after provisioning to ensure everything obeys repo-specific bootstraps |

Export env vars before persona-specific commands:

```bash
export SOPS_AGE_KEY_FILE=$HOME/.config/age/keys.txt
export LOCKBOX_ENV=prod
export LOCKBOX_STACK=all
export LOCKBOX_OUT=$HOME/code/infra/harbor
mani run lockbox-render
```

## Integration notes (for Anvil)

- Run `mani sync --tags infra` near the end of the playbook, after git identity and SOPS age keys are available.
- Expect `SOPS_AGE_KEY_FILE` to be set by the bootstrap bundle. `lockbox-render` and `harbor-validate` will fail fast if it is missing.
- Lockbox is cloned into `code/infra/lockbox` before Mani commands run, so `lockctl` only needs the key file; no extra args are required.
- Provide `LOCKCTL_BIN` if the binary isn’t on `PATH` (e.g., `~/.local/bin/lockctl`).
- Use tags to differentiate personas:
  - `infra` — everything in this manifest (default)
  - `server` — harbor, lockbox, bootstrap, anvil
  - `dev` — anvil, dotfiles, flakes
  - `secrets` — lockbox, key
  - `ops` — ops, harbor

## Testing expectations

- Minimal validation: run `mani list projects`, `mani sync --dry-run --tags infra` (>=0.27) or `mani sync --tags infra` inside an Ubuntu 24.04 VM created via the bootstrap Proxmox flow.
- Functional smoke: `mani run status --tags infra`, `mani run lockbox-render` (with a test `LOCKBOX_ENV=dev` + dummy age key) to ensure shell hooks work without manual tweaks.
- Long term add CI that runs `mani lint` (when available) or `mani list` + `mani sync --dry-run` to catch manifest drift.

## Contributing

- Edit `mani.yaml` when adding/removing repos or commands; keep descriptions/tags up to date so personas can target the right subsets.
- Avoid storing decrypted materials here—Lockbox renders into `.rendered/` inside consumer repos which remain gitignored.
- Document new commands in this README so future Anvil roles can rely on them.
