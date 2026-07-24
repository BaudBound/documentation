---
title: Target Runtimes and Platform Support
description: Select the exact Windows and Linux environments where a package is allowed to run.
tags: [editor, runner, compatibility, platforms]
---
# Target Runtimes and Platform Support

A target runtime tells BaudBound exactly where a script is allowed to run. Every project must select at least one target. A project can select several targets when the same script is intended to work in several environments.

**Desktop** means a user is signed in to a graphical Windows or Linux session. Desktop workflows can interact with visible applications when the selected operating system supports the requested node.

**Headless** means the runner works without a graphical desktop session. This is common on servers. Headless workflows can use files, network requests, schedules, and other background features, but they cannot click a mouse, type into a window, show a desktop notification, or read a screen pixel.

The editor offers four targets:

| Target | Intended use |
| --- | --- |
| **Windows Headless** | Windows service or non-interactive automation without desktop APIs |
| **Windows Desktop** | Signed-in Windows session and Windows-native desktop behavior |
| **Linux Headless** | Linux service automation without a graphical session |
| **Linux Desktop** | Linux graphical session using currently implemented cross-platform desktop adapters |

## Choose targets

1. Select the operating system that will run the script.
2. Select **Desktop** when the script runs inside a signed-in graphical session.
3. Select **Headless** when the script runs as a service or from the normal CLI execution path.
4. Select both Windows and Linux only when the complete graph works on both systems.
5. Select both Desktop and Headless only when the complete graph avoids desktop-only features.

For example, a portable background script can select **Windows Headless** and **Linux Headless**. A portable desktop script can select **Windows Desktop** and **Linux Desktop** when all its nodes have implementations on both systems.

Paths, program names, serial port names, shell syntax, and external dependencies are not translated between operating systems. A graph may use portable nodes and still depend on operating system specific values.

> Selecting more targets makes the compatibility rules stricter. Every node and relevant option must work on every selected target.
{.is-warning}

## Desktop-only nodes

These nodes require a Desktop target:

- Hotkey trigger.
- Open Application.
- Clipboard.
- Keyboard and Type Text.
- Show Notification.
- Mouse Click and Move Mouse.
- Play Sound.

The editor checks node definitions, and the runner independently applies its corresponding compatibility rules.

## Windows Desktop-only behavior

These nodes support only Windows Desktop:

- MessageBox.
- Get Active Window.
- Window Focus.
- Get Pixel Color.

Process Status, Kill Process, and App / Process Started are otherwise portable, but selecting **Window title** match mode makes that configured node Windows Desktop-only.

## Definition rules

A node definition can constrain support in two ways:

- `desktopOnly: true` allows desktop targets and rejects headless targets.
- `supportedTargetRuntimes` lists an explicit narrower set.

When `supportedTargetRuntimes` is omitted, the node is available to all targets unless another rule such as `desktopOnly` or configuration-specific validation narrows it. Omission does not promise that arbitrary machine paths or external dependencies are portable.

## Enforcement stages

1. **Palette:** a node is offered only when it supports every selected target.
2. **Editor verification:** incompatible nodes produce blocking errors after a target change or package import.
3. **Package declaration:** export records every selected target with calculated capabilities.
4. **Runner package validation:** Rust recalculates node and option compatibility.
5. **Runner host validation:** the current runner mode must appear in the package target list.
6. **Native adapter:** a required adapter must still be available in the actual desktop session.

A manually edited package cannot bypass these stages.

## Runner target configuration

The desktop application executes packages for the current operating system's **Desktop** target. Normal `baudbound serve`, `baudbound script run`, and trigger dispatch commands execute packages for the current operating system's **Headless** target.

Package inspection and installation commands can manage both modes for the current operating system. They do not make a package executable in a mode that the package did not select.

`runner.target_runtimes` can restrict the allowed targets further. For example, a Linux service can explicitly allow only:

```toml
[runner]
target_runtimes = ["Linux Headless"]
```

Configuration cannot grant a target or action that the built runner does not implement.

## Examples

| Workflow | Suitable target |
| --- | --- |
| Scheduled HTTP request and log on either OS | Windows Headless and Linux Headless |
| Linux file watcher using `/srv/inbox` | Linux Headless |
| Windows process automation without desktop input | Windows Headless |
| Cross-platform notification workflow | Windows Desktop and Linux Desktop |
| Focus a Windows application and type text | Windows Desktop |
| Serial listener with Linux `/dev/ttyUSB0` mapping | Linux Headless or Linux Desktop, depending on other nodes |

For per-node configuration and support, use [Node Reference](node-reference.md).
