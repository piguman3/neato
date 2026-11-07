# NEATO `headsup` API Specification

Extension: `ext.headsup`

Version: 1

Requires: `core`

---

An environment that reports `ext.headsup` must provide the global `headsup` table with the same functions, parameters
and return types as the NEET Computers API. It does not necessarily have to reach the physical heads-up display: an
operating system may point a program's `headsup` at a virtual display of its own, as long as the functions behave the
same from the program's point of view.

`headsup` keeps the NEET Computers API unchanged, including how it reports its own failures, so it does not use the
codes in [errors.md](../common/errors.md): a mistake raises a plain Lua error, as
[Drawing](#drawing) describes.

| Name              | Description                                                                                                                                                       | Arguments                              | Returns                   |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- | ------------------------- |
| headsup.drawPixel | Turns on the single pixel at column `x`, row `y`.                                                                                                                 | x (int), y (int)                       | nil                       |
| headsup.getSize   | Returns the width and height of the display in pixels.                                                                                                            | none                                   | width (int), height (int) |
| headsup.drawLine  | Turns on every pixel of the straight line from (`x1`, `y1`) to (`x2`, `y2`), endpoints included.                                                                   | x1 (int), y1 (int), x2 (int), y2 (int) | nil                       |
| headsup.drawRec   | Turns on every pixel of the filled rectangle whose opposite corners are (`x1`, `y1`) and (`x2`, `y2`).                                                             | x1 (int), y1 (int), x2 (int), y2 (int) | nil                       |
| headsup.draw      | Publishes the buffer to the display, replacing everything the display shows.                                                                                      | none                                   | nil                       |
| headsup.clear     | Resets the buffer: every pixel in it becomes off. What the display shows does not change until the next `headsup.draw`.                                            | none                                   | nil                       |

```c
headsup
  drawPixel
  drawLine
  drawRec
  draw
  clear
  getSize
```

---

### Drawing

- Coordinates. Every coordinate is an integer counted from 1, and bounds are inclusive: `x` runs from 1 to the width
  and `y` from 1 to the height returned by `headsup.getSize`, with 1, 1 at the top left corner, as in the
  [terminal](core/term.md).

- The buffer and the display. Every drawing call acts on an off-screen buffer that belongs to this program, not on the
  display itself. `headsup.draw` publishes: the display is replaced, in one atomic step, with the buffer as it stands
  at that moment. Until `headsup.draw` is called, no drawing call affects what the display shows, and after it the
  display does not change again until the next `headsup.draw`.

- Range errors. `headsup.drawPixel` range-checks `x` and `y`; `headsup.drawLine` range-checks both endpoints;
  `headsup.drawRec` range-checks both corners. If any coordinate falls outside 1 to the width or 1 to the height, the
  call raises a Lua error with the message `values not in allowed range`. The check happens before anything is drawn,
  so a call that raises leaves the buffer unchanged. Passing arguments of the wrong type raises an error as well.

- Lines. `headsup.drawLine` turns on exactly the pixels that the classical integer Bresenham line algorithm, with its
  error term initialized to `dx - dy`, visits between the two endpoints, endpoints included. Two equal endpoints turn
  on a single pixel.

- Rectangles. `headsup.drawRec` fills the rectangle between the two corners, edges included, so a pair of corners
  naming a single row, column or pixel draws that row, column or pixel. The order of the corners does not matter:
  either one may be given first.

- Only on. Drawing never turns a pixel off, no function reads a pixel back, and `headsup.clear` is the only call that
  turns pixels off, turning off every pixel of the buffer at once. It acts on the buffer only: the display keeps
  showing the last published picture until the next `headsup.draw`.

- Size. `headsup.getSize` returns the same width and height on every call: the size of the display the operating
  system has pointed this program's `headsup` at, fixed while the program runs.

---

### Sharing the display

The display is one small surface and many programs may want it at once, which is the question this extension has to
settle. The settlement is structural, not a protocol: the API already gives every program its own buffer and reserves
the display for `headsup.draw`, so an operating system can multiplex programs without the function set changing under
any of them.

An operating system may give each program a private buffer and its own notion of the display, and may itself decide
whose published buffer the display shows: the last program to call `headsup.draw` may simply win, the display may
follow the program that has focus, or sole ownership of the display may be granted to one program at a time. Whatever
it chooses, the drawing calls must behave the same from the program's point of view. NEATO defines no interleaving
guarantees between concurrent programs: another program cannot touch a program's buffer, not even in the middle of a
clear, redraw and publish, but which published picture is on the display at any moment, and for how long, is up to the
operating system. A program can hold a stable picture across other programs' draws only where the operating system
promises it, such as under sole ownership.

This is the same spirit as [ext.screen](screen.md), where an operating system may direct drawing into a window or onto
a layer of its own as long as the functions behave the same from the program's point of view: a multiplexed heads-up
display is such a layer, one bit deep.

Granting the display to one program at a time is a special case of the same rules. An operating system that does this
reports `ext.headsup` only to the program that owns the display; to every other program the extension must not appear
to be half there, with no `headsup` global and `sys.hasExtension("ext.headsup")` returning `false`, as the
[main README](../README.md) requires.

---

### Example usage

```lua
local w, h = headsup.getSize()

headsup.drawRec(1, 1, w, h)      -- fill everything, corners in any order
headsup.clear()                  -- empty the buffer again; the display is unchanged

headsup.drawLine(1, 1, w, 1)     -- top edge
headsup.drawLine(1, h, w, h)     -- bottom edge
headsup.drawLine(1, 1, 1, h)     -- left edge
headsup.drawLine(w, 1, w, h)     -- right edge

headsup.draw()
```

```lua
local w, h = headsup.getSize()
local cx, cy = math.ceil(w / 2), math.ceil(h / 2)

while true do
  headsup.drawPixel(cx, cy)
  headsup.draw()                 -- dot on
  sys.sleep(0.5)
  headsup.clear()
  headsup.draw()                 -- the buffer was cleared, so the dot goes off
  sys.sleep(0.5)
end
```
