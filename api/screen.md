# NEATO `screen` API Specification

Extension: `ext.screen`

Version: 1

---

An environment that reports `ext.screen` must provide the global `screen` table with the same functions, parameters
and return types as the NEET Computers API. It does not necessarily have to access the main screen: an operating
system may direct drawing into a window or onto a layer of its own, as long as the functions behave the same from the
program's point of view. This is separate from the [terminal](term.md) in `core`, and the two are not required to
share the same surface.

`screen` keeps the NEET Computers API unchanged, including how it reports its own failures, so it does not use the
codes in [errors.md](../common/errors.md).

Requires: `core`

```c
screen
  readData
  draw
  clone
  writePixel
  writeLine
  readPixel
  writeData
  createLayer
  substitute
  fill
  getSize
  set
```
