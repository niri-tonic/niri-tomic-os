# Fedora 45 migration and qualification

The production recipe now targets Fedora 45, with an extra
Fedora 45 module installing `pam_oo7`. The Fedora 45 greetd PAM stack references
this module, but it was absent from the initial build with weak dependencies
disabled. The separate candidate recipe and workflow have been removed.
Production selection does not imply that the runtime checks below have passed.

## Local validation — 2026-11-08

Both corrected candidate builds completed, including initramfs generation and
systemd unit enablement. AMD64 ran natively; ARM64 used QEMU user emulation.
In each final image, `niri validate --config /etc/niri/config.kdl` succeeded and
`pam_oo7.so` was present and dynamically loadable.

Local image IDs (configuration digests, not published registry manifest digests):

| Architecture | Image ID |
| --- | --- |
| AMD64 | `91282344d58247bd5b993f1e5439626d0c027ea4e25c7a436420218d402d5545` |
| ARM64 | `2db914877198238dc62a81046ed25a2ae5a3ff67e20d8596702e27a690e5d1b7` |

Base index: `sha256:947ba3183f4d165f67f3c232f77218b6444c2b3665096a65ca4da4f82995d07d`.
The local signing module configured the `localhost` image identity; these builds
were not published or signed for GHCR. Separate single-platform builds replace
BlueBuild's common local manifest tags, so use the architecture-specific tags or
the IDs above rather than assuming local `latest` contains both architectures.
The candidate workflow passed actionlint 1.7.12. Those initial checks did not
qualify boot, login or installation. Subsequent prototype BIOS/UEFI installation
and greetd/Niri startup evidence is recorded in the
[November 15 audit](audit-2026-11-15.md); user login, keyring migration, physical
hardware and rollback still require pilot qualification.

## Build

For a local native AMD64 build:

```bash
bluebuild build --build-driver podman --platform linux/amd64 recipes/recipe.yml
```

Use `linux/arm64` on an ARM64 builder. Branch CI qualifies native images without
production publication. Explicit **release** preparation on `main` builds and
qualifies both platforms before publishing `ghcr.io/niri-tonic/niri-tomic-os`.
Cosign uses the existing `SIGNING_SECRET`. Ensure GHCR is publicly readable for
pilot hosts. See [release operations](release-ci.md).

On SELinux-enforcing hosts, run local Podman builds sequentially, or use separate
source copies and temporary directories for each architecture. Concurrent builds
sharing BlueBuild script bind mounts can relabel the same files with different
private `:Z` labels and fail with `script file read error: Permission denied`.
Check AVC records before attributing this error to a Fedora package transaction.

Record the source commit, BlueBuild CLI/modules versions, base and output digests,
build logs and RPM inventory. Confirm that the published index contains both
AMD64 and ARM64. Verify the output digest with `cosign.pub` before deploying it.
An emulated build does not qualify native ARM64 boot or hardware behavior.

## Baseline and pilot deployment

Use a disposable VM first, then a representative existing Fedora 44 installation.
Record its signed image origin and digest from `rpm-ostree status`, local package
layering, and changes reported by `ostree admin config-diff`. Identify and pin the
working deployment with `ostree admin pin INDEX` before rebasing; do not assume
that deployment index 0 is the booted deployment.

Back up home data, keyrings and container volumes. Record `podman info --debug`,
`podman ps -a` and `podman volume ls` for both rootless and rootful workloads.
Migrate legacy BoltDB state with the supported Podman upgrade procedure before
moving to Podman 6. Do not use `podman system reset` to migrate existing state.

Pause the pilot user's automatic updates while qualifying an exact image:

```bash
systemctl --user mask --now topgrade.timer topgrade.service
```

Use the published Fedora 45 image's verified digest rather than its mutable
`latest` tag. Inspect the pilot's container signature policy first: older hosts
may authorize only `ghcr.io/tdesaules/niri-tomic-os`. Configure trust for
`ghcr.io/niri-tonic/niri-tomic-os` using the verified public key and the BlueBuild
signing procedure before attempting a signed rebase. Signing with the same key
does not automatically authorize a new repository in every existing policy.

The signed transport, once trust is configured, is:

```text
ostree-image-signed:docker://ghcr.io/niri-tonic/niri-tomic-os@sha256:DIGEST
```

Rebase with `rpm-ostree rebase` to that reference and reboot. Replace `DIGEST`
with the recorded, verified candidate digest. Record the deployed digest again
after boot. A digest-pinned pilot will not follow subsequent candidate builds.

## Qualification record

Record pass/fail, architecture, image digest and relevant logs for each scenario:

| Area | Required checks |
| --- | --- |
| Package transaction | Both architectures build; GPG verification succeeds; record RPM versions and origins, especially Qt/Quickshell, Ghostty and mise |
| Repositories | DNF5 discovers Fedora repos under `/usr`; Terra/COPR cleanup works; inherited `/etc` overrides and layered RPMs remain usable |
| Boot | UEFI, initramfs, Plymouth, LUKS where used, rescue VT, greetd on VT1, Secure Boot where used |
| Session | Fresh account and existing account; login/logout, lock/unlock; `niri validate --config /etc/niri/config.kdl`; separately validate the user's Niri config |
| DMS | Panel, spotlight, wallpaper set/clear/next, greeter sync, search service, Qt/QML loading |
| Desktop | GTK/Qt file dialogs, screen sharing, audio, Hangul input, multiple displays and scaling |
| Secrets | Identify Secret Service provider and greetd PAM integration; unlock and read existing keyrings, including after rollback |
| Bootstrap | Ghostty launches chezmoi; completion marker suppresses repeat initialization; inspect external dotfile compatibility |
| Containers | Existing rootless/rootful containers, SQLite state, networking, volumes, Docker API sockets, Compose, Distrobox, GPU access |
| Strix Halo | Speakers/headphones/mic, repeated suspend/resume, Wi-Fi/Bluetooth, battery threshold, power profiles, graphics and representative LLM workload |
| System services | No new failed system/user units or relevant SELinux denials; journal reviewed |

Do not remove the audio or firmware workarounds until testing shows that Fedora
provides the required behavior. Check whether copied TAS2783 firmware masks newer
packaged firmware. Keep weak dependencies disabled and add any required missing
runtime dependency explicitly.

## ISO qualification

Release preparation constructs the AMD64 installer from its exact source-bound
OS archive using the published Fedora 45/Lorax action. Media verification compares
the embedded native manifest and blobs with the qualified OS. Tag verification
downloads and retests the published installer without rebuilding.

A successful media check alone does not prove that installation works. Test boot,
partitioning, encryption, account creation, signed installation, first reboot and
subsequent update. ARM64 OS images are supported; ARM64 ISO output is not advertised.

## Rollback

Select the pinned Fedora 44 deployment from the boot menu if the candidate cannot
boot. Otherwise inspect `rpm-ostree status` and, when the rollback deployment is
the recorded Fedora 44 deployment, run:

```bash
sudo rpm-ostree rollback
systemctl reboot
```

Confirm the booted version, signed origin and image digest. Restore the intended
production origin if needed before resuming updates. OSTree rollback does not
restore home or `/var`: verify keyrings, container state and chezmoi-managed files
against backups. Test these workloads after rollback, not just the OS boot.

Once the intended origin and update behavior are confirmed:

```bash
systemctl --user unmask topgrade.timer topgrade.service
systemctl --user start topgrade.timer
```

## Production rollout

Require successful AMD64/ARM64 builds, runtime qualification, an existing-system
upgrade and rollback, installer qualification, and several days of pilot use.
Keep a recoverable signed Fedora 44 image and pinned deployments during rollout.

`recipes/recipe.yml` now selects 45 through the normal workflow. Once published,
Fedora 45 is available to machines following production `latest`. The Fedora 45
PAM module is included and the installer constructor targets Fedora 45. A rebuild against
floating repositories is a new artifact: compare and qualify its exact digest
before treating it as tested. Hosts following the old personal namespace or
the `:44` tag require an explicit origin change.
