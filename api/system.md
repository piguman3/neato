# NEATO `system` API Specification

Extension: `ext.system`

Version: 1

Requires: `core`

---

A NEATO compliant operating system that reports this extension must provide a global `system` API with the few
facts about the computer itself that programs such as `hostname`, `uname` and `uptime` are written from. The
operating system's name and version are already covered by [`sys.getOSName` and
`sys.getOSVersion`](sys.md); this extension covers the computer those programs run on.

| Name               | Description                                                                                     | Arguments       | Returns                            |
| ------------------ | ----------------------------------------------------------------------------------------------- | --------------- | ---------------------------------- |
| system.hostname    | Returns the computer's hostname.                                                                | none            | hostname (string)                  |
| system.setHostname | Sets the computer's hostname. May be refused.                                                   | hostname (string)| true, or nil, code and message    |
| system.machine     | Returns a short lowercase name for the kind of computer, or `nil` when there is no meaningful answer. | none       | machine (string), or nil           |
| system.uptime      | Returns the number of seconds since the operating system started.                                | none            | seconds (number)                   |

The hostname names this computer on a network of computers. It is a non-empty string, it may change while a program
runs, and a program that wants to know it again must ask again.

`system.setHostname` takes a non-empty string. An operating system may refuse the change, typically unless the
program's user is an administrator (see [`ext.user`](user.md)), which fails with `EACCES`; an empty name fails with
`EINVAL` and a name the operating system will not store, such as one that is too long, fails with `ELIMIT`.

`system.machine` is for the kind of "hardware" the operating system runs on, in the sense of `uname -m`: a short
lowercase name such as `"neet"`. Many NEET Computers are the same kind of machine, and a computer whose kind has no
name answers `nil`, which is an ordinary answer and not a failure.

`system.uptime` counts from the moment the operating system started. It uses the same kind of clock as
[`time.monotonic`](time.md): it never counts backwards. It is the one number here that is a measurement and not a
name, so it may have a fractional part.

---

### Failure codes

| Function            | Condition                                     | Code     |
| ------------------- | --------------------------------------------- | -------- |
| system.setHostname  | the name is empty                             | `EINVAL` |
| system.setHostname  | the name is too long, or will not be stored   | `ELIMIT` |
| system.setHostname  | the change is not allowed                     | `EACCES` |

---

### Example usage

```lua
print(system.hostname())
  -> workshop

print(system.machine() or "unknown machine")
  -> neet

print(("up %d s"):format(system.uptime()))
  -> up 4021 s
```
