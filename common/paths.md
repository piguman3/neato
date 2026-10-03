# NEATO Paths Specification

Written by piguman3

Extension: `core`

Version: 2

---

NEET Computers by default uses "partition:filepath" for most of its functions, but, to lower the number of arguments
and the complexity of many things in NEATO we instead have adopted the format of "disk:partition:filepath".

---

### Format

A full path is written `disk:partition:filepath`.

- `disk` identifies a disk in one of three ways:
  - a disk ID, a string of letters, digits, `-` and `_` that stays the same for as long as the disk exists, and
    that contains at least one character that is not a digit. Disk IDs are listed by `fs.getDisks`. Programs and
    configuration files that need to refer to a disk over time should use its ID.
  - the word `any`, which is not a disk but a search. It refers to the first disk, in ascending order of slot, that
    has a partition with the given name. If there is no such disk the path is invalid for lookups. `any` is decided
    once for each operation, before the operation starts.
  - a disk slot number, a non-negative integer that identifies the disk by its current slot. Slot numbers may change
    when disks are added, removed, or reordered, so programs and configuration files that need to refer to a disk
    over time should use its ID instead.
- `partition` is the name of a partition.
- `filepath` always begins with `/`, and is a list of names separated by `/`. A name cannot be empty or contain `/`,
  `:` or control characters. A path that ends in `/` refers to a directory. `.` refers to the same directory, and `..`
  to the parent directory (at the root of a partition, `..` is the root itself). Names are case sensitive.

The partition name cannot contain `:` or `/`.

### Relative paths

A path given to a NEATO function does not have to be a full path. If it does not have the two `:` separators of a full
path, it is instead:

- a path starting with `/`, which is a path on the disk and partition of [`CWD`](../api/cwd.md), or
- any other path, which is relative to `CWD`.

Both are turned into a full path by `fs.resolve`, and a path must never contain `:` unless it is a full path.
Resolving never checks whether anything exists at the path.

---

### Examples

- `0:bios:/hello.lua` -> file named "hello.lua", at the root of partition "bios" on the disk in slot 0.
- `4:hello:/my/epic/file.lua` -> file named "file.lua" inside of directory "epic", inside of "my", which is at the root
  of partition "hello" on the disk in slot 4.
- `hd0:user:/home/` -> the directory "home" at the root of partition "user" on the disk with ID "hd0".
- `any:system:/boot/cfg/boot.lua` -> "boot.lua" in the first disk that has a partition named "system".
- If `CWD` is `hd0:user:/home/`, then `notes/a.txt` is `hd0:user:/home/notes/a.txt`, `/etc/x` is `hd0:user:/etc/x`, and
  `../y` is `hd0:user:/y`.
