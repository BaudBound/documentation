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

Variable-aware fields show matching suggestions after typing `{{`. Select a suggestion or press Tab to insert its complete token. Suggestions are filtered by the field's declared type contract: numeric fields show numeric values, datetime fields show datetime values, and so on.

The editor highlights known compatible references in green. A known reference with the wrong type is cyan and produces an inline validation error. Amber means the path may exist but its type cannot be established; typed fields reject that uncertainty, while fields that explicitly accept any value may use it. Red means the reference is unavailable. Whole-script verification repeats these checks and blocks simulation or export when a typed input is invalid.

## Variable types

Every variable, Script Setting, and node output uses one of ten types. Matching is exact. A value must be exactly the declared type. A color is not usable where a string is required, and an integer is not usable where a float is required.

| Type | Accepts | Rejects |
| --- | --- | --- |
| `string` | Any text | A number, a boolean, a list, or an object |
| `integer` | A whole number from -9007199254740991 through 9007199254740991 | A fractional number, or text such as `"42"` |
| `float` | A number that can include a fractional part | A whole number produced as an integer |
| `boolean` | `true` or `false` | Text such as `"true"` |
| `object` | A JSON object | A JSON array |
| `list` | A JSON array | A JSON object |
| `color` | Six digit hex text such as `#2F80ED`, case insensitive | A three digit hex code, or a named color such as `red` |
| `hotkey` | A key or chord validated against the [shared Windows key contract](node-reference.md#supported-windows-node-keys), such as `F5` or `Ctrl+S` | An unknown key name, an empty expression, or a repeated key |
| `datetime` | An object with `type: "datetime"` and an RFC 3339 `value` | A bare date string without the surrounding object |
| `duration` | An object with `type: "duration"`, a `unit`, and a numeric `value` | An object that is missing its `unit` |

A Script Setting, a default variable, a Variable Operation value, and a node's Runtime Data all read and write this same ten type vocabulary. A value produced by one can be declared or compared against another without translation.

### Integer and float are separate types

`integer` and `float` are both numbers, but they do not mix. `42` is an integer, not a float. `3.7` is a float, not an integer. Moving a value from one to the other needs a cast. See [Casting a value to another type](#casting-a-value-to-another-type) for the exact syntax.

A number produced outside the script carries the sender's type, and the editor cannot know which type it will be at design time. An HTTP response body `{"count": 3}` produces an integer. The same field sent as `{"count": 3.0}` produces a float. A value read from an HTTP response, a webhook payload, or a Form Dialog number field usually needs a cast before it can be compared against or stored as the other numeric type.

Numeric config fields are exactly typed for the same reason. A field that takes a float, such as the Beep node's frequency, accepts only a float variable. A field that takes an integer, such as Repeat's count, accepts only an integer variable. A cast is how an author bridges the two.

When the type is not known until the script runs, an If/Else or While condition can test it. **Is integer** passes for a whole number, **Is float** passes for a fractional one, and **Is numeric** passes for either. These read the type of the value, so text that reads as a number is still text. Cast the value first to test it as a number.

A value read out of an object or a list has no type until the script runs, so it needs a cast before a typed field accepts it. See [Nested data](#nested-data).

### Casting a value to another type

Add `|target` inside the braces to convert a value to a different type before it is used:

```text
{{name|target}}
```

`target` is one of the ten types: `string`, `integer`, `float`, `boolean`, `object`, `list`, `color`, `hotkey`, `datetime`, or `duration`. A cast can be used anywhere a reference is used, including inside a longer string:

```text
https://spools.example.com/api/v1/spool/{{spool_number|string}}
{{payload.count|integer}}
```

`{{x|float}}` is how an integer becomes a float. `{{x|integer}}` is how a whole float becomes an integer. `3.7` has no whole number to become, so casting it to `integer` fails.

`null` fails every target without exception. An unset variable and a missing object field both resolve to `null`, so casting either one always fails instead of silently producing an empty string or a placeholder value.

| Target | Accepts | Rejects |
| --- | --- | --- |
| `string` | Any non-null value, serialized to text unless it is already a string | `null` |
| `integer` | An integer, a float with no fractional part, or text that parses as one of those | `null`, a fractional value, a value outside the integer range above, or non-numeric text |
| `float` | An integer, a float, or non-empty numeric text | `null`, non-numeric text, or a value that is not finite |
| `boolean` | `true`, `false`, or the text `"true"` or `"false"` in any letter case | `null`, any other text, a number, a list, or an object |
| `list` | An existing list, or text that parses as a JSON array | `null`, or a value that does not resolve to a list |
| `object` | An existing object, or text that parses as a JSON object | `null`, or a value that does not resolve to an object |
| `color` | Text satisfying the `color` rule above | `null`, a value that is not text, or text that fails the rule |
| `hotkey` | Text satisfying the `hotkey` rule above | `null`, a value that is not text, or text that fails the rule |
| `datetime` | An existing datetime value, or a bare RFC 3339 string wrapped into the datetime object automatically | `null`, text that is not a valid date, or any other shape |
| `duration` | An existing duration value | `null`, or any other shape |

Duration accepts only a value already shaped like the object above. There is no conversion from a plain number or a unit name into a duration.

A failed cast stops the run. It does not take the node's `failed` output, there is no retry, and there is no fallback value. This is checked before the node runs, so a failed cast on an HTTP Request node's URL sends no request, and a failed cast on a Write File node's content writes no file.

> A cast in a webhook or schedule driven script turns one bad payload into a stopped run. There is no failure branch to catch it. Use Convert Value instead when the source of a value cannot be trusted to already have the right type.
{.is-warning}

Choose between the two based on whether a failure should be handled or should stop the run:

| | Convert Value node | Inline cast |
| --- | --- | --- |
| Failure | Takes the failure output, which you can branch on | Stops the run |
| Use when | The value may legitimately be wrong | The value must be right |

### A float always renders with a decimal

Interpolating a float into text always includes a decimal point, even when the value is a whole number. A float variable holding `42` renders as `42.0`. A URL built from `/spool/{{spool_number}}` produces `/spool/42.0` when `spool_number` is a float. Use an integer, or cast the value, when the destination expects bare digits.

### Arithmetic preserves the type

Incrementing preserves the type it started with. An integer that increments by a whole amount stays an integer, so a counter that starts at `0` and increments by `1` each run renders `3`, not `3.0`. A missing counter starts from integer `0`. Incrementing by a fractional amount produces a float. Calculate always produces a float, because its expression evaluator works in floating point regardless of the operands.

### A declared float may hold a whole value

A default variable or Script Setting declared `float` accepts a whole value such as `300` or `300.0`, and holds it as a float. The type written beside the value is what settles it, the same way a typed language reads `300` into a variable declared as a decimal number.

This applies only where a value is declared next to its type. A value flowing through a run still has one exact type, and there a whole number is an integer, not a float: `{{count}}` holding `300` fails **Is float** and passes **Is integer**. Cast it with `{{count|float}}` to move it across.

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

An unknown reference is left unchanged as literal `{{name}}` text at runtime. This makes the failure visible instead of silently replacing it with an empty value. The editor marks it red and blocks verification so it cannot be exported accidentally.

## Nested data

Use dot-separated paths to read object fields and zero-based list indexes:

```text
{{profile.name}}
{{payload.users.0.email}}
{{n-mr3zyt6f-12.json.user.name}}
```

Reading paths use numeric dot segments such as `.0`. Bracket notation is reserved for the Variable Operation node's **Set object field** path, where paths such as `users[0].name` are supported.

### A nested read needs a cast

An HTTP Request node's `json` output is an `object`, and so is a webhook payload. Nothing can know the type of a field inside one until the data arrives, so a nested read carries no type of its own and a typed field will not accept it. The editor reports this as a type that is only known when the script runs.

A cast is how you say what the field holds:

```text
{{n-mr3zyt6f-12.json.ip|string}}
```

The cast is checked like any other, so a field that takes a string still refuses `{{n-mr3zyt6f-12.json.ip|integer}}`. A cast that turns out not to match at run time stops the run, so prefer Convert Value when the payload comes from a source you do not control. See [Casting a value to another type](#casting-a-value-to-another-type).

References retrieve data. They do not evaluate arithmetic or arbitrary expressions. Use Calculate for mathematics and Format Text for text transformations.

Use Convert Value when an action returns the right content in the wrong type. For example, it can convert the text `42` to a number or JSON text to a list or object.

## Variable sources

### User variables

The Variables tab shows both the plain variable name and its `{{reference}}`. Both values can be selected, and each has its own copy button.

Variable Operation creates and changes named user variables. Variable, Default Variable, Script Setting, and Secret names use the same portable identifier characters: letters `A-Z` or `a-z`, numbers `0-9`, hyphens, and underscores. No special first character is required. Spaces, dots, slashes, and other characters are rejected by the editor, package schema, and runner.

The `manifest_` and `system_` prefixes are reserved, and the exact name `settings` is reserved for the Script Settings object. Names are case sensitive and must be unique within the declaration group that owns them.

The selected scope controls ownership and lifetime:

| Scope | Lifetime and visibility | Approval impact |
| --- | --- | --- |
| `runtime` | Exists only inside the current run. A new run starts without the value. | Low-risk local write |
| `persistent` | Stored for this script and loaded into later runs of the same script. | Medium-risk persistent write |
| `global` | Stored once by name and shared with every script using that global name. | High-risk cross-script write |

By default the runner executes one run of a specific installed script at a time, and a later activation of the same script waits for the active run to finish. That is the `limits.max_active_runs_per_script` default of `1`, not a rule: raising it lets runs of one script overlap. Different scripts always run at the same time. Global writes remain versioned so two different scripts cannot silently overwrite an update made between reading and writing. Use global variables sparingly because unrelated scripts can intentionally share and change the same value.

A stored variable is loaded at run start when the script contains a Variable Operation declaration for that name and scope. Do not declare the same variable name with conflicting scopes in one script.

### A condition reads current stored state

A persistent or global variable is loaded once when the run starts. A condition then reloads the stored variables its rows name, so a value changed by another run, or by the same script on a later activation, is seen rather than missed. This applies to **If / Else** and **While** alike, and to a **While** on every pass.

The reloaded value is kept. Every node after the condition reads what the condition read, not the copy taken at run start, so a **Log** placed after an **If / Else** shows the same value the branch was chosen from.

This is what lets a loop end on a flag something else set:

```text
While  {{running}} is true  ->  ...work...
```

Only the variables a condition actually names are reloaded, so the cost follows what the condition uses. A runtime variable is never reloaded, because nothing outside the run can change it.

### Default variables

Open **Settings**, then choose **Default Variables**. This section defines typed starting values that are saved in the `.bbs` package. The bottom **Variables** tab remains a read only view of values that can exist during a run.

Every default variable must have an explicit value. String, color, and hotkey defaults cannot be blank, while values such as `false`, `0`, an empty list, and an empty object are valid explicit defaults. The editor changes to match the selected type:

| Type | Editor |
| --- | --- |
| `string` | Resizable text field |
| `integer` | Number field that accepts only whole values |
| `float` | Number field that accepts finite values, including fractions |
| `boolean` | `true` or `false` selector |
| `object` | JSON editor with line numbers |
| `list` | Item type selector and ordered item rows |
| `color` | Color picker and text field restricted to `#RRGGBB` |
| `hotkey` | Captures a canonical Windows key or chord such as `Ctrl+Shift+F8` |
| `datetime` | Local date and time field stored in RFC 3339 format |
| `duration` | Amount field and unit selector |

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

Each declaration contains a name, type, description, required option, optional package default, and optional simulation override. Script Settings use the same ten types as default variables.

Use `hotkey` when a Hotkey trigger or Keyboard action should be configurable on each runner. Those node fields offer Literal key and Variable modes and list only compatible `hotkey` settings. Use `color` for configurable color comparisons. A color setting used in Color Match updates the field's color swatch after the variable resolves.

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
| `{{system_datetime}}` | Current runner-local date and time, as a datetime |

`{{system_datetime}}` is a datetime rather than text, so a script can read one part of it with a [derived part](#derived-metadata) or render it with the **Format date and time** operation of [Text Transform](node-reference.md#text-transform). Every reference within one run reads the same moment: the runner takes one clock reading when the run starts.

Simulation uses browser-derived or clearly simulated system values. Do not assume that simulator host data matches the machine where the package will run.

### Secrets

Open **Settings**, then choose **Secrets** to manage declarations. Secrets are read-only text variables declared by name in the editor. Their values are not included in the project or `.bbs` package. The runner stores values separately for each installed script and blocks a run when a required secret is missing.

Simulation values are entered explicitly in the Secrets panel, remain in memory for that simulation session, and are not saved. Secret values and values derived from them are redacted from runner reports and logs where detected. Derived metadata is deliberately not created for secrets.

See [Secrets](../runner/secrets.md) for runner configuration and lifecycle commands.

## Datetime and duration values

Both are structured objects rather than plain strings or numbers. See [Variable types](#variable-types) for their required fields.

```json
{ "type": "datetime", "value": "2026-07-11T12:00:00Z" }
```

New datetime values start with the current date and time. The timezone selector controls how the editor displays and interprets the value. Changing the timezone keeps the same instant and only changes its displayed time. Editing the date or time converts the selected timezone back to the UTC timestamp stored in the project and package.

```json
{ "type": "duration", "unit": "seconds", "value": 10 }
```

Node outputs use the same ten types as default variables, Script Settings, and Variable Operations. The Runtime Data panel identifies the exact type produced by each node.

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

Operations that need input accept either a typed value or a variable reference. Text, integer, float, and object inputs use the same code editor for both forms. Other typed inputs provide a **Raw value** source for variable references. The runner validates each resolved value before changing runtime or stored state.

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

### Datetime and duration parts

A datetime and a duration each expose their components as well, so a script can read one field without any string handling:

| Datetime suffix | Result |
| --- | --- |
| `.$year` | Calendar year |
| `.$month` | Month, 1 through 12 |
| `.$day` | Day of the month |
| `.$hour` | Hour, 0 through 23 |
| `.$minute` | Minute, 0 through 59 |
| `.$second` | Second, 0 through 59 |
| `.$weekday` | Day of the week, Monday `1` through Sunday `7` |

| Duration suffix | Result |
| --- | --- |
| `.$days` | Whole days in the span |
| `.$hours` | Hours left after the days |
| `.$minutes` | Minutes left after the hours |
| `.$seconds` | Seconds left after the minutes |
| `.$milliseconds` | Milliseconds left after the seconds |
| `.$total_milliseconds` | The whole span in milliseconds |

Duration parts are a breakdown, not the same span counted six ways: a duration of 90 seconds reads as `.$minutes` `1` and `.$seconds` `30`, with `.$total_milliseconds` `90000`.

A datetime part is read in the offset the value carries rather than converted to the machine's own zone, so `{{system_datetime.$hour}}` is the hour on the runner's wall clock.

```text
{{system_datetime.$hour}}
{{system_datetime.$weekday}}
{{n-mr3zyt6f-12.elapsed.$total_milliseconds}}
```

Metadata is refreshed whenever the underlying runtime value changes. The `$` names are reserved. An object property with the same name cannot be addressed through this derived-token form.

## Datetime format patterns

The **Format date and time** operation of [Text Transform](node-reference.md#text-transform) renders a datetime as text. Its pattern is a run of tokens; everything that is not a token is kept as written.

| Token | Example |
| --- | --- |
| `yyyy` | `2026` |
| `yy` | `26` |
| `MMMM` | `July` |
| `MMM` | `Jul` |
| `MM` | `07` |
| `M` | `7` |
| `dd` | `03` |
| `d` | `3` |
| `EEEE` | `Friday` |
| `EEE` | `Fri` |
| `HH` | `14`, 24-hour |
| `H` | `14`, 24-hour |
| `hh` | `02`, 12-hour |
| `h` | `2`, 12-hour |
| `a` | `PM` |
| `mm` | `05` |
| `m` | `5` |
| `ss` | `09` |
| `s` | `9` |

So `yyyy-MM-dd HH:mm` gives `2026-07-03 14:30`, and `EEEE d MMMM yyyy` gives `Friday 3 July 2026`.

To keep a letter as text, wrap it in single quotes: `HH:mm 'on' EEEE` gives `14:30 on Friday`. Two single quotes in a row write one quote.

A run of letters that is not a token is refused where it is written rather than emitted as itself, so a mistyped `YYYY` is reported in the editor instead of reaching the output as the literal text `YYYY`.

A pattern is read in the offset the value carries, matching the [datetime parts](#datetime-and-duration-parts). Month and weekday names are English.

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

**A derived field does not resolve:** place the suffix after the complete value name, such as `{{items.$length}}`. Every value offers `$length`, `$count`, `$type`, and `$is_empty`; a datetime or duration also offers its [parts](#datetime-and-duration-parts). The older `.$meta.*` form is not supported.

For node-specific output names and types, use [Node Reference](node-reference.md). For stored-state backup and recovery behavior, use [Storage, Backups, and Recovery](../runner/storage-backups.md).
