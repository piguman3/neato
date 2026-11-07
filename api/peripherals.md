# NEATO `io` API Specification

Extension: `ext.peripherals`

Version: 1

Requires: `core`

---

In NEET Computers, the global `io` table is the machine's window onto its hardware. It is how a program sees the
peripherals attached to the computer, asks what they are and calls them, and it is how the computer speaks to every
other computer on its network cables. An operating system that reports `ext.peripherals` must provide this global
`io` table with the same functions, parameters and return types as the NEET Computers API, and defines it exactly
when it reports the extension, as [globals.md](globals.md) describes.

NEET Computers hands this window to every program on the computer: any program can enumerate every peripheral,
rename another program's tagged drive with `setTag`, call anything a peripheral exposes, and shout on the local
network. An operating system will usually not want that. It will want privilege levels, so that a program reaches
the hardware only as far as the operating system allows, rather than directly. The answer of this specification is
not a different API: the functions stay NEET's, unchanged in name, parameters and return types. The answer is a
visibility and access model, in [Access and privilege](#access-and-privilege), under which the operating system
decides, for each program, which peripherals exist and which calls may be made.

### The `io` API

| Name               | Description                                                                                                                      | Arguments                            | Returns                       |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ | ----------------------------- |
| io.getPeripherals  | Returns the UUIDs of the attached peripherals this program may see, in no particular order.                                      | none                                 | uuids (table of strings)      |
| io.isCompatibility | Returns whether a peripheral comes from the ComputerCraft compatibility layer.                                                    | uuid (string)                        | compat (boolean)              |
| io.broadcastLocal  | Sends the values to every other computer on the same network cable graph, and returns nothing.                                    | ... (any)                            | none                          |
| io.getType         | Returns the type name of a peripheral, namespaced like a Minecraft id, such as `neetcomputers:access_point`.                      | uuid (string)                        | type (string)                 |
| io.queryTag        | Returns the UUIDs of the visible peripherals whose tag is the given one. The argument is trimmed first, and a blank tag is an error. | tag (string)                       | uuids (table of strings)      |
| io.setTag          | Assigns a tag to a peripheral. The tag is trimmed, and `nil` or a blank tag clears it.                                            | uuid (string), tag (string?)         | none                          |
| io.wrapPeripheral  | Returns a table with one function for each function the peripheral exposes, each bound to that peripheral.                        | uuid (string)                        | functions (table)             |
| io.getTag          | Returns the tag of a peripheral, or `nil` when it has none.                                                                       | uuid (string)                        | tag (string), or nil          |
| io.callFunction    | Calls one of a peripheral's functions directly and returns whatever it returns.                                                    | uuid (string), name (string), ... (any) | the function's return values |
| io.queryType       | Returns the UUIDs of the visible peripherals of the given type. A type without a `:` has `neetcomputers:` put in front of it.     | type (string)                        | uuids (table of strings)      |
| io.isViewed        | Returns whether a player is currently viewing this computer.                                                                       | none                                 | viewed (boolean)              |

A UUID is one of the strings `io.getPeripherals`, `io.queryTag` and `io.queryType` returned. Every function that
takes one checks it before it looks at any peripheral, so a string that is not a well-formed UUID fails with
`UUID invalidly formatted` no matter what else is true, and a well-formed one that no peripheral this program may
see has fails with `Peripheral not found`, as [Failures](#failures) lists.

`io.isViewed` is not about peripherals. It answers whether a player is currently viewing this computer's interface
- in stock NEET Computers, whether its GUI is open on their screen. It is the one function here about the room the
computer sits in, and a program that puts secrets on the screen uses it to decide whether to: a password prompt
drawn while `io.isViewed()` is `true` is read by whoever is watching, not only by the program. The operating system
answers the question for its own environment, but the answer stays about the player's eyes on this computer, not
about which program has a window.

---

### Failures

`io` keeps the NEET Computers failure convention and does not use the codes in [errors.md](../common/errors.md).
A failure is a plain Lua error carrying one of these messages, exactly as NEET Computers raises it:

| Message                     | Raised when                                                                                                                                                          |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `UUID invalidly formatted`  | the `uuid` argument is not a well-formed UUID string.                                                                                                                 |
| `Peripheral not found`      | no peripheral this program may see has the given UUID. `io.callFunction` with a name the peripheral does not expose fails the same way, and so does calling a function out of a `wrapPeripheral` table whose peripheral has since been detached or hidden. |
| `tag cant be blank`         | `io.queryTag` was given a tag that is empty or only whitespace.                                                                                                       |

The spelling and wording of the messages are NEET's and are not corrected here, because programs match on them.
`io.getPeripherals`, `io.queryType`, `io.isViewed` and `io.broadcastLocal` do not fail, and `io.queryTag` fails
only for the blank tag. A query that finds nothing returns an empty table, which is an ordinary answer.

---

### Access and privilege

In NEET Computers, every program on a computer sees the whole peripheral network and every call in this API is
allowed. An operating system that reports `ext.peripherals` MAY instead restrict, for each program, what that
program sees and what it may call. Two kinds of restriction exist, and an operating system may use either or both:

**Visibility.** The operating system MAY hide peripherals from a program. A peripheral that is hidden from a
program must be indistinguishable from one that does not exist. It does not appear in the results of
`io.getPeripherals`, `io.queryTag` or `io.queryType`, and every function that takes a UUID - `io.getType`,
`io.getTag`, `io.setTag`, `io.isCompatibility`, `io.wrapPeripheral` and `io.callFunction` - fails on it with
`Peripheral not found`, the same answer an unknown UUID gets. This is the rule the NEATO registry applies to
extensions an operating system does not report - nothing must appear to be half there - applied one level down, to
the peripheral network. A program can therefore trust what it finds: a peripheral it can see is a peripheral it
may ask things of, and a peripheral it cannot find is not for it.

**Denial.** The operating system MAY additionally refuse individual calls to a program - keeping `io.setTag` or
`io.callFunction` from unprivileged programs, for example. A refused call raises a Lua error whose message the
operating system chooses. Whether that message reads as a refusal or as anything else is the operating system's
choice too, since the message is its own.

An operating system that reports `ext.peripherals` and restricts nothing grants a program the whole API, which is
the plain NEET Computers behavior. Which programs get which restrictions is the operating system's decision alone.
This specification defines no way for a program to ask for a restriction to be lifted, and no tag, type or name of
peripheral that must always be hidden or always allowed.

---

### The local broadcast medium

`io.broadcastLocal(...)` hands the given values to every computer on the same network cable graph. The sender is
not among the receivers. The call returns nothing, does not wait for anyone, and does not say whether anybody was
listening: it is fire-and-forget, and a frame can be lost when a receiver's queue is full. What a frame may
contain, and what survives the trip, is defined once in [wire.md](../network/wire.md); this specification adds
nothing to it.

The NEATO Network protocols are built on this one primitive and on nothing else (see
[the NEATO Network README](../network/README.md)). A program whose operating system permits it can therefore speak
`dpp` or `spp` with peers on the same cable graph through `io.broadcastLocal` alone, without asking the operating
system to route anything for it.

A program that is permitted to receive broadcasts receives one as the raw NEET Computers event, unchanged and in
order, through the [`event`](core/event.md) API of `core`:

| Name           | Values    | Description                                                                                                  |
| -------------- | --------- | ------------------------------------------------------------------------------------------------------------- |
| networkMessage | ... (any) | A broadcast arrived over the network cables. The values are exactly those the sender passed to `io.broadcastLocal`. |

Which programs a copy of a broadcast reaches is part of the access model above and is the operating system's
decision, as with input events in `core`. An operating system may deliver raw broadcasts only to programs it
trusts: everything spoken on a cable graph reaches every permitted listener, in the clear, and no receiver is told
who spoke.

---

### Example usage

```lua
-- find the redstone controller by type and drive it
local uuid = io.queryType("redstone_controller")[1]
if uuid then
  io.callFunction(uuid, "setOutput", "north", 15)
end
```

```lua
-- everything tagged as the house printer, and what each really is
for _, uuid in ipairs(io.queryTag("printer")) do
  print(uuid, io.getType(uuid))
end

-- hand a peripheral a name of its own
io.setTag(io.queryType("access_point")[1], "uplink")
```

```lua
-- a tiny broadcast: sender on one computer
io.broadcastLocal("chime", 3)

-- receiver on every other computer on the cable graph
local name, what, n = event.pull("networkMessage")
```

---

### Footnotes

The model is the part that needs the testing: the pass-through functions are the ones NEET Computers already has.

A hidden peripheral is answered with NEET's own `Peripheral not found` rather than a distinct message such as
`peripheral hidden`, on purpose. A distinct message would let a program prove a peripheral exists by telling the
answers apart, which is exactly what hiding must not leak. For the same reason the UUID check comes first: a
malformed UUID fails with `UUID invalidly formatted` whether or not a hidden peripheral would have matched it.

Reception of broadcasts is defined as the raw NEET Computers event, unchanged, rather than left to the network
extensions, for two reasons. NEET Computers already delivers a broadcast as a plain named event to the receiving
computer, so this specification only names what happens anyway and invents no delivery mechanism of its own. And an
operating system that reports only `ext.peripherals` would otherwise hold a radio it may speak into and never hear
out of, which is the half-there the registry forbids. The network extensions keep their place above it: an
operating system that runs `ext.dpp` takes the `networkMessage` events for itself and hands programs `dpp` events
instead, which the access model permits and which is how those extensions are specified.

`io.isViewed` is missing from the function list in the issue this specification was written from, but both
implementations have it - the mod and the reference emulator - so it is specified. Version 1 likewise specifies
none of the other raw NEET Computers events of this area, such as the notices a computer or peripheral queues when
a peripheral comes and goes: an operating system that has them may deliver them, and a later version, or another
extension, may standardize them.
