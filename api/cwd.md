# NEATO `CWD` API Specification

Written by piguman3

Revision 1 of July 15, 2026

---

A NEATO compliant environment must provide the current working directory as a path to the application via `_G.CWD`. The path must be in the format defined in [paths.md](../common/paths.md).

---

### Example usage

```lua
print(CWD)
  -> "0:user:/home/"
```