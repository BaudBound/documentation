---
title: Node Reference
description: Configuration, flow, outputs, access, compatibility, failures, and simulation behavior for every BaudBound node.
tags: [editor, nodes, reference]
---
# Node Reference

This page explains every node available in the editor. Use it when you need to know what a node does, what you must enter, what data it produces, or whether it works on your target machine.

Each node entry uses the same labels:

| Label | Meaning |
| --- | --- |
| **Action type** | The internal name stored in the exported package. You normally do not type or change it. |
| **Configuration** | The settings you enter in the node properties. |
| **Output** or **Data** | Values that later nodes can read. The exact reference uses the node ID shown in the editor. |
| **Flow** | The execution connection that runs next, such as `out`, `success`, or `failed`. |
| **Use** | A common reason to choose the node. |
| **Runtime** | What happens when the native runner executes the node. |
| **Simulation** | What the browser editor can test without performing the real native action. |
| **Permission** | An operation that the runner shows for human approval. |
| **Capability** | An internal feature category that the runner verifies against the graph. You do not grant capabilities manually. |
| **Risk** | The review priority calculated from the operations used by the package. |

A setting described as **variable aware** accepts either a fixed value or a reference such as `{{customer_name}}`. A fixed value stays the same for every run. A variable reference is replaced with current runtime data. [Variables and Data](variables.md) explains how references work.

Exact package field names are enforced by the linked [node schemas](../package-format/index.md). Ordinary editor users do not need to read or edit those schemas.

## Common node behavior

- Every node may have an optional **Custom name** for editor readability.
- Triggers begin runs and expose one `out` execution handle.
- Ordinary actions continue through `out`.
- Fallible actions expose `success` and `failed`. The failed route includes `error.message`, `error.code`, `error.type`, `error.retryable`, and `error.details`.
- Runtime output references use the real node ID, for example `{{n-example.status_code}}`.
- When one output connects to multiple destinations, the numbered connections execute sequentially in their explicit execution order. Canvas position does not select that order.
- Simulation descriptions are controlled tests. Native runner paths, permissions, devices, processes, and desktop state still require target-machine testing.
- Platform support defaults to all targets, then narrows through desktop-only, explicit target, and configuration-specific rules.

Risk and permission meanings are defined in [Approvals, Capabilities, and Risk](../security/approvals-capabilities.md).

## Triggers

### Manual

- **Action type:** `trigger.manual`. Capability `trigger.manual`. Low risk.
- **Configuration:** none. Only one Manual trigger is allowed per project.
- **Output:** no trigger-specific data. Continues through `out`.
- **Use:** start a script on demand from Scripts or `baudbound script run SCRIPT`.
- **Simulation:** trigger card starts immediately with optional test payload.

### Schedule

- **Action type:** `trigger.schedule`. Capability `trigger.schedule`. Low risk.
- **Configuration:** required variable-aware **Every** positive whole number. **Unit** `milliseconds`, `seconds`, `minutes`, `hours`, or `days`, default `minutes`. The resolved interval must be at least one millisecond. A fraction is refused rather than rounded; choose a smaller unit for a shorter interval, so half a second is `500` milliseconds.
- **Output:** runner payload includes interval and due-time information. Graph continues through `out`.
- **Use:** recurring work while a background service is active.
- **Runtime:** unchanged registrations preserve due timing across reload. A delayed poll emits every occurrence that came due rather than dropping any, and `missed_intervals` reports how many were still due behind each one. Millisecond schedules use operating-system timers and are not hard real-time guarantees.
- **Simulation:** fires automatically while simulation remains active.

### File Watch

- **Action type:** `trigger.file_watch`. Capability `trigger.file_watch`. Permission `file.watch.limited` for a bounded relative path or `file.watch.any` for an unbounded path. Medium or Dangerous risk.
- **Configuration:** variable-aware **Path**. Optional **Include subdirectories** for directory targets. Trigger fields can use only values available before a run begins, such as defaults and Script Settings.
- **Outputs:** `path` and normalized `event` (`created`, `modified`, `deleted`, or `renamed`).
- **Use:** react to one file or a directory tree.
- **Runtime:** target must exist and be accessible when listener registration starts. Absolute, parent-traversing, sensitive, or runtime-selected paths require `file.watch.any`. OS save behavior may emit multiple events.
- **Simulation:** supplied path/event become outputs. No filesystem watcher is opened.

### Webhook

- **Action type:** `trigger.webhook`. Capability `trigger.webhook`. Permission `network.webhook`. High risk.
- **Configuration:** HTTP **Method**. Required **Hook name** using letters `A-Z` or `a-z`, numbers `0-9`, hyphens, or underscores. Optional wait switch, positive response timeout, fallback status `100-599`, content type, and body.
- **Outputs:** `method`, `path`, `headers`, `query`, raw `body`, parsed `json`, and `response` state.
- **Use:** receive HTTP at `/events/HOOK_NAME` while webhooks are enabled.
- **Flow rule:** waiting mode requires a reachable Webhook Response node. Timeout uses the configured fallback.
- **Simulation:** creates a request and response state from supplied payload. It does not open a network port.

### WebSocket

- **Action type:** `trigger.websocket`. Capability `trigger.websocket`. Permission `network.websocket`. High risk.
- **Configuration:** required **Path**, for example `events/messages`. The editor displays the leading `/` and stores the normalized path as `/events/messages`, so you do not need to type the slash.
- **Outputs:** message text, path, `connection_id`, headers, query, and remote address.
- **Use:** begin one run per inbound text message on a matched connection.
- **Runtime:** requires enabled WebSocket listener and connection capacity. Use WebSocket Write and select this trigger as its connection source.
- **Simulation:** supplies a synthetic connection when one is not entered.

### Hotkey

- **Action type:** `trigger.hotkey`. Capability `trigger.hotkey`. Medium risk. Windows Desktop only.
- **Configuration:** choose **Literal key** to capture a chord or **Variable** to select a pre-trigger `hotkey` Script Setting. The variable selector shows only compatible hotkey values and displays the setting name without requiring braces. A single literal key such as `G`, `F1`, or `MediaPlayPause` is valid. Any distinct supported keys can form a chord, including `K+L`, `F1+T`, and `Ctrl+Shift+B`. See [Supported Windows node keys](#supported-windows-node-keys) for the exact names.
- **Output:** canonical `key` expression and timestamp.
- **Use:** start a script when the complete physical key chord is held while the Windows desktop background runner is active.
- **Matching:** the held keys must match the configured chord exactly. The run starts when the final required key is pressed, regardless of the order in which the keys were pressed. Holding the chord does not repeatedly start runs.
- **Platform:** Windows Desktop only. Firmware-only `Fn` keys and Windows secure-attention input such as `Ctrl+Alt+Delete` are not available.
- **Simulation:** uses the supplied or configured expression. It does not register a global OS hook.

### Serial Input

- **Action type:** `trigger.serial_input`. Capability `trigger.serial_input`. Permission `serial.input`. High risk.
- **Configuration:** required logical **Device id** using letters `A-Z` or `a-z`, numbers `0-9`, hyphens, or underscores.
- **Outputs:** `device_id`, received `data`, byte count, and runner `timestamp`.
- **Use:** start when a runner-mapped serial device emits data.
- **Runner framing:** the runner mapping decides whether a complete message ends after an idle gap, at a line ending, or at each raw operating system chunk. Multiple Serial Input nodes using the same logical ID share one native reader.
- **Graph rule:** the same device ID cannot be used by two Serial Input triggers in one project.
- **Simulation:** supplied text produces output and UTF-8 byte length without opening a port.

### Startup

- **Action type:** `trigger.startup`. Capability `trigger.startup`. Permission `trigger.startup`. High risk.
- **Configuration:** none.
- **Output:** startup reason and service context where available.
- **Use:** run once when an eligible script is loaded by a newly started service session.
- **Simulation:** reports a synthetic `runner_startup` reason.

### App / Process Started

- **Action type:** `trigger.process_started`. Capability `trigger.process_started`. Permission `process.observe`. Medium risk.
- **Configuration:** **Match by** process name, executable path, or window title. Required variable-aware **Target** using pre-trigger string values.
- **Outputs:** process name, process ID, executable path, and window title where available.
- **Platform:** window-title matching requires Windows Desktop. Other modes support compatible Windows/Linux targets.
- **Runtime:** polling dispatches when a process first appears, not continuously while the same process remains present.
- **Simulation:** uses supplied process facts or stable sample values.

## Control Flow

### If / Else

- **Action type:** `control.if`. Capability `runtime.if`. Low risk.
- **Configuration:** one or more condition rows with a value, operator, optional inversion, and AND/OR combinator. Most comparisons show one target. **Is between** shows an inclusive start value and end value.
- **Flow:** named `true` and `false` outputs.
- **Text operators:** equality, ordering, contains, prefix, suffix, regular expression, and case insensitive equality or contains checks.
- **Number range:** **Is between** passes when Value is equal to the start value, equal to the end value, or anywhere between them. The start must be less than or equal to the end.
- **Collection operators:** **has key** checks an object key. **contains item** checks a list item.
- **Type operators:** **Is numeric**, **Is integer**, **Is float**, **Is string**, **Is boolean**, **Is list**, and **Is object** check the type of the resolved value. **Is integer** passes for a whole number and **Is float** passes for a fractional one, so **Is numeric** passes for either. Text that reads as a number is still text. `"42"` passes **Is string** and fails **Is numeric**. Cast the value first to test it as a number.
- **Presence operator:** **Is null or missing** passes when a standalone variable reference has a null value or does not exist.
- **Checks without a target:** empty, null or missing, boolean, and type checks inspect Value directly, so the editor does not show a Target field for them. Boolean checks match only real boolean values. Text such as `"true"` and `"false"` does not match.
- **Simulation/runtime:** values are resolved with their types before comparison. A numeric variable or calculated result matches an equivalent numeric literal, so calculated `1.0` equals the literal `1`. Two text values still require exactly the same text. Inversion applies to one row before combinators.
- **Regular expressions:** regex conditions use the shared linear-time Editor/Runner subset. Unsupported constructs and oversized patterns or inputs fail validation instead of running through the browser JavaScript regex engine.
- **Example:** `{{status_code}} >= 400` routes errors to `true`.

### Color Match

- **Action type:** `control.color_match`. It uses capability `runtime.color_match`, has low risk, and does not request a permission.
- **Configuration:** variable-aware **Actual color** and **Expected color**, **Comparison mode**, and a variable-aware **Tolerance** from `0` through `100` percent. A resolved `color` Script Setting updates the color swatch beside its field.
- **Accepted colors:** canonical hex such as `#2F80ED`, RGB text such as `rgb(47, 128, 237)`, or a typed RGB object with exactly `r`, `g`, and `b` integer channels from `0` through `255`.
- **Flow:** `match` runs when the measured difference is less than or equal to the tolerance. `no match` runs for any valid pair outside the tolerance. Exactly one branch runs.
- **Per channel mode:** compares the largest red, green, or blue channel difference against the percentage tolerance. This is useful when no individual channel may drift too far.
- **Total RGB distance mode:** compares the normalized three-dimensional distance between the colors. This allows channel differences to contribute to one overall similarity value.
- **Outputs:** `matches`, `difference_percent`, `red_difference`, `green_difference`, and `blue_difference` remain available to later nodes through the Color Match node ID.
- **Validation:** `0` percent requires exact equality. `100` percent accepts every pair of valid RGB colors. Invalid or dynamically resolved malformed colors stop the node with an execution error instead of following `no match`.
- **Get Pixel Color example:** set **Actual color** to `{{n-pixel.rgb}}`, set **Expected color** to `#2F80ED`, choose a comparison mode, and enter the acceptable tolerance. The RGB object is passed directly without converting it to text.
- **Other examples:** compare two literals such as `#101820` and `rgb(16, 24, 32)`, or compare RGB object variables from any source. Color Match is available on every target runtime because it only compares data and does not read the screen.

### Switch

- **Action type:** `control.switch`. Capability `runtime.switch`. Low risk.
- **Configuration:** variable-aware **Value** and ordered case rows with stable IDs, labels, and expected values.
- **Validation:** every case needs a unique non-empty name and value. Duplicate names, duplicate values, malformed rows, and a missing Value block verification.
- **Flow:** one named output per case and an always available `default` output.
- **Runtime:** the first equal case wins. When no case matches, the runner follows `default`. Numeric variables and calculated results match equivalent numeric literals even when their displayed formatting differs. Two text values still require exactly the same text.
- **Example:** route `{{event_type}}` to `created`, `updated`, or default.

### Repeat

- **Action type:** `control.repeat`. Capability `runtime.repeat`. Low risk.
- **Configuration:** a variable-aware whole-number **Repeat count** from `1` through `18446744073709551615`.
- **Flow:** `repeat` executes the body once for every iteration. `done` continues after every iteration finishes or after Break Loop ends the Repeat early.
- **Graph rule:** let the body end naturally. Do not connect the body back to Repeat because the runner starts each new iteration automatically.
- **Simulation:** repeats the body with the same count and flow behavior used by the runner.

### Break Loop

- **Action type:** `control.break_loop`. Capability `runtime.break_loop`. Low risk.
- **Placement:** connect it inside the body of Repeat, While, or For Each.
- **Flow:** ends the nearest active loop immediately and continues from that loop's `done` output.
- **Nested loops:** only the innermost active loop ends. An outer loop continues normally.
- **Ports:** it has an input but no output because execution resumes from the loop's `done` output.
- **Failure:** verification rejects a Break Loop that is not inside a loop body. The runner also stops with a clear error if an invalid package reaches this state.

### Continue Loop

- **Action type:** `control.continue_loop`. Capability `runtime.continue_loop`. Low risk.
- **Placement:** connect it inside the body of Repeat, While, or For Each.
- **Flow:** skips every remaining step in the current iteration and starts the nearest loop's next iteration.
- **Nested loops:** only the innermost active loop advances.
- **Ports:** it has an input but no output because execution returns directly to the loop.
- **Completion:** when no next iteration exists, the loop continues from its `done` output.
- **Failure:** verification rejects a Continue Loop that is not inside a loop body. The runner also stops with a clear error if an invalid package reaches this state.

### While

- **Action type:** `control.while`. Capability `runtime.while`. Low risk.
- **Configuration:** the same condition rows and inversion behavior as If/Else.
- **Flow:** `loop` executes while conditions pass. `done` follows the first false result.
- **Graph rule:** no explicit edge returns to While.
- **Safety:** ensure body state can make the condition false. Runtime limits and cancellation remain important.

### For Each

- **Action type:** `control.for_each`. Capability `runtime.for_each`. Low risk.
- **Configuration:** variable-aware **Items** resolving to a list.
- **Flow:** `loop` runs once per item. `done` follows completion.
- **Data:** `{{node-id.item}}` holds the current item and `{{node-id.index}}` holds its zero-based index. Replace `node-id` with the ID shown in the node's Runtime Data section.
- **Failure:** non-list input fails control-flow validation/execution.
- **Example:** for a node with ID `n-example`, read the active item with `{{n-example.item}}` and its position with `{{n-example.index}}`.

## Data and Output

### Variable Operation

- **Action type:** `runtime.set_variable`. Capabilities `runtime.variables` and, for stored scopes, `runtime.persistent_storage`. Fallible.
- **Configuration:** operation `set`, `increment`, `toggle_boolean`, `append_list`, `remove_list_items`, `set_object_field`, `remove_object_field`, `merge_object`, `clear`, or `delete`. Name. Scope `runtime`, `persistent`, or `global`. Set also declares a value type, and Set list declares one item type. Other operations use operation-specific values, removal modes, and field paths. Object field writes declare the field value type.
- **Clear and Delete:** require a variable name and scope. Neither requires a variable type or value. Clear derives the empty value from the existing variable and fails if it does not exist.
- **List operations:** Append infers the item type and rejects an item that differs from existing entries. Remove matching list items compares the resolved value and type exactly.
- **Access:** runtime scope requires `variable.local.set` at Low risk, persistent requires `variable.persistent.set` at Medium risk, and global requires `variable.global.set` at High risk.
- **Data:** writes `{{name}}` and refreshes `$length`, `$count`, `$type`, and `$is_empty`.
- **Validation:** names use letters `A-Z` or `a-z`, numbers `0-9`, hyphens, or underscores. The `manifest_` and `system_` prefixes and the exact name `settings` are reserved.
- **Failure:** invalid values, mixed list item types, incompatible existing values, invalid object paths, and storage write errors continue through `failed` with structured error details. A failed operation does not modify the variable. Removing a field that is already missing succeeds without changing the object.
- **Simulation:** updates current simulation state. Runner persistence must be tested separately.

### Calculate

- **Action type:** `action.calculate`. Capability `action.calculate`. Permission `calculate`. Low risk. Fallible.
- **Configuration:** variable-aware numeric **Expression**. The editor requires a valid supported formula and rejects arbitrary text before simulation or export.
- **Output:** `result` float on success. Structured error on failure.
- **Use:** arithmetic supported by the runtime expression evaluator, not arbitrary code.
- **Simulation:** evaluates with current values using the same supported expression rules.

### Convert Value

- **Action type:** `action.value.convert`. Capability `action.value`. Permission `value.convert`. Low risk. Fallible.
- **Configuration:** a variable-aware **Value** and a target type of `string`, `integer`, `float`, `boolean`, `object`, `list`, `color`, `hotkey`, `datetime`, or `duration`.
- **Null:** `null` is rejected for every target. An unset variable or a missing object field resolves to `null`.
- **String rules:** accepts any non-null value. An existing string passes through unchanged, and every other type is converted to its JSON text form.
- **Integer rules:** the value must already be a whole number within the safe integer range. Convert Value does not round decimal values.
- **Float rules:** accepts an integer, a float, or non-empty numeric text. The result always renders with a decimal point.
- **Boolean rules:** accepts a boolean value or the text `true` or `false` without regard to letter case.
- **List and object rules:** accepts an existing value of the selected type or JSON text whose top-level value has the selected type.
- **Color and hotkey rules:** accepts only text, checked against the same [color](variables.md#variable-types) or [hotkey](#supported-windows-node-keys) rule as the matching variable type.
- **Datetime rules:** accepts an existing datetime value, or a bare RFC 3339 string, which is wrapped into the datetime object automatically.
- **Duration rules:** accepts only an existing duration value. There is no conversion from a plain number or text.
- **Output:** `value`, `source_type`, and `target_type` on success. Structured error details are available from `failed`.

### Text Transform

- **Action type:** `action.text.format`. Capability `action.text`. Permission `text.transform`. Low risk. Fallible.
- **Configuration:** one initial input followed by an ordered list of operations. Drag operations to change their order.
- **Operations:** template, trim, uppercase, lowercase, sentence case, capitalize words, literal/regex replace, split, join, substring, padding, URL/Base64 encode/decode, and JSON escape/unescape.
- **Order:** each operation receives the result from the operation above it. Split changes text into a list. Join changes a list back into text.
- **Output:** the final value is available as `text` or `items` according to its type.
- **Failure:** invalid regex, encoding, indexes, or an incompatible value type continues through `failed`.
- **Regular expressions:** replacement patterns use the same bounded linear-time subset in the Editor and Runner. Simulation executes regex work outside the UI thread and can be cancelled.

### Parse URL

- **Action type:** `action.url.parse`. Capability `action.text`. Permission `url.parse`. Low risk. Fallible.
- **Configuration:** a variable-aware absolute URL. Standard protocols such as `https` and custom protocols such as `ptr` are supported.
- **Outputs:** `protocol`, `host`, `port`, `path`, raw `query`, decoded `query_parameters`, and `fragment`.
- **Query parameters:** an ordered list of objects with `name` and `value` fields. Repeated names remain separate entries and keep their original order.
- **Failure:** missing, relative, and malformed URLs continue through `failed` with structured error details.
- **Example:** `ptr://command/move?param=value1` produces protocol `ptr`, host `command`, path `/move`, and query parameters `[ { "name": "param", "value": "value1" } ]`.

### Log

- **Action type:** `action.log`. Capability `action.log`. Permission `log`. Low risk.
- **Configuration:** level `info`, `warn`, `error`, or `debug`. Variable-aware **Message**.
- **Output:** no action-specific data. Continues through `out`.
- **Runtime:** stores the resolved message with node and run identity.
- **Security:** do not intentionally log secrets or untrusted payloads without review.

### Delay

- **Action type:** `action.delay`. Capability `action.delay`. Permission `delay`. Low risk. Fallible.
- **Configuration:** variable-aware positive whole **Amount** and unit milliseconds, seconds, minutes, hours, or days. The resolved duration must be at least one millisecond. A fraction is refused rather than rounded; choose a smaller unit for a shorter pause, so half a second is `500` milliseconds.
- **Flow:** `success` continues after the cancellable wait. An invalid resolved duration continues through `failed` with structured error details. Cancelling a run stops execution instead of following `failed`.
- **Simulation:** validates the resolved duration and records the simulated delay without blocking the UI thread.

### Beep

- **Action type:** `action.beep`. Capability `action.sound`. Permission `beep`. Low risk. Desktop only. Fallible.
- **Configuration:** required variable-aware positive frequency Hz and duration ms.
- **Flow/data:** success or failed with structured error.
- **Runtime:** plays a generated tone through the default desktop audio output.
- **Simulation:** Web Audio sine tone clamped to safe editor bounds. Browsers may block audio before interaction.

### Show Notification

- **Action type:** `action.notification`. Capability `action.notification`. Permission `notification.show`. Medium risk. Desktop only. Fallible.
- **Configuration:** variable-aware **Title** and **Message**.
- **Output:** success/failure state.
- **Simulation:** editor toast rather than native notification-center behavior.

### Message Dialog

- **Action type:** `action.message_box`. Capability `action.message_box`. Permission `messageBox.show`. Medium risk. Windows Desktop and Linux Desktop. Fallible.
- **Configuration:** variable-aware title and message, type Info/Warning/Error, window size, buttons OK, OK/Cancel, Cancel/Confirm, Yes/No, or Yes/No/Cancel, and an optional positive timeout from greater than `0` through `86400` seconds.
- **Runtime:** opens a BaudBound desktop dialog while the main runner window stays responsive. Dialog requests are queued first-in, first-out, with one active request at a time. When Dialog Console mode is enabled, the active request is rendered in the persistent console instead of a transient window.
- **Output:** `button` is `ok`, `cancel`, `confirm`, `yes`, `no`, or `timeout`. Non-timeout values exactly match a button from the configured set. Closing OK behaves as OK. Closing a set that contains Cancel behaves as Cancel. A Yes/No dialog must be answered explicitly and cannot be closed into an ambiguous result. A configured timeout is displayed as a countdown and returns `timeout` when it expires.
- **Cancellation:** stopping the run, restarting the runner, or closing the runner cancels the waiting action. These are runtime cancellations, not button results.
- **Simulation:** uses a blocking editor modal and returns the selected button. Stopping simulation aborts the wait.

### Form Dialog

- **Action type:** `action.form_dialog`. Capability `action.form_dialog`. Permission `formDialog.show`. Medium risk. Windows Desktop and Linux Desktop. Fallible.
- **Common configuration:** required variable-aware window title, optional variable-aware description, an ordered draggable component list, and an optional positive timeout from greater than `0` through `86400` seconds. A form must contain from 1 through 50 components.
- **Input components:** Text input, Password input, Multiline text, Number input, Checkbox, Single choice, Multi choice, Dropdown, Date, Time, Date and time, Color picker, File picker, Folder picker, and Slider. The editor shows only settings owned by the selected component type. Component types are selected when a component is added and cannot be changed afterward.
- **Display components:** Information, Section heading, Divider, and Image. Information, Section heading, and Divider have configurable variable-aware accent colors. Images use packaged project image assets, support contain or cover fitting, and are limited to 8 MiB.
- **Field keys:** every input component has a literal output key. Keys must be unique identifiers of at most 64 characters and use letters `A-Z` or `a-z`, numbers `0-9`, hyphens, or underscores. Display components have no key or output.
- **Typed values:** Number and Slider return floats; Checkbox returns a boolean; Multi choice and a multi-file File picker return lists; Color picker returns a normalized `#RRGGBB` color; File picker and Folder picker return filesystem paths as text; Date and Time return their displayed ISO-shaped values; Date and time returns an ISO 8601 UTC timestamp; the remaining inputs return text.
- **Path selection:** the desktop runner opens an authenticated native operating-system picker. BaudBound does not restrict which files or folders the signed-in user can select. Canceling a picker leaves the existing value unchanged.
- **Defaults and variables:** Text and Multiline defaults are variable-aware, while Password has no default value. Number defaults accept float variables and must resolve to a finite supported number. Date, Time, and Date and time defaults use either their literal picker or one complete `datetime` variable reference; string variables are rejected even when their text resembles a date. A typed datetime is displayed in local time for Date and Time; Date and time uses the component's configured timezone. Checkbox has a boolean default.
- **Choices:** each choice has a variable-aware key and displayed value. The dialog shows only the displayed value and returns only the key. Both must resolve to non-empty unique text. Each choice component supports 1 through 100 choices, and multi-choice results preserve configured order rather than click order.
- **Required values:** required text must not be empty, required numbers must be present and finite, required checkboxes must be checked, and required choice components need a selection. An optional blank number is omitted from `values`.
- **Buttons and outputs:** every Form Dialog displays Cancel and Submit. `values` is an object whose typed fields are derived from component keys, `submitted` is true only after Submit, and `button` is `ok`, `cancel`, or `timeout`.
- **Cancel and timeout:** Cancel, Escape, or window close returns `submitted=false`, `button="cancel"`, and an empty `values` object. Timeout returns the same empty values with `button="timeout"`. Stopping the runtime is a cancellation and produces no normal outputs.
- **Password handling:** each submitted password field is transient sensitive data. It remains usable through its nested reference during the active run, but its value is redacted from logs and cannot be copied into persistent or global variables. The containing `values` output is omitted from retained run variables; non-secret sibling values can still be copied to ordinary downstream variables. This is an application window, not an operating-system secure desktop or credential vault; other software with desktop capture or input-monitoring access may still observe it.
- **Validation boundary:** the editor, simulator, package schema, desktop renderer, and runner independently reject malformed components, duplicate keys, wrong variable and response types, unknown response fields, and values outside configured choices. Editor suggestions expose only compatible variable types, and whole-script verification blocks known type mismatches or typed references whose type cannot be established.
- **Runtime and simulation:** dialog requests are queued first-in, first-out, with one active request at a time. Dialog Console mode uses its persistent window. Simulation uses a blocking editor form with the same component behavior, typed output shape, configured choice ordering, timeout behavior, and password redaction as the runner.

### Play Sound

- **Action type:** `action.sound.play`. Capability `action.sound`. Permission `sound.play` for package audio. A filesystem source also declares `file.read` for a bounded relative path or `file.read.any` for an unbounded path. Medium or Dangerous risk. Desktop only. Fallible.
- **Configuration:** source package asset or file path and corresponding selected path.
- **Output:** played source/path information and failure data.
- **Simulation:** plays package audio in the browser. A runner filesystem path cannot be tested there.

## Network and Serial

### HTTP Request

- **Action type:** `action.http`. Capability `action.http`. Permission `http.request`. Medium risk. Fallible.
- **Configuration:** method, body format, variable-aware URL, headers, body, timeout `1-300` seconds, and user agent.
- **JSON bodies:** JSON mode parses the body before variables are inserted. Put variable references inside JSON string quotes. A reference that fills the complete string keeps its original type. Strings are escaped safely, including quotes, backslashes, carriage returns, and newlines.
- **Text bodies:** Text mode inserts variables directly and sends the resulting text without JSON processing.
- **Existing projects:** when body format is not present, the runner treats the body as JSON if its `Content-Type` is `application/json` or an `application/*+json` media type.
- **Outputs:** status code/text, headers, body, optional parsed `json`, duration, or structured network error.
- **Runtime:** native HTTP client behavior may differ from browser redirects, CORS, forbidden headers, cookies, and TLS stores.
- **Private destinations:** the runner blocks loopback, private, link-local, and other non-public addresses by default. The operator must explicitly enable `security.policy.allow_private_http_requests` when an approved workflow needs them.
- **Simulation:** mock mode is the default and performs no network access or secret interpolation. Live mode requires per-run approval of the literal destination origins, then sends through the bounded Editor service with Runner-equivalent address, redirect, header, timeout, cancellation, and response-size policy.

### Webhook Response

- **Action type:** `action.webhook_response`. Capability `action.webhook_response`. Permission `webhook.response`. Low risk. Fallible.
- **Configuration:** variable-aware status `100-599`, content type, headers, and body.
- **Outputs:** `sent`, status, content type, headers, body, owning `trigger_id`, or error.
- **Graph rule:** must be reachable from a Webhook trigger with waiting enabled.
- **Runtime:** exactly one response owns one pending request.

### WebSocket Write

- **Action type:** `action.websocket.write`. Capability `action.websocket`. Permission `websocket.write`. Medium risk. Fallible.
- **Configuration:** select the **Connection** from an available WebSocket Trigger and enter a variable-aware **Message**. The editor stores the selected trigger's `connection_id` reference.
- **Outputs:** send result/byte information or connection error.
- **Use:** reply through the connection emitted by the selected WebSocket Trigger for the current run.
- **Graph rule:** the selected WebSocket Trigger must still exist and must be able to reach the WebSocket Write node through the graph.
- **Failure:** unknown, stale, or disconnected IDs are rejected.

### Serial Write

- **Action type:** `action.serial.write`. Capability `action.serial`. Permission `serial.write`. Medium risk. Fallible.
- **Configuration:** logical device ID selected from Serial Input triggers, variable-aware data, and line ending none/LF/CRLF.
- **Output:** write result or structured serial error.
- **Runner connection:** Serial Write uses the same logical-device connection as Serial Input. It does not open a competing port handle when a reader is active.
- **Graph rule:** requires a Serial Input trigger using the same logical ID.
- **Simulation:** reports intended bytes and line ending without opening hardware.

## Files

### Read File

- **Action type:** `action.file.read`. Capability `action.file`. Permission `file.read` for a bounded relative path. Medium risk. Fallible.
- **Configuration:** variable-aware path. Files are always decoded as UTF-8, so there is no encoding selector.
- **Outputs:** content, byte count, resolved path, or error.
- **Access:** absolute, sensitive, parent-traversing, or runtime-selected paths require the dangerous `file.read.any` permission instead of `file.read`.
- **Simulation:** sample output only. Runner account permissions and file existence remain untested.

### Write File

- **Action type:** `action.file.write`. Capability `action.file`. Permission `file.write.limited` for a bounded relative path. High risk. Fallible.
- **Configuration:** variable-aware path/content and mode overwrite or append.
- **Outputs:** resolved path, bytes written, mode, or error.
- **Access:** absolute, sensitive, parent-traversing, or runtime-selected paths require the dangerous `file.write.any` permission instead of `file.write.limited`.
- **Review:** paths can be influenced by variables. Confirm they cannot escape intended storage.

### Download File

- **Action type:** `action.file.download`. Capabilities `action.file` and network behavior. Permission `file.download`. Medium risk. Fallible.
- **Configuration:** variable-aware URL and destination path. Overwrite switch.
- **Outputs:** destination, transferred size/status information, or error.
- **Access:** an absolute, sensitive, parent-traversing, or runtime-selected destination also requires `file.write.any`.
- **Review:** validate both remote source and local overwrite consequences.

### Delete File

- **Action type:** `action.file.delete`. Capability `action.file`. Permission `file.delete.limited` for a bounded relative path or `file.delete.any` for an unbounded path. High or Dangerous risk. Fallible.
- **Configuration:** variable-aware path.
- **Output:** deleted path or error.
- **Warning:** deletion is not a recycle-bin operation. Restrict input and test on disposable data.

### Copy File

- **Action type:** `action.file.copy`. Capability `action.file`. Permission `file.copy` plus path-dependent read and write permissions. Medium risk before path escalation. Fallible.
- **Configuration:** variable-aware source and destination. Overwrite switch.
- **Outputs:** resolved paths, copy result, or error.
- **Access:** a broad source requires `file.read.any`. A broad destination requires `file.write.any`.

### Move File

- **Action type:** `action.file.move`. Capability `action.file`. Permission `file.move` plus path-dependent read and write permissions. Medium risk before path escalation. Fallible.
- **Configuration:** variable-aware source and destination. Overwrite switch.
- **Outputs:** resolved paths, move result, or error.
- **Access:** a broad source requires `file.read.any`. A broad destination requires `file.write.any`.
- **Review:** a successful move removes the original path.

## Processes and Windows

### Run Process

- **Action type:** `action.process.run`. Capability `action.process`. Permission `process.run`. Dangerous risk. Fallible.
- **Configuration:** variable-aware executable, arguments, optional working directory, and optional timeout from `1` to `86400` seconds, default `300`.
- **Outputs:** process ID, exit code, success flag, captured standard output, captured standard error, or action error.
- **Runtime:** uses native process APIs, not shell parsing. Arguments must match the target executable's contract.
- **Program lookup:** a name without a path separator is looked up on `PATH`, so `git` works. The working directory is never searched, which stops a file placed there from being run in place of the intended program. To run a program from a specific directory, give a path such as `./tool.exe` or a full path.

### Process Status

- **Action type:** `action.process.status`. Capability `action.process`. Permission `process.query`. Medium risk. Fallible.
- **Configuration:** match by process name, executable path, or window title. Variable-aware target.
- **Outputs:** running flag, matching process information, or error.
- **Platform:** window-title mode requires Windows Desktop.

### Kill Process

- **Action type:** `action.process.kill`. Capability `action.process`. Permission `process.kill`. High risk. Fallible.
- **Configuration:** match by process name, executable path, window title, or PID. Variable-aware target.
- **Outputs:** matched/terminated process information or error.
- **Platform:** window-title mode requires Windows Desktop.

### Open Application

- **Action type:** `action.application.open`. Capability `action.window`. Permission `process.run`. Dangerous risk. Desktop only. Fallible.
- **Configuration:** variable-aware application name/ID/shortcut/desktop entry and arguments.
- **Outputs:** resolved application ID and process ID when exposed.
- **Authorization:** launching an application with arguments is process execution and is subject to the same dangerous-command policy as Run Process immediately before creation.
- **Simulation:** returns sample IDs without opening an application.

### Get Active Window

- **Action type:** `action.window.active`. Capability `action.window`. Permission `window.query`. Medium risk. Windows Desktop only. Fallible.
- **Configuration:** none.
- **Outputs:** window title, process ID/name, executable path, and native handle where available.
- **Simulation:** sample window data. No native lookup.

### Window Focus

- **Action type:** `action.window.focus`. Capability `action.window`. Permission `window.focus`. High risk. Windows Desktop only. Fallible.
- **Configuration:** match by process name, executable path, or window title. Variable-aware target.
- **Outputs:** focused window/process details or error.
- **Review:** focus changes can redirect subsequent keyboard or mouse actions.

### Get Pixel Color

- **Action type:** `action.pixel.get`. Capability `action.pixel`. Permission `screen.pixel.read`. Medium risk. Windows Desktop only. Fallible.
- **Configuration:** variable-aware signed integer screen X and Y coordinates from `-2147483648` through `2147483647`.
- **Outputs:** coordinates, RGB channels, hex color, and error on failure.
- **Simulation:** deterministic sample color derived for testing, not a real screenshot read.
- **Runtime:** coordinates use the Windows virtual desktop. Negative values address monitors to the left of or above the primary display. Points in gaps between monitors are rejected.
- **Tools:** use the desktop runner coordinate picker to select a point and copy its X coordinate, Y coordinate, pair, or sampled color.

## Input Control

### Set Clipboard

- **Action type:** `action.clipboard.set`. Capability `action.clipboard`. Permission `clipboard.write`. Medium risk. Desktop only. Fallible.
- **Configuration:** variable-aware value to write.
- **Output:** written text, its UTF-8 byte length, or an error.
- **Review:** replaces the user's current clipboard and may expose data to other applications.

### Get Clipboard

- **Action type:** `action.clipboard.get`. Capability `action.clipboard`. Permission `clipboard.read`. Medium risk. Desktop only. Fallible.
- **Configuration:** none.
- **Output:** clipboard text or an error when text is unavailable.
- **Review:** clipboard text can contain passwords or other sensitive data. Avoid sending it to logs or external services unless that is intentional.
- **Platform:** requires a signed-in Windows or Linux desktop session with a working native clipboard provider.

### Keyboard

- **Action type:** `action.keyboard`. Capability `action.keyboard`. Permission `keyboard.control`. High risk. Windows Desktop only. Fallible.
- **Configuration:** choose an input action and either a literal key chord or a compatible `hotkey` Script Setting. Literal mode can capture keys or use the key reference buttons. Variable mode lists only compatible settings and displays the setting name without requiring braces. Separate literal chord members with `+`, for example `G`, `F1`, `K+L`, or `Ctrl+Shift+S`.
- **Supported keys:** uses the same [Supported Windows node keys](#supported-windows-node-keys) as the Hotkey trigger. Unsupported names are rejected instead of being guessed.
- **Platform:** Windows Desktop only.
- **Output:** the normalized key expression, the performed input action, or an error.
- **Review:** ensure the intended application has focus. Use Type Text for words and arbitrary text.

The input action controls what happens to every key in the configured chord.

| Input action | Behavior |
| --- | --- |
| Press and release | Performs a normal key press. Modifier keys are pressed first, the final key is pressed and released, then the modifiers are released. |
| Press down | Holds every configured key. A later Keyboard node can release the same keys. |
| Release | Releases only keys held by the current run. Using Release without a matching Press down is safe and does nothing. |

Held keys belong to the run that pressed them. The runner releases them automatically when that run completes, fails, or is stopped. When concurrent runs hold the same key, the physical key is released only after every owning run has released it or ended.

#### Supported Windows node keys

Hotkey and Keyboard nodes use one shared Windows key contract. The editor, exported package checks, global hotkey service, and native Keyboard action all validate the same names. The editor captures held keys and also provides a button for every canonical key name.

| Group | Supported canonical names |
| --- | --- |
| Modifiers | `Ctrl`, `Alt`, `Shift`, `Windows` |
| Letters and digits | `A` through `Z`, `0` through `9` |
| Function | `F1` through `F24` |
| Navigation and editing | `Escape`, `Enter`, `Space`, `Tab`, `Backspace`, `Delete`, `Insert`, `Home`, `End`, `PageUp`, `PageDown`, `ArrowUp`, `ArrowDown`, `ArrowLeft`, `ArrowRight` |
| System and lock | `CapsLock`, `NumLock`, `ScrollLock`, `PrintScreen`, `Pause`, `ContextMenu` |
| Punctuation | `Semicolon`, `Equal`, `Comma`, `Minus`, `Period`, `Slash`, `Backquote`, `BracketLeft`, `Backslash`, `BracketRight`, `Quote`, `IntlBackslash` |
| Numpad | `Numpad0` through `Numpad9`, `NumpadMultiply`, `NumpadAdd`, `NumpadSeparator`, `NumpadSubtract`, `NumpadDecimal`, `NumpadDivide` |
| Browser | `BrowserBack`, `BrowserForward`, `BrowserRefresh`, `BrowserStop`, `BrowserSearch`, `BrowserFavorites`, `BrowserHome` |
| Media and volume | `VolumeMute`, `VolumeDown`, `VolumeUp`, `MediaNext`, `MediaPrevious`, `MediaStop`, `MediaPlayPause` |
| Application launch | `LaunchMail`, `LaunchMedia`, `LaunchApp1`, `LaunchApp2` |

Firmware-managed keys such as `Fn` and Windows secure-attention input such as `Ctrl+Alt+Delete` cannot be captured or generated. Browser, media, and Windows-key combinations may also be reserved by the browser, Windows, or another application. Build those combinations with the key-reference buttons when the browser cannot capture them. For example, pressing `Ctrl+W` normally asks the browser to close the tab, so use the `Ctrl` and `W` buttons instead.

### Type Text

- **Action type:** `action.keyboard.type_text`. Capability `action.keyboard`. Permission `keyboard.control`. High risk. Windows Desktop only. Fallible.
- **Configuration:** variable-aware text.
- **Platform:** Windows Desktop only.
- **Output:** typed length/status or error.
- **Security:** never type secrets into an unverified foreground target.

### Mouse Click

- **Action type:** `action.mouse`. Capability `action.mouse`. Permission `mouse.control`. High risk. Windows Desktop only. Fallible.
- **Configuration:** choose left, right, middle, back, or forward and then choose an input action. Single and double click are available for Press and release.
- **Output:** the mouse button, the performed input action, click type when applicable, or an error.
- **Platform:** Windows Desktop only.
- **Runtime:** acts at the current pointer position.

| Input action | Behavior |
| --- | --- |
| Press and release | Performs a single or double click. |
| Press down | Holds the selected mouse button. |
| Release | Releases the selected button only when the current run holds it. |

Held mouse buttons use the same run ownership and automatic cleanup as held keyboard keys. The runner releases them when the run completes, fails, or is stopped.

### Move Mouse

- **Action type:** `action.mouse.move`. Capability `action.mouse`. Permission `mouse.control`. High risk. Windows Desktop only. Fallible.
- **Configuration:** variable-aware signed integer X/Y and relative switch.
- **Output:** final coordinates/movement details or error.
- **Platform:** Windows Desktop only.
- **Runtime:** absolute coordinates use the Windows virtual desktop and must belong to a connected monitor. Relative values remain signed offsets from the current pointer position.
- **Review:** use the runner Tools tab to inspect monitor ranges and display scaling or select an exact point with the coordinate picker.

## Scripts and System

### Sub-script

- **Action type:** `action.script.run`. Capability `action.sub_script`. Permission `script.run`. High risk. Fallible.
- **Configuration:** installed child script name or ID.
- **Outputs:** child run ID/status/report summary or error.
- **Runtime:** child must be independently installed, valid, current, approved, and manually runnable. Parent approval does not approve the child.

### Shell Command

- **Action type:** `action.shell`. Capability supplied through process execution. Permission `process.shell`. Dangerous. Fallible.
- **Configuration:** variable-aware command string interpreted by the target shell and optional timeout from `1` to `86400` seconds, default `300`.
- **Outputs:** exit code, stdout, stderr, or error.
- **Platform:** syntax is platform-specific even under a Generic target.
- **Warning:** prefer a native node or Run Process. Shell interpolation can turn data into arbitrary commands and has independent runner policy gates.

## Related references

- [Variables and Data](variables.md) for output syntax and types.
- [Target Runtimes](target-runtimes.md) for compatibility enforcement.
- [Background Service and Triggers](../runner/service-triggers.md) for listener operation.
- [Webhooks, WebSockets, and Network Access](../runner/network-listeners.md) for network exposure.
- [Configuration and Serial Devices](../runner/configuration.md) for logical hardware mapping.
