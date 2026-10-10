# NEATO `statfs` API Specification

Extension: `ext.statfs`

Version: 1

Requires: `core`

Status: draft. An OS must not report `ext.statfs` until this notice is removed.

---

A NEATO compliant operating system that reports this extension must provide one function in its [`fs`](fs.md)
API, `fs.statfs`, which answers how full the filesystem holding a path is. It is what the `df` core utility is
written from.

This extension adds one function to the `fs` API. An operating system defines it exactly when it reports this
extension, as [fs.md](fs.md) describes.

| Name      | Description                                                     | Arguments       | Returns                                     |
| --------- | --------------------------------------------------------------- | --------------- | ------------------------------------------- |
| fs.statfs | Returns the size and free space of the filesystem holding a path. | path (string)  | info (table), or nil, code and message      |

The table has two fields, both in bytes:

| Field   | Type           | Meaning                                                                          |
| ------- | -------------- | -------------------------------------------------------------------------------- |
| `total` | int, optional  | How much storage the filesystem holds in total.                                   |
| `free`  | int, optional  | How much of it may still be written by this program.                              |

Either field may be `nil` when the filesystem does not report it, so a filesystem with a fixed size but no free
space accounting, or the other way around, still fits this specification. The space used is `total - free` when
both are present. What "free" means for a program whose user has a quota, or a filesystem that reserves space, is
up to the operating system: `free` is what this program may actually use.

Everything at or below one partition is one filesystem. Two paths on the same partition get the same answer, and a
path on another partition gets that partition's.

---

### Failure codes

| Function   | Condition                                        | Code     |
| ---------- | ------------------------------------------------ | -------- |
| fs.statfs  | nothing exists at the path                       | `ENOENT` |
| fs.statfs  | a component of the path is not a directory       | `ENOTDIR`|
| fs.statfs  | the path is not allowed to be examined           | `EACCES` |
| fs.statfs  | the filesystem cannot be examined for another reason | `EIO` |

---

### Example usage

```lua
local info = assert(fs.statfs(fs.getPoint("storage.persistent") or CWD))
if info.free and info.free < 1024 * 1024 then
  print("warning: less than a megabyte left")
end
```

---

### Footnotes

The draft notice comes off when a second operating system has implemented the fields. There are deliberately no
per-file block counts (the `du` question): a program answers that from [`fs.stat`](fs.md#file-information) sizes,
which overestimate on filesystems that pad, and that is accepted for now.
