# NEATO `fs` API Specification

Extension: `core`

Version: 2

---

A NEATO compliant environment must provide a global `fs` API for working with files and directories. It replaces the
raw NEET Computers `files` API, whose administrative functions (creating, deleting and hiding partitions, choosing the
boot partition, and so on) are not part of `core`.

Every path taken by `fs` is in the format defined in [paths.md](../common/paths.md). It may be a full path
(`disk:partition:/dir/file`) or a path relative to [`CWD`](cwd.md). What a program may see or change is decided by the
operating system, which may refuse any operation.

On failure, functions that return a result return `nil`, then an error code, then a message, as defined in
[errors.md](../common/errors.md): the code is one of the required ones listed under [Failure codes](#failure-codes), and
the message is for humans and must not be compared. Functions that answer a question with a boolean, and
[`fs.getPoint`](#filesystem-points), which answers `nil` for a point that is not provided, do not fail this way. A path
that is not valid according to `paths.md`, or a value of the wrong type, raises an error instead.

| Name             | Description                                                                                                                                           | Arguments                    | Returns                                            |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- | -------------------------------------------------- |
| fs.exists        | Returns whether a file or directory exists at the path.                                                                                               | path (string)                | boolean                                            |
| fs.isFile        | Returns whether the path is a file.                                                                                                                   | path (string)                | boolean                                            |
| fs.isDir         | Returns whether the path is a directory.                                                                                                              | path (string)                | boolean                                            |
| fs.list          | Returns the names of everything inside a directory, in no particular order. Names do not include a path.                                              | path (string)                | names (table of strings), or nil, code and message |
| fs.makeDir       | Creates a directory. The parent directory must already exist, and the path must not.                                                                  | path (string)                | true, or nil, code and message                     |
| fs.delete        | Deletes a file or an empty directory. A partition root cannot be deleted.                                                                             | path (string)                | true, or nil, code and message                     |
| fs.open          | Opens a file. See below.                                                                                                                              | path (string), mode (string) | handle (table), or nil, code and message           |
| fs.resolve       | Returns the full, normalized form of a path, without checking that it exists. See [paths.md](../common/paths.md). The disk is always given as its ID. | path (string)                | full path (string), or nil, code and message       |
| fs.getDisks      | Returns the disks that are present, in ascending order of slot.                                                                                       | none                         | disks (table of `{slot = int, id = string}`)       |
| fs.getPartitions | Returns the names of the partitions on a disk.                                                                                                        | disk (int or string)         | names (table of strings), or nil, code and message |
| fs.getPoint      | Returns the path the operating system uses for a [filesystem point](#filesystem-points). A point the operating system does not provide returns `nil`. | point (string)               | path (string), or nil                              |
| fs.getPoints     | Returns every filesystem point the operating system provides, as a table from the point's name to its path.                                           | none                         | points (table from str to str)                     |

---

### Opening files

`mode` is one of:

| Mode | Meaning                                                                                 |
| ---- | --------------------------------------------------------------------------------------- |
| "r"  | Read. The file must exist.                                                              |
| "w"  | Write. The file is created if it does not exist, and emptied if it does.                |
| "a"  | Append. The file is created if it does not exist. All writes go to the end of the file. |

A trailing `b` (`"rb"`, `"wb"`, `"ab"`) is accepted and has no effect. Files are always binary, and no newline
translation happens.

A handle has the following methods, called with `:`. Calling any of them on a closed handle raises an error.

| Name         | Description                                                                                                                                                                                                                                                                  | Arguments                       | Returns                                                       |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------------- |
| handle:read  | Reads from the current position. `"l"` (the default) reads a line without its newline, `"a"` reads everything that is left (an empty string at the end of the file), and a number `n` reads up to `n` bytes. Returns `nil` at the end of the file for `"l"` and for numbers. | format (string or int?)         | data (string), or nil (end of file), or nil, code and message |
| handle:write | Writes strings or numbers at the current position.                                                                                                                                                                                                                           | ... (string or number)          | the handle, or nil, code and message                          |
| handle:seek  | Moves the position. `whence` is `"set"` (from the start), `"cur"` (the default) or `"end"`. Returns the new position, counted from the start.                                                                                                                                | whence (string?), offset (int?) | position (int), or nil, code and message                      |
| handle:flush | Makes sure everything written so far is stored.                                                                                                                                                                                                                              | none                            | true, or nil, code and message                                |
| handle:close | Closes the handle. Handles that are still open when the program ends are closed by the operating system.                                                                                                                                                                     | none                            | true                                                          |

Reading from a handle opened for writing, or writing to a handle opened with `"r"`, fails with `EBADF`.

---

### Failure codes

A function that can fail uses the exact code below for each condition, as defined in [errors.md](../common/errors.md).
The code is the second result, after the `nil`. `fs.exists`, `fs.isFile` and `fs.isDir` answer with a boolean and do
not fail; `fs.getDisks` and `fs.getPoints` always succeed; and `fs.getPoint` answers `nil` for a point that is not
provided.

| Function         | Condition                                          | Code           |
| ---------------- | -------------------------------------------------- | -------------- |
| fs.list          | the directory does not exist                       | `ENOENT`       |
| fs.list          | the path is not a directory                        | `ENOTDIR`      |
| fs.list          | the directory is not allowed to be read            | `EACCES`       |
| fs.list          | the directory cannot be read                       | `EIO`          |
| fs.makeDir       | the path already exists                            | `EEXIST`       |
| fs.makeDir       | the parent directory does not exist                | `ENOENT`       |
| fs.makeDir       | a component of the path is not a directory         | `ENOTDIR`      |
| fs.makeDir       | the directory is not allowed to be created         | `EACCES`       |
| fs.makeDir       | the filesystem is read-only                        | `EROFS`        |
| fs.makeDir       | there is no space left                             | `ENOSPC`       |
| fs.makeDir       | the name is too long                               | `ENAMETOOLONG` |
| fs.makeDir       | the directory cannot be created for another reason | `EIO`          |
| fs.delete        | nothing exists at the path                         | `ENOENT`       |
| fs.delete        | the directory is not empty                         | `ENOTEMPTY`    |
| fs.delete        | the path is a partition root                       | `EPERM`        |
| fs.delete        | the path is not allowed to be deleted              | `EACCES`       |
| fs.delete        | the filesystem is read-only                        | `EROFS`        |
| fs.delete        | the path is in use                                 | `EBUSY`        |
| fs.delete        | the path cannot be deleted for another reason      | `EIO`          |
| fs.open          | the file does not exist and the mode is `"r"`      | `ENOENT`       |
| fs.open          | the path is a directory                            | `EISDIR`       |
| fs.open          | a component of the path is not a directory         | `ENOTDIR`      |
| fs.open          | the mode is not a valid mode                       | `EINVAL`       |
| fs.open          | the file is not allowed to be opened               | `EACCES`       |
| fs.open          | the filesystem is read-only and the mode writes    | `EROFS`        |
| fs.open          | there is no space left and the mode writes         | `ENOSPC`       |
| fs.open          | the file is too large                              | `EFBIG`        |
| fs.open          | the name is too long                               | `ENAMETOOLONG` |
| fs.open          | the file cannot be opened for another reason       | `EIO`          |
| fs.resolve       | the disk cannot be identified                      | `ENODEV`       |
| fs.resolve       | the path is too long                               | `ENAMETOOLONG` |
| fs.getPartitions | the disk does not exist                            | `ENODEV`       |
| fs.getPartitions | the disk argument is not a valid slot or ID        | `EINVAL`       |
| fs.getPartitions | the partitions cannot be read                      | `EIO`          |
| handle:read      | the format is not `"l"`, `"a"` or a number         | `EINVAL`       |
| handle:read      | the handle was opened for writing                  | `EBADF`        |
| handle:read      | the file cannot be read                            | `EIO`          |
| handle:write     | the handle was opened for reading                  | `EBADF`        |
| handle:write     | there is no space left                             | `ENOSPC`       |
| handle:write     | the file is too large                              | `EFBIG`        |
| handle:write     | the filesystem is read-only                        | `EROFS`        |
| handle:write     | the data cannot be written for another reason      | `EIO`          |
| handle:seek      | the whence is not `"set"`, `"cur"` or `"end"`      | `EINVAL`       |
| handle:seek      | the position is out of range                       | `ERANGE`       |
| handle:seek      | the position cannot be reached                     | `EIO`          |
| handle:flush     | there is no space left                             | `ENOSPC`       |
| handle:flush     | the filesystem is read-only                        | `EROFS`        |
| handle:flush     | the data cannot be stored                          | `EIO`          |
| handle:close     | the handle cannot be closed cleanly                | `EIO`          |

---

### Filesystem points

A filesystem point is a name for a directory whose job is fixed, while the directory it actually lives in is left up
to the operating system. It is NEATO's abstraction of the layout described by the [Filesystem Hierarchy
Standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html) (FHS 3.0). NEATO does not require an operating
system to follow FHS, or any other layout, and does not try to reproduce one. A program asks for a directory by what it
is for, and the operating system answers with the path it uses. For example, `storage.runtime` is `/run` on an FHS
system and might be `/apps/store/runtime` on one that is not.

`fs.getPoint(point)` returns the full path of a point. It returns `nil` if the operating system does not provide that
point; a point that does not apply to a computer, such as `mount.removable` on one with no removable media, also
returns `nil`. This is not a failure, so no code or message comes with it. A name that is not one of the points below
returns `nil` as well, so that an operating system that predates a point and one that chooses not to provide it behave
the same, and a program must treat `nil` as "not available". Passing anything other than a string raises an error.

`fs.getPoints()` returns a table from the name of every point the operating system provides to its path. It is the
authoritative list of what is available, and it never contains a point that is not supported. A program that can accept
more than one point, such as any persistent storage, picks the first one that is present:

```lua
local dir = fs.getPoint("storage.state") or fs.getPoint("storage.persistent")
```

Every path returned is a full path in the [paths](../common/paths.md) format, names a directory, and ends in `/`. The
operating system chooses it, it is not derived from `CWD`, and it does not change while the program runs. The directory
may or may not already exist, and a program that needs it can check with `fs.exists` or create it with `fs.makeDir` if
the operating system allows it; support for a point also says nothing about whether the program may write there. The
same directory may be the answer for more than one point.

The following points are defined. A point may stand for more than one FHS directory, in which case the operating system
returns whichever one it actually uses.

| Point                         | FHS 3.0 directory    | Purpose                                                                              |
| ----------------------------- | -------------------- | ------------------------------------------------------------------------------------ |
| `system.root`                 | `/`                  | The root of the operating system's own filesystem.                                   |
| `system.bin`                  | `/bin`, `/usr/bin`   | Commands used by users.                                                              |
| `system.sbin`                 | `/sbin`, `/usr/sbin` | Commands for system administration.                                                  |
| `system.libraries`            | `/lib`, `/usr/lib`   | Shared libraries and loadable modules.                                               |
| `system.include`              | `/usr/include`       | Header files for compiled programs.                                                  |
| `system.source`               | `/usr/src`           | Source code.                                                                         |
| `system.config`               | `/etc`               | Host-specific system configuration.                                                  |
| `system.boot`                 | `/boot`              | Files needed before the operating system starts, such as a kernel or boot loader.    |
| `system.device`               | `/dev`               | Device files.                                                                        |
| `software.opt`                | `/opt`               | Add-on application packages.                                                         |
| `software.opt.config`         | `/etc/opt`           | Host-specific configuration for `software.opt`.                                      |
| `software.opt.data`           | `/var/opt`           | Variable data for `software.opt`.                                                    |
| `software.local`              | `/usr/local`         | Software installed locally by the administrator.                                     |
| `software.localData`          | `/usr/local/share`   | Architecture-independent data belonging to `software.local`.                         |
| `software.data`               | `/usr/share`         | Architecture-independent data shared by programs, such as documentation and locales. |
| `storage.persistent`          | `/var`               | Variable data that survives reboots.                                                 |
| `storage.runtime`             | `/run`, `/var/run`   | Data about the running system, cleared at boot.                                      |
| `storage.state`               | `/var/lib`           | State that programs keep between runs.                                               |
| `storage.cache`               | `/var/cache`         | Cached data that can be regenerated.                                                 |
| `storage.log`                 | `/var/log`           | Log files.                                                                           |
| `storage.spool`               | `/var/spool`         | Data queued for later processing.                                                    |
| `storage.lock`                | `/var/lock`          | Lock files.                                                                          |
| `storage.mail`                | `/var/mail`          | User mailboxes.                                                                      |
| `storage.crash`               | `/var/crash`         | System crash dumps.                                                                  |
| `storage.games`               | `/var/games`         | Variable data for games.                                                             |
| `storage.temporary`           | `/tmp`               | Temporary files that are not kept across reboots.                                    |
| `storage.temporaryPersistent` | `/var/tmp`           | Temporary files that are kept across reboots.                                        |
| `user.home`                   | `/home`              | User home directories.                                                               |
| `user.admin`                  | `/root`              | The home directory of the administrative account, if there is one.                   |
| `mount.removable`             | `/media`             | Mount points for removable media.                                                    |
| `mount.temporary`             | `/mnt`               | A mount point for a filesystem mounted by hand.                                      |
| `service.data`                | `/srv`               | Data served by services on this computer.                                            |

An operating system may define points of its own under the `x.<vendor>.` prefix; `fs.getPoints` reports them, and a
program must ignore any point it does not know. Interior directories that only contain the ones above, such as `/usr`
itself, do not get a point of their own, and neither do optional directories that programs do not generally use, such as
`/var/account` and `/var/yp`.

---

### Example usage

```lua
local f = assert(fs.open("notes.txt", "w"))
f:write("hello\n", 42, "\n")
f:close()

local f = assert(fs.open("notes.txt", "r"))
print(f:read("l"))   -> "hello"
print(f:read("a"))   -> "42\n"
f:close()
```

```lua
for _, d in ipairs(fs.getDisks()) do
  print(d.slot, d.id, table.concat(fs.getPartitions(d.id), ", "))
end
```

```lua
print(fs.getPoint("storage.runtime"))
  -> "0:system:/run/"

print(fs.getPoint("mount.removable"))   -- no removable media on this computer
  -> nil

for point, path in pairs(fs.getPoints()) do
  print(point, path)
end
  -> storage.runtime    0:system:/run/
     storage.temporary  0:system:/tmp/
     system.config      0:system:/etc/
```
