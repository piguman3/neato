# NEATO `utimes` API Specification

Extension: `ext.utimes`

Version: 1

Requires: `core`

---

A NEATO compliant operating system that reports this extension must provide one function in its [`fs`](core/fs.md)
API, `fs.utimes`, which sets the recorded modification time of a file or directory. It is what the `touch` core
utility is written from, and what copying a file while keeping its date needs.

The time is the one [`fs.stat`](core/fs.md#file-information) reports: the optional `mtime` field, in seconds on the
[`ext.time`](time.md) wall clock, exactly as `fs.stat` returns it and exactly as [`time.wall`](time.md) measures it.
`fs.stat` defines no other time field, and this extension sets no other.

This extension adds one function to the `fs` API. An operating system defines it exactly when it reports this
extension, as [fs.md](core/fs.md) describes.

| Name      | Description                                                  | Arguments                  | Returns                        |
| --------- | ------------------------------------------------------------ | -------------------------- | ------------------------------ |
| fs.utimes | Sets the recorded modification time of a file or directory.  | path (string), mtime (int) | true, or nil, code and message |

`fs.utimes(path, mtime)` sets the modification time of the object at `path` to `mtime`; afterwards,
`fs.stat(path)` reports that `mtime`. `mtime` must be an integer, on the same wall clock `fs.stat`'s `mtime` is on:
a number that is not an integer fails with `EINVAL`, as [`fs.chmod`](perms.md) does for a mode that is out of range,
and a value that is not a number at all raises an error, as a wrong type always does. There is no meaning for `nil`:
the function has one time to set, so it always sets it.

Setting the time is not a modification of the file. No byte of it is read or written, the call does not make the
file newer in any other respect, and it must not update the very time it sets: `fs.utimes(path, t)` followed by
`fs.stat(path)` reports exactly `t`. The set time holds until the object is next actually modified, when the
operating system records a new `mtime` as it usually does.

The function works on files and directories, and on any other object the operating system keeps a time for. Where
`fs.stat` reports no `mtime` - because the operating system does not track times at all, or keeps none for that kind
of object - there is nothing to set, and `fs.utimes` fails with `ENOTSUP`, the answer
[`ext.perms`](perms.md) gives for an object with no permission bits behind it. A program can look before it leaps:
`fs.stat(path).mtime` being `nil` is what `ENOTSUP` will say.

Who may set a time is up to the operating system. When [`ext.perms`](perms.md) is reported, the natural rule is the
Unix one, that the owner of the object and an administrator may; what an operating system decides otherwise, with or
without an ownership model, is its own policy, and anything this program may not change fails with `EACCES`.

---

### Failure codes

| Function  | Condition                                    | Code           |
| --------- | -------------------------------------------- | -------------- |
| fs.utimes | nothing exists at the path                   | `ENOENT`       |
| fs.utimes | a component of the path is not a directory   | `ENOTDIR`      |
| fs.utimes | `mtime` is a number that is not an integer   | `EINVAL`       |
| fs.utimes | the program may not change this object's time | `EACCES`      |
| fs.utimes | the object has no recorded modification time | `ENOTSUP`      |
| fs.utimes | the filesystem is read-only                  | `EROFS`        |
| fs.utimes | the name is too long                         | `ENAMETOOLONG` |
| fs.utimes | the time cannot be changed for another reason | `EIO`         |

---

### Example usage

```lua
-- touch: make it look just written.
assert(fs.utimes("notes.txt", time.wall()))
```

```lua
-- The timestamp part of "cp -p": copy the original's time onto the copy.
local original = assert(fs.stat(src))
if original.mtime then
  assert(fs.utimes(dst, original.mtime))
else
  -- This operating system does not track times; there is nothing to copy.
end
```