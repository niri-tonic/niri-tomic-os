# Fedora 45 pilot — November 15, 2026

This is the historical pre-release pilot record. Use the
[release 1.0.2 pilot guide](pilot-fedora45-1.0.2.md) for the finalized artifacts.

## Verified image

The existing production workflow published Fedora `45.20261115.0` from
`0f2d51a220f036b64068b3ac0839cfcef87d5a90`.

```text
ghcr.io/niri-tonic/niri-tomic-os@sha256:fb708fbd3a1ec6b7c2e20aedf701326185e0c9a72db15b916a2bfec16f921ae6
```

The index contains AMD64 and ARM64. Its signature was verified with this
repository's `cosign.pub`. AMD64 userspace checks passed for Niri configuration,
PAM library dependencies, required desktop packages and enabled greeter/GPU
profile services. A local installer prototype based on the same OS filesystem
also passed disposable VM installations and BIOS/UEFI boot with greetd/Niri running.
The new CI, graphical user login, Secure Boot and physical hardware
qualification remain incomplete; see [the audit](audit-2026-11-15.md).

## Host readiness

The current machine still boots `44.20261028.0` from
`ghcr.io/tdesaules/niri-tomic-os:44`. Its 16 GiB BIOS GPU reservation is confirmed.
Rootless and rootful Podman already use SQLite. Dependency resolution of all requested
layered packages succeeded in the published Fedora 45 image, without installing
them. The actual rpm-ostree rebase remains the definitive layering transaction.

The following commands are migration instructions, not commands already executed
on the host. Save home data, keyrings and container volumes before the pilot,
including both rootless and rootful workloads.

## Preserve the running deployment

Check that no unrelated deployment is pending:

```bash
rpm-ostree status
```

Identify and pin the booted index rather than assuming index zero:

```bash
BOOTED_INDEX=$(rpm-ostree status --json |
  jq -r '.deployments | to_entries[] | select(.value.booted) | .key')
sudo ostree admin pin "$BOOTED_INDEX"
systemctl --user mask --now topgrade.timer topgrade.service
```

## Configure the new repository's signature policy

Run from this trusted source checkout, whose public key was used for the
verification above. Preserve the current policy before merging the new entry:

```bash
sudo cp -a /etc/containers/policy.json \
  "/etc/containers/policy.json.before-fedora45-$(date +%Y%m%d%H%M%S)"
sudo install -m 0644 cosign.pub /etc/pki/containers/niri-tonic-os.pub
sudo python3 - <<'PY'
import json
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
path.write_text(json.dumps(policy, indent=2) + "\n")
PY
sudo tee /etc/containers/registries.d/niri-tonic-os.yaml >/dev/null <<'YAML'
docker:
  ghcr.io/niri-tonic/niri-tomic-os:
    use-sigstore-attachments: true
YAML
```

This adds explicit trust for the organization namespace without removing the
old repository's policy or changing unrelated registry entries.

## Rebase and verify

After choosing to start the pilot:

```bash
sudo rpm-ostree rebase \
  ostree-image-signed:docker://ghcr.io/niri-tonic/niri-tomic-os@sha256:fb708fbd3a1ec6b7c2e20aedf701326185e0c9a72db15b916a2bfec16f921ae6
rpm-ostree status
systemctl reboot
```

After boot:

```bash
cat /etc/os-release
rpm-ostree status
systemctl --failed
systemctl --user --failed
journalctl -u niri-strix-halo-128g-memory.service -b --no-pager
rpm-ostree kargs
```

Test graphical login, keyring access, audio, Wi-Fi, suspend/resume, gaming and a
representative GPU workload. If the profile staged its argument, reboot once
more and check:

```bash
cat /sys/module/ttm/parameters/pages_limit
grep . /sys/class/drm/card*/device/mem_info_{vram,gtt}_total
```

Expected: `20971520` pages, approximately 80 GiB GTT, 16 GiB VRAM.
The dynamic ceiling is not an allocation guarantee.

## Update tracking and rollback

The pilot reference pins one immutable digest. It will not follow future
releases. After qualification, explicitly choose the intended update channel
(`:45` stays on Fedora 45; `:latest` follows the production Fedora version),
then restore Topgrade:

```bash
systemctl --user unmask topgrade.timer topgrade.service
systemctl --user start topgrade.timer
```

If the new deployment cannot boot, select the pinned October Fedora 44 deployment
from the boot menu. If it boots but needs rollback, first inspect
`rpm-ostree status`: the GPU-profile transaction can create another Fedora 45
deployment, so `rpm-ostree rollback` does not necessarily select Fedora 44.
Use the explicitly pinned Fedora 44 deployment when necessary.
OSTree rollback does not restore home or `/var` application data.
