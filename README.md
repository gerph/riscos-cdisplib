# CDispLib

CDispLib is a C implementation of the RISC OS `DispLib` text-display
library. It manages a scrollable display made from lines and coloured text
runs, and can provide a Wimp window, selections, clipboard transfer, menus,
text saving, and direct screen rendering.

The library is suitable for log windows, status displays, viewers, and any
application that needs simple formatted text without a full text-editor
component.

## Features

- Lines containing one or more coloured text runs.
- System-font and Font Manager rendering.
- Optional Wimp window management, resize handling, menus, selection, and
  clipboard ownership.
- Saving all text or the current selection as a Text file.
- A renderer that accepts a plain screen rectangle and has no Wimp interface
  dependency.

## Building

Build the normal CDispLib command and its test support with:

```sh
riscos-amu
```

Run the regression tests with:

```sh
riscos-build-run aif32 --command 'aif32.CDispLib --selftest'
```

The standalone standard-rendering smoke test does not link RISC_OSLib or
start a Wimp task. It draws known coloured regions directly to the screen:

```sh
riscos-amu STANDARD_RENDER=1
riscos-build-run aif32 --command aif32.StandardRender
```

## Basic use

Include `cdisplib.h`, initialise the library once after initialising your
Wimp task, and shut it down before closing that task.

```c
CDispDisplay *display;

cdisplib_init();
cdisplib_set_task_handle(task_handle);

display = cdisplib_display_new("Build output");
cdisplib_display_colours(display, 7, -1);
cdisplib_display_add(display, "Compiling");
cdisplib_display_add_to(display, " completed", 2, 7);
cdisplib_display_open(display);
```

Dispose of a single display when it is no longer needed:

```c
cdisplib_display_kill(&display);
```

Before the application exits, call:

```c
cdisplib_shutdown();
```

`cdisplib_shutdown()` releases clipboard ownership and completes any pending
clipboard transfer before freeing its data.

## Wimp event integration

The caller owns the Wimp poll loop. Route relevant events to CDispLib:

- On redraw, find the display with `cdisplib_display_find()` and call
  `cdisplib_display_redraw()`.
- On open-window requests, call `cdisplib_display_reopen()` for display
  windows so that a bottom-aligned display stays bottom aligned.
- On mouse clicks, call `cdisplib_display_drag_check()` to enable selection
  dragging, Adjust extension, and double-click line selection.
- On null polls during a selection drag, convert the pointer position with
  `cdisplib_display_rel_xy()` and call `cdisplib_display_selection_modify()`.
- On drag completion, call `cdisplib_display_selection_end()`.
- Pass clipboard messages to `cdisplib_display_clipboard_message()` before
  handling other application messages.
- Call `cdisplib_display_mode_change()` after palette or mode changes.

The `c/wimp_smoke` source is a compact working example of the redraw path.

## Lines, runs, and colours

`cdisplib_display_add()` appends a new line. `cdisplib_display_add_to()`
appends another run to the final line; `cdisplib_display_add_to_at()` does the
same for a specified line handle.

The colour arguments to `cdisplib_display_add_to()` are background followed
by foreground. Use `-1` to retain the preceding colour and `-2` as a mottled
background marker. Standard Wimp colours (`0` to `15`), ColourTrans colours,
and private palette references are supported. Add private entries with
`cdisplib_display_set_palette()`.

## Direct rendering without a Wimp window

`cdisplib_render.h` provides `CDispRenderBox` and
`cdisplib_display_render()`. The caller supplies the screen position of the
work-area origin and the clipped rectangle to redraw:

```c
CDispRenderBox clip;

clip.x0 = 64;
clip.y0 = 700;
clip.x1 = 1024;
clip.y1 = 900;
cdisplib_display_render(display, 64, 900, &clip);
```

The Wimp-specific redraw iterator is kept in `c/display_wimp`; it converts
Wimp redraw rectangles into `CDispRenderBox` values and calls this renderer.

## Selection, menus, and saving

Use `cdisplib_display_select_all()` or
`cdisplib_display_select_range()` for programmatic selection. The selected
text can be copied with `cdisplib_display_copy_selection()` or saved with
`cdisplib_display_save_selection()`.

`cdisplib_display_edit_menu()` and `cdisplib_display_menu()` return shared
menu structures. Call their corresponding `*_setup()` functions before
opening a menu and their `*_choice()` functions when processing a menu
selection.

Use `cdisplib_display_save_text()` to write the complete display as a RISC OS
Text file.
