# Screen Dimmer: Blackout Overlay and Brightness Control

## Overview

Two Python approaches to darken connected monitors. The first creates a full interactive blackout overlay across all displays. The second adjusts the physical monitor backlight directly.

---

## Prerequisites

### Blackout overlay
```bash
pip install pyglet screeninfo
```

### Brightness control
```bash
pip install screen_brightness_control
```

---

## Method 1: Blackout Overlay

Creates a fullscreen black window on every connected monitor. Underlying windows are not clickable. Press `Esc` or `Enter` to exit.
```python
import pyglet
from screeninfo import get_monitors

windows = []

def close_all_windows():
    for window in windows:
        window.close()

for monitor in get_monitors():
    screen_index = get_monitors().index(monitor)
    screen = pyglet.canvas.Display().get_screens()[screen_index]
    window = pyglet.window.Window(fullscreen=True, screen=screen)

    pyglet.gl.glClearColor(0, 0, 0, 1)

    @window.event
    def on_key_press(symbol, modifiers):
        if symbol == pyglet.window.key.ESCAPE or symbol == pyglet.window.key.ENTER:
            close_all_windows()

    windows.append(window)

pyglet.app.run()
```

**Behaviour:**

- Covers all connected monitors
- Blocks mouse interaction with underlying windows
- Exit with `Esc` or `Enter`

---

## Method 2: Brightness Control

Adjusts monitor backlight level directly. Does not block interaction.
```python
import screen_brightness_control as sbc

# Set brightness to 25%
sbc.set_brightness(25)
```

| Value | Effect |
|---|---|
| `0` | Minimum brightness |
| `50` | Half brightness |
| `100` | Full brightness |

---

## Comparison

| | Blackout Overlay | Brightness Control |
|---|---|---|
| Blocks interaction | Yes | No |
| Affects all monitors | Yes | Depends on driver support |
| Requires display server | Yes | No |
| Use case | Instant lockout, distraction blocking | Eye strain reduction, ambient dimming |

---

## Known Limitations

- `screen_brightness_control` requires driver support. Behaviour varies by monitor and OS.
- The blackout overlay requires a running display server. Not usable on headless systems.
- `pyglet` may require additional system packages on some Linux distributions.