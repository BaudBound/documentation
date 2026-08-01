---
title: Release Engineering
description: Version, verify, sign, package, publish, and update BaudBound releases.
tags: [developers, releases]
---
# Release Engineering

BaudBound versions must agree across the Cargo workspace, Tauri configuration, and desktop UI package metadata. Release tags use `vMAJOR.MINOR.PATCH` and are created from a clean, verified `master` commit.

`scripts/verify-release-version.mjs` checks that the tag, root Cargo workspace, `tauri.conf.json`, and `ui/package.json` agree before packaging.

The release workflow builds a Windows NSIS installer plus Linux AppImage, Debian, and RPM packages. It signs updater artifacts, creates a draft GitHub release, publishes `latest.json` with platform URLs and signatures, and generates `SHA256SUMS`. The private updater key and password live in the protected `runner-release` GitHub environment. Only the public key is committed in application configuration.

Clone the `tooling` repository beside the runner repository, then use its interactive release helper:

```powershell
./release.ps1
```

It can check versions and prerequisites, run verification, create the tag, inspect artifacts, and publish only after explicit confirmation.

## Release workflow

1. Verify a clean `master` worktree and synchronized version metadata.
2. Run Rust, editor contract/schema, and desktop UI release gates.
3. Create and push `vMAJOR.MINOR.PATCH`.
4. GitHub verifies that the version tag is annotated and points to a commit contained in protected `master`, then repeats the quality gate on Ubuntu.
5. Windows builds the NSIS installer. Ubuntu 22.04 builds the AppImage, Debian package, and RPM package from the same revision.
6. Tauri signs updater artifacts and uploads platform signatures.
7. The workflow creates a **draft** GitHub release and generates `latest.json`.
8. It inspects the Debian and RPM metadata, dependencies, installed files, desktop entry, and package scripts.
9. A read-only job verifies every release asset.
10. A protected write job generates and uploads `SHA256SUMS` after verification succeeds.
11. A maintainer checks artifacts on clean supported machines, release notes, updater metadata, and install/update behavior.
12. Publish the draft only after every artifact passes.

The draft prevents a partially uploaded release from becoming the automatic update target.

## Installer entry points

The public commands at `https://get.baudbound.app/linux` and `https://get.baudbound.app/windows` are static scripts served by the `baudbound-get` container. The scripts request the latest published GitHub release, require exactly one matching platform installer, and verify the asset against GitHub's SHA-256 release digest before installation.

Linux release metadata is parsed with `jq`. The script identifies Debian, Ubuntu, or Fedora before downloading anything. It selects the matching native package, verifies its digest and metadata, then asks APT or DNF to install it. Unsupported distributions stop with links to the manual instructions and GitHub Releases. The hosted installer never installs the AppImage. Windows uses PowerShell's structured `Invoke-RestMethod` response and `Get-FileHash`.

Changes in the [`BaudBound/get`](https://github.com/BaudBound/get) repository run Linux, Windows, shell lint, and container endpoint tests. A successful non-pull-request build publishes:

```text
ghcr.io/baudbound/get:latest
```

The production server can use `compose.yaml` from the get repository as its deployment baseline. It binds the container to `127.0.0.1:8086`, so the public hostname must terminate HTTPS at a reverse proxy and forward to that loopback address. Verify all three endpoints after deployment:

```bash
curl --fail https://get.baudbound.app/healthz
curl --fail https://get.baudbound.app/linux
curl --fail https://get.baudbound.app/windows
```

The scripts deliberately use `Cache-Control: no-store` behavior so updates become available without stale proxy copies. Keep caching disabled for these endpoints.

## Protected settings

| Setting | Purpose |
| --- | --- |
| `GITHUB_TOKEN` | Create the draft and upload artifacts in the repository |
| `TAURI_SIGNING_PRIVATE_KEY` | Sign updater packages. Never commit it |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | Decrypt the private signing key in CI |

Loss of the private updater key prevents producing updates trusted by existing installations. Exposure allows an attacker with release publication access to sign malicious updates. Back it up offline, restrict the GitHub environment, and rotate only through a deliberately designed migration.

### Configure the protected environment

1. Open `BaudBound/baudbound` on GitHub.
2. Open **Settings**.
3. Open **Environments**.
4. Create an environment named `runner-release`.
5. Add `TAURI_SIGNING_PRIVATE_KEY` as an environment secret.
6. Add `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` as an environment secret.
7. Restrict deployment branches and tags to the version tags accepted by the release workflow.
8. Add a required reviewer when another trusted maintainer is available.
9. Keep **Prevent self-review** disabled while there is only one maintainer.
10. Remove the same two values from repository-level Actions secrets after a protected draft rehearsal succeeds.

The signed `package` jobs and final checksum publication job reference this environment. They pause before publication access is provided when approval is required. Only the package jobs reference the signing values. The checksum job can update the verified draft but does not receive the signing key through its environment. Quality and artifact verification jobs receive read-only repository access and no signing key.

Environment approval protects publication before a write-capable job starts. It cannot make approved third-party code trustworthy. Every external Action is therefore pinned to a complete commit SHA. When updating an Action, review the upstream release and commit, replace the SHA while preserving its readable version comment, then run the complete affected workflow.

### Release security checklist

- [ ] The release commit and tag contain the intended source and submodule revisions.
- [ ] Both target-specific `cargo deny check` gates pass.
- [ ] The production UI dependency audit passes.
- [ ] Formatting, Clippy, Rust tests, UI tests, and the production UI build pass.
- [ ] Every write-capable release job waits for the `runner-release` environment approval.
- [ ] No quality or verification job receives signing secrets.
- [ ] Windows and Linux packages are signed by the expected updater key.
- [ ] Native package metadata and clean installation tests pass.
- [ ] `latest.json`, signatures, checksums, and every expected asset pass inspection.
- [ ] The GitHub release remains a draft until manual platform checks are complete.

## Review and rollback

Install the Windows artifact on a clean supported Windows machine. Install the Debian package on clean Debian and Ubuntu machines. Install the RPM package on a clean Fedora machine. Launch the AppImage on representative compatible systems. Check first launch, config initialization, import, approval, execution, tray behavior, background operation, update discovery, package-specific update actions, and resulting version.

If the draft is broken, leave it unpublished, fix the source, and create a new patch version. Do not replace published artifact bytes under an existing version because clients and signatures may already reference them. For a published defect, remove it from update discovery when necessary and ship a new signed patch release.
