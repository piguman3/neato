# NEATO `term` API Specification

Written by piguman3 July 10th, 2026

Revision 1

Added in NEATO version alpha

Last updated in NEATO version alpha

---

Programs running under NEATO are able to access a "term" API that provides functions for interacting with a terminal emulator.

This terminal emulator is defined by a matrix of characters (each with colored background cells) that can be addressed with X and Y positions (starting at 1, at the top left corner), and it should internally keep track of a cursor that the program can control to select where to perform operations on the screen, and keep track of the current background and foreground color things will be printed in. This is very similar to Computercraft's terminal API.

RGB values work the same as in the default Neetcomputers API, going from 0 to 255.

| Name | Description | Arguments | Returns |
|---|---|---|---|
| term.clear | Clears the terminal screen with the current background color. | none | nil |
| term.write | Prints a string of text on the screen with no formatting. | text (string) | nil |
| term.getSize | Returns the current width and height of the terminal in characters. | none | width (int), height (int) |
| term.clearLine | Clears the current line the cursor is on. | none | nil |
| term.setCursorPos | Sets the cursor's position. | x (int), y (int) | nil |
| term.setBackgroundColor | Sets the background color to the provided RGB value. | r (int), g (int), b (int) | nil |
| term.setTextColor | Sets the foreground text color to the provided RGB value. | r (int), g (int), b (int) | nil |
| term.setCursorPos | Gets the cursor's position. | none | x (int), y (int) |
| term.setBackgroundColor | Gets the background color's RGB value. | none | r (int), g (int), b (int) |
| term.setTextColor | Gets the foreground text color's RGB value. | none | r (int), g (int), b (int) |
| term.scroll | Scrolls the entire screen vertically up by `n` characters. | n (int) | none |
| print | Prints all of its arguments (converted to strings), separated by a tab character (`\t`), as a formatted string of text on screen, starting a new line on `\n` and wrapping at the edge of the terminal screen. | args, ... (any) | nil |

Note on `print`: obviously, this function overrides the original function that prints to the Minecraft console. We decided on this mostly because this feature is disabled by default, doesn't work in multiplayer servers where you can't see the terminal, (or vanilla singleplayer clients) and because even then it feels like something only the OS developer should have access to. Also, to be clear, `print` not being prefixed by `term.` is intentional, as in vanilla Lua `print` isn't part of any table, and shouldn't be in a NEATO environment.
