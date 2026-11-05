# @lango-project/lango-ui

`lango-ui` is the shared server-driven UI protocol for Lango applications.
It is written in Lango and builds form-scoped, atomic UI responses for
synchronous API dispatch. Browser rendering and navigation live in `lango-client`.

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

A Form exposes an `inherit` map, included by `definition()`:

```lango
details = new ui/Form("details")
details.inherit = {
    customerId: "customerId"
    city: {
        path: ["address", "city"]
        writable: true
    }
}
```

The nearest parent Form supplies these concrete control values at send time.
String/path-array entries are read-only; `writable: true` permits response
`vals` to update the original source. In an event handler, use
`form.customerId.value` or assign `form.city.value`. The client rejects a write
to a read-only alias before applying changes; the event proxy itself receives
ordinary client values and is not an authorization boundary. Aliases cannot
be structural targets; writable aliases accept only saved-value acknowledgements
and field-error commands. The browser coordinates overlapping managed
requests; form tokens and later claim changes are independent.

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
state explicitly in a hidden `Value` control. Event handlers run synchronously;
Promise/thenable results are rejected.

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

When no explicit replay adapter is supplied, Dispatcher first checks the
request's `context state`. A configured API Host provides its shared runtime
store here, so SQLite state configuration automatically makes UI reservations
persistent. The same context remains available inside event handlers. An
explicit `replayStore` takes precedence.

Without a host state store, `MemoryReplayStore` holds at most 10000 streams per Dispatcher until
their signed absolute session deadlines. It is process-local and loses records
on restart. Legacy calls without a session identity have no replay guarantee;
generic API endpoints outside Dispatcher are also outside this guard.
The API's `SqliteStateStore` is a ready-made persistent adapter for a local
filesystem shared by processes; see `lango-api` runtime state configuration.

Supply a shared, durable adapter through `Dispatcher(appRoot, { replayStore:
store })` for multiple server processes. The `./replay` entry exports
`MemoryReplayStore(maxForms)` to share an in-process store or change capacity.

| Method | Required contract |
| --- | --- |
| `claim(key, seqno, expiresAt)` | Atomically reserve a higher sequence when idle, returning `Accepted`, `AlreadyConsumed`, `InProgress`, `Capacity`, or `Expired`, synchronously. The key is an opaque JSON tuple; expiry is Unix seconds. |
| `complete(key, seqno)` | Atomically clear only the matching running reservation while retaining its consumed sequence until expiry. Returns synchronously; expired records may already be absent. |

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
The browser renderer and URL navigation are provided by `lango-client`. Its demo
serves a GET shell and delivers form-scoped URL events. The `Pages` registry
selects exact pages and the longest matching template prefix.

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

## Synchronous dispatch

`Dispatcher(appRoot)` executes event handlers synchronously. The previous
`synchronous: true` option remains accepted; `synchronous: false` raises
`UiAsyncDispatchRemoved`. Register direct event functions,
module objects, or module/function names through `event()` or `routesFromXml()`.
Call `preload()` during startup to resolve named modules using the application's
package context and validate every named export. Each resolved module is imported
once per dispatcher preparation; the completed registry stores callable handlers.
`handle()` then returns a plain UI result, using the same `context form`,
values/commands, and replay semantics as ordinary dispatch. No file access,
module resolution, or dynamic import occurs during prepared dispatch.

```lango
dispatcher = new ui/Dispatcher(appRoot, {
    synchronous: true
})
dispatcher.routesFromXml(uiFile)
```

The XML uses the existing event schema:

```xml
<ui>
    <event path="/app/save" mod="@shop/ui" func="save" />
</ui>
```

In a `lango-api` worker factory, return the dispatcher as a preload resource:

```lango
return {
    routes: routes.values()
    preload: [dispatcher]
}
```

The host calls each resource's `preload()` and waits for it before accepting
requests. The factory and request handlers remain synchronous. API wildcard
routes still forward requests to `dispatcher.handle(input)`. Create the dispatcher
inside the factory so every worker owns its registry.

Outside that host, `preload()` returns a Promise for the dispatcher. Concurrent
calls share the same preparation; calls after success are idempotent. Registration
is blocked while loading and after success. Failed preparation rejects without
publishing a partially resolved registry and can be retried. Synchronous dispatch
with unresolved module names, dispatch while loading, and dispatch after failed
preparation throw `UiDispatcherNotPreloaded` before replay reservation. Direct
functions/module objects can still be used synchronously without preparation.
Module names always require preparation before dispatch. XML is read by
`routesFromXml()`; prepared registries keep working after
the XML or package files are removed. Changes require a new dispatcher/restart.

The replay store's `claim` and `complete` methods and every event handler must
return synchronously. A Promise or thenable raises `UiSynchronousResultRequired`.
An accepted sequence remains consumed on handler failure, and the reservation is
released in `finally`. A failed or rejected Promise result consumes its accepted
sequence too. Worker hosts should provide a shared store through
`context state`; independent per-worker memory does not coordinate replay.

## Page templates and URL routes

`Pages(appRoot = ".")` is a separate UI registry. It does not change API endpoint
matching. Register exact pages and templates whose paths end in `/`:

```lango
pages = new ui/Pages(appRoot)
pages.template({
    path: "/"
    function: createPublicTemplate
})
pages.template({
    path: "/admin/"
    function: createAdminTemplate
})
pages.page({
    path: "/admin/login"
    function: createLoginForm
})
```

`match(url)` returns null for an unknown page, otherwise a descriptor with `url`,
`pathname`, `template`, and `page`. Query/fragment do not affect selection. Page
paths are exact; the longest matching template prefix wins. `/admin/` does not
match `/administrator` or `/admin`. Only local absolute URLs are accepted;
native URL pathname normalization applies. A match seals registration. Matching
does not invoke factories, so the GET handler can use the same registry to decide
whether to serve its fixed HTML shell; missing assets need not become HTML.

Factories receive that descriptor, run synchronously in the existing request
contexts, and return wire control definitions. A template must be a `Container`
with one direct child `Container` named `__content__`. The registry installs the
page there and names the outer template `content`. Template factories run before
page factories on first entry or a template switch. Definitions are copied before
composition, so factory-owned objects are not mutated.

The root browser Form needs hidden `Value` controls named `url` and `template`
(initially null), plus an empty `Container` named `content`. In the URL handler:

```lango
context form
context response
rendered = pages.render({
    url: form.url.value
    template: form.template.value
})
result = new ui/Response(input.formId, input.seqno)
result.replace(rendered.target, rendered.control)
result.value("template", rendered.template)
response.send(result.result())
return null
```

`render()` returns `{ template, target, control }`. With no matching current
marker, it replaces the entire `content` template. With the same marker, it
replaces only `["content", "__content__"]`; the template DOM, values and other
nested forms survive. The page Form always receives a fresh instance. Apply the
marker and replacement atomically using the normal response protocol. A failed
patch must retain the previous marker. An unknown page throws `UiPageNotFound`.

The marker describes browser state; it is not authorization or a server cache.
Every page factory must enforce its permissions even when a template is reused.
Template factories are layout builders, not guards for subsequent pages. No
worker retains the UI tree: another worker uses the same registry and browser
marker. Leaving a prefix discards its template; returning recreates it. Refresh
starts a new root Form. Clear the marker and rerun the URL event after logout or
another identity change. Screen caching and scroll restoration are not provided.

Registration accepts direct functions, module objects with a function name, or
npm module names with exported function names, like Dispatcher. For npm modules,
include `pages` in the worker factory's `preload` list; loading finishes at startup
before requests. `preload()` is idempotent, validates exports and freezes the
registry. Registries with unresolved modules reject rendering. Module resolution
starts at `appRoot`.

`pages.routesFromXml(file)` reads a separate page registry:

```xml
<ui>
  <template path="/" mod="@shop/pages" func="createPublicTemplate" />
  <template path="/admin/" mod="@shop/pages" func="createAdminTemplate" />
  <page path="/admin/login" mod="@shop/pages" func="createLoginForm" />
</ui>
```

Use Dispatcher for event entries and Pages for page/template entries. API routes
continue to choose the URL event endpoint independently; a template prefix never
captures an ordinary API request.

## Control presentation

Control, Container, Form and Button builders include `classes` and `style` in
their wire definitions. `classes` is a list of CSS class tokens; `style` maps
kebab-case CSS property names (including custom properties such as `--accent`)
to string values. Presentation belongs to definitions or commands, not `vals`.

```lango
form.saveButton.classes = ["primary"]
form.saveButton.style = {
    "background-color": "navy"
}
```

For event handlers, the same assignments through `context form` become
`setProperty` commands. The client validates `classes`/`style` before applying the
response and supports state rollback. A style command replaces server-managed
inline declarations; a classes command replaces the full class list. Null clears
the setting. Browser custom classes and theme defaults are selected explicitly
with `lango-client/Types`; wire definitions carry type names, never executable
code. Theme CSS is loaded by the application's HTML shell.

## Select and Checkbox builders

`new Select(options)` and `new Checkbox(options)` accept maps and produce the
native input definitions supported by `lango-client`:

```lango
country = form.add(new ui/Select({
    name: "country"
    label: "Country"
    value: "tr"
    event: "/preferences/change"
    options: [{
        value: "tr"
        text: "Türkiye"
    }, {
        value: "gb"
        text: "United Kingdom"
    }]
}))
subscribed = form.add(new ui/Checkbox({
    name: "subscribed"
    label: "Receive updates"
    value: false
    event: "/preferences/change"
}))
```

Select option values/text are strings; option values must be unique. Each option
may specify a Boolean `disabled`. The selected value must occur in that list or
be null for no selection. An empty string is a valid distinct option value.
Without options, the list is empty. Checkbox values are Boolean, defaulting to
false when the initial value is absent/null. Builders validate at construction
and `definition()`, including edits made through the public `options`/`value`
properties. They retain CSS classes/styles and support `event` or explicit
`events` mappings. `label` supplies an accessible name in the browser, not a
visible caption.

In event handlers, `context form.country.value` is a string/null and
`form.subscribed.value` is Boolean. Reads and writes use the normal scalar value,
inheritance and response contracts. The UI builder's option list is not a server
validation registry: handlers must validate the incoming client data separately.
Change an option list with a structural replacement. This first version supports
single selection and a two-state checkbox.

## Validation and save acknowledgement

Control builders carry `validation` and optional `trackChanges` metadata. A
Button can opt into client form validation with `validate: true`:

```lango
email.validation = {
    required: true
    maxLength: 254
    pattern: "[^ @]+@[^ @]+[.][^ @]+"
    message: "Enter a valid email address."
}
save = form.add(new ui/Button({
    name: "save"
    text: "Save"
    validate: true
}))
```

The initial rule set supports required, min/max string lengths and a full-value
RegExp pattern; optional empty fields skip length/pattern checks. Builder
`definition()` validates rule metadata. These are browser checks; request
handlers remain responsible for server validation and permissions.

A successful save must explicitly acknowledge the accepted value. Using
`context form`:

```lango
form.email.savedValue = canonicalEmail
form.country.savedValue = form.country.value
form.subscribed.savedValue = form.subscribed.value
```

These assignments emit state commands, not values. Return canonical `vals`
when changing a displayed value, and acknowledge that same value. The client
compares live data with the accepted baseline, retaining dirty state for newer
unacknowledged edits. Ordinary event responses do not imply persistence.

`form.email.error = "This email is unavailable."` sends a field error without
acknowledging a save; null clears it. The Response builder exposes the equivalent
`markClean(target, value)` and `fieldError(target, message)` helpers. Writable
inherited aliases accept only these two metadata commands in addition to value
writes; other state commands and structural patches remain forbidden. Read-only
aliases accept no state commands. Validation/dirty metadata does not enter
request `values` or change JWT/replay semantics.

A Button with `confirmDirty: true` asks before its event discards unsaved form
edits. The client handles this before queueing; canceling sends no HTTP and
consumes no seqno. Use it for actions such as New/Open in the contact-book example
under `lango-client/examples`. It is separate from `validate: true`: loading a
different record may discard an incomplete form after confirmation. The server
still validates all reads/writes and record versions.

## UI protocol 1

Dispatcher event results, Response builder output and Dispatcher HTTP errors now
include top-level numeric `protocol: 1`. `protocolVersion()` returns 1. Client
Session sends that value on UI events; versioning does not change normal API,
bootstrap, ping or logout contracts.

An explicitly unsupported request version is rejected before a replay claim or
handler execution: verified HTTP callers receive 400/`UiProtocolUnsupported`
with the supported protocol tag; standalone legacy calls throw that error.
Missing tags are accepted as version 1 for the unversioned preview transition.
New results always emit the tag. Unsupported responses are rejected by the client
before UI changes. A missing tag is a compatibility bridge, not feature
negotiation. Required fields or changes to controls, commands, value semantics or
ordering guarantees need a new protocol version. Safe optional metadata may be
added under version 1.

The compatible starter release set is compiler 0.1.10 with API/UI/client 0.2.0.
The application generator in `lango-client` pins that set and includes server
validation, shared Pages, a same-origin BrowserHost and a browser import map.
See the client README for generation and packed-package smoke verification.
