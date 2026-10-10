# NEATO `term` API Specification

Written by piguman3

Extension: `core`

Version: 3

---

NEATO compatible environments must define a `term` API that provides functions for interacting with a terminal emulator.

This terminal emulator is defined by a matrix of characters (each with colored background cells) that can be addressed
with X and Y positions starting at 1, 1 at the top left corner. It should internally keep track of a cursor that the
program can control to select where to perform operations on the screen, and keep track of the current background and
foreground color for text. This is aimed to be similar to the ComputerCraft `term` API for developer familiarity.

RGB values work the same as in the default NEET Computers API, going from 0 to 255.

| Name                    | Description                                                                                                                                                                                                    | Arguments                 | Returns                   |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------- | ------------------------- |
| term.clear              | Clears the terminal screen with the current background color. The cursor does not move.                                                                                                                        | none                      | nil                       |
| term.write              | Writes a string of text at the cursor with no formatting, and moves the cursor past it.                                                                                                                        | text (string)             | nil                       |
| term.getSize            | Returns the current width and height of the terminal in characters.                                                                                                                                            | none                      | width (int), height (int) |
| term.clearLine          | Clears the current line the cursor is on. The cursor does not move.                                                                                                                                            | none                      | nil                       |
| term.setCursorPos       | Sets the cursor's position.                                                                                                                                                                                    | x (int), y (int)          | nil                       |
| term.setBackgroundColor | Sets the background color to the provided RGB value.                                                                                                                                                           | r (int), g (int), b (int) | nil                       |
| term.setTextColor       | Sets the foreground text color to the provided RGB value.                                                                                                                                                      | r (int), g (int), b (int) | nil                       |
| term.getCursorPos       | Gets the cursor's position.                                                                                                                                                                                    | none                      | x (int), y (int)          |
| term.getBackgroundColor | Gets the background color's RGB value.                                                                                                                                                                         | none                      | r (int), g (int), b (int) |
| term.getTextColor       | Gets the foreground text color's RGB value.                                                                                                                                                                    | none                      | r (int), g (int), b (int) |
| term.scroll             | Scrolls the entire screen vertically up by `n` lines (down if `n` is negative). The cursor does not move.                                                                                                      | n (int)                   | nil                       |
| print                   | Prints all of its arguments (converted to strings), separated by a tab character (`\t`), as a formatted string of text on screen, starting a new line on `\n` and wrapping at the edge of the terminal screen. | args, ... (any)           | nil                       |

---

### Behavior

- When a program starts, the background color is 0, 0, 0 (black) and the text
color is 255, 255, 255 (white).

- `term.setCursorPos` accepts any integers, including positions outside the screen. Cells outside the
screen do not exist: nothing is drawn to them, but the cursor keeps the position that was set. `term.getCursorPos`
returns exactly the position last set or reached by the cursor moving.

- `term.setBackgroundColor` and `term.setTextColor` only affect cells that are written or cleared
afterwards; cells already on the screen keep their colors. Each of `r`, `g` and `b` must be an integer from 0 to 255,
otherwise an error is raised.

- `term.write` draws each character of `text` in its own cell starting at the cursor, using the current colors, and
moves the cursor right by one cell per character. It does not interpret control characters such as `\n`, does not
wrap, and never changes the cursor's row. Characters that would land outside the screen are discarded, but the cursor
still advances past them.

- `term.clear` and `term.clearLine`. Every affected cell becomes a space with the current background color.
`term.clearLine` does nothing if the cursor's row is outside the screen.

- `term.scroll`. For a positive `n`, the content of row `n + 1` moves to row 1, and so on, the top `n` rows are
discarded, and the bottom `n` rows are cleared as by `term.clear`. A negative `n` scrolls the other way by `-n`
rows. If `n` is 0 nothing happens, and if `n` is at least the terminal height the result is the same as
`term.clear`.

- `print`. Writes the converted arguments, separated by tab characters, starting at the cursor and using the
current colors. A `\n` moves the cursor to column 1 of the next line. If a character would fall past the right edge
of the screen it is placed at column 1 of the next line instead. Whenever the cursor would move below the last row,
the terminal first scrolls up by one line, as with `term.scroll(1)`, and the cursor stays on the last row. After the
last argument the cursor moves to column 1 of the next line, as if the output ended with `\n`. If the cursor is
outside the screen when `print` is called, it behaves as if the cursor had been moved to the nearest cell on the
screen first.

- Passing an argument of the wrong type to any `term` function raises an error. No `term` function fails with an error
  code: a mistake raises, and everything else succeeds, as described in [errors.md](../common/errors.md).

- When the operating system also reports [`ext.stdio`](stdio.md), `print` writes to the standard output instead of
  the terminal, as [stdio.md](stdio.md) defines; when the standard output is not a terminal this replaces the wrapping
  and scrolling described above. Every `term` function except `print` always acts on the terminal itself.

---

### Example usage

```lua
term.setBackgroundColor(0, 0, 0)
term.clear()
term.setCursorPos(1, 1)
print("Hello, World!")
```

---

### Footnotes

On `print`: This function overrides the original function that prints to the Minecraft console. We decided on
this mostly because this feature is disabled by default, doesn't work in multiplayer servers where you can't
see the terminal, (or vanilla singleplayer clients) and even then it is something only the OS maintainer should
have access to. To be clear, `print` not being prefixed by `term.` is intentional, as in vanilla Lua `print`
isn't part of any table, and this preserves some native Lua program compatibility.
