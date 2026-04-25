# Eqoi Framework

Eqoi is a high-level native application framework for Dolet. It owns the
window + event pump and layers widgets on top of `ui`.

It is built on top of:

- `window` — native windows and event pump
- `ui` — pure-software drawing (rects, text, mouse state, draw commands)
- `input` — mouse/keyboard/wheel accumulators

Eqoi is aimed at app builders who want a cleaner API than raw drawing
calls while keeping the native stack small and direct.

Current version: **0.4.0**

## Quick start

```dolet
import eqoi

app: EqoiApp = EqoiApp.create("Hello Eqoi", 800, 600)
if app.is_valid() == 0:
    print("failed to create app")
else:
    count: i64 = app.alloc_int(0)

    while app.running():
        if app.begin() == 1:
            app.title(32, 32, "Eqoi")
            app.muted_label(32, 72, "Native apps with a small framework layer.")

            if app.button(32, 112, 180, 44, "Click") == 1:
                app.set_int(count, app.get_int(count) + 1)

            app.label(32, 176, Convert.i32_to_str(app.get_int(count)))
            app.end()

    app.destroy()
```

## Widget catalogue

### Text / decoration
- `title(x, y, value)`
- `label(x, y, value)` / `muted_label(x, y, value)` / `text(x, y, value, color)`
- `rect(x, y, w, h, color)` / `outline(x, y, w, h, color)` / `separator(x, y, w)`
- `panel(x, y, w, h)` — single surface panel

### Inputs
- `button(x, y, w, h, label) -> i32` / `secondary_button(...)`
- `checkbox(x, y, label, value_ptr) -> i32`
- `toggle(x, y, label, value_ptr) -> i32`
- `slider(x, y, w, min, max, value_ptr) -> i32`
- `progress_bar(x, y, w, h, percent)`
- `text_input(x, y, w, h, buf, max_len) -> i32`
- `tooltip(widget_id, text)`

### Responsive containers
- `panel_begin(x, y, w, h)` / `panel_end()` — nested clip stack (16 levels).
  Anything drawn between the two calls is auto-clipped to the panel.
- `modal_begin(w, h, title) -> i32` / `modal_end()` — auto-centres on the
  current canvas, so it re-centres on window resize.

### Advanced widgets
- `scrollbar_vertical(x, y, h, view_h, content_h, scroll_ptr) -> i32`
- `scrollable_begin(x, y, w, h, scroll_ptr, content_h)` / `scrollable_end()`
- `tab_bar_begin(x, y, w, h, active_ptr)` / `tab_bar_item(label, w) -> i32` / `tab_bar_end() -> i32`
- `dropdown_begin(x, y, w, h, selected_ptr, label) -> i32` / `dropdown_item(label) -> i32` / `dropdown_end()`
- `drag_source(x, y, w, h, payload) -> i32` / `drop_target(x, y, w, h) -> i64`

## Managed strings

Prefer `a + b` or `a.concat(b)` when you need to build a string in a
label or any transient UI code. Inside a bracketed scope (the `while`
body, an `if` body, a function body that doesn't return `str`) the
compiler dispatches both forms to an arena-backed builtin, so the
intermediate result is freed automatically when the scope exits.

```dolet
# safe — freed on the next iteration pop
app.label(48, 308, "level: " + Convert.i32_to_str(app.get_int(volume)))
app.label(48, 320, "name: ".concat(name_str))
```

Never call `Str.concat()` from widget code. It is the static-method
form on the `Str` struct and it always heap-allocates — the caller
is responsible for `Memory.free`. Using it inside the frame loop
produces a per-frame leak that grows forever (see the `simple-app-eqoi`
and `FileManager` incidents).

## Managed state

No manual `Memory.malloc` / `Memory.free` pairs in user code. Allocate
slots on the app; eqoi frees them in `destroy()`.

```dolet
dark:    i64 = app.alloc_bool(0)
volume:  i64 = app.alloc_int(40)
name:    i64 = app.alloc_buf(60)

app.checkbox(48, 180, "Dark mode", dark)
app.slider(48, 280, 360, 0, 100, volume)
app.text_input(48, 414, 360, 30, name, 60)

if app.get_bool(dark) == 1:
    # apply dark theme
```

## Layout

```text
mod.dlt         # module declaration, exports, load statements
module.meta     # package metadata + deps + link settings
theme.dlt       # EqoiTheme struct + dark theme preset
app.dlt         # EqoiApp — window, frame loop, state, clip stack, theme
widgets.dlt     # every widget method on EqoiApp
```

## Dependencies

```ini
window = required
ui = required
input = required
```

## Link (Windows)

```ini
system_libs = user32, gdi32
```

Linux and macOS backends are planned once the matching `window`/`ui`
backends land.

## Direction

- More layout helpers (rows/columns, alignment) — currently the caller
  gives absolute coordinates; the clip stack already prevents overflow.
- Keyboard navigation / accessibility hooks.
- Animation timelines for transitions.
- Richer text (wrap, alignment, variable-width glyphs — needs fonts upgrade).

Low-level drawing stays in `ui`. Window creation and event waiting stay
in `window`. Eqoi only composes them.
