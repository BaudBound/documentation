---
title: Publishing Script Repositories
description: Publish one or more BaudBound scripts through GitHub or another public HTTPS file host.
tags: [editor, scripts, repositories, publishing]
---
# Publishing Script Repositories

A script repository is a public `repository.json` file that lists BaudBound scripts and tells the runner where their `.bbs` packages can be downloaded.

A repository can contain one script or many scripts. It does not need a server application, database, account system, or GitHub API token. You can host the JSON file and packages in a public GitHub repository or on another public HTTPS file host.

Repository refreshes only discover information. They never install, approve, enable, or run a script. A person must review every downloaded package before it is installed.

## Files you publish

A repository contains:

1. One file named `repository.json`.
2. One versioned `.bbs` package for each published script version.

A simple layout looks like this:

```text
repository.json
packages/
  SCRIPT_ID/
    inventory-scanner-1.0.0.bbs
    inventory-scanner-1.1.0.bbs
```

Keep every published package file unchanged. When you release another version, add a new file with the new version in its name. Then change `repository.json` to point to the new file.

## Prepare the editor project

Open **Settings** in the editor and complete these fields:

| Field | What to enter |
| --- | --- |
| **Script version** | A semantic version such as `1.0.0` or `1.2.0` |
| **Repository URL** | The final public HTTPS address of `repository.json` |
| **Project ID** | Keep this value unchanged for every version of the same script |
| **Source** | The optional source code or project page for the script |

The repository URL must end in `repository.json`. It may point to a file that you have not published yet. The editor checks the address format without contacting the server.

New projects use version `1.0.0`. Increase the version whenever published package bytes change.

Use:

1. `1.0.1` for a compatible fix.
2. `1.1.0` for a compatible feature.
3. `2.0.0` for an incompatible change.

The runner rejects an older version. It also rejects a changed package that reuses an existing version.

## Export the package

1. Choose **Export**.
2. Review the project information and calculated access.
3. Complete verification.
4. Generate the package.
5. Choose **Download package**.
6. Keep the export window open.
7. Choose **Create repository entry**.
8. Enter the final public package URL.
9. Write optional Markdown release notes.
10. Review the generated JSON.
11. Choose **Copy repository JSON**.

The editor calculates the package size and SHA256 from the exact `.bbs` bytes offered by **Download package**. The JSON and package therefore describe the same export.

The editor shows the JSON for review and copying. It does not download a separate JSON file. Create or update `repository.json` in your publishing repository and paste the reviewed content there.

## Repository format

A one-script repository looks like this:

```json
{
  "format": "baudbound.repository",
  "format_version": 1,
  "name": "Inventory Scanner",
  "description": "Scripts published by Example Author.",
  "homepage": "https://example.com/baudbound",
  "scripts": [
    {
      "script_id": "6db0f09c-2d76-4ea3-bb6b-9a093a04d8f7",
      "name": "Inventory Scanner",
      "summary": "Moves scanned inventory items between locations.",
      "description": "Reads a configured serial scanner and sends inventory commands.",
      "author": "Example Author",
      "website": "https://example.com/inventory",
      "source": "https://github.com/example/inventory-script",
      "license": "PolyForm-Noncommercial-1.0.0",
      "target_runtime": "Generic Desktop",
      "minimum_runner_version": "2.0.0",
      "risk_level": "high",
      "tags": [
        "inventory",
        "serial"
      ],
      "permissions": [
        "serial_input",
        "http_request"
      ],
      "capabilities": [
        "trigger.serial_input",
        "action.http"
      ],
      "latest": {
        "version": "1.2.0",
        "package_url": "https://raw.githubusercontent.com/example/baudbound-scripts/main/packages/6db0f09c-2d76-4ea3-bb6b-9a093a04d8f7/inventory-scanner-1.2.0.bbs",
        "sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
        "size": 123456,
        "published_at": "2026-07-23T12:00:00Z",
        "release_notes": "Adds the new inventory workflow."
      }
    }
  ]
}
```

The top-level fields describe the repository:

| Field | Meaning |
| --- | --- |
| `format` | Must be `baudbound.repository` |
| `format_version` | Repository contract version supported by the runner |
| `name` | Repository name shown in the runner |
| `description` | Short explanation of who publishes the scripts |
| `homepage` | Optional public page for the repository |
| `scripts` | List containing every published script |

Each script entry contains:

| Field | Meaning |
| --- | --- |
| `script_id` | Stable project ID from the editor |
| `name` | Script name |
| `summary` | Short text used in browser results |
| `description` | Longer Markdown description |
| `author` | Publisher or author name |
| `website` | Optional script website |
| `source` | Optional source code or project URL |
| `license` | License identifier or name |
| `target_runtime` | Runtime required by the package |
| `minimum_runner_version` | Oldest compatible runner version |
| `risk_level` | Preview of the package risk |
| `tags` | Search terms |
| `permissions` | Preview of declared package permissions |
| `capabilities` | Preview of required package capabilities |
| `latest` | Current published package |

The `latest` object contains:

| Field | Meaning |
| --- | --- |
| `version` | Exact package version |
| `package_url` | Public HTTPS address that returns the `.bbs` file |
| `sha256` | Lowercase SHA256 of the exact package bytes |
| `size` | Exact package size in bytes |
| `published_at` | Publication time in UTC |
| `release_notes` | Markdown notes shown during review |

## Publish several scripts

The editor generates a complete repository with one script. To publish several scripts from the same repository:

1. Export each script and create its repository JSON.
2. Keep one top-level `format`, `format_version`, `name`, `description`, and `homepage`.
3. Copy each generated item from its `scripts` list.
4. Put all copied items inside one `scripts` list.
5. Make sure every `script_id` is unique.
6. Validate the final JSON before publishing it.

Do not merge two complete JSON documents by placing one after another. A JSON file has one top-level object and one `scripts` list.

## Publish with GitHub raw files

GitHub can host the complete repository without GitHub Pages.

1. Create a public GitHub repository.
2. Add `repository.json` at its root.
3. Add each package under `packages/SCRIPT_ID/`.
4. Commit and push the files.
5. Open each raw address in a browser and confirm that it returns the file itself.

The repository URL looks like:

```text
https://raw.githubusercontent.com/USERNAME/REPOSITORY/main/repository.json
```

A package URL looks like:

```text
https://raw.githubusercontent.com/USERNAME/REPOSITORY/main/packages/SCRIPT_ID/inventory-scanner-1.2.0.bbs
```

Do not use a URL containing `/blob/`. A normal GitHub file page returns an HTML website instead of the requested JSON or package bytes.

GitHub Pages can also serve the same files. Raw GitHub URLs are usually simpler because no Pages configuration is needed.

## Publish on another host

Any public static HTTPS host can be used when:

1. The repository address ends in `repository.json`.
2. Package addresses return their `.bbs` files directly.
3. Redirects stay on public HTTPS destinations.
4. Files do not require cookies, login, or private authentication.
5. The server permits ordinary download requests from the runner.

Localhost, private network addresses, plain HTTP, URLs with embedded credentials, and private authenticated repositories are not supported.

## Safe publishing order

Publish a new version in this order:

1. Export and download the new `.bbs` package.
2. Add the new package under a new versioned filename.
3. Confirm that its public URL downloads the expected bytes.
4. Create the repository entry from the same editor export.
5. Update `repository.json` last.

Publishing the package first prevents runners from discovering a version whose package is not available yet.

## Verify the package hash

On Windows PowerShell, run:

```powershell
Get-FileHash .\inventory-scanner-1.2.0.bbs -Algorithm SHA256
```

On Linux, run:

```text
sha256sum ./inventory-scanner-1.2.0.bbs
```

Compare the lowercase result with `latest.sha256`. Also confirm that `latest.size` matches the file size.

## What the runner trusts

Repository descriptions, permissions, capabilities, runtime, and risk are browsing previews. They do not grant access and are never used as package approval.

Before installation or update, the runner:

1. Downloads the exact package through its protected HTTPS client.
2. Checks the expected size and SHA256.
3. Validates package schemas and compatibility.
4. Calculates permissions, capabilities, and risk from the package.
5. Compares the package with the repository claims.
6. Blocks the operation when the information differs.
7. Opens the normal package review when every check succeeds.

An information mismatch may mean that the repository is stale, the publisher made a mistake, or someone changed a file. The runner flags the repository and does not install the package.

> Install scripts only from publishers you trust. Validation can confirm structure, integrity, compatibility, and declared access. It cannot determine whether a publisher or script has good intentions.
{.is-warning}

## Privacy

Refreshing a repository contacts its hosting server. The server can observe the runner's public IP address, request time, and ordinary request metadata.

The runner does not send script secrets, browser credentials, cookies, or authentication headers to the repository.

Automatic checks are disabled for each installed script by default. Enabling them requires confirmation.

## Common problems

| Problem | What to check |
| --- | --- |
| Repository URL is not configured | Add the final public `repository.json` address in editor Settings and export again |
| URL returns a GitHub page | Replace the `/blob/` address with a raw file address |
| Repository is invalid | Validate its JSON shape, field names, values, and duplicate script IDs |
| Script is missing | Confirm that the installed script ID still exists in the repository |
| Package hash or size differs | Publish the exact exported package or recreate the entry from those exact bytes |
| Package information differs | Correct the repository entry and refresh the repository |
| Version is not newer | Increase the script version before publishing changed package bytes |
| Destination is restricted | Use a public HTTPS host instead of a local or private address |
| Refresh failed | Confirm that the repository and package addresses are publicly available |

Read [Script Management](../runner/script-management.md#browse-and-manage-repositories) for repository controls in the runner.
