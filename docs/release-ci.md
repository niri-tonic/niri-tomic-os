# Explicit OS and installer releases

The workflows use catalog 1.8.0 at
`211bdb660789b484f8af7d086ad216fad9e4da08`. This includes digest-preserving zstd
index publication and complete source-bound rootfs SBOM generation.
The first finalized OS release is
[1.0.2](https://github.com/niri-tonic/niri-tomic-os/releases/tag/1.0.2).
Its [pilot guide](pilot-fedora45-1.0.2.md) records the exact published digests.

## Observed activation results

- Native OS construction and userspace qualification passed on both architectures
  in [run 43949628860](https://github.com/niri-tonic/niri-tomic-os/actions/runs/43949628860).
  BlueBuild rendering must set `BB_BUILD_DRIVER=docker` for BuildKit mount syntax.
- Organization App token creation and Renovate dependency extraction passed in
  [dry-run 43949205855](https://github.com/niri-tonic/niri-tomic-os/actions/runs/43949205855).
  The old documentation links that failed maintenance were subsequently corrected.
- First release preparation
  [43953653795](https://github.com/niri-tonic/niri-tomic-os/actions/runs/43953653795)
  passed OS/installer construction and qualification, then failed while publishing
  the multi-platform index because Skopeo requested a format conversion under
  `--preserve-digests`. The release was not finalized. Catalog 1.8.0 declares the
  zstd capability annotation before hashing its new index, preserving exact
  native artifacts.
- Direct image-source Syft scanning exhausted memory before cataloging. Errors
  remain advisory and are recorded as failures. A complete extracted-rootfs scan
  passed locally at a 7 GiB scan limit with identical filesystem package/file
  coverage. The new dedicated `oci/new-atomic-sbom` action verifies native
  manifest and layer hashes, unpacks without running the target OS, and requires
  coverage of the installed RPM inventory before exposing the SBOM.
- Corrected maintenance
  [44011407312](https://github.com/niri-tonic/niri-tomic-os/actions/runs/44011407312)
  passed App authentication, Renovate dry-run, links, complete SBOM generation and
  Grype for both architectures. Scanner RSS was 1.662 GiB/1.726 GiB; RPM coverage
  was 1,559/1,559 and 1,548/1,548. The original summary's 32 MiB input cap rejected
  the complete SBOMs; the raw artifacts and scanner outcomes remained valid.
- Preparation
  [44025291332](https://github.com/niri-tonic/niri-tomic-os/actions/runs/44025291332)
  and tag verification
  [44030839334](https://github.com/niri-tonic/niri-tomic-os/actions/runs/44030839334)
  finalized 1.0.2. Earlier 1.0.0/1.0.1 tags remain immutable records of preparations
  whose tag lint failed; they were not moved to repair generated changelogs.
- Anonymous full ISO download, all three payload checksums, and both OCI
  signatures were independently verified for 1.0.2.

## Development

Pushes run applicable source checks and native AMD64/ARM64 OS qualification.
They do not publish production channels. A generated Containerfile is built with
the dedicated Atomic BuildKit action. The exact archive and its source-bound
receipt are transferred to a separate test job.

## Release preparation and tag verification

From `main`, dispatch **release**. For the first official release, explicitly
enable `first-release` to select `1.0.0`. Later versions follow Conventional
Commits: features are minor, fixes/performance changes patch, breaking changes
major. A routine-only change is a successful no-op.

Preparation builds both native OS archives and an AMD64 ISO containing the exact
AMD64 archive. Qualification verifies userspace, media integrity and embedded OCI
identity. The release job verifies the complete bundle before publishing the OS
index and installer OCI artifact. A job-local App token atomically advances
`main` and creates the annotated version tag with generated release evidence.

The tag workflow downloads the immutable published objects and qualifies them
again without rebuilding. It then publishes the GitHub Release and promotes
`:45` and `:latest` for the newest stable version. Older stable releases do not
move those channels. Fedora's major version stays 45; Git release SemVer is a
separate identity.

ISO tar-export checksums may differ after a registry round-trip. Tag verification
compares native manifests while retaining the original AMD64 archive checksum in
the ISO build receipt. The original ISO receipt travels with the ISO and checksum
sidecar and is never replaced by a fabricated tag-time build receipt.

## Maintenance and credentials

Dispatch **maintenance** on `main` with `dry-run: true` to test App access and
Renovate extraction first. Scheduled maintenance performs real dependency
updates. Fedora remains constrained to version 45 in Renovate.

The organization must expose `BOT_APP_ID` and `BOT_APP_PRIVATE_KEY` to this
repository and install the App with the required repository permissions.
Each job creates and revokes its own scoped installation token. `SIGNING_SECRET`
is passed only to Cosign's environment; it is never included in artifacts.

The former installer gist and ttl.sh publication are retired. Their old
repository variables and `GIST_TOKEN` are not consumed by these workflows.

## Evidence and retries

Candidate artifacts expire after seven days. Advisory reports are retained for
30 days in development/maintenance and 90 days for releases.
Failed-job reruns may reuse successful producer artifacts only from the same
workflow run, source and compatible attempt. Missing or expired required
artifacts require rerunning their producers. Publication failures are blocking.

Security scans and signatures are advisory. Reports distinguish errors,
skipped operations, findings and successful verification rather than treating
missing evidence as clean or signed.

Media qualification reports `boot_tested: false`: it checks the ISO and embedded
OS, not a VM installation. The separately observed BIOS/UEFI installation and
greetd/Niri startup results are in the
[Fedora 45 audit](audit-2026-11-15.md). Secure Boot, real user login, keyring
migration and physical GPU behavior still require pilot checks.
