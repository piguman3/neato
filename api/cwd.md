# NEATO `CWD` API Specification

Extension: `core`

Version: 2

---

A NEATO compliant environment must provide the current working directory as a path to the application via `_G.CWD`.
The path must be a full path in the format defined in [paths.md](../common/paths.md), must name an existing directory,
must end in `/`, and must not use `any` as its disk.

`CWD` is decided when the application is launched, and does not change while it runs. The operating system that
launches an application chooses its value (for example, a shell would use the directory the user was in). Relative
paths given to [`fs`](fs.md) are resolved against the value the operating system supplied at launch, so assigning
to the `CWD` global does not change how they are resolved.

---

### Example usage

```lua
print(CWD)
  -> "0:user:/home/"
```
