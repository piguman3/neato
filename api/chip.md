# NEATO `chip` API Specification

Extension: `ext.chip`

Version: 1

Requires: `core`

---

A NEATO compliant operating system that reports this extension must provide the global `chip` table with the same
functions, parameters and return types as the NEET Computers API. The functions are about the computer itself, not
about the operating system running on it: its identity, its clocks, and switching it off. They keep their raw NEET
names so that a program written for NEET Computers works unchanged on an operating system that reports this
extension.

`chip` keeps the NEET Computers API unchanged, including how it reports its own failures: a failure is a Lua error
with a plain string message, and there is nothing for an operating system to redirect. The one thing an operating
system may do about the raw API, blocking its destructive calls, is defined below.

```c
chip
  version
  getMachine
  getUUID
  getTime
  getUnixTime
  getLunarTime
  sleep
  shutdown
  reboot
  crash
```

| Name              | Description                                                                       | Arguments        | Returns          |
| ----------------- | --------------------------------------------------------------------------------- | ---------------- | ---------------- |
| chip.version      | Returns the version string of the NEET Computers implementation the computer runs. | none             | version (string) |
| chip.getMachine   | Returns the model name of the machine, as the hardware reports it.                 | none             | name (string)    |
| chip.getUUID      | Returns the UUID of this computer.                                                 | none             | UUID (string)    |
| chip.getTime      | Returns how long the computer has been running, in seconds.                        | none             | seconds (number) |
| chip.getUnixTime  | Returns the current Unix epoch time, in seconds.                                   | none             | seconds (number) |
| chip.getLunarTime | Returns the in-game lunar time of the world the computer is in.                    | none             | time (int)       |
| chip.sleep        | Yields for `ms` milliseconds.                                                      | ms (number)      | nothing          |
| chip.shutdown     | Powers the computer off. Every program on it stops running.                        | none             | nothing          |
| chip.reboot       | Reboots the computer. Every program on it stops and the boot process starts over.  | none             | nothing          |
| chip.crash        | Crashes the computer. The crash screen shows `message` as the reason.              | message (string) | nothing          |

`chip.version` names the NEET Computers implementation, not the operating system; the operating system's name and
version come from [`sys`](core/sys.md). An implementation may be the mod itself or something that emulates it, and
the answer names whichever it is.

`chip.getMachine` returns the machine's model name and must stay that: an operating system must not substitute its
own name here. A program that wants the operating system's name uses [`sys.getOSName`](core/sys.md), and one that
wants the kind of machine, in the sense of `uname -m`, uses [`system.machine`](system.md).

`chip.getUUID` is the computer's own identity and may not change while it stays switched on. What an operating
system calls the computer on a network is a different name entirely; that is [`system.hostname`](system.md).

`chip.getTime` counts while the computer is on and resets when it is switched off or crashes. It counts from when
the computer last started running, not from when the operating system started; that number is
[`system.uptime`](system.md).

`chip.getUnixTime` is the computer's wall clock against the Unix epoch. It is not the NEATO wall clock
[`time.wall`](time.md), which an operating system may adjust and may not have at all. `chip.getTime` and
`chip.getUnixTime` stay the raw counters they are; an operating system or a program that wants portable time uses
[ext.time](time.md).

`chip.sleep` yields until at least `ms` milliseconds have passed. A `ms` of 0 yields exactly once, and a negative
`ms` returns without yielding. A value that is not a number raises an error. The sleep ends early if the computer
stops running.

`chip.getLunarTime` is in-game time and has no meaning outside the game; it is passed through unchanged.

---

### Blocking destructive calls

`chip.shutdown`, `chip.reboot` and `chip.crash` destroy the environment they are called in: an honored call ends
every program on the computer, and a crash ends it with a message a program chose. This specification expects an
operating system to keep programs from doing that to each other casually, and gives it the means to.

An operating system MAY block any of the three, for every program or only for some, for example unless the
program's user is an administrator (see [ext.user](user.md)), or always. The rule is the same for all three, and
`chip.crash` is not treated apart from `chip.shutdown` and `chip.reboot`: a program that is denied one of them must
not be left a path to the others. Which programs may call these functions, and why a call was blocked, is entirely
the operating system's policy and is not defined here.

A blocked call must have no effect: the computer must not power off, must not reboot and must not crash, and the
program that made the call must keep running. A blocked call must also fail in a way the program can detect: it
raises a Lua error, carrying a plain string message the operating system chooses. A program that is allowed the
call gets exactly the NEET Computers behavior: the computer powers off, reboots, or crashes with `message` as the
reason on the crash screen. The call itself returns like any other; the power off, reboot or crash follows on the
computer's own schedule.

The error, rather than the nil, code and message convention of [errors.md](../common/errors.md), is chosen because
`chip` is a raw passthrough extension like [ext.screen](screen.md) and keeps NEET's own failure reporting, which
is errors with plain string messages. It also cannot be ignored by accident: an uncaught error stops the program,
so a script that was denied cannot carry on as if it had shut the computer down.

The other functions must always work when this extension is reported. `chip.version`, `chip.getMachine`,
`chip.getUUID`, `chip.getTime`, `chip.getUnixTime`, `chip.getLunarTime` and `chip.sleep` are not destructive, and
an operating system must not block them.

An operating system that cannot honor `chip.shutdown` or `chip.reboot`, one that manages its own power and has no
way to answer the call, must still report this extension when it implements the rest. Blocking is the mechanism
this specification gives it for that: an operating system does not report a partial `ext.chip`, and it does not
withhold `ext.chip` over the calls it blocks.

---

### Example usage

```lua
local ok, err = pcall(chip.reboot)
if not ok then
  print("reboot refused: " .. tostring(err))
end
```

```lua
print(chip.getUUID())
  -> 5f8d1e2c-9a41-4b7e-8c3d-2f6a90b1c4d7

print(("the computer has been running for %d s"):format(chip.getTime()))
  -> the computer has been running for 4021 s
```