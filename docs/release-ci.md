# Explicit OS and installer releases

The workflows use catalog 1.7.3 at
`45c9ce9160b50d49a7167543f2286e000656e565`. Its preparation and tag qualification
succeeded before consumer activation. The first end-to-end OS release remains
to be exercised; catalog qualification does not prove organization credentials
or production installer publication.

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
