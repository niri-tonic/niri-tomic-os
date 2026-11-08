# AGENTS.md

## Project overview

**Niri Tonic OS** (image/repo name: `niri-tomic-os`, hostname: `niritonicos`) is a personal immutable Fedora Atomic desktop image built with [BlueBuild](https://blue-build.org) on top of the Fedora OSTree Desktops `base-atomic` base image (Fedora 44, see `recipes/recipe.yml`). It provides a Niri Wayland session with Dank Material Shell (DMS / Dank Linux), `greetd` + `dms-greeter`, and a Nord theme.

Fedora 45 qualification uses `recipes/recipe-fedora45.yml`, sharing the production modules and files. The manual `build-fedora45.yml` workflow optionally publishes to the separate `niri-tomic-os-fedora45` package and does not trigger the production ISO workflow. See `docs/fedora45-migration.md` before promoting the candidate.

The build is fully declarative: the image is described by `recipes/recipe.yml` + `recipes/modules/*.yml` (BlueBuild modules) and static files under `files/system/` (copied verbatim to `/`). There are no imperative build scripts in this repository.

User-side configuration (dotfiles, shell, personal tooling) is NOT part of this image: `chezmoi-first-init.timer` schedules it after the first graphical login through `chezmoi-first-init.service` using <https://github.com/tdesaules/chez-moi>. Initialization retries on later sessions until chezmoi writes `~/.local/state/chezmoi/bootstrap-complete`; automatic Topgrade updates require that marker.

## Repository map

- `recipes/recipe.yml` — main recipe: base image, Fedora version, platforms, module list
- `recipes/modules/` — BlueBuild modules:
  - `dnf.yml` — Fedora, Terra, and Dank Linux package install/remove configuration
  - `files.yml` — copies `files/system` to `/`
  - `systemd.yml` — enabled system and user units
  - `initramfs.yml`, `os-release.yml`, `signing.yml`
- `files/system/` — static files merged into the image:
  - `etc/niri/config.kdl` — Niri compositor configuration (KDL syntax)
  - `etc/greetd/config.toml` — greeter configuration
  - `etc/dms/`, `etc/topgrade/`, `etc/wireplumber/`, `etc/sudoers.d/`
  - `usr/lib/systemd/{system,user}/` — unit files
  - `usr/lib/tmpfiles.d/`, `usr/lib/sysusers.d/`, `usr/lib/firmware/`
  - `usr/share/{backgrounds,fonts,icons,applications}/` — assets
- `.github/workflows/` — CI pipelines (image + ISOs)
- `cosign.pub` — image signing public key (the private key exists only in CI secrets)

## Build & validation

There is no test suite; validation means building the image successfully.

- Local build (requires podman/docker and the BlueBuild CLI), from the repo root:
  ```bash
  bluebuild build recipe.yml
  ```
- CI (`build-distro.yml`) builds on pushes to `main` touching `recipes/`, `files/` or `cosign.pub`, nightly, and on manual dispatch; it signs with cosign and pushes to `ghcr.io/niri-tonic/niri-tomic-os`.
- After editing a Niri KDL config, check syntax against the Niri docs: <https://niri-wm.github.io/niri/>. `dms-greeter` also parses this file (`-C /etc/niri/config.kdl`), so a syntax error breaks the greeter too.
- For BlueBuild module schemas, refer to <https://blue-build.org/> before adding or editing modules.

## Conventions

- Naming: the OS is called **Niri Tonic OS** / `niri-tonic-os`; the image and repository are named `niri-tomic-os`.
- Comments and commit messages must be in English.
- Follow the existing style of each file type: YAML (recipes/modules), KDL (Niri), TOML (greetd, topgrade), systemd unit syntax, JSON (DMS theme).
- Theme is **Nord**: keep colors consistent with `files/system/etc/dms/themes/nord/theme.json` and the Niri config (e.g. `#81a1c1` focus, `#bf616a` urgent).
- The image is built for both `linux/amd64` and `linux/arm64`: never add architecture-specific packages or repos without verifying arm64 availability.
- Immutable-image friendly changes only: prefer `/etc` files, systemd units, `tmpfiles.d`/`sysusers.d` over anything assuming runtime mutation. User-level changes belong in the chezmoi repository, not here.
- Packages are intentionally removed in `dnf.yml` (`flatpak`, `firefox`, `nano`) and firmware updates are intentionally disabled in topgrade (`disable = ["firmware"]`); do not reintroduce them without being asked.
- Weak dependencies are disabled (`install-weak-deps: false`): if a package needs an extra dependency, add it explicitly.
- COPR repositories prune old builds, so stable Dank packages float in `dnf.yml`; Niri comes from Fedora. Verify package availability and compatibility on both `x86_64` and `aarch64` before changing repositories or package names.

## Common tasks

- Add/remove a package: edit `recipes/modules/dnf.yml`.
- Add a file to the image: place it under `files/system/` mirroring its target path (it is copied to `/` by `recipes/modules/files.yml`).
- Add a systemd user unit: drop the unit in `files/system/usr/lib/systemd/user/` AND enable it in `recipes/modules/systemd.yml`.
- Change Niri behavior: edit `files/system/etc/niri/config.kdl` (binds, layout, window rules, input).

## Security

- Never commit signing material: `cosign.key` / `cosign.private` are git-ignored for a reason; only `cosign.pub` belongs in the repo.
- `%wheel` has passwordless sudo by design (`files/system/etc/sudoers.d/wheel-nopasswd`); keep this in mind when reviewing privilege-related changes.

## CI notes

- `build-distro.yml` — `blue-build/github-action@v1.11`, cosign signing via `SIGNING_SECRET`, push to GHCR.
- `build-iso.yml` — installer ISO after successful image workflows or manual dispatch, signed with Sigstore, pushed to `ttl.sh` with 24h TTL; the current UUID is published to a gist.
