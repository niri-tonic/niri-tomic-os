# Fedora 45 pilot — release 1.0.2

Release [1.0.2](https://github.com/niri-tonic/niri-tomic-os/releases/tag/1.0.2)
passed native AMD64/ARM64 OS qualification and AMD64 installer-media qualification.
The tag workflow retested the published artifacts without rebuilding.
The full ISO download and OS/ISO signatures were independently verified without
registry credentials.

## Exact artifacts

| Artifact | Immutable identity |
| --- | --- |
| OS index | `ghcr.io/niri-tonic/niri-tomic-os@sha256:e921e3e7a1df53b83bcc50e2e5ca80ef0b60e11212157c4698e2d41fdf5b98fb` |
| AMD64 manifest | `sha256:035bcdc11095a5277900fbb18cbd8fdb434d01ffec7a99e348fe18efdb3e8576` |
| ARM64 manifest | `sha256:959eca414cd05011a5f873c476b2a46a3ea4e31f645bb65027dfd7de668dc487` |
| Installer artifact | `ghcr.io/niri-tonic/niri-tomic-os-iso@sha256:55a206b52da8d444b62a1c01c4b47b85dd418b520ef319741df7f3d20974cef3` |
| ISO SHA-256 | `6c4a8e2211b8e98577389f91829bd80f5c7e8821199cbd41f433b2452ec7b11c` |

The ISO is **4,747,362,304 bytes**. Source commit:
`5515d1febc7ec3d7644100bb40a13852bfc6303f`; release metadata commit:
`c8104348db49d86a17aa3d3b3ab0ba2416d534fe`.

Use the repository's trusted `cosign.pub`:

```bash
OS_REFERENCE=ghcr.io/niri-tonic/niri-tomic-os@sha256:e921e3e7a1df53b83bcc50e2e5ca80ef0b60e11212157c4698e2d41fdf5b98fb
ISO_REFERENCE=ghcr.io/niri-tonic/niri-tomic-os-iso@sha256:55a206b52da8d444b62a1c01c4b47b85dd418b520ef319741df7f3d20974cef3
cosign verify --key cosign.pub "$OS_REFERENCE"
cosign verify --key cosign.pub "$ISO_REFERENCE"
oras pull "$ISO_REFERENCE"
sha256sum -c niri-tomic-os-x86_64.iso-CHECKSUM
```

## Preserve the current system

These are instructions for the pilot, not host changes already performed.
On November 17 the actual machine still booted Fedora `44.20261028.0` from the
old personal `:44` namespace, with no staged deployment. Both Podman stores use
SQLite. The 16 GiB BIOS UMA reservation and 128 GiB installed RAM were confirmed.

Back up home data, keyrings and rootless/rootful container volumes. Inspect the
deployment list again and pin the currently booted entry:

```bash
rpm-ostree status
BOOTED_INDEX=$(rpm-ostree status --json |
  jq -r '.deployments | to_entries[] | select(.value.booted) | .key')
sudo ostree admin pin "$BOOTED_INDEX"
systemctl --user mask --now topgrade.timer topgrade.service
```

## Trust the organization image

From this trusted source checkout, preserve the existing policy and add the
organization namespace without replacing unrelated trust entries:

```bash
sudo cp -a /etc/containers/policy.json \
  "/etc/containers/policy.json.before-fedora45-$(date +%Y%m%d%H%M%S)"
sudo install -D -m 0644 cosign.pub /etc/pki/containers/niri-tonic-os.pub
sudo python3 - <<'PY'
import json
import os
import shutil
from pathlib import Path

path = Path("/etc/containers/policy.json")
policy = json.loads(path.read_text())
policy.setdefault("transports", {}).setdefault("docker", {})[
    "ghcr.io/niri-tonic/niri-tomic-os"
] = [{
    "type": "sigstoreSigned",
    "keyPath": "/etc/pki/containers/niri-tonic-os.pub",
    "signedIdentity": {"type": "matchRepository"},
}]
temporary = path.with_name("policy.json.fedora45-new")
with temporary.open("x") as output:
    output.write(json.dumps(policy, indent=2) + "\n")
shutil.copymode(path, temporary)
os.replace(temporary, path)
PY
sudo tee /etc/containers/registries.d/niri-tonic-os.yaml >/dev/null <<'YAML'
docker:
  ghcr.io/niri-tonic/niri-tomic-os:
    use-sigstore-attachments: true
YAML
```

## Explicit pilot rebase

When ready to migrate the host:

```bash
sudo rpm-ostree rebase \
  ostree-image-signed:docker://ghcr.io/niri-tonic/niri-tomic-os@sha256:e921e3e7a1df53b83bcc50e2e5ca80ef0b60e11212157c4698e2d41fdf5b98fb
rpm-ostree status
systemctl reboot
```

After boot, check the deployment and failed units, then exercise login, keyring
unlock, audio, Wi-Fi, suspend/resume and a representative GPU workload:

```bash
rpm-ostree status
systemctl --failed
systemctl --user --failed
journalctl -u niri-strix-halo-128g-memory.service -b --no-pager
rpm-ostree kargs
```

The hardware-specific service may stage the memory argument for a further reboot.
After that reboot, verify:

```bash
cat /sys/module/ttm/parameters/pages_limit
grep . /sys/class/drm/card*/device/mem_info_{vram,gtt}_total
```

Expected: `20971520` pages, approximately 80 GiB GTT and 16 GiB VRAM.
The ceiling is dynamic, not an allocation guarantee.

## Rollback and update tracking

Select the pinned Fedora 44 deployment from the boot menu if needed. Inspect
`rpm-ostree status` before using `rpm-ostree rollback`: the GPU argument transaction
can create another Fedora 45 deployment, so the preceding entry need not be
Fedora 44. Deployment rollback does not restore home or `/var` application data.

This pilot pins one immutable digest. After hardware and application validation,
explicitly choose the signed `:45` channel to remain on Fedora 45, or `:latest`
to follow the recipe's future Fedora major version, then restore automatic updates:

```bash
systemctl --user unmask topgrade.timer topgrade.service
systemctl --user start topgrade.timer
```

Secure Boot, real user login, keyring migration, physical GPU behavior and
host rollback remain pilot work. Earlier BIOS/UEFI installation tests used the
documented prototype; they do not claim a new boot test of this exact release ISO.
