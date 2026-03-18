# Screen Dimmer with Python

## Method 1: Blackout Overlay

Creates a full black overlay on every connected monitor. Windows underneath are not clickable. Press `Esc` or `Enter` to exit.
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

- Covers all monitors
- Blocks mouse clicks
- Exit with `Esc` or `Enter`

---

## Method 2: Adjust Actual Brightness

Uses `screen_brightness_control` to directly change the monitor backlight. Does not block interaction.
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

## When to Use Each

| Method | Use Case |
|---|---|
| Overlay blackout | Instant lockout, blocks all clicks and distractions |
| Brightness control | Subtle dimming for eye strain or nighttime use |