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
    return { message: "Saved." }
```

Value writes and the returned map become `vals` (returned entries win).
Other property writes emit ordered `setProperty` commands with path-array
targets, such as `["saveButton"]`. Reading a property that has not been set
during the current request raises `UiFormPropertyUnavailable`. Persist such
state explicitly in a hidden `Value` control. Capture `form` in asynchronous
callbacks that need it. Handler Promise results are supported.

Use `definition()` to serialize a control tree without server indexes or
property aliases. Containers may be unnamed. `Button` accepts either a name
or `{ name, text, event }`. `value`, `then`, and prototype-related control names
are reserved in event value trees.

The current milestone covers server event dispatch and response construction.
The browser renderer, initial GET shell, and template routing are pending.

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

Nonempty structural patches are explicitly rejected in this milestone. Control
`setValue()` and adapter methods must accept validated changes; rollback of
adapter failures is not yet implemented. Browser builds use the compiler's
`--target browser` option; see the client project's demo and build instructions.
