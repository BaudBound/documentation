---
title: Official Blacklist and Personal Block List
description: Understand security advisories, restrictions, quarantine, and local repository blocks.
tags: [security, repositories, blacklist]
---
# Official Blacklist and Personal Block List

BaudBound uses an Official blacklist to respond to reviewed security concerns involving a repository, publisher, domain, script, or exact package file.

The list comes from the BaudBound API at `api.baudbound.app`. The runner downloads only active public entries. It does not send your installed script list, package hashes, machine identity, or repository choices to the API.

The Official blacklist is different from your Personal block list.

| List | Controlled by | Purpose |
| --- | --- | --- |
| Official blacklist | BaudBound maintainers | Apply reviewed security advisories and mandatory restrictions |
| Personal block list | You | Hide and disable a repository on one runner installation |

Official restrictions cannot be ignored in the runner. A personal block can be removed whenever you choose.

## What the severity means

Every entry has one severity.

| Severity | What the runner does |
| --- | --- |
| Advisory | Shows the reason and advisory link without blocking the content |
| Restricted | Hides matching browser entries and blocks matching downloads, imports, approvals, installations, and updates |
| Quarantined | Applies the restricted behavior, disables matching installed scripts, removes their queued runs, and prevents new runs |
| Critical quarantine | Applies quarantine and requests cancellation of matching active runs |

Quarantine does not delete the package, approval history, variables, secrets, settings, or run history. This keeps the information available for inspection and recovery.

A run that is already active can finish when the severity is **Quarantined**. A **Critical quarantine** requests cooperative cancellation. The current node is allowed to return safely, then the runtime stops before starting another node.

When an entry is removed or its severity becomes lower, BaudBound removes the current restriction. It does not automatically enable a script that was quarantined. Review the script and enable it yourself when you are satisfied that it is safe.

## What can be matched

An entry can match one of these identities:

1. One exact repository address.
2. One publisher account on a supported hosting provider.
3. One domain name.
4. One stable script ID.
5. One exact package SHA256 value.

Domain matching uses the parsed hostname. It never uses a simple text prefix. A domain entry matches only the exact domain unless the advisory explicitly includes subdomains.

GitHub publisher entries recognize normal repositories, raw files, the GitHub repository API, GitHub Pages, Gists, and raw Gist files. The account name is taken from the trusted request address recorded by the runner. It is not taken from package text that a publisher can edit.

The runner checks the original address and every redirect when it downloads a repository or package. A restricted redirect stops the download before the next request is sent.

## Exact package restrictions and safe updates

A package entry applies only to one exact package hash.

The affected package cannot be approved, enabled, or run when the entry requires quarantine. You can still check for a replacement update when the repository, publisher, domain, and script are not restricted.

The replacement does not inherit trust from the old package. BaudBound downloads it, calculates its hash, validates its script ID and repository information, and checks the Official blacklist again. The replacement is accepted only when the new package passes every normal package and security check.

This behavior lets a publisher replace one affected package without removing protection from the affected file.

## Security tab

Open **Security**, then find **Blacklist**.

This section shows:

1. Whether the last API check succeeded.
2. Whether the local cache is current or stale.
3. How many active entries were received.
4. Matching incidents for installed scripts and configured repositories.
5. The title, reason, severity, publication time, affected identity, and advisory link.

Choose **Check now** to request a fresh copy.

If the API is unavailable, the runner keeps the last complete and valid cache. A failed request never clears known entries. On a first offline start with no cache, the runner continues and reports that blacklist information is unavailable.

The cache is checked during startup before triggers are registered. New entries are also applied when a background refresh completes.

## Repository Management

Open **Browse Scripts**, then choose **Repository management**.

An affected repository remains visible there even when its scripts are hidden from the browser. The row shows its state and the complete public advisory.

For a restricted repository:

1. Refresh and enable controls are unavailable.
2. The repository can be removed after confirmation.
3. Removing it clears its cached browser entries.
4. Installed scripts and their stored data remain on the runner.

Removing a repository cannot bypass an Official blacklist entry. Scripts already quarantined remain disabled.

## Personal block list

Use the block control in **Repository management** when you no longer want one runner installation to use a repository.

A personal block:

1. Disables the repository.
2. Stops repository refreshes.
3. Hides its scripts from the browser.
4. Keeps installed scripts and their data.
5. Does not affect other BaudBound users.

The repository remains visible in management while it is configured. If the repository has been removed, its address remains in a separate blocked repository row so you can remove the personal block later.

Removing a personal block does not automatically enable the repository. It also does not override an active official restriction.

## Privacy

An Official blacklist request contains an ordinary request for the active public records. The API server can observe the runner public IP address, request time, runner user agent, and normal connection metadata.

The request does not include:

1. Installed scripts.
2. Package hashes.
3. Repository selections.
4. Script secrets.
5. Persistent variables.
6. Run history.
7. Machine identifiers.

The runner requests an explicit public field list. Hidden maintainer notes are not requested, cached, logged, or displayed.

Continue with [Script Management](../runner/script-management.md) for repository and update tasks, or [Security Model](index.md) for the complete trust model.
