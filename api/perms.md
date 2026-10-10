# NEATO `perms` API Specification

Extension: `ext.perms`

Version: 1

Requires: `core`

---

A NEATO compliant operating system that reports this extension must provide permission bits for the objects of its
[`fs`](fs.md) filesystem, and a global set of `fs` functions to read and change them. This is the Unix model in its
smallest useful form, and an operating system that has no ownership or permissions at all, such as one where every
program may do everything, does not report this extension at all.

Every file and directory may carry permission bits. The bits are an integer from 0 to 511, written in octal in the
usual way, with three groups of `rwx`:

| Bits (octal) | Who    | On a file                                     | On a directory                                                     |
| ------------ | ------ | --------------------------------------------- | ------------------------------------------------------------------ |
| `400`        | owner  | may be read                                   | may be listed                                                      |
| `200`        | owner  | may be written                                | may have entries created and deleted in it                          |
| `100`        | owner  | may be run as a program                       | may be entered: its contents and the objects in it may be reached   |
| `40`, `20`, `10` | group | the same three, for the group               | the same three, for the group                                      |
| `4`, `2`, `1` | everyone | the same three, for everyone else           | the same three, for everyone else                                  |

Who the owner is: every object with permission bits has an owning user, by id, and an owning group, by id, as
[`ext.user`](user.md) defines users and groups. An object may carry no bits at all, in which case
[`fs.stat`](fs.md#file-information) reports no `mode`, `uid` or `gid` fields and this extension's functions answer
as if the object were fully accessible.

This extension adds four functions to the `fs` API. An operating system defines them exactly when it reports this
extension, as [fs.md](fs.md) describes, and a program tests for them with
[`sys.hasExtension("ext.perms")`](sys.md).

| Name      | Description                                                                                  | Arguments                              | Returns                            |
| --------- | -------------------------------------------------------------------------------------------- | -------------------------------------- | ---------------------------------- |
| fs.access | Returns whether this program may use a path in the given ways.                                | path (string), ways (string)           | boolean                            |
| fs.chmod  | Sets the permission bits of a path.                                                           | path (string), mode (int)              | true, or nil, code and message     |
| fs.chown  | Sets the owning user, the owning group, or both.                                              | path (string), user, group             | true, or nil, code and message     |
| fs.umask  | Returns the current umask, and sets a new one when one is given.                              | mask (int?)                            | mask (int)                         |

`fs.access(path, ways)` answers one question: may this program, right now, use this path in all of the ways named?
`ways` is a string of the letters `r`, `w` and `x`, with the meanings of the table above, such as `"r"` or `"rw"`;
an empty string asks only that the path can be reached at all. It answers `false` when the path does not exist or
the program may not, and it never fails: it is a question, like [`fs.exists`](fs.md), not an operation.

`fs.chmod(path, mode)` sets the bits of the object at the path to exactly `mode`, an integer from 0 to 511. A
program may chmod an object it owns, an administrator may chmod anything, and the operating system decides for its
own objects; anything this program may not change fails with `EACCES`, and an object that carries no permission
bits at all, such as some virtual files, fails with `ENOTSUP`.

`fs.chown(path, user, group)` sets the owner to the given user and the owner group to the given group. Either may
be `nil`, which leaves it unchanged. A user or group is given by id, or by name when the operating system knows
names for them. Changing the owner to somebody else is for administrators; anything this program may not change
fails with `EACCES` or `ENOTSUP`, as above.

`fs.umask` returns the calling program's umask, and sets it first when `mask` is given. The umask holds the bits
that are removed from the `mode` of every file and directory this program creates, whatever mode the creation would
otherwise give it: a program with umask `0o022` that creates a file that would be `0o666` gets `0o644`, and one
that would be `0o777` gets `0o755`. It applies to [`fs.open`](fs.md#opening-files) in the writing modes and to
[`fs.makeDir`](fs.md), and a child program starts with a copy of its parent's umask. Every program has a umask from
the moment it starts; the operating system chooses the first one.

An operating system enforces the bits on every [`fs`](fs.md) operation and on [`proc.spawn`](proc.md), and decides
for itself what an administrator may override; the table above says what each bit promises, not who may hold it.

---

### Failure codes

| Function | Condition                                                | Code           |
| -------- | -------------------------------------------------------- | -------------- |
| fs.access |                                                         | (never fails)  |
| fs.chmod | nothing exists at the path                               | `ENOENT`       |
| fs.chmod | the mode is not an integer from 0 to 511                 | `EINVAL`       |
| fs.chmod | the program may not change this object's bits            | `EACCES`       |
| fs.chmod | no permission bits model applies to this object          | `ENOTSUP`      |
| fs.chown | nothing exists at the path                               | `ENOENT`       |
| fs.chown | the user or group does not exist                         | `EINVAL`       |
| fs.chown | the program may not change this object's owner           | `EACCES`       |
| fs.chown | no permission bits model applies to this object          | `ENOTSUP`      |
| fs.chmod | the filesystem is read-only                              | `EROFS`        |
| fs.chown | the filesystem is read-only                              | `EROFS`        |
| fs.chmod | the bits cannot be changed for another reason            | `EIO`          |
| fs.chown | the owner cannot be changed for another reason           | `EIO`          |

---

### Example usage

```lua
-- Is there anything at this path that we may run?
if fs.access("tools/build", "x") then
  assert(proc.spawn("tools/build"))
end

-- Make a script runnable by everyone: rwxr-xr-x, octal 755.
-- A mode is an integer; write it with tonumber("755", 8), since Lua
-- has no octal literals.
assert(fs.chmod("tools/build", tonumber("755", 8)))

-- Private by default: files this program creates from now on lose group
-- and everyone bits, whatever mode they are created with (octal 077).
fs.umask(tonumber("077", 8))
```

---

### Footnotes

Only the nine bits of the table are defined. Setuid, sticky and the rest of the Unix bits are deliberately left
out: an operating system may store them, but no program can rely on them, and [`fs.stat`](fs.md#file-information)
must not report them through `mode`. The `x` bit on a file is a request to the operating system to treat the file
as a program, which is why [`proc.spawn`](proc.md) refuses a file without it with `EACCES`.
