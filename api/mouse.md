# NEATO `mouse` API Specification

Extension: `ext.mouse`

Version: 1

Requires: `core`

---

This extension defines the mouse pointer events of a graphics screen as [events](core/event.md) of `core`. On real
NEET Computers hardware, a graphics screen fires these events to the computer it belongs to while the user works the
cursor over it; `ext.mouse` gives those events their names in NEATO's event namespace and states exactly when each
one fires, so a program can rely on them on every operating system that reports the extension. The names and the
values are the ones NEET Computers uses, unchanged, in the same way [ext.screen](screen.md) keeps the `screen` API
itself unchanged. NEATO does not expose the raw NEET Computers `event` API, so without this extension these events
have no defined names and an operating system must not deliver them.

Which program receives mouse events, and which screen they come from when a machine has more than one, is up to the
operating system, exactly as [core/event.md](core/event.md) says for input events. Usually they go to the program
that has focus. A program the events are not delivered to receives none.

---

### Standard events

| Name          | Values                                                   | Description                                       |
| ------------- | -------------------------------------------------------- | ------------------------------------------------- |
| mouseMoved    | x (int), y (int)                                         | The cursor moved to a new position on the screen. |
| mouseClicked  | x (int), y (int), key (int)                              | A mouse button was pressed on the screen.         |
| mouseDragged  | x (int), y (int), key (int)                              | The cursor moved while a mouse button is held.    |
| mouseReleased | x (int), y (int), key (int)                              | A mouse button was released.                      |
| mouseScrolled | x (int), y (int), horizontal (number), vertical (number) | The scroll wheel was moved.                       |

---

### Coordinates

In `mouseMoved`, `mouseClicked`, `mouseDragged`, `mouseReleased` and `mouseScrolled`, `x` and `y` are graphics
coordinates, in the same space the `screen` API of [ext.screen](screen.md) draws in when both extensions are
reported. One unit is one pixel, `(0, 0)` is the top left, and on a screen of size `(width, height)` from
`screen.getSize` the coordinates run from `0` to `width - 1` and from `0` to `height - 1`. A coordinate is never
negative. An operating system that redirects drawing, as [ext.screen](screen.md) allows it to, reports the events in
the redirected space too, so the positions a program is given are the positions it draws at.

Every mouse event is delivered only while the cursor is on the screen. While it is not, no mouse event is delivered,
and the next one reports wherever the cursor has come to.

---

### Buttons

`key` is the mouse button number. `0` is the left button, `1` the right one and `2` the middle one. Buttons past the
middle, where the hardware has them, take the numbers from there. Each button is tracked on its own. Holding two
buttons delivers drags and releases for each, every event carrying its own `key`.

---

### mouseMoved

Delivered when the cursor arrives at a graphics position on the screen that it did not have before. It is not
delivered again while the cursor stays within one position, however long, and the hardware samples the cursor
between frames, so movement that happens within one frame may be delivered as the position it ended at. A position
the cursor returns to later is delivered again like any other new position.

### mouseClicked

Delivered when a mouse button is pressed while the cursor is on the screen, at the position of the press.

A click is also delivered without a press when a drag begins on the screen for a button the screen did not see go
down. When the cursor starts to move on the screen while `key` is already held and no press was delivered for it,
the first event of that drag is a `mouseClicked` at the position the movement begins at, and the `mouseDragged`
events follow it. Every drag this extension reports therefore begins with a click. From a program's point of view
the two kinds of click are the same event. The button came down at that position.

### mouseDragged

Delivered when the cursor moves to a new graphics position while the button `key` is held down, after the click that
began the drag. Like `mouseMoved` it is not delivered again while the position is unchanged, and it is measured per
button: where one button's drag has already been delivered does not hold back another's.

### mouseReleased

Delivered when the button `key` is released while the cursor is on the screen, at the position of the release.

When the cursor leaves the screen while a button is held, a `mouseReleased` for that button is delivered at the
moment it leaves, reporting the last position the button was known at, and releasing the button off the screen after
that delivers nothing. A button that leaves the screen held is therefore reported as released exactly once.

### mouseScrolled

Delivered when the scroll wheel is moved while the cursor is on the screen, at the position of the cursor. The
`horizontal` and `vertical` values are how far the wheel was moved. They are numbers and may be fractional. An
ordinary wheel notch is `1.0`, and a touchpad or a high resolution wheel may report smaller steps in between.
`vertical` is positive when the wheel is pushed away from the user and negative when it is pulled towards them;
`horizontal` is positive to the right.

---

### Example usage

```lua
-- let the user paint with the left button, onto a screen drawn through ext.screen
while true do
  local name, x, y, key = event.pull()
  if name == "terminate" then return end
  if key == 0 and (name == "mouseClicked" or name == "mouseDragged") then
    screen.writePixel(x, y, 255, 0, 0)
  end
end
```

```lua
-- what is held where
local held = {}
while true do
  local name, x, y, key = event.pull()
  if name == "mouseClicked" then held[key] = { x = x, y = y }
  elseif name == "mouseReleased" then held[key] = nil
  elseif name == "terminate" then return end
end
```
