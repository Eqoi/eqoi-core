# Eqoi

Eqoi is the native application framework for Dolet. It owns the application
window and event loop, translates native input into `ui` snapshots, manages the
software surface, and provides themes, layouts, widgets, clipping, and managed
widget state.

On Linux the public API is display-server neutral. The shared `window` adapter
uses native Wayland/xdg-shell when available and falls back to X11/XWayland;
Eqoi does not contain display-server-specific rendering or input code.

Current version: **0.21.0**

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

The model is single-pass, because an immediate-mode frame places a widget the
moment it is asked for and cannot look ahead at items that have not been
requested yet. The API is built inside that constraint rather than pretending
it is not there: cross-axis work needs no lookahead, so it is automatic;
flexible sizing needs the plan, so `flex` takes it; main-axis alignment needs
the content extent, so `align_main` takes it.

**Flexible sizing.** `flex(total_weight, item_count)` declares a run and
reserves the gaps between its items, so equal weights come out equal and the
run ends exactly on the far edge. `flex_even(n)` is the equal-share shorthand.

```dolet
bar: EqoiLayout = EqoiLayout.row(0, 0, 300, 100, 10, 10)
bar.flex(3, 2)
narrow: UIRect = bar.next_flex(1)      # one third
wide: UIRect = bar.next_flex(2)        # two thirds
```

**Alignment.** `align(mode)` places items that give an explicit cross size:
`EQOI_ALIGN_START` (the default, and the original behaviour), `CENTER`, `END`,
or `STRETCH` to ignore the requested size and fill. `align_main(mode, extent)`
shifts the starting cursor so a known amount of content sits centred or
end-aligned along the main axis.

**Constraints.** `margin(n)` reserves space around every item by growing its
slot. `limits(min, max)` clamps each item's main-axis size, with a maximum of
0 meaning unbounded.

**Wrapping.** `wrap(1)` moves to a new line (row) or column (column) when an
item does not fit. The item that did not fit starts the new line at its full
size rather than being squeezed into what was left.

**Grids.** `EqoiLayout.grid(x, y, w, h, padding, gap, columns)` flows cells
across and then down. `next_cell(height)` takes one cell and
`next_cell_span(span, height)` takes several; a span that would overflow the
row starts a new one.

**Nesting.** `column_in(rect, padding, gap)`, `row_in`, and `grid_in` build a
child layout inside a rectangle a parent produced.

```dolet
slot: UIRect = page.next(0, 100)
panel: EqoiLayout = EqoiLayout.column_in(slot, 5, 5)
```

**Measuring.** `content_main()` reports the main-axis extent consumed so far
and `used_rect()` the rectangle it occupies, so a container can be sized to
what was just laid out inside it. `skip(n)` advances without producing a
rectangle.

## Widgets

- text, labels, titles, panels, separators, and raw rectangles
- primary and secondary buttons
- checkboxes, toggles, sliders, progress bars, and text input
- scrollbars and clipped scrollable regions
- tabs, dropdowns, modals, tooltips, drag sources, and drop targets
- a virtualized data table with columns, selection, and sorting hooks
- radio groups, number spinners, tree rows, context menus, and toasts

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

## State

A widget that outlives a frame keeps its value somewhere the application
owns. That worked through a bare `i64`: every checkbox, slider and text field
took the same anonymous number, and nothing stopped a text buffer being
handed to a slider or a length being passed as a value.

Handles are the same pointers with a type on them.

```dolet
enabled: EqoiBool = app.new_bool(1)
volume:  EqoiInt  = app.new_int(64)
name:    EqoiText = app.new_text(64)

app.checkbox(x, y, "Enable", enabled)
app.slider(x, y, 200, 0, 100, volume)
app.text_input(x, y, 240, 32, name)      # no length to get wrong
```

A text handle carries its own capacity, which removes the `max_len` argument
entirely — a length passed separately is a length that can be passed wrongly.
`get`, `set`, `add`, `toggle`, `str`, `length` and `clear` read and write
without going through the app.

An unset handle answers rather than reaching through a null pointer, so a
struct that was never initialised reads 0 and ignores writes instead of
faulting.

**The untyped form still works.** Every widget takes both, so nothing written
against the old shape has to change, and the two can be mixed in one frame.

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

## Arabic and other scripts

Write the string you mean:

```dolet
app.label(x, y, "السلام عليكم")
app.button(x, y, w, h, "اضغط هنا")
```

`ui` chooses the cursive forms and the drawing order on the way to the pixels,
so Arabic goes through labels, buttons, table cells and text fields the same
way Latin does. Mixed content orders correctly too: numbers inside an Arabic
line still read left to right.

Eqoi's built-in face covers ASCII, so drawing Arabic needs a face that has it:

```dolet
face: FontFace = font_face_load_ttf("assets/fonts/arabic.ttf", 22)
app.use_font(face)
```

`examples/arabic.dlt` is a working application that does exactly this and
prepares nothing itself.

## Display scale

An application writes one set of coordinates. A denser display needs
everything larger. Reconciling those is the whole of display scaling, and
there are exactly two honest places to do it: where a coordinate becomes a
pixel, and where a pixel becomes a coordinate.

Eqoi does it at both and nowhere else. Layout, widgets, hit testing and the
whole application work in one set of units and never ask what the display is
doing — **nothing above `scale.dlt` contains a scale factor.**

```dolet
app.set_scale_percent(200)          # or let it come from the window
face: FontFace = font_face_load_ttf(path, app.font_pixels(16))
```

- `app.width`/`app.height` and pointer positions are **logical**; the surface
  stays physical, at the resolution the window actually has.
- Text metrics are reported logically too, because the face is built at the
  physical size. Without that every centred label would drift by half the
  scale factor.
- A one-pixel border never rounds away to nothing at a fractional scale.
- The built-in face is rebuilt at the new size. An application that supplied
  its own face sizes it with `font_pixels`, and Eqoi stops rebuilding.

**Where it comes from.** Windows reports the display's dots per inch and Eqoi
adopts it before the first frame. The Linux backend does not yet listen for
the Wayland output scale or read Xft.dpi, so it reports one to one rather
than guessing — an application there sets the scale outright and everything
else behaves identically.

## Controls

Beyond the basics: `radio` for one of several, `spinner` for a number you
nudge, `tree_row` for a hierarchy, `menu_*` for a context menu, and `toast`
for a message that leaves on its own.

They keep the shape the rest of Eqoi uses — state in a value the caller owns,
a return that says whether it changed, and mirroring when the direction turns
over — with two decisions worth knowing:

**A tree row is two targets.** It returns 1 when the twisty was hit and 2 when
the row itself was chosen, because opening a branch and selecting it are
different intentions and a row that conflates them is annoying to use. The
application owns the tree and walks it; Eqoi draws a row at a depth.

**A toast is not the application's to remember.** It is posted once and Eqoi
expires it on the frame clock. A toast on its way out has no OS event behind
it, so it also keeps the frame loop awake until it is gone, the same way an
animation does.

```dolet
app.toast("Saved", 2000)
...
app.toasts_draw()          # once, late in the frame
```

## Reading direction

A right-to-left interface is not right-aligned text. The whole surface turns
over, and one call does it:

```dolet
app.use_rtl()
```

Rows fill from the right, a checkbox puts its box on the right with the label
running away from it, a slider fills from the right, tabs run the other way,
table columns start on the right, and the scrollbar moves to the left edge.

Two things carry it. **Layout**: every rectangle is mirrored inside its
layout's bounds, so rows, columns and grids all flow the other way from one
transform rather than a direction test in every calculation. **Widgets**: a
widget with a side asks `mirror_x` for it; a symmetric one — a button, a
panel, a progress bar — needs nothing, which is most of them.

Build layouts through the app so nested ones stay consistent:

```dolet
page: EqoiLayout = app.column(x, y, w, h, padding, gap)
row: EqoiLayout = app.row_in(slot, 0, 12)
```

`EqoiLayout.column(...)` still builds a left-to-right layout, so the direction
travels with the application rather than leaking globally.

For text that fills a row, the reading edge moves with the direction, so pass
the rectangle instead of a point:

```dolet
app.label_in(rect, "السلام عليكم")
app.title_in(rect, "العنوان")
```

Text editing works in right-to-left too. A caret lives at a position in the
text as stored and has to be drawn somewhere in the text as shown, and after
shaping and reordering nothing arithmetic connects the two. Eqoi carries the
correspondence: every drawn glyph remembers the character it came from and the
direction it runs in.

The caret sits on the **leading edge** of the character it precedes — the left
side of a left-to-right glyph, the right side of a right-to-left one. That one
rule is what makes a caret behave in mixed text: it walks leftwards through
Arabic and rightwards through a Latin run in the same line, following the text
rather than the pixels.

Clicking maps back the same way, and a selection is painted as the visual runs
it actually occupies: a Latin word inside an Arabic line is one run and the
Arabic either side of it is another, which a single rectangle between two ends
cannot express.

## Widget identity

State that outlives a frame — focus, drag ownership, hover age, whether a
dropdown is open — is keyed by an identity hash. With nothing pushed, that hash
is derived from the widget's position, which is all an immediate-mode loop can
infer on its own.

Position stops being a usable identity the moment a widget moves. A slider
dragged while the window resizes, a focused text input inside a scrolling
panel, an open dropdown in a reflowing layout: each becomes a different widget
mid-gesture and drops the state the gesture depended on.

`push_id` gives the caller a say. While an identity is pushed, the hash is the
scope plus the widget's place within it, and position drops out — so the widget
keeps its identity wherever layout puts it, and two widgets in one scope stay
two widgets. Counting restarts on every push, so adding a control to one panel
cannot renumber another panel's.

```dolet
app.push_id_str("volume")
app.slider(x, y - scroll, width, 0, 100, volume)   # survives scrolling
app.pop_id()
```

Pushes nest, so a reusable component can seed itself once per item and every
widget inside it gets a distinct scope:

```dolet
i: i32 = 0
while i < row_count:
    app.push_id_int(i)
    app.checkbox(x, y, "Selected", row_flag(i))
    app.pop_id()
    i = i + 1
```

`push_id(seed: i64)`, `push_id_int`, and `push_id_str` all fold into the scope
already in force. `pop_id()` restores the parent and is harmless when the stack
is empty; `clear_id_stack()` drops every scope, which is worth calling between
top-level sections if a frame can leave a nested block early.

Callers that push nothing behave exactly as they did before `push_id` existed.

## Keyboard

Tab order is the order widgets are drawn in. There is no retained tree to walk,
so it is rebuilt every frame: each focusable widget enters itself into the
order, and the traversal settles in one pass.

- **Tab** / **Shift+Tab** move focus forward and backward, wrapping at both
  ends. With nothing focused, Tab enters at the first widget.
- **Enter** fires the focused widget: a button returns 1, a checkbox or toggle
  flips, a tab or dropdown opens.
- **Escape** closes an open dropdown.
- **Left/Right** move a focused slider by a twentieth of its range, so a slider
  is usable without a pointer at any range size.
- A focus ring is drawn outside the focused widget, because keyboard focus
  nobody can see is not keyboard support.

If the focused widget stops being drawn — a tab switched, a panel collapsed —
focus falls to the first focusable rather than vanishing.

Space is deliberately not an activation key yet: it also arrives as a typed
character, and until text input owns a caret there is no unambiguous way to
tell a press from a typed space. Delete waits on the same work.

## Animation

Widget states move rather than snap: button and checkbox fills fade, the
toggle knob slides, the tab underline grows from its centre.

An animated value is keyed by widget identity, so a widget declares what it
wants to be and the value walks there over time.

```dolet
t: i32 = app.anim(id, hovered * 100, 700)       # 0..100 at 700 per second
fill: UIColor = app.mix_colors(base, hot, t)
```

The first time an identity is seen its value starts **at** the target, so
nothing fades in from nowhere on the frame it appears. Rates are per second
and `app.delta_us()` is the frame duration, so speed does not change with
frame rate.

The clock is the platform layer's, available on every target with no import,
so animation costs Eqoi no new dependency.

**The part that makes it work.** The frame loop blocks on `Window.wait()`,
which returns only when the operating system has an event — and an animation
has none. So the loop asks whether anything is still moving and blocks with a
timeout while it is, then goes fully idle again once everything settles. That
keeps the property worth having: an Eqoi application costs nothing while it
sits on screen.

Offscreen, timing comes from `inject_delta_us` instead of a clock, so
animation is exactly reproducible in tests.

## Table

`table_*` is a virtualized data table. The application never hands over its
data: it is told which rows are on screen and fills only those, so a hundred
thousand rows cost the same frame as eight.

```dolet
app.table_begin(x, y, w, h, 24, row_count, scroll, selected)
if app.table_column("Name", 220) == 1:
    sort_by_name()
app.table_column("Size", 90)
app.table_headers_end()
i: i32 = app.table_first_visible()
while i <= app.table_last_visible():
    if app.table_row(i) == 1:
        app.table_cell(0, name_of(i))
        app.table_cell(1, size_of(i))
    i = i + 1
changed: i32 = app.table_end()
```

Selection is a single row index in a caller-owned `i32`, `-1` for none, the
same shape every other stateful widget here takes. Clicking a row selects it;
Up, Down, Home and End move the selection and scroll the least amount that
brings it into view. Sorting is the application's job — `table_column` returns
1 on the frame its header is clicked and what that means is for the caller to
decide.

The keyboard and the wheel are both applied before the visible window is
worked out. That matters more than it sounds: the event loop only wakes on
input, so a scroll applied after the window was computed would not appear
until some later, unrelated event.

## Text input

`text_input` owns a caret and a selection rather than only appending at the
end. Only one input can hold keyboard focus, so one set of caret state serves
them all; the caller's contract is unchanged — still a byte buffer and a
maximum length.

- **Left/Right** move the caret, **Home/End** jump to either end, and
  **Shift** with any of them extends the selection.
- **Backspace** and **Delete** remove the selection when there is one, and
  otherwise the character before or after the caret.
- Typing inserts at the caret, replacing the selection.
- **Ctrl+A** selects everything.
- Clicking places the caret at the pointer, and dragging extends a selection.
- The selection is drawn behind the text, and the caret sits at its measured
  pixel position rather than at a fixed advance, so it stays correct under a
  proportional face.

Positions are byte offsets, which is correct for the ASCII the input layer
delivers today. When the text package lands they become grapheme offsets, and
no caller has to change, because none of them index the buffer themselves.

Copy, cut and paste are not wired: they need an OS clipboard, which belongs in
the `window` package. Nothing here pretends they work.

## Scrolling

`scrollable_begin`/`scrollable_end` take the wheel anywhere over the region,
not only over the scrollbar column. A standalone `scrollbar_vertical` still
takes the wheel over its own track; `scrollbar_vertical_area` accepts the
wheel-sensitive rectangle explicitly.

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
