# @lango-project/lango-ui

`lango-ui` is the shared server-driven UI protocol for Lango applications.
It is written in Lango. Its initial library builds form-scoped, atomic UI
responses; browser rendering and API integration follow in later milestones.

## Response builder

```lango
import js "@lango-project/lango-ui" as ui

response = new ui/Response("profile", 12)
response.value("email", "ada@example.com")
response.setEnabled("saveButton", false)
response.focus("email")
return response.result()
```

The resulting response contains only form values in `vals`. Structural changes
belong in `ui`; control state and client actions belong in ordered `cmds`.
Every response identifies its target form and request sequence number.

## Controls and forms

Control names are unique within their immediate container. A container retains its controls in
wire-order under `controls`, while its name index remains server-only. Adding a
named control also exposes it as a property, so application code can use:

```lango
form = new ui/Form("profile")
form.add(new ui/Button("saveButton"))
form.saveButton.enabled = false
```

Generated JavaScript is published under `dist/` and is never edited manually.

## Event dispatch

`Dispatcher(appRoot)` owns UI event routes. Call `event({ path, function })`
with a function reference, or `routesFromXml(file)` with an XML definition:

```xml
<ui>
  <event path="/app/save" mod="@shop/profile" func="save" />
</ui>
```

Delegate an API wildcard endpoint to `dispatcher.handle(input)`. The dispatcher
uses `context request.path` and builds `context form` from the incoming
`values` tree. Each named control contributes its value, including `null` for
controls without a data value. Nested containers contribute nested maps.

```lango
save(input)
    context form
    form.saveButton.enabled = false
    form.address.city.value = "Ankara"
    return {
        message: "Saved."
    }
```

Value writes and the returned map become `vals` (returned entries win).
Other property writes emit ordered `setProperty` commands with path-array
targets, such as `["saveButton"]`. Reading a property that has not been set
during the current request raises `UiFormPropertyUnavailable`. Persist such
state explicitly in a hidden `Value` control. Capture `form` in asynchronous
callbacks that need it. Handler Promise results are supported.

## Event replay protection

With a verified API token context, Dispatcher reserves each request before
module loading or handler execution. Its key is the signed `sessionId`, incoming
`formId`, and incoming `formInstanceId`. The client generates a new instance ID
for every Session, including replacements and logout restarts. Protected requests
require a positive safe-integer `seqno` and an instance ID of 1–128 characters.

| Condition | Result |
| --- | --- |
| Higher sequence, idle stream | Reserve it and execute the handler; gaps are allowed. |
| Consumed sequence, including one still running | 409 `UiReplayConflict`, reason `AlreadyConsumed`; no execution. |
| Higher sequence while another event is running | 409 `UiReplayConflict`, reason `InProgress`; the higher sequence is not consumed. |
| Full store | 503 `UiReplayUnavailable`; no live record is evicted. |
| Expired session | 401; no execution. |

Handler exceptions, rejected Promises, and invalid results consume their sequence
and release the running flag. Changing the path or values cannot reuse a consumed
sequence. Streams for different sessions, forms, and replacement instances are
independent. Ping does not consume a sequence. API auth callbacks and request
audits are outside this guard and may run for repeated HTTP requests.

The client does not automatically retry. A lost response may hide a completed
operation; a duplicate returns a conflict, not a cached response. A new sequence
or instance ID is a new event, not a safe retry. These fields are client-supplied
stream identifiers, not authorization or protection against a stolen bearer
token. Business idempotency and durable outcomes require application handling.

The default `MemoryReplayStore` holds at most 10000 streams per Dispatcher until
their signed absolute session deadlines. It is process-local and loses records
on restart. Legacy calls without a session identity have no replay guarantee;
generic API endpoints outside Dispatcher are also outside this guard.

Supply a shared, durable adapter through `Dispatcher(appRoot, { replayStore:
store })` for multiple server processes. The `./replay` entry exports
`MemoryReplayStore(maxForms)` to share an in-process store or change capacity.

| Method | Required contract |
| --- | --- |
| `claim(key, seqno, expiresAt)` | Atomically reserve a higher sequence when idle, returning `Accepted`, `AlreadyConsumed`, `InProgress`, `Capacity`, or `Expired`, directly or through a Promise. The key is an opaque JSON tuple; expiry is Unix seconds. |
| `complete(key, seqno)` | Atomically clear only the matching running reservation while retaining its consumed sequence until expiry. May return a Promise; expired records may already be absent. |

All dispatchers serving the same sessions must share the store and namespace.
Claims must be committed before returning `Accepted`; rejected claims must not
advance the sequence. Errors and unknown outcomes fail closed. Never evict a
live record to free capacity. Crash recovery must not blindly release and
re-execute a reservation: external effects may already have happened.

Use `definition()` to serialize a control tree without server indexes or
property aliases. Containers may be unnamed. `Button` accepts either a name
or `{ name, text, event }`. `value`, `then`, and prototype-related control names
are reserved in event value trees.

The current milestone covers server event dispatch and response construction.
The browser renderer is provided by `lango-client`; the initial GET shell and
template routing are pending.

## Client protocol core

The client core now lives in the separate `@lango-project/lango-client`
project. Its protocol entry exports `collectValues(controls)` and
`applyResponse(controls, response, formId, seqno, effects)`.
Controls provide `name`, `getValue()`, and `setValue(value)`; containers expose
their children as `controls` and collect their own nested value map.

The response consumer checks form and request sequence identity and validates
all value and command targets before applying changes. It currently supports
`setEnabled`, `setVisible`, and `setProperty` for Boolean `enabled`/`visible`,
plus ordered `focus` and `toast` effects supplied by the host adapter. Nested
targets use path arrays; a string addresses a top-level control.

The client supports structural replacement/removal through its `Renderer`.
`Response.replace(["address", "city"], definition)` builds a nested replacement
map; `Response.remove(["address", "warning"])` retains the path as an array.
Top-level string targets remain supported. The client rolls back structural
and value changes on failure. Commands run afterward in list order; a failing
state command restores its own state, while earlier commands remain committed.
Custom adapters provide capture/restore hooks; see the client documentation.
Browser builds use the compiler's
`--target browser` option; see the client project's demo and build instructions.
