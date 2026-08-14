# Eqoi

Eqoi is the native application framework for Dolet. It owns the application
window and event loop, translates native input into `ui` snapshots, manages the
software surface, and provides themes, layouts, widgets, clipping, and managed
widget state.

Current version: **0.5.0**

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

## Boundaries

- `ui` remains pure and platform-neutral.
- Eqoi owns native application lifecycle and presentation.
- `window` and `input` contain operating-system adapters.
- GPU presentation can be added behind Eqoi without rewriting layout or widgets.

Windows is verified natively. Linux/X11 uses the same Eqoi/UI layers, a cached
`XImage` framebuffer path, X11 keyboard and pointer snapshots, and native text
events. Linux is verified through a native WSLg/X11 integration smoke test in
addition to cross-compilation; a desktop session needs `libX11` at runtime.
