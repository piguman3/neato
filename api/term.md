# NEATO `Term` API Specification

Written by piguman3

Revision 2 of July 12, 2026

---

NEATO compatible environments must define a `Term` API that provides functions for interacting with a terminal emulator.

This terminal emulator is defined by a matrix of characters (each with colored background cells) that can be addressed with X and Y positions starting at 1, 1 at the top left corner. It should internally keep track of a cursor that the program can control to select where to perform operations on the screen, and keep track of the current background and foreground color for text. This is aimed to be similar to the ComputerCraft `term` API for developer familiarity.

RGB values work the same as in the default NEET Computers API, going from 0 to 255.

| Name | Description | Arguments | Returns |
|---|---|---|---|
| Term.Clear | Clears the terminal screen with the current background color. | none | nil |
| Term.Write | Prints a string of text on the screen with no formatting. | text (string) | nil |
| Term.GetSize | Returns the current width and height of the terminal in characters. | none | width (int), height (int) |
| Term.ClearLine | Clears the current line the cursor is on. | none | nil |
| Term.SetCursorPos | Sets the cursor's position. | x (int), y (int) | nil |
| Term.SetBackgroundColor | Sets the background color to the provided RGB value. | r (int), g (int), b (int) | nil |
| Term.SetTextColor | Sets the foreground text color to the provided RGB value. | r (int), g (int), b (int) | nil |
| Term.SetCursorPos | Gets the cursor's position. | none | x (int), y (int) |
| Term.SetBackgroundColor | Gets the background color's RGB value. | none | r (int), g (int), b (int) |
| Term.SetTextColor | Gets the foreground text color's RGB value. | none | r (int), g (int), b (int) |
| Term.Scroll | Scrolls the entire screen vertically up by `n` characters. | n (int) | none |
| print | Prints all of its arguments (converted to strings), separated by a tab character (`\t`), as a formatted string of text on screen, starting a new line on `\n` and wrapping at the edge of the terminal screen. | args, ... (any) | nil |

---

### Example usage

```lua
Term.SetBackgroundColor(0, 0, 0)
Term.Clear()
Term.SetCursorPos(1, 1)
print("Hello, World!")
```

---

### Footnotes

On `print`: This function overrides the original function that prints to the Minecraft console. We decided on this mostly because this feature is disabled by default, doesn't work in multiplayer servers where you can't see the terminal, (or vanilla singleplayer clients) and even then it is something only the OS maintainer should have access to. To be clear, `print` not being prefixed by `Term.` is intentional, as in vanilla Lua `print` isn't part of any table, and this preserves some native Lua program compatibility.
