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

Generated JavaScript is published under `dist/` and is never edited manually.
