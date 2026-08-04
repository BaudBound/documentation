---
title: Verification and Simulation
description: Understand editor verification rules, simulation controls, trigger payloads, side effects, and runner differences.
tags: [editor, verification, simulation]
---
# Verification and Simulation

Verification checks whether a project is internally valid. Simulation executes its graph in the browser with test input. Runner validation and approval happen later on the target machine. Passing one stage never skips the others.

## Compare the stages

| Stage | Runs where | Performs native production actions | Main purpose |
| --- | --- | --- | --- |
| **Verification** | Editor | No | Find graph, config, variable, asset, access, and target problems |
| **Simulation** | Editor browser | Mostly no. See side effects below | Exercise branches and data with controlled input |
| **Runner validation** | Native runner | No | Reparse package and recalculate integrity, security, target, and semantic contracts |
| **Runner execution** | Native runner | Yes, after checks and approval | Perform the approved automation |

Graph or project changes invalidate the previous editor verification state.

## Verification checks

The Verify command and export gate run these rule families:

| Check | What it validates |
| --- | --- |
| **Script metadata** | Required name and at least one selected target runtime |
| **Secret references** | Unique declarations and no collision with writable variables |
| **Target runtimes** | Every node supports every selected operating system and session target |
| **Graph structure** | Runnable nodes exist and at most one Manual trigger is present |
| **Entry points** | At least one trigger and valid trigger configuration |
| **Connections** | Source/target nodes and named handles exist |
| **Permissions** | Medium, high, or dangerous access is surfaced for review |
| **Variables** | Writable names, references, calculations, node config, and graph-specific rules |
| **Assets** | Package paths, allowed files, duplicates, limits, and node references |
| **Serial devices** | Logical IDs, duplicate triggers, and write targets |
| **Export readiness** | Combined blocking conditions permit package creation |

Node definitions add their own config and graph rules. Examples include a waiting webhook requiring a reachable response node, Serial Write requiring a matching Serial Input logical ID, valid loop structure, and condition-row syntax.

## Failures, warnings, and passing checks

**Failed** checks block simulation or export when the graph cannot be executed safely or serialized consistently. Fix them before continuing.

**Warnings** require review but do not necessarily make the graph invalid. Medium-or-higher permissions are warnings because the editor cannot decide whether intended access is acceptable.

**Passed** means the tested contract succeeded with current editor information. It does not prove that paths, credentials, ports, hardware, desktop sessions, or external services will exist on the runner.

## Simulation controls

Open **Simulation** in the inspector. Each trigger has a card containing fields appropriate to that trigger and a button to fire it. Schedule triggers run automatically while the simulation session remains active.

| Control | Behavior |
| --- | --- |
| **Trigger button** | Runs every blocking verification check first. Only a passing project enters the active state and fires the trigger payload |
| **Simulation pacing** | **Real time** adds no artificial pause. Slowdown options add `100`, `300`, or `700` milliseconds after every streamed step |
| **Runtime override** | Forces a selected fallible node to succeed or fail for branch testing |
| **Stop** | Cancels the active run or waiting session |
| **Simulation output** | Shows verification, trigger, node, branch, side-effect, and failure traces |
| **Runtime data** | Shows current variable and node-output snapshots |
| **Taken path** | Connections used by the simulation turn green as the workflow runs |

Only one trigger executes at a time. Stop an active run before firing another. Editing the graph cancels and resets the previous simulation state.

Simulation executes and reports one node at a time. The active highlight, taken connection, output entry, and Runtime Data changes are published before the simulator advances to the next node. It does not execute the complete graph in the background and reveal buffered results afterward. Real time still waits for the simulated behavior of nodes such as Delay, schedules, authorized live HTTP requests, dialogs, and audio. The slowdown choices add review time on top of that behavior.

The green connections remain visible after the simulation finishes so you can inspect the complete path. Starting another trigger or editing the graph clears the previous path.

Runtime variables reset to their package defaults before each simulated run. Persistent variables keep their latest simulated values for later runs in the current editor session. Changing a Variable Operation node from runtime scope to persistent scope makes later simulated writes use persistent state. Reloading the editor restores the saved package defaults because simulated persistent values are not exported.

## Trigger payload examples

Payload fields are test input, not saved production events.

| Trigger | Useful simulation fields |
| --- | --- |
| Manual | Optional generic payload supplied by the card |
| Schedule | Fires automatically from configured interval |
| File Watch | `path`, `event` such as `modified` |
| Webhook | `method`, `path`, headers, query, raw `body`, JSON body |
| WebSocket | `path`, `message`, `connectionId`, headers, query, `remoteAddress` |
| Hotkey | `key` |
| Serial Input | `data`. Output also calculates byte length and timestamp |
| Process Started | `processName`, `processId`, `executablePath`, `windowTitle` |
| Startup | `reason`, normally represented as runner startup |

Example webhook body:

```json
{
  "event": "simulation",
  "message": "hello"
}
```

Use the trigger node's runtime output browser to insert its real ID into downstream references.

Schedule simulation uses the configured interval instead of adding the callback duration to every cycle. If the browser runs a callback late, the next due time remains anchored to the original cadence. Browser and operating-system scheduling can still make a callback late, so simulation and the runner are not hard real-time systems.

## Secrets in simulation

Secret declarations appear in the output console's secret manager. Each declaration can use a placeholder or a value you enter for the current browser session.

Entered values are type-checked, supplied to simulation, redacted from simulation reports where applicable, and never written into the exported package. Reloading, importing, or clearing the session can remove them.

HTTP Request simulation defaults to **Mock**, which performs no network request and does not resolve secret-bearing request fields. **Live** mode identifies the literal destination origins before interpolation and requires explicit authorization for those origins for the current simulation run. Authorization is not stored for later runs. A live request can transmit entered values to an authorized origin, so use test credentials unless real access is deliberate.
{.is-warning}

## Side effects and fidelity

The browser performs a small controlled set of visible simulation effects:

- notification actions show editor toasts.
- Message Dialog shows a blocking editor dialog and returns the selected configured button or `timeout`. Its size, button set, and timeout countdown match the runner contract.
- Form Dialog requires at least one component, shows its ordered components with Cancel and Submit in a blocking editor dialog, and returns the same typed `values`, `submitted`, and `button` outputs as the runner. Choice results contain configured keys in configured order, not displayed values. Packaged images render from the project asset.
- Browser security prevents the editor simulator from reading absolute local filesystem paths. File and Folder components therefore accept explicit simulated paths; the desktop runner uses native operating-system pickers. Their submitted value shapes remain the same.
- Beep uses browser Web Audio.
- Play Sound can play a packaged audio asset.
- HTTP Request defaults to a deterministic mock response. Live mode sends the request through the editor's bounded same-origin server endpoint after origin authorization.

Live HTTP simulation accepts only public HTTP or HTTPS destinations, resolves and pins the selected public address, revalidates every redirect, rejects HTTPS-to-HTTP downgrade, strips all caller headers on cross-origin redirects, and does not forward a request body across origins unless redirect semantics convert the request to a bodyless GET. It bounds request bodies, response bodies, headers, authorized origins, and redirect hops, requests identity encoding, and supports cancellation. It does not send browser cookies.

The simulation endpoint follows the shared HTTP simulation policy contract. It is still not proof of the target runner's native TLS store, account environment, destination availability, or operator policy. Test native network behavior on the intended runner before release.

Other native actions are described and assigned simulated outputs without controlling the machine. This includes files, processes, shell, keyboard, mouse, clipboard, windows, physical serial ports, application opening, and WebSocket listener connections.

Simulation cannot prove:

- runner filesystem paths or account permissions.
- executable or process availability.
- desktop focus and session behavior.
- serial protocol or device identity.
- listener binding, firewall, proxy, or port ownership.
- package approval, runner policy, or encrypted secret storage.
- native error timing and operating-system-specific output.

## Runtime overrides

Add an override for a fallible action to test its **success** or **failed** edge without reproducing the external failure. Failed overrides populate structured error output so downstream error handling can be exercised.

Remove overrides before considering the simulation representative. Overrides are editor test settings and are not production runner policy.

## Logs and variables

Simulation traces identify the node and branch being executed. Log actions also produce output entries at `debug`, `info`, `warn`, or `error` level.

The runtime-data view updates after each step with current variables and node outputs. Use it to confirm types and nested fields rather than relying only on formatted log text.

## Pre-export checklist

- [ ] Every verification check passes or has an understood warning.
- [ ] Every trigger has been tested with representative valid and invalid payloads.
- [ ] True, false, switch, loop, done, success, and failed branches behave as intended.
- [ ] Temporary runtime overrides are removed or accounted for.
- [ ] Secret simulation values use test credentials unless a reviewed Live request explicitly needs real access.
- [ ] HTTP Request nodes use Mock unless their displayed Live origin authorization is intentional.
- [ ] Runtime data has the expected types and nested paths.
- [ ] Native behavior still requiring runner testing is listed for the operator.
- [ ] Export access review contains only expected permissions and capabilities.

Continue with [Projects, Assets, and Export](projects-assets-export.md) and [Variables and Data](variables.md).
