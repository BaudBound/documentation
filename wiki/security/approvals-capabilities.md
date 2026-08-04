---
title: Approvals, Capabilities, and Risk
description: Review BaudBound permissions, capabilities, calculated risk, policy gates, and per-revision approval.
tags: [security, approval, permissions, capabilities, risk]
---
# Approvals, Capabilities, and Risk

Before a script can run, BaudBound calculates what the script needs and shows that information for review.

**Permissions** describe operations that matter to the person approving the script. Examples include reading a file, controlling the keyboard, or starting a process.

**Capabilities** are internal feature categories used to confirm that package declarations match the executable graph. Users do not add or approve capabilities separately.

**Risk** shows how carefully the package should be reviewed. It is calculated from the highest risk operation in the package.

**Approval** records that the current package revision and its calculated access were accepted. A changed package needs a new approval. Approval never disables validation or platform checks.

## Risk levels

| Level | Meaning | Examples |
| --- | --- | --- |
| **Low** | Local workflow behavior with limited external impact | logging, delay, calculation, text transformation |
| **Medium** | Reads, controlled writes, communication, or visible desktop effects | file read/copy, HTTP request, clipboard, notification, serial write |
| **High** | Broad mutation, input control, persistent or global data, secrets, listeners | keyboard/mouse, file write, webhook bind, secret read |
| **Dangerous** | Arbitrary process or command behavior and irreversible operations requiring focused review | run process, shell command, file deletion |

Risk is a review priority, not a verdict. A low-risk workflow can still disclose data through careless logging, and a high-risk workflow can be legitimate when its scope is understood.

## Permission reference

The runner derives permissions from executable action types and variable scopes. It rejects missing declarations, unused declarations, duplicates, or a declared risk lower or higher than the calculated value.

| Permission | Risk | Behavior that requires it |
| --- | --- | --- |
| `beep` | Low | Generate a system beep |
| `calculate` | Low | Evaluate supported arithmetic |
| `delay` | Low | Pause execution |
| `log` | Low | Write a runner log entry |
| `text.transform` | Low | Transform text or structured text input |
| `url.parse` | Low | Parse a URL into structured fields |
| `value.convert` | Low | Convert a value between supported types |
| `variable.local.set` | Low | Write run-scoped data |
| `webhook.response` | Low | Complete the current waiting webhook |
| `application.open` | Medium | Open an application through a native adapter |
| `clipboard.read` | Medium | Read text from the native clipboard |
| `clipboard.write` | Medium | Replace native clipboard content |
| `file.copy` | Medium | Copy a file |
| `file.download` | Medium | Download network content to a file |
| `file.move` | Medium | Move or rename a file |
| `file.read` | Medium | Read a bounded relative file path |
| `file.watch.limited` | Medium | Observe changes beneath a statically bounded script-workspace path |
| `http.request` | Medium | Send an outbound HTTP request |
| `messageBox.show` | Medium | Display an interactive desktop dialog |
| `notification.show` | Medium | Display a desktop notification |
| `formDialog.show` | Medium | Interrupt the desktop user with a configured form or confirmation dialog |
| `process.query` | Medium | Inspect process state |
| `process.observe` | Medium | Observe process-start metadata for a configured Process Started trigger |
| `screen.pixel.read` | Medium | Read a screen pixel on supported desktops |
| `serial.write` | Medium | Send data to a configured serial device |
| `sound.play` | Medium | Play package or filesystem audio |
| `variable.persistent.set` | Medium | Store script data between runs |
| `websocket.write` | Medium | Write to a connection associated with a WebSocket run |
| `window.query` | Medium | Read active-window information |
| `file.delete.limited` | High | Delete a bounded relative file path |
| `file.write.limited` | High | Create, overwrite, or append within a bounded relative path |
| `keyboard.control` | High | Send native keys or text |
| `mouse.control` | High | Move or click the native pointer |
| `network.webhook` | High | Register an inbound HTTP route. Public exposure is controlled separately by runner policy |
| `network.websocket` | High | Register an inbound WebSocket route. Public exposure is controlled separately by runner policy |
| `process.kill` | High | Terminate a matching process |
| `script.run` | High | Execute another installed and approved script |
| `secret.read` | High | Resolve runner-supplied secret declarations |
| `serial.input` | High | Read unattended input from a serial mapping |
| `trigger.startup` | High | Start a script automatically when its service loads |
| `variable.global.set` | High | Change runner-global data |
| `window.focus` | High | Change foreground-window focus |
| `file.delete.any` | Dangerous | Permanently remove a file at an unbounded path |
| `file.read.any` | Dangerous | Read from an absolute, sensitive, parent-traversing, or runtime-selected path |
| `file.watch.any` | Dangerous | Observe an absolute, sensitive, parent-traversing, or runtime-selected path |
| `file.write.any` | Dangerous | Write to an absolute, sensitive, parent-traversing, or runtime-selected path |
| `process.run` | Dangerous | Start an executable with arguments, wait for completion, and capture its output |
| `process.shell` | Dangerous | Execute a command through a shell interpreter |

Manual, Schedule, and Hotkey triggers do not need an additional user-data observation permission. File Watch requires `file.watch.limited` when its configured path is provably bounded to the script workspace and `file.watch.any` when it is absolute, sensitive, parent-traversing, runtime-selected, or otherwise cannot be proven bounded. Process Started requires `process.observe` because it continuously inspects process metadata.

The editor resolves statically knowable trigger inputs before deriving these permissions. If a path depends on data that cannot be proven at package-validation time, derivation fails toward `file.watch.any`; it never silently treats an unknown path as limited. The runner independently derives and enforces the same result before registration.

File permissions depend on the configured path. A bounded relative path uses `file.read`, `file.write.limited`, or `file.delete.limited`. It resolves inside `workspaces/SCRIPT_ID` under the runner home, giving each installed script its own limited workspace. Parent traversal and symbolic-link components that escape this directory are rejected.

An absolute path, a sensitive system location, a path containing runtime variables, or another path whose location cannot be proved in advance requires `file.read.any`, `file.write.any`, or `file.delete.any` according to the operation. Copy and Move also declare `file.copy` or `file.move`. Download declares `file.download`. These permissions do not prevent an approved script from using the selected path. They make the broader access explicit during review.

## Capability reference

Capabilities describe runtime subsystems, not user consent by themselves. Current categories are:

| Family | Capabilities |
| --- | --- |
| Action and data | `action.calculate`, `action.delay`, `action.log`, `action.text`, `action.value`, `runtime.variables`, `runtime.persistent_storage`, `runtime.secrets` |
| Files and network clients | `action.file`, `action.http` |
| Processes and scripts | `action.process`, `action.sub_script` |
| Desktop | `action.clipboard`, `action.keyboard`, `action.message_box`, `action.mouse`, `action.notification`, `action.pixel`, `action.sound`, `action.form_dialog`, `action.window` |
| Serial | `action.serial`, `trigger.serial_input` |
| Network replies/listeners | `action.webhook_response`, `action.websocket`, `trigger.webhook`, `trigger.websocket` |
| Control flow | `runtime.break_loop`, `runtime.color_match`, `runtime.continue_loop`, `runtime.for_each`, `runtime.if`, `runtime.repeat`, `runtime.switch`, `runtime.while` |
| Other triggers | `trigger.file_watch`, `trigger.hotkey`, `trigger.manual`, `trigger.process_started`, `trigger.schedule`, `trigger.startup` |

The editor node registry generates the node-to-capability contract embedded in the Rust security crate. During import, the runner recalculates required capabilities from `program.json` and compares them with package declarations. A package cannot gain access by declaring an unrelated capability, and it cannot omit one required by its program.

## Per-revision approval

Approval records:

- stable script identity.
- exact package hash.
- calculated permissions accepted during review.
- approval time.

An approval is **current** only while the installed package hash and permission set match. Updating package content invalidates the previous approval. An unchanged display name does not preserve trust.

Revocation removes consent for the current revision without removing the package. The runner removes its active trigger registrations and rejects new manual, automatic, queued, and sub-script execution attempts until that revision is approved again.

Disabling a script is separate from revoking approval. A disabled script remains approved and installed, but the runner removes its active trigger registrations and rejects every new execution attempt. A run that was already executing is not forcefully cancelled.

## Review in the desktop application

1. Open **Scripts** or **Security**.
2. Select the script requiring attention.
3. Confirm package integrity and stable identity.
4. Compare the target runtime with the actual machine and session.
5. Review every node, permission, capability, risk level, and secret declaration.
6. Inspect paths, URLs, process targets, shell text, serial IDs, and listener routes in context.
7. Approve only when the revision matches the intended behavior.

The approval button remains blocked when validation, integrity, target, or policy checks make approval unsafe.

## Review with the CLI

Replace `SCRIPT` with the installed name or ID:

```text
baudbound script inspect SCRIPT
baudbound script approval SCRIPT
baudbound secret list SCRIPT
baudbound script approve SCRIPT
```

To revoke approval:

```text
baudbound script revoke-approval SCRIPT
```

Use `--json` on supported inspection commands for automation. Do not parse decorative table spacing.

## Runner policy gates

The operator controls four independent switches under `[security.policy]`:

| Setting | What `false` blocks |
| --- | --- |
| `allow_shell_commands` | Every action requiring `process.shell` |
| `allow_dangerous_permissions` | Every package that requires a Dangerous permission |
| `allow_public_network_listeners` | Webhook and WebSocket triggers that bind beyond loopback |
| `allow_private_http_requests` | HTTP Request destinations that resolve to loopback, private, link-local, or other non-public addresses |

Shell commands must pass both the shell-specific setting and the Dangerous-permission setting. A current approval does not bypass these checks. Policy is applied during inspection, import, approval, trigger registration, and execution. A blocked script reports the exact setting involved.

The first three settings default to `true` so an update does not silently disable packages that a user already reviewed. `allow_private_http_requests` defaults to `false` to block server-side request forgery into local services and private networks. Set a switch to the narrowest value that supports the machine's intended workflows. Policy is defense in depth. It must not be used to make package declarations inaccurate, and a package or repository cannot change it.

## Operator review checklist

- [ ] **Filesystem:** Are read, write, move, copy, download, and delete paths limited to intended data?
- [ ] **Processes:** Are executables, arguments, working directories, match modes, and kill targets exact?
- [ ] **Shell:** Is a native action unavailable, and has every expansion and quoting boundary been reviewed?
- [ ] **Input control:** Could keyboard, mouse, clipboard, or focus actions affect the wrong foreground application?
- [ ] **Network client:** Are URLs, headers, bodies, redirects, and returned data trusted appropriately?
- [ ] **Network server:** Are bind address, route, method, authentication, proxy, firewall, limits, and TLS intentional?
- [ ] **Serial:** Does the logical ID resolve to hardware whose USB identity and protocol settings are verified?
- [ ] **Secrets:** Are values configured only on the runner and kept out of package assets, variables, logs, and commands?
- [ ] **Persistence:** Can stored or global data influence later unattended runs safely?
- [ ] **Sub-scripts:** Is the child script independently installed, current, approved, and compatible?

Continue with [Security Model](index.md), [Secrets](../runner/secrets.md), and [Network Access](../runner/network-listeners.md).
