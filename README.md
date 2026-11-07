# niri-tomic-os &nbsp; [![Build distro](https://github.com/niri-tonic/niri-tomic-os/actions/workflows/build-distro.yml/badge.svg)](https://github.com/niri-tonic/niri-tomic-os/actions/workflows/build-distro.yml)

**Niri Tonic OS** is a personal immutable [Fedora Atomic](https://fedoraproject.org/atomic-desktops/) desktop image, built with [BlueBuild](https://blue-build.org) on top of the [Universal Blue](https://universal-blue.org/) `base-atomic` image (Fedora 44).

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
- 128 GB LPDDR5X (96 GB VRAM + 32 GB RAM)

Primary use cases: local LLM serving with [Lemonade](https://lemonade-server.ai), gaming with Steam, and Development / DevOps / SRE work.

## Installation

> [!WARNING]
> [This is an experimental feature](https://www.fedoraproject.org/wiki/Changes/OstreeNativeContainerStable), try at your own discretion.

To rebase an existing atomic Fedora installation to the latest build:

- First rebase to the unsigned image, to get the proper signing keys and policies installed:
  ```
  rpm-ostree rebase ostree-unverified-registry:ghcr.io/niri-tonic/niri-tomic-os:latest
  ```
- Reboot to complete the rebase:
  ```
  systemctl reboot
  ```
- Then rebase to the signed image, like so:
  ```
  rpm-ostree rebase ostree-image-signed:docker://ghcr.io/niri-tonic/niri-tomic-os:latest
  ```
- Reboot again to complete the installation
  ```
  systemctl reboot
  ```

The `latest` tag will automatically point to the latest build. That build will still always use the Fedora version specified in `recipe.yml`, so you won't get accidentally updated to the next major version.

### Migrating from the personal namespace

The repository now lives at `niri-tonic/niri-tomic-os`. Existing installations following
`ghcr.io/tdesaules/niri-tomic-os` do not automatically switch registries after the
GitHub transfer. Once the new image has been published and is publicly readable,
follow the rebase steps above to install its signing policy and switch to the new
signed image. Keep the existing `SIGNING_SECRET` paired with `cosign.pub`.

## ISO

![Dynamic JSON Badge](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgist.githubusercontent.com%2Ftdesaules%2F93883cb28c830015a10172604676af73%2Fraw%2Fniri-tomic-os.uuid.json&query=%24.uuid&label=uuid&color=blue)

```bash
oras pull ttl.sh/niri-tomic-os-x86_64-$(curl -s https://gist.githubusercontent.com/tdesaules/93883cb28c830015a10172604676af73/raw/niri-tomic-os.uuid.json | jq -r '.uuid'):24h
oras manifest fetch ttl.sh/niri-tomic-os-x86_64-$(curl -s https://gist.githubusercontent.com/tdesaules/93883cb28c830015a10172604676af73/raw/niri-tomic-os.uuid.json | jq -r '.uuid'):24h
cosign verify-blob \
  --bundle niri-tomic-os-x86_64.iso.sigstore.json \
  --certificate-identity "https://github.com/niri-tonic/niri-tomic-os/.github/workflows/build-iso.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  niri-tomic-os-x86_64.iso
```

## Verification

These images are signed with [Sigstore](https://www.sigstore.dev/)'s [cosign](https://github.com/sigstore/cosign). You can verify the signature by downloading the `cosign.pub` file from this repo and running the following command:

```bash
cosign verify --key cosign.pub ghcr.io/niri-tonic/niri-tomic-os
```

## Repository structure

```
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

- `build-distro.yml`: builds and signs the image on every push to `main` touching `recipes/`, `files/` or `cosign.pub`, plus a nightly rebuild, and pushes to `ghcr.io/niri-tonic/niri-tomic-os`.
- `build-iso.yml`: installer ISO build after each successful image workflow, or manually, signed with Sigstore and pushed to `ttl.sh` (24h TTL).

Both workflows derive the GHCR namespace from the repository owner. After a repository
transfer, run `Build distro` manually to validate publication in the new namespace;
documentation-only pushes do not trigger it. Ensure the organization allows package
creation and that the new GHCR package is public and grants this repository Actions
access. A successful image build automatically triggers the ISO build.

CI requires the repository secrets `SIGNING_SECRET` and `GIST_TOKEN`, and variables
`ARCH`, `IMAGE_NAME`, `VERSION`, `VARIANT`, and `GIST_ID`. The ISO discovery gist remains
under `tdesaules`: `GIST_TOKEN` must retain write access to it. ISO signatures created
before the transfer use the old repository URL as their certificate identity.

Local build with the [BlueBuild CLI](https://blue-build.org/how-to/build-locally/):

```bash
bluebuild build recipe.yml
```

## User configuration

The image stays intentionally lean on the user side: on the first graphical login, `chezmoi-first-init.timer` schedules the dotfiles bootstrap from <https://github.com/tdesaules/chez-moi> and retries on later logins until setup succeeds. Automatic Topgrade updates start only after that bootstrap completes. Personal tooling and shell configuration live in the chezmoi repository, not in this image.

## License

[GPL-3.0](LICENSE)
