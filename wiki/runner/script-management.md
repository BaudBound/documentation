---
title: Script Management
description: Import, inspect, approve, enable, update, run, and remove BaudBound packages.
tags: [runner, scripts, getting-started]
---
# Script Management

The desktop Scripts view and `baudbound script` commands operate on the same installed scripts.

In the commands below, `SCRIPT` means either the script name shown by `baudbound script list` or its manifest ID. `PACKAGE` means the path to an exported `.bbs` file. Do not type these placeholders literally.

For normal desktop use, open **Scripts** to perform these operations without entering commands. Use the CLI examples when operating a headless runner or diagnosing a problem.

## Lifecycle

```text
.bbs file -> validated import -> installed -> reviewed -> approved -> enabled -> triggered
                                      |                         |
                                      +---- manual run ---------+
                                      |
                                      +---- update or remove
```

**Installed** means the runner has accepted and copied a package. **Approved** means the operator accepts that exact hash and permission set. **Enabled** means eligible listener triggers may register. These states are independent.

## Import and inspect

```text
baudbound script import PACKAGE
baudbound script list
baudbound script inspect SCRIPT
```

Run the commands in that order. Import validates and stores the package. List shows the name or ID accepted by later commands. Inspect displays the exact imported revision, target runtime, nodes, requested access, integrity state, and approval state.

Example package paths:

```powershell
baudbound script import "C:\Users\Alice\Downloads\desk-lights.bbs"
```

```text
baudbound script import "$HOME/Downloads/desk-lights.bbs"
```

Import validates the archive, manifest, executable graph, node configuration, integrity hashes, minimum runner version, target runtime, permissions, and capabilities before installation. A successful import prints the installed script name and ID. Import does not create Webhook or WebSocket credentials. Those credentials are created only after you review and approve the package. A failed import does not install a partially accepted package.

In the desktop app, choose **View details** on a script row to read the descriptive information stored in the verified package manifest. The dialog shows the author, description, website, source, tags, creation details, minimum BaudBound version, target runtimes, package identity, health, permissions, triggers, and recent runs when those values are available. These fields describe the package but do not replace the approval review. Always review permissions, capabilities, risk, and package integrity separately.

The runner stages validation before durable installation. Rejection leaves existing script records, package files, approvals, variables, secrets, and registrations unchanged.

Display names are not guaranteed unique. The manifest ID is stable and unambiguous. When a name matches more than one installed script, use the ID printed by `baudbound script list` or inspection rather than guessing.

**Verified** means package content matches its integrity information. **Not verified** means the package lacks valid integrity proof or its bytes no longer match. Re-export an unverified package from the current editor instead of modifying its archive.

## Approve

```text
baudbound script approval SCRIPT
baudbound script approve SCRIPT
```

Run `approval` first and review its output. Check author information, target runtime, requested capabilities, risk, secret declarations, and included nodes. If the information matches the workflow you expect, run `approve`.

**Approved for this version** means the decision matches the installed content and capabilities. **Review required** means approval is missing or no longer matches. Approval does not fix validation, compatibility, policy, or missing-secret errors.

Approval belongs to an exact package revision, not a filename. Updating package content invalidates the previous approval.

When approval creates credentials for Webhook or WebSocket triggers, the CLI prints each new token and the desktop app opens a token dialog. Save the values immediately because the runner stores only their hashes. Approving an unchanged trigger again does not replace its existing token.

## Run and enable

Run a manual trigger:

```text
baudbound script run SCRIPT
baudbound script logs --script SCRIPT
```

`run` starts the script's manual trigger. It fails when the package has no manual trigger. Expand the script in the desktop Scripts view or run `baudbound script triggers SCRIPT` to see the available trigger node IDs.

To run a specific trigger with controlled JSON input:

```text
baudbound script run SCRIPT --trigger TRIGGER --payload-json '{"test":true}'
```

Replace `TRIGGER` with the node ID shown by trigger inspection. Listener-generated production events normally use the background service instead.

The JSON after `--payload-json` is test data for the selected trigger. It is not a runner setting. See [Supplying trigger test data](cli-reference.md#supplying-trigger-test-data) for a complete example and the variable names created from the JSON.

Enable the package when its listener-based triggers should load in the background:

```text
baudbound script enable SCRIPT
baudbound serve
```

Enablement does not bypass validation or approval. A disabled script remains installed and inspectable, but it cannot start a manual run, automatic run, queued trigger event, or sub-script run. Its schedule, webhook, serial, file, process, startup, WebSocket, and hotkey listeners are not loaded by the background runner. Disabling does not force-cancel a run that is already executing.

The foreground `baudbound serve` command is mainly for headless operation and testing. Desktop users can start the desktop-owned background runner from **Service** instead.

## Update or remove

Export the edited project from the editor, then update using the new package path:

```text
baudbound script update PACKAGE
baudbound script inspect SCRIPT
baudbound script approve SCRIPT
```

Update succeeds only when the package has the same manifest identity as an installed script. It invalidates the old approval because the content changed. Inspect and approve the replacement before running it.

Open the same saved editor project when creating an update. Its later exports retain the manifest identity automatically. Duplicating the editor project or choosing **Import copy** creates a different identity and must be imported as another script instead of used with Update.

Update preserves the script's enabled state, import identity, persistent variables, configured secret values, network tokens for unchanged trigger node IDs, and historical runs. A newly added Webhook or WebSocket trigger receives a token only after the updated package is approved. The CLI prints that token once and the desktop app shows it once. The previous approval remains recorded against its old hash and is reported as stale until the new revision is approved. When the imported filename changes, the old stored package copy is removed after the replacement is committed.

Remove an installed script with:

```text
baudbound script remove SCRIPT
```

Confirm the script is no longer listed with `baudbound script list`. Review or export any run information you need before removal.

Removal deletes the installed package copy and script row. Database foreign-key cleanup removes its approval, script-scoped persistent variables, and encrypted secret values. Global variables remain because they are runner-wide. Historical run records are retained by current storage so past diagnostics remain available, but they no longer make the removed script executable.

> Removal is not a temporary disable operation. Use `baudbound script disable SCRIPT` when you want to keep package state and stop unattended listeners.
{.is-warning}

## Direct remote packages

The desktop **Import** command accepts a local `.bbs` file or a direct public HTTPS address that returns a `.bbs` package. The desktop **Update** command accepts the same two package sources.

Repository addresses do not belong in Import or Update. Add a `repository.json` address through **Browse Scripts** instead.

A remote package is downloaded into protected temporary storage. The runner validates it and displays its identity, version, SHA256, target, risk, permissions, and capabilities before installation. Installing does not approve, enable, or run it.

Remote addresses must use public HTTPS destinations. The runner rejects credentials in URLs, unsafe redirects, local addresses, private network addresses, oversized responses, unexpected filenames, hash mismatches, reused versions with changed bytes, and downgrades.

## Browse and manage repositories

Open **Browse Scripts** to view scripts published through configured repositories.

The official BaudBound repository is configured automatically. It can be disabled, but it cannot be removed. The Official label identifies who manages the list. It does not bypass package validation or guarantee that a script is harmless.

To add another repository:

1. Choose **Repository management**.
2. Choose **Add repository**.
3. Enter a public HTTPS address ending in `repository.json`.
4. Choose **Preview repository**.
5. Review its name, description, address, and script count.
6. Choose **Add repository** to confirm.

Adding or refreshing a repository contacts its hosting server. The server can observe your public IP address and request time. BaudBound does not send script secrets, browser credentials, cookies, or private authentication headers.

Enabled repositories refresh in the background when the desktop runner starts and after the configured update check interval has elapsed. The refresh runs outside the interface thread, so you can continue using BaudBound while it downloads and validates the repository.

Repository controls can:

1. Enable or disable a repository.
2. Refresh one repository.
3. Refresh every enabled repository.
4. Remove a user-added repository after confirmation.

Removing a repository deletes its cached browser entries. It does not uninstall scripts that were installed from it. Those scripts remain installed, but their repository update source becomes unavailable.

If a refresh fails, the last valid cached script list stays available and the repository shows its latest error. Invalid or incomplete data never replaces a valid cache.

The background repository refresh updates the Browse Scripts catalog. It does not download script packages and it does not install anything. The **Automatic update checks** setting for an installed script is separate. That setting controls whether BaudBound checks the repository entry for that installed script and reports a newer version.

Search is available directly above the script list. Choose **Filters** to narrow scripts by repository, target, risk, permission, capability, and installation state. Select the filters you need, then choose **Apply filters**. Open a result to read its complete description, links, compatibility, permissions, capabilities, version, and release notes.

Browser entries can show:

| State | Meaning |
| --- | --- |
| **Not installed** | The script is available for package review |
| **Installed** | This version is already installed from the same repository |
| **Update available** | The same repository publishes a newer version |
| **Incompatible** | The script target is not supported by this runner |
| **Unavailable** | Repository information did not match the downloaded package |
| **Installed from another repository** | A repository with the same script ID cannot take over the installed script |

Choose **Install** or **Review update** to download a selected package. The runner checks its exact size, SHA256, identity, version, runtime, permissions, capabilities, and risk before opening package review.

Repository permissions, capabilities, target, and risk are previews. The downloaded `.bbs` package remains authoritative.

When repository information differs from the validated package, BaudBound blocks the operation and flags the repository. A user-added repository can be kept or removed. The official repository is also blocked on mismatch, but it cannot be removed.

## Installed script update checks

When an installed package declares a repository URL, its details dialog contains an **Updates** section. Choose **Check for updates** to check only that script. Choose **Check updates** on the main Scripts page to check every configured script.

The main page can show:

| State | Meaning |
| --- | --- |
| **Not configured** | The package does not declare a repository URL |
| **Not checked** | A repository URL exists but no successful check matches it yet |
| **Up to date** | The repository does not publish a newer version |
| **Update available** | The repository publishes a newer version for the exact script ID |
| **Check failed** | The repository could not be fetched or validated |
| **Unavailable** | The installed package metadata cannot be inspected |

Choose **Review update** to download the discovered package. The runner fetches the repository and verifies the package again before showing the review. Replacing the installed package makes the previous approval stale. Review and approve the new exact hash before enabling or running it.

**Update all** creates a review queue. It does not approve packages as a group. Review, install, skip, or cancel each package separately.

Automatic checks are disabled for every script by default. Enabling **Automatic update checks** requires confirmation because the repository server can observe the runner's public IP address and request time. Automatic checks only discover a new version. They never download or install a package.

> A valid package is not automatically trustworthy. Install scripts only from publishers you trust. BaudBound checks structure, integrity, compatibility, declared access, and exact package bytes. It cannot determine the intentions of a publisher or script.
{.is-warning}

Authors can read [Publishing Script Repositories](../editor/publishing-script-repositories.md) for the complete repository and publishing workflow.

## Desktop equivalents

| CLI task | Scripts view |
| --- | --- |
| `baudbound script import PACKAGE` | **Import** |
| `baudbound script list` / `baudbound script inspect SCRIPT` | Script row and **View details** |
| `baudbound script approve SCRIPT` | Approval review dialog |
| `baudbound script enable SCRIPT` / `baudbound script disable SCRIPT` | Row action menu |
| `baudbound script run SCRIPT` | **Run** button |
| `baudbound script update PACKAGE` | **Update package** action |
| `baudbound script revoke-approval SCRIPT` | Revoke approval action |
| `baudbound script remove SCRIPT` | Remove action and confirmation dialog |

## Recovery links

- Package or hash problem: [Runs, Logs, and Troubleshooting](runs-logs-troubleshooting.md#package-hash-is-not-verified)
- Approval or risk question: [Approvals, Capabilities, and Risk](../security/approvals-capabilities.md)
- Missing secret: [Secrets](secrets.md)
- Trigger not active: [Background Service and Triggers](service-triggers.md)
- Backup before removal or migration: [Storage, Backups, and Recovery](storage-backups.md)
