# Eqoi

Eqoi is the native application framework for Dolet. It owns the application
window and event loop, translates native input into `ui` snapshots, manages the
software surface, and provides themes, layouts, widgets, clipping, and managed
widget state.

On Linux the public API is display-server neutral. The shared `window` adapter
uses native Wayland/xdg-shell when available and falls back to X11/XWayland;
Eqoi does not contain display-server-specific rendering or input code.

Current version: **0.8.0**

```text
Eqoi application
  -> Eqoi lifecycle, layout, widgets, theme, state
  -> ui commands, input snapshots, software surface
  -> window/input native adapters
  -> operating system
```

## Quick start

```dolet
import eqoi

app: EqoiApp = EqoiApp.create("Hello Eqoi", 800, 600)
if app.is_valid() == 1:
    count: i64 = app.alloc_int(0)

    while app.running():
        if app.begin() == 1:
            layout: EqoiLayout = EqoiLayout.column(24, 24, app.width - 48, app.height - 48, 16, 12)
            title_rect: UIRect = layout.next(0, 32)
            button_rect: UIRect = layout.next(180, 44)

            app.title(title_rect.left, title_rect.top, "Eqoi")
            if app.button(button_rect.left, button_rect.top, 180, 44, "Click") == 1:
                app.set_int(count, app.get_int(count) + 1)

            app.end()

    app.destroy()
```

## Layout

`EqoiLayout.column(...)` and `EqoiLayout.row(...)` produce bounded `UIRect`
values. A non-positive cross-axis size fills the inner area; a non-positive
main-axis size consumes the remaining area. Layout only computes geometry, so it
can be tested without opening a window.

## Widgets

- text, labels, titles, panels, separators, and raw rectangles
- primary and secondary buttons
- checkboxes, toggles, sliders, progress bars, and text input
- scrollbars and clipped scrollable regions
- tabs, dropdowns, modals, tooltips, drag sources, and drop targets

All widget text measurement comes from the active `ui` surface FontFace. Eqoi
does not hard-code glyph dimensions, so centering, carets, tabs, dropdowns, and
tooltips remain correct when the font face or pixel size changes. Coverage-font
text and anti-aliased primitives are rasterized by `ui`; Eqoi stays focused on
native application behavior and component composition.

An application may create a face with the independent `fonts` package and pass
it to `app.use_font(face)`. Eqoi borrows the atlas, so the application keeps the
face alive and destroys it after `app.destroy()`; switching faces does not copy
glyph pixels or add per-frame rasterization.

```dolet
app_font: FontFace = font_face_load_ttf("assets/fonts/Inter-Regular.ttf", 16)
if app_font.is_valid() == 1:
    app.use_font(app_font)

# ...event loop...
app.destroy()
app_font.destroy()
```

The showcase uses the official Inter 4.1 static TTF and includes its OFL
license under `examples/assets/fonts`.

`panel_begin/end` and scrollable regions use the dynamic nested clip stack from
`ui`. Draw commands and command text grow safely instead of being silently
dropped at a fixed capacity.

## State and themes

`alloc_bool`, `alloc_int`, and `alloc_buf` create application-owned state that is
released by `app.destroy()`. The pointer registry grows as required, so an
application is not limited to a fixed widget count.

`EqoiTheme` centralizes application colors, spacing, padding, borders, and
corner radii. `EqoiTheme.dark()` is the built-in default and
`EqoiTheme.light()` provides the matching light palette.

## Testing without a window

`EqoiApp.headless(width, height)` is the offscreen twin of `EqoiApp.create`. It
owns the same surface and command list and runs the same frame loop, but opens
no window, pumps no events, and presents nothing. Input arrives from the
`inject_*` helpers instead of the operating system, so widget behaviour can be
driven and asserted on with no display server.

```dolet
app: EqoiApp = EqoiApp.headless(400, 300)

app.inject_mouse_move(60, 60)
app.begin_offscreen()
app.button(40, 40, 120, 40, "OK")     # hovered, not pressed -> 0
app.end()

app.inject_left_down()
app.begin_offscreen()
clicked: i32 = app.button(40, 40, 120, 40, "OK")   # press edge -> 1
app.end()

app.destroy()
```

Injected state follows the native path exactly. Buttons and modifiers are
**held** until cleared, which is what makes press and release edges behave the
way they do under a real mouse. Wheel, typed text, backspace, and the
navigation keys are **one-shot** and drain after the frame that reads them,
mirroring the native `window_consume_*` calls. A click is therefore two frames,
not one, because that is what it is in an immediate-mode loop.

`pixel_at(x, y)` reads back a rendered pixel once `end()` has flushed the frame,
and `resize_offscreen(width, height)` moves the canvas so responsive layout can
be exercised without a window manager. `tests/widgets.dlt` uses all of it.

## Boundaries

- `ui` remains pure and platform-neutral.
- Eqoi owns native application lifecycle and presentation.
- `window` and `input` contain operating-system adapters.
- GPU presentation can be added behind Eqoi without rewriting layout or widgets.

Windows is verified natively. Linux uses the same Eqoi/UI layers through the
display-server-neutral `window` API. Native Wayland/xdg-shell is preferred;
X11/XWayland is the runtime fallback. Both paths use cached software surfaces,
native keyboard/pointer/text events, and are covered by forced-backend WSLg
integration smoke tests in addition to cross-compilation.

## Component showcase

`examples/showcase.dlt` is a complete native test application. It exercises
responsive panels, tabs, both built-in themes, buttons, checkboxes, toggles,
sliders, progress, text input, dropdowns, scrolling, drawing primitives,
tooltips, drag and drop, modal layering, and application-managed state.

```powershell
doletc examples/showcase.dlt -o examples/eqoi-showcase.exe --target windows/x86_64 --no-console
```

```bash
doletc examples/showcase.dlt -o examples/eqoi-showcase --target linux/x86_64
```
