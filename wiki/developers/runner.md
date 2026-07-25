---
title: Runner Development
description: Extend the Rust runner, Tauri bridge, native actions, triggers, storage, and CLI.
tags: [developers, runner, rust]
---
# Runner Development

## Official blacklist administration

The production runner reads active records from the PocketBase `blacklist` collection at `api.baudbound.app`.

The collection uses these public fields:

| Field | Purpose |
| --- | --- |
| `scope` | Selects repository, publisher, domain, script, or package matching |
| `target` | Stores the normalized identity for the selected scope |
| `subdomains` | Includes descendant hostnames for a domain entry |
| `title` | Provides the short public advisory title |
| `reason` | Explains the reviewed security concern |
| `severity` | Selects low, medium, high, or critical enforcement |
| `advisory_url` | Links to public supporting information |
| `published_at` | Records when the advisory became public |
| `active` | Publishes and applies the entry |

The hidden `private_notes` field is for maintainers. The runner never requests it.

PocketBase list and view rules must remain `active = true`. Create, update, and delete operations remain available only to PocketBase superusers. New records should stay inactive until the scope, normalized target, reason, severity, advisory, and publication time have been reviewed.

Use a unique constraint for `scope, target` and an index for `active, scope`. Deactivate an entry instead of deleting it when possible. This preserves the PocketBase record ID used by runner security incidents.

Before activating an entry:

1. Confirm that the target matches the intended scope.
2. Confirm that `subdomains` is enabled only for a domain entry.
3. Confirm that a package target is the lowercase SHA256 of the exact package.
4. Confirm that a publisher target uses a supported identity such as `github:account`.
5. Publish an advisory page that explains the evidence and recovery steps.
6. Test the intended severity against a local runner.

The runner keeps its last complete valid snapshot when a refresh fails. Increasing severity takes effect on the next successful refresh. Lowering severity or deactivating an entry removes the active restriction but never enables a previously quarantined script.

Keep `src/main.rs` in the runner repository limited to top-level parsing and dispatch. Commands live under `commands/`. Service options, runtime, status, webhooks, and trigger loading live under `service/`. Tauri commands bridge UI requests into shared application services rather than reimplementing CLI behavior.

Crates define ownership boundaries. Avoid large crate-root implementation files and organize source into domain folders. Public APIs should expose validated types and narrow operations. Storage, policy, and native adapters remain replaceable behind explicit interfaces.

Native actions must use Rust crates or operating-system APIs. Do not construct PowerShell, Bash, xdotool, or similar command scripts to simulate a native feature. If no production-quality implementation exists on a target, mark the node unsupported there and update editor definitions, generated contracts, runner checks, and tests.

Every run path must validate package integrity, compatibility, approval, policy, secrets, and node configuration before side effects. Error messages should identify script, run, and node without leaking secret values.

Runtime graph construction must reject missing, duplicate, negative, or gapped edge execution orders. Fan-out destinations from one source handle execute sequentially by `execution_order`. Do not derive their order from map iteration, node IDs, package array position, or editor coordinates.

## Application modules

| Module | Responsibility |
| --- | --- |
| `cli.rs` | Clap command and option contract |
| `commands/` | Config, package, script, status, hotkey, and doctor task handlers |
| `service/` | Long-lived options, preflight, runtime loop, status, listeners, and trigger loading |
| `desktop_ui.rs` and modules | Tauri command state, dashboard models, background runner, updater bridge |
| `desktop_actions/` | Desktop-only native adapters and platform modules |
| `paths.rs` | Config and runner-home resolution |
| `output.rs` | Human and JSON CLI presentation |

Command handlers and Tauri commands should call shared core/application services. Do not maintain a desktop implementation and CLI implementation of the same lifecycle rule.

## Common change procedures

### Add an action

1. Define and document the editor contract.
2. Add security derivation and target restrictions.
3. Implement the action in the owning `baudbound-actions` domain module or a desktop native adapter.
4. Register it with core runtime dispatch.
5. Validate all config before side effects and return structured fallible outputs where the node contract requires them.
6. Add unit, runtime integration, rejection, cancellation, and supported-platform tests.

### Add a trigger

Define registration config and output payload, implement its service in `baudbound-triggers`, integrate loading/reloading in the app service, isolate listener failure, and test dispatch identity plus graceful shutdown. Network and device services require explicit resource limits.

### Add configuration

Add a serde field with a safe default to `baudbound-core::config`, validate cross-field invariants, include it in the generated template and desktop simple/advanced editor, decide whether service restart is required, and update [Configuration and Devices](../runner/configuration.md). Coverage fails when a serialized config field is absent from that page.

### Add durable state

Add a forward SQLite migration in `baudbound-storage`, repository methods, row conversions, transaction/error tests, backup implications, and UI/CLI access through shared services. Do not create JSON sidecar state or let callers edit SQL rows directly.

### Add a CLI or Tauri command

Clap owns command syntax and typed options. Tauri commands use serializable request/response DTOs and authorize in Rust. Update the typed TypeScript client and tests. CLI options are coverage-checked against [CLI Reference](../runner/cli-reference.md).

## Concurrency and lifecycle

Long-lived services must support cancellation, bounded work, graceful listener shutdown, and deterministic resource ownership. One runner home can be accessed by independent CLI processes through SQLite, while the active `baudbound serve` process receives live reload/control signals through authenticated loopback IPC. Tauri is the desktop UI bridge, not the service transport for headless clients.

Avoid holding database transactions across native calls or network waits. Trigger registration reload should replace changed registrations without duplicating active listeners. Errors in one script or registration must not terminate unrelated services.

## Error and secret handling

Add context at ownership boundaries: action type, node ID, script ID, listener, path category, or device ID. Do not include secret values, encryption keys, authorization headers, complete private payloads, or unredacted command environments. Test redaction both for success-path logs and nested errors.

Use [Repository Architecture](architecture.md) for the full execution and process flow and [Testing and CI](testing-ci.md) to select mandatory gates.
