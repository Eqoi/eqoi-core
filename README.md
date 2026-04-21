# Eqoi Framework

Eqoi is a high-level native application framework for Dolet.

It is built on top of:

- `window` for native windows and event handling
- `ui` for native drawing, colors, text, mouse state, and draw commands

Eqoi is intended for app builders who want a cleaner API than raw drawing calls while still keeping the native stack small and direct.

## First API

```dolet
import window
import ui
import eqoi

app: EqoiApp = EqoiApp.create("Eqoi App", 800, 600)
if app.is_valid() == 0:
    print("failed to create app")
else:
    count: i32 = 0

    while app.should_close() == 0:
        if app.begin() == 1:
            app.title(32, 32, "Eqoi")
            app.muted_label(32, 72, "Native apps with a small framework layer.")

            if app.button(32, 112, 180, 44, "Click") == 1:
                count = count + 1

            app.label(32, 176, Convert.i32_to_str(count))
            app.end()

    app.destroy()
```

## Layout

```text
mod.dlt
module.meta
theme.dlt
app.dlt
widgets.dlt
```

## Direction

Eqoi should grow as the app framework layer:

- buttons
- labels
- panels
- text input
- layout helpers
- themes
- app lifecycle
- eventually routing/navigation or document-style app helpers

Low-level drawing remains in `ui`. Window creation and event waiting remain in `window`.

Current compiler note: consumers should import `window` and `ui` before `eqoi` so the compiler resolves native structs and color aliases correctly.
