# niri-tomic-os &nbsp; [![main](https://github.com/niri-tonic/niri-tomic-os/actions/workflows/main.yml/badge.svg)](https://github.com/niri-tonic/niri-tomic-os/actions/workflows/main.yml)

**Niri Tonic OS** is a personal immutable [Fedora Atomic](https://fedoraproject.org/atomic-desktops/) desktop image, built with [BlueBuild](https://blue-build.org) on top of the [Fedora OSTree Desktops](https://quay.io/repository/fedora-ostree-desktops/base-atomic) `base-atomic` image (Fedora 45).

It ships a [Niri](https://niri-wm.github.io/) scrollable-tiling Wayland session powered by [Dank Material Shell (DMS)](https://danklinux.com/), with `greetd` + `dms-greeter` and a full [Nord](https://www.nordtheme.com/) theme.

## Highlights

- **Niri** Wayland compositor from Fedora with a Nord-flavored KDL configuration
- **DMS** (Dank Material Shell): panel, spotlight/app launcher (`Mod+Space`), wallpaper rotation, greeter sync
- **greetd** + **dms-greeter** graphical login
- **Ghostty** terminal
- **fcitx5** input method (including Hangul)
- **chezmoi**: user dotfiles auto-applied on first login from [tdesaules/chez-moi](https://github.com/tdesaules/chez-moi)
- **topgrade** automated updates (rpm-ostree + chezmoi; firmware updates deliberately disabled)
- **Podman** (`podman-docker`, `podman-compose`) and **distrobox** for containerized workflows
- Hardware-specific tweaks for Strix Halo: WirePlumber AMD SoundWire audio fix, battery charge threshold capped at 85%
- Multi-arch builds: `linux/amd64` and `linux/arm64`
- Images signed with [Sigstore cosign](https://www.sigstore.dev/)

## Target hardware & usage

Tuned for an **ASUS ProArt PX13 (HN7306)**:

- AMD Ryzen AI MAX+ 395 (Strix Halo)
- AMD XDNA NPU
- AMD Radeon 8060S Graphics
- 128 GB shared LPDDR5X; an automatically detected [Strix Halo memory profile](docs/strix-halo-memory.md)
  combines a 16 GiB BIOS GPU reservation with an 80 GiB dynamic GPU memory ceiling.

Primary use cases: local LLM serving with [Lemonade](https://lemonade-server.ai), gaming with Steam, and Development / DevOps / SRE work.

## Installation

> [!WARNING]
> [This is an experimental feature](https://fedoraproject.org/wiki/Changes/OstreeNativeContainerStable), try at your own discretion.

To rebase an existing atomic Fedora installation to the latest build:

- First rebase to the unsigned image, to get the proper signing keys and policies installed:

  ```bash
  rpm-ostree rebase ostree-unverified-registry:ghcr.io/niri-tonic/niri-tomic-os:latest
  ```

- Reboot to complete the rebase:

  ```bash
  systemctl reboot
  ```

- Then rebase to the signed image, like so:

  ```bash
  rpm-ostree rebase ostree-image-signed:docker://ghcr.io/niri-tonic/niri-tomic-os:latest
  ```

- Reboot again to complete the installation

  ```bash
  systemctl reboot
  ```

The `latest` tag follows the Fedora version specified in `recipes/recipe.yml`.
Changing that version and publishing the image upgrades machines following this
tag on their next system update and reboot, including updates staged by Topgrade.

### Fedora 45 migration

Fedora 45 uses `recipes/recipe.yml` and explicit **release** preparation,
publishing to `ghcr.io/niri-tonic/niri-tomic-os`. The separate
candidate recipe and workflow have been removed. See the
[qualification and rollback procedure](docs/fedora45-migration.md).

### Migrating from the personal namespace

The repository now lives at `niri-tonic/niri-tomic-os`. Existing installations following
`ghcr.io/tdesaules/niri-tomic-os` do not automatically switch registries after the
GitHub transfer. Once the new image has been published and is publicly readable,
follow the rebase steps above to install its signing policy and switch to the new
signed image. Keep the existing `SIGNING_SECRET` paired with `cosign.pub`.

## ISO

AMD64 installers are distributed as OCI artifacts in
`ghcr.io/niri-tonic/niri-tomic-os-iso`. Use the immutable installer reference
listed in the [release notes](https://github.com/niri-tonic/niri-tomic-os/releases).
The first release through this pipeline is still awaiting end-to-end validation.

Set `ISO_REFERENCE` to that digest-qualified reference, then download and check:

```bash
oras pull "${ISO_REFERENCE:?Set the immutable installer reference from release notes}"
sha256sum -c niri-tomic-os-x86_64.iso-CHECKSUM
cosign verify --key cosign.pub "$ISO_REFERENCE"
```

Cosign verifies the OCI artifact manifest. The checksum sidecar verifies the
downloaded ISO; the accompanying `.build.json` preserves its source binding.
ARM64 OS images are supported, but ARM64 installer media is not produced.

## Verification

These images are signed with [Sigstore](https://www.sigstore.dev/)'s [cosign](https://github.com/sigstore/cosign). You can verify the signature by downloading the `cosign.pub` file from this repo and running the following command:

```bash
cosign verify --key cosign.pub ghcr.io/niri-tonic/niri-tomic-os
```

## Repository structure

```text
├── recipes/
│   ├── recipe.yml              # Main image recipe (base image, platforms, modules)
│   └── modules/                # BlueBuild modules: dnf, files, systemd, signing, ...
├── files/
│   └── system/                 # Static files copied verbatim to / (etc/, usr/)
│       ├── etc/                # niri, greetd, dms, topgrade, wireplumber, sudoers
│       └── usr/                # systemd units, fonts, icons, wallpapers, firmware
├── .github/workflows/          # Image and ISO build pipelines
└── cosign.pub                  # Public key used to verify images
```

## Builds

- `main.yml`: branch-push checks, selected native AMD64/ARM64 builds and tests.
- `release.yml`: main-only manual preparation, then stable tag verification and
  GitHub Release publication. Tags retest published artifacts without rebuilding.
- `maintenance.yml`: scheduled dependency updates, advisory rescans and link
  checks; main-only manual execution supports a Renovate dry-run.

Publication is explicit, never nightly. Actions are pinned to the qualified
[`tdesaules/github-actions` 1.7.3](https://github.com/tdesaules/github-actions/releases/tag/1.7.3)
catalog. The OS builder uses the approved Atomic-only BuildKit exception; ISO
construction uses the separate Lorax-only privileged worker.

CI uses `vars.BOT_APP_ID` and `secrets.BOT_APP_PRIVATE_KEY` from the shared
repository-scoped GitHub App, plus `SIGNING_SECRET` paired with `cosign.pub`.
The job token needs GHCR package access. Both OS and installer packages must be
public for anonymous installation. Signing and vulnerability reports remain
advisory, with failures visible; functional and artifact-integrity failures block
publication. See [release operations](docs/release-ci.md).

Local build with the [BlueBuild CLI](https://blue-build.org/how-to/local/):

```bash
bluebuild build recipe.yml
```

## User configuration

The image stays intentionally lean on the user side: on the first graphical login, `chezmoi-first-init.timer` schedules the dotfiles bootstrap from <https://github.com/tdesaules/chez-moi> and retries on later logins until setup succeeds. Automatic Topgrade updates start only after that bootstrap completes. Personal tooling and shell configuration live in the chezmoi repository, not in this image.

## License

[GPL-3.0](LICENSE)
