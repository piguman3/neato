# NEATO `partitions` API Specification

Extension: `ext.partitions`

Version: 1

Requires: `core`

---

A NEATO compliant operating system that reports this extension must provide the global `partitions` table with the
same functions, parameters and return types as the NEET Computers API. In stock NEET Computers these functions live in
the global `files`, mixed in with the file functions; NEATO's [`fs`](core/fs.md) deliberately does not carry them, and
this extension gives them a global of their own. The name is of the hardware, not of paths: its functions take disk
slots and partition names, which is machine-level work, and `fs` is a view an operating system builds on top of what
they manage. The one kind of path its functions take, the boot entrypoint, is a NEET path on the disk named by the
`disk` parameter, not a NEATO [path](../common/paths.md); [Boot selection](#boot-selection) describes its exact form.

`partitions` keeps the NEET Computers API unchanged, including how it reports its own failures. A failure is a Lua
error with a plain string message, and there is nothing for an operating system to redirect. The two things an
operating system may do about the raw API, hiding whole disks and blocking its destructive calls, are defined below.

### Disks and partitions

A NEET Computers machine stores its files on disks. Every machine has exactly one of its own, the disk it came with,
in slot 0; more can be attached to it from outside, and each of those takes the next free slot number. A disk is
identified by its slot number, which can change as disks come and go, and by an ID, a string that stays the same for
as long as the disk exists - the same two notions the [paths](../common/paths.md) format writes as the first
component of a full path.

A partition is a named sub-tree of a disk: paths on the disk start with the partition's name, and everything below
that name belongs to the partition. A partition carries two flags. `readonly` marks its contents as not to be
written: the hardware honors at least this much, refusing to delete a read-only partition, and an operating system
that mounts a read-only partition must present it read-only, the `EROFS` of [fs](core/fs.md). `hidden` constrains
nothing in the hardware itself: the disk keeps the flag, reports it through `partitions.getPartition`, and lists
hidden partitions to this API like any other. Whoever presents the partition to a program is who honors it - in
stock NEET Computers that is the shell, which displays the flag; in a NEATO environment it is the operating system,
and a partition it hides is one it does not mount.

NEATO's [fs](core/fs.md) presents one merged tree and has none of this machinery. It answers which disks and
partitions exist, in NEATO form (`fs.getDisks`, `fs.getPartitions`), but creating, deleting and hiding partitions,
ejecting a disk and choosing what the machine runs when it is switched on are hardware operations, and an operating
system mounts from them whatever it decides a program may see. This extension is that hardware view.

```c
partitions
  getPartitions
  getPartition
  createPartition
  setPartitionHidden
  setPartitionReadOnly
  deletePartition
  getNumberOfDisks
  getDisks
  getDiskID
  removeDisk
  getBootPath
  setBoot
```

| Name                        | Description                                                                                                          | Arguments                              | Returns                 |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------- | -------------------------------------- | ----------------------- |
| partitions.getPartitions    | Returns the names of the partitions on a disk, in no particular order. Hidden partitions are listed like any other.  | disk (int?)                            | names (table of strings) |
| partitions.getPartition     | Returns a table describing one partition, or nil when the disk has none of that name. See below.                     | name (string), disk (int?)             | info (table), or nil    |
| partitions.createPartition  | Creates an empty, writable, visible partition of the given name. An invalid or already-used name returns false.      | name (string), disk (int?)             | created (boolean)       |
| partitions.setPartitionHidden | Sets the partition's hidden flag to `state`. Returns whether the disk has such a partition.                        | name (string), state (boolean), disk (int?) | set (boolean)      |
| partitions.setPartitionReadOnly | Marks the partition read-only. There is no function here that clears the flag again. Returns whether the disk has such a partition. | name (string), disk (int?)  | set (boolean)          |
| partitions.deletePartition  | Deletes the partition and everything stored in it. A read-only partition refuses with an error instead.              | name (string), disk (int?)             | deleted (boolean)       |
| partitions.getNumberOfDisks | Returns how many disks this program may see are attached.                                                            | none                                   | count (int)             |
| partitions.getDisks         | Returns the slot numbers of the disks this program may see, in no particular order.                                  | none                                   | slots (table of ints)   |
| partitions.getDiskID        | Returns the ID of a disk, the string the hardware identifies it by. It stays the same for as long as the disk exists. | disk (int)                            | id (string)             |
| partitions.removeDisk       | Ejects a disk, as if it were pulled from its drive bay. The machine's own disk cannot be ejected.                    | disk (int)                             | removed (boolean)       |
| partitions.getBootPath      | Returns the entrypoint the disk boots from, or nil when it does not boot.                                            | disk (int)                             | path (string), or nil   |
| partitions.setBoot          | Makes the disk bootable from `entrypoint`, a path on that disk.                                                      | entrypoint (string), disk (int)        | set (boolean)           |
| partitions.setBoot          | Makes a disk stop booting, returning whether it was bootable before; the machine's own disk answers `false` and keeps its boot selection. | disk (int)                     | was (boolean)           |

Every partition function takes its disk as an optional trailing argument, and the omitted disk is the machine's own,
slot 0. A disk number that is not attached fails every function that takes one with `Disk not found`, as
[Failures](#failures) lists. Slot numbers change as disks come and go; IDs do not, and `partitions.getDiskID` is how
one is turned into the other.

`partitions.getPartition` answers nil for a partition the disk does not have, which is an ordinary answer and not a
failure. Names are matched without regard to case. The table has three fields:

| Field      | Type    | Meaning                                                                   |
| ---------- | ------- | ------------------------------------------------------------------------- |
| `name`     | string  | The partition's name, as it is written at the head of a path on the disk. |
| `readonly` | boolean | Whether the partition refuses writes.                                     |
| `hidden`   | boolean | Whether the partition asks to be kept out of the views presented to programs. |

---

### Failures

`partitions` keeps the NEET Computers failure convention and does not use the codes in [errors.md](../common/errors.md).
A failure is a plain Lua error carrying one of these messages, exactly as NEET Computers raises it:

| Message                                       | Raised when                                                                                |
| --------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `Disk not found`                              | a function was given a disk number that is not a disk this program may see (see [Seeing disks](#seeing-disks)). |
| `Invalid entrypoint`                          | `partitions.setBoot` was given an entrypoint that is not a valid path on its disk.         |
| `entrypoint doesn't refer to an assigned partition` | the first name of the entrypoint is not a partition of that disk.                     |
| `Access denied`                               | `partitions.deletePartition` was aimed at a read-only partition.                           |

The spelling and wording of the messages are NEET's and are not corrected here, because programs match on them.
Everything else in this API answers by return value and does not fail: `false` for a partition that does not exist
or a change that was not made, `nil` from `partitions.getPartition` for a partition that does not exist, and `nil`
from `partitions.getBootPath` for a disk that does not boot.

---

### Boot selection

A disk that boots carries an entrypoint: the path, on that disk, of the file the machine runs when it is switched
on. `partitions.getBootPath` and `partitions.setBoot` pass this selection through. `setBoot(entrypoint, disk)` makes
the disk boot from `entrypoint`. A path that is not valid, or whose first name is not a partition of the disk, is an
error; on the machine's own disk the file must already exist, and a missing one is answered with `false` rather than
an error. On any other disk the file does not have to exist yet. Called with only the disk number,
`setBoot(disk)` makes that disk stop booting and returns whether it was bootable before; a call that takes effect
returns `true`. The machine's own disk is the one exception. It refuses this form, answering `false` and keeping
whatever boot selection it has.

The path `partitions.getBootPath` answers, and `partitions.setBoot` accepts, is the disk's own entrypoint path in
NEET's normalized form, backslashes and all; it is passed through, not converted into the NEATO
[paths](../common/paths.md) format.

This is the hardware's power-on selection, not the NEATO bootloader's: a NEATO bootloader is configured through
`boot.lua` instead (see [boot/README.md](../boot/README.md)).

---

### Blocking destructive calls

Six of the functions here change the machine rather than the caller's view of it: `partitions.createPartition`,
`partitions.deletePartition`, `partitions.setPartitionHidden`, `partitions.setPartitionReadOnly`,
`partitions.removeDisk` and both forms of `partitions.setBoot`. They rearrange storage every program shares, hide or
erase what other programs may be using, eject hardware, and choose what the machine runs at power-on. This
specification expects an operating system to keep programs from doing that to each other casually, and gives it the
means to.

An operating system MAY block any of the six, for every program or only for some, for example unless the program's
user is an administrator (see [ext.user](user.md)), or always. Which programs may call these functions, and why a
call was blocked, is entirely the operating system's policy and is not defined here.

A blocked call must have no effect: no partition is created, changed or deleted, no disk is ejected, no boot
selection changes. A blocked call raises a Lua error, carrying a plain string message the operating system chooses;
a program that is allowed the call gets exactly the NEET Computers behavior.

The error, rather than the nil, code and message convention of [errors.md](../common/errors.md), is because
`partitions` is a raw passthrough extension like [ext.chip](chip.md) and keeps NEET's own failure reporting, which
is errors with plain string messages.

The other functions must always work when this extension is reported. `partitions.getPartitions`,
`partitions.getPartition`, `partitions.getNumberOfDisks`, `partitions.getDisks`, `partitions.getDiskID` and
`partitions.getBootPath` are not destructive, and an operating system must not block them.

An operating system that answers none of the destructive calls, one that manages its disks and its boot selection
itself and has no way to answer the calls, must still report this extension when it implements the rest. Blocking is
the mechanism this specification gives it for that: an operating system does not report a partial `ext.partitions`,
and it does not withhold `ext.partitions` over the calls it blocks.

---

### Seeing disks

In NEET Computers, every program sees every attached disk and may call the whole API. An operating system that
reports `ext.partitions` MAY instead hide disks from a program, the way [ext.peripherals](peripherals.md) may hide
peripherals. A disk that is hidden from a program must be indistinguishable from one that is not attached: it does
not appear in the results of `partitions.getDisks`, it is not counted by `partitions.getNumberOfDisks`, and every
function that takes its slot number fails on it with `Disk not found`, the same answer an unknown number gets. This
is the rule the NEATO registry applies to extensions an operating system does not report - nothing must appear to be
half there - applied one level down, to the disks. A program can therefore trust what it finds: a disk it can see is
a disk it may ask things of, and one it cannot find is not for it.

The two models divide the work. Visibility is for keeping a program away from a disk entirely; blocking is for a
program that may see a disk and use it, but not rearrange it. For partitions there is no third model: which
partitions of a visible disk a program reaches is decided by the operating system's [fs](core/fs.md) view - it
mounts what it wants the program to have and leaves the rest out - and `hidden`, the flag the disk itself carries,
is one input to that decision, not a rule of this specification.

An operating system that reports `ext.partitions` and restricts nothing grants a program the whole API, which is
the plain NEET Computers behavior.

---

### Example usage

```lua
-- what is in the machine?
for _, slot in ipairs(partitions.getDisks()) do
  print(slot, partitions.getDiskID(slot))
  for _, name in ipairs(partitions.getPartitions(slot)) do
    local p = partitions.getPartition(name, slot)
    local flags = p.readonly and "read-only" or "writable"
    if p.hidden then flags = flags .. ", hidden" end
    print("  " .. name, flags)
  end
end
```

```lua
-- an installer carving itself a partition to live in
if not partitions.getPartition("system") then
  if not partitions.createPartition("system") then
    error("could not create the system partition")
  end
end
print(partitions.getPartition("system").name)
  -> system
```

```lua
-- boot repair: point the machine's own disk at its boot file again
if partitions.getBootPath(0) == nil then
  partitions.setBoot("bios:/boot.lua")
end

-- and taking an attached disk's bootability away
print(partitions.setBoot(1))
  -> true    (it was bootable, and now it is not)
print(partitions.getBootPath(1))
  -> nil
```

```lua
-- a partition tool running without the privilege to destroy
local ok, err = pcall(partitions.deletePartition, "system")
if not ok then
  print("refused: " .. tostring(err))
end
```
