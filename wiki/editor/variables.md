---
title: Variables and Data
description: Create, reference, inspect, persist, and safely compare data in BaudBound workflows.
tags: [editor, variables, data]
---
# Variables and Data

Variables carry data between triggers, actions, and control-flow nodes. BaudBound uses double braces to reference a value:

```text
{{status}}
```

Variable-aware fields show matching suggestions after typing `{{`. Select a suggestion or press Tab to insert its complete token.

## Choose the right data kind

| Need | Use | Why |
| --- | --- | --- |
| Value only during the current trigger run | Runtime variable | Starts clean for each run and has the narrowest write scope |
| Value retained for later runs of the same script | Persistent variable | Stored under that script identity |
| Deliberately shared value across scripts on one runner | Global variable | One runner-level name. Requires high-risk review |
| Password, token, private key, or credential | Secret declaration | Value stays outside the project and package |
| Runner configurable value that is safe to include in the package | Script Setting | Runner users can override it without editing the script |
| Data produced by a trigger or action | Node output | Read-only and namespaced by stable node ID |

Prefer the narrowest lifetime that satisfies the workflow. Do not use global data merely to avoid passing node outputs, and never use ordinary persistent/global variables for credentials.

## References and interpolation

A field containing only one reference preserves the value's original type:

```text
{{request.retry_count}}
```

If `request.retry_count` is the number `3`, the resolved field receives a number. This matters for numeric comparisons, lists, objects, booleans, and action fields that expect structured data.

A reference combined with other text produces a string:

```text
Attempt {{request.retry_count}} failed
```

Lists and objects embedded in text are serialized as compact JSON. Use a standalone reference when the destination must receive the original list or object.

Whitespace inside a token is accepted by the runner, but the editor highlights it as a warning. Prefer the canonical form `{{status}}` rather than `{{ status }}`.

An unknown reference is left unchanged as literal `{{name}}` text. This makes the failure visible instead of silently replacing it with an empty value, but it can also reach an action as unintended input. Resolve red tokens before export.

## Nested data

Use dot-separated paths to read object fields and zero-based list indexes:

```text
{{profile.name}}
{{payload.users.0.email}}
{{n-mr3zyt6f-12.json.user.name}}
```

Reading paths use numeric dot segments such as `.0`. Bracket notation is reserved for the Variable Operation node's **Set object field** path, where paths such as `users[0].name` are supported.

References retrieve data. They do not evaluate arithmetic or arbitrary expressions. Use Calculate for mathematics and Format Text for text transformations.

Use Convert Value when an action returns the right content in the wrong type. For example, it can convert the text `42` to a number or JSON text to a list or object.

## Variable sources

### User variables

The Variables tab shows both the plain variable name and its `{{reference}}`. Both values can be selected, and each has its own copy button.

Variable Operation creates and changes named user variables. Names must start with a letter or underscore and may contain only letters, numbers, and underscores. The `manifest_` and `system_` prefixes are reserved.

The selected scope controls ownership and lifetime:

| Scope | Lifetime and visibility | Approval impact |
| --- | --- | --- |
| `runtime` | Exists only inside the current run. A new run starts without the value. | Low-risk local write |
| `persistent` | Stored for this script and loaded into later runs of the same script. | Medium-risk persistent write |
| `global` | Stored once by name and shared with every script using that global name. | High-risk cross-script write |

The runner executes only one run of a specific installed script at a time. A later run of the same script waits until the active run finishes, so persistent variables provide predictable state between its runs. Different scripts can run at the same time. Global writes remain versioned so two different scripts cannot silently overwrite an update made between reading and writing. Use global variables sparingly because unrelated scripts can intentionally share and change the same value.

A stored variable is loaded at run start when the script contains a Variable Operation declaration for that name and scope. Do not declare the same variable name with conflicting scopes in one script.

### Default variables

Open **Settings**, then choose **Default Variables**. This section defines typed starting values that are saved in the `.bbs` package. The bottom **Variables** tab remains a read only view of values that can exist during a run.

Every default variable must have an explicit value. String and file path defaults cannot be blank, while values such as `false`, `0`, an empty list, and an empty object are valid explicit defaults. The editor changes to match the selected type:

| Type | Editor |
| --- | --- |
| `string` | Resizable text field |
| `number` | Number field that accepts only finite values |
| `boolean` | `true` or `false` selector |
| `object` | JSON editor with line numbers |
| `list` | Item type selector and ordered item rows |
| `datetime` | Local date and time field stored in RFC 3339 format |
| `duration` | Amount field and unit selector |
| `file_path` | Path text field |

Every list has one declared item type. All items must use that type. A list cannot contain another list directly. Use objects when nested data is needed.

This makes the variable available from the beginning of a run, even when no earlier node has assigned it.

Choose one of these scopes:

| Scope | Start-of-run behavior |
| --- | --- |
| `runtime` | Every run starts from the saved package value. Changes made by Variable Operation nodes last only for that run. |
| `persistent` | The runner saves the package value only when that script has no stored value. Later runs use the stored value, including changes made by Variable Operation nodes. |

A Variable Operation node can change a default variable by using the same name, type, and scope. Export verification rejects a mismatch so the editor and runner cannot interpret one name in two different ways.

Updating a script package does not replace an existing persistent value. The new default applies only when no value has been stored for that script. Removing and importing the script again creates a new persistent state lifecycle.

Default values are ordinary package data. Anyone who receives the package can read them. Use a [secret declaration](../runner/secrets.md) for passwords, access tokens, private keys, and other sensitive values.

### Script Settings

Open **Settings**, then choose **Script Settings**. A Script Setting is a package declaration that a runner user can configure without changing the script graph.

Each declaration contains a name, type, description, required option, optional package default, and optional simulation override. Script Settings support every Default Variable type plus two setting-only types:

| Setting-only type | Editor and validation |
| --- | --- |
| `hotkey` | Captures a canonical Windows key or chord such as `Ctrl+Shift+F8` |
| `color` | Color picker and text field restricted to `#RRGGBB` |

Use `hotkey` when a Hotkey trigger or Keyboard action should be configurable on each runner. Those node fields offer Literal key and Variable modes and list only compatible hotkey settings. Use `color` for configurable color comparisons. A color setting used in Color Match updates the field's color swatch after the variable resolves.

Use the setting through the read only `settings` object:

```text
{{settings.Endpoint}}
{{settings.RetryCount}}
{{settings.Options.enabled}}
```

The runner chooses the value in this order:

1. A value configured on that runner
2. The package default
3. `null` when the setting is optional and neither value exists

A required setting must have a runner value or a package default. Otherwise the script cannot run. An enabled script with a missing required setting also prevents the background service from starting.

The simulation override is editor data. It is used only by simulation and is not included in the `.bbs` package. When no simulation override exists, simulation uses the package default. A missing optional value becomes `null`.

Script Settings are normal package data and runner configuration. They are not secret. Use **Secrets** for passwords, tokens, and credentials.

### Node outputs

Actions and triggers expose read-only runtime data using the node ID and output name:

```text
{{n-mr3zyt6f-12.status_code}}
{{n-mr3zyt6f-12.error.message}}
```

Select a node and open **Runtime Data** in Properties to see its outputs, field types, descriptions, and complete tokens. A custom node display name does not change these references. The stable node ID remains the data namespace.

Trigger payload fields use the trigger node ID in the same way. Action outputs become available only after that node executes on the current branch. Referencing an output before execution leaves the token unresolved.

Fallible actions expose structured `error` data when execution continues through the failed branch. Common fields include `message`, `code`, `retryable`, and `details`.

### For Each variables

For Each exposes the current item and zero-based index through its node runtime data. If the For Each node ID is `n-example`, use `{{n-example.item}}` for the current item and `{{n-example.index}}` for its position. These values update for every iteration.

### Manifest variables

Manifest values are read-only and come from the exported package and project settings:

| Variable | Value |
| --- | --- |
| `{{manifest_name}}` | Script name |
| `{{manifest_version}}` | Package format version |
| `{{manifest_author}}` | Author |
| `{{manifest_description}}` | Description |
| `{{manifest_website}}` | Project website |
| `{{manifest_source}}` | Source URL |
| `{{manifest_minimum_runner_version}}` | Minimum compatible runner version |

### System variables

System values are read-only and describe the runner environment at execution time:

| Variable | Value |
| --- | --- |
| `{{system_os}}` | Operating system |
| `{{system_arch}}` | CPU architecture |
| `{{system_hostname}}` | Host name |
| `{{system_user}}` | Runner user when available |
| `{{system_locale}}` | Runner locale |
| `{{system_timezone}}` | Runner time zone |
| `{{system_date}}` | Current runner-local ISO date |
| `{{system_time}}` | Current runner-local 24-hour time |

Simulation uses browser-derived or clearly simulated system values. Do not assume that simulator host data matches the machine where the package will run.

### Secrets

Open **Settings**, then choose **Secrets** to manage declarations. Secrets are read-only text variables declared by name in the editor. Their values are not included in the project or `.bbs` package. The runner stores values separately for each installed script and blocks a run when a required secret is missing.

Simulation values are entered explicitly in the Secrets panel, remain in memory for that simulation session, and are not saved. Secret values and values derived from them are redacted from runner reports and logs where detected. Derived metadata is deliberately not created for secrets.

See [Secrets](../runner/secrets.md) for runner configuration and lifecycle commands.

## Default Variable and Variable Operation types

| Type | Accepted value | Empty value after Clear |
| --- | --- | --- |
| `string` | Plain text | Empty string |
| `number` | Finite number | `0` |
| `boolean` | `true` or `false` | `false` |
| `list` | Ordered values that all use one declared item type | `[]` |
| `object` | JSON object | `{}` |
| `file_path` | Non-empty path string | Empty string |
| `datetime` | Date and time stored as RFC 3339 | Unix epoch |
| `duration` | Object containing `type: "duration"`, a `unit`, and numeric `value` | Zero-second duration object |

Structured examples:

```json
{ "type": "datetime", "value": "2026-07-11T12:00:00Z" }
```

New datetime values start with the current date and time. The timezone selector controls how the editor displays and interprets the value. Changing the timezone keeps the same instant and only changes its displayed time. Editing the date or time converts the selected timezone back to the UTC timestamp stored in the project and package.

```json
{ "type": "duration", "unit": "seconds", "value": 10 }
```

Node outputs can use additional specialized types such as HTTP responses, file content, HTTP headers, status codes, process IDs, and exit codes. These runtime output types cannot be selected for a Default Variable, Script Setting, or Variable Operation. `hotkey` and `color` are Script Setting types only. They cannot be selected for Default Variables or Variable Operation. The Runtime Data panel identifies the exact type produced by each node.

## Variable operations

| Operation | Behavior |
| --- | --- |
| **Set** | Creates the variable or replaces its value after coercion to the selected type. |
| **Increment** | Adds a finite numeric amount. A missing value starts at zero. |
| **Toggle boolean** | Changes true to false or false to true. A missing value starts as false and becomes true. |
| **Append list** | Appends one item. A missing value starts as an empty list. The first item establishes the list item type, and later items must use that same type. |
| **Remove matching list items** | Removes the first or every item that exactly matches the provided value and type. The variable must already contain a list. |
| **Set object field** | Writes a typed nested object field and creates missing objects or list positions. |
| **Remove object field** | Removes a nested field or list position when the path exists. The variable must already contain an object. |
| **Merge object** | Performs a shallow merge. Incoming top level fields replace fields with the same name. A missing value starts as an empty object. |
| **Clear** | Resets an existing variable to the empty value for its current type without deleting its stored declaration. Clear fails when the variable does not exist. |
| **Delete variable** | Removes the value completely. It requires only the variable name and scope because deletion is independent of type. Deleting a persistent default allows its package default to initialize again on a later run. |

Only **Set** asks for a variable type. A Set operation for a list also asks for the one item type used by that list. Fixed operations derive their required type from the operation. Clear derives the type from the existing value. Append infers the item type from the value and existing list, while Remove matching list items uses exact value and type equality.

Operations that need input accept either a typed value or a variable reference. Text, number, object, and file path inputs use the same code editor for both forms. Other typed inputs provide a **Raw value** source for variable references. The runner validates each resolved value before changing runtime or stored state.

## Derived metadata

Every non-secret value exposes read-only metadata:

| Token suffix | Result |
| --- | --- |
| `.$length` | UTF-16 length for strings, item count for lists, key count for objects, otherwise `0` |
| `.$count` | Alias for `.$length` |
| `.$type` | `string`, `number`, `boolean`, `list`, `object`, or `null` |
| `.$is_empty` | `true` for null, empty strings, empty lists, and empty objects |

Examples:

```text
{{message.$length}}
{{records.$count}}
{{n-mr3zyt6f-12.response.$type}}
{{payload.$is_empty}}
```

Metadata is refreshed whenever the underlying runtime value changes. The `$` names are reserved. An object property with the same name cannot be addressed through this derived-token form.

## Conditions and typed values

If/Else and While resolve both sides before comparing them. Keep a standalone reference when comparing numbers, booleans, null, lists, or objects. Combining a reference with text converts that side to a string.

Available comparisons include equality, numeric ordering, inclusive number ranges, text matching, and collection checks. If/Else also provides strict boolean checks and value type checks. Checks that inspect Value directly hide the Target field. **Is null or missing** matches a null value or a variable reference that does not exist. **Is between** uses a start value and end value, and includes both boundary values. Text such as `"true"` is not a boolean and does not match **Is True**. Rows are combined in order with AND or OR, and each row can be inverted.

Numeric comparisons require numeric operands. **Is between** also requires the start to be less than or equal to the end. Regex patterns are limited to 256 characters. The runner rejects invalid patterns, and simulation treats patterns rejected by its safety checks as a non-match.

## Inspecting runtime data

The bottom **Variables** tab combines declared variables, built-ins, secrets, node outputs, and the most recent simulation snapshot. It can:

- show or hide derived metadata.
- show or hide manifest and other built-in values.
- show or hide system values.
- sort recently changed values first.

These controls only affect the panel. They do not change the project, exported package, or runtime state. Values shown before simulation may be declarations without a current value. Run a relevant trigger to inspect actual data produced along that path.

## Common mistakes

**A token remains visible in output:** the name is unknown, the producing node did not execute, or the reference runs before assignment.

**A number behaves like text:** use a standalone token and ensure the source is typed as a number.

**An object or list becomes JSON text:** remove surrounding text from the field so the reference can preserve its original type.

**A nested field does not resolve:** verify every path segment and use numeric dot notation for list indexes.

**A value unexpectedly survives another run:** the variable is persistent or global rather than runtime-scoped.

**Scripts affect each other's values:** they use the same global variable name. Change the name or use persistent scope for script-local storage.

**The editor marks a reference red:** select a suggestion from the variable browser and compare the stable node ID, spelling, case, and nested path. For example, change `{{request.users[0].name}}` to the supported read form `{{request.users.0.name}}`.

**A derived field does not resolve:** place `$length`, `$count`, `$type`, or `$is_empty` after the complete value name, such as `{{items.$length}}`. The older `.$meta.*` form is not supported.

For node-specific output names and types, use [Node Reference](node-reference.md). For stored-state backup and recovery behavior, use [Storage, Backups, and Recovery](../runner/storage-backups.md).
