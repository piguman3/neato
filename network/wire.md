# NEATO Network Wire Format Specification

Extension: none. These are shared definitions that [`ext.rpp`](rpp.md) and [`ext.sppRemote`](spp-remote.md) are
written against, and that [`ext.dpp`](dpp.md) needs to be revised to use.

Version: 1

---

Every protocol in NEATO Network that crosses between computers eventually has to hand a list of values to the one
thing NEET Computers gives programs for that: `io.broadcastLocal(...)` or the `broadcast(...)` function of an Access
Point (see [Media](spp-remote.md#media)). Those functions are strict about what they take and lossy in what they give
back. This file states exactly what survives, so that no protocol has to rediscover it, and defines the one encoding
that the protocols use to put richer values into a frame.

---

### Frames

A frame is the list of values given to one call of `broadcastLocal` or `broadcast`. A frame, and every value in it,
must follow all of these rules. A receiver drops a frame that breaks one, without answering.

1. A frame holds at most 16 values, and at least 1. This stays well under what Lua guarantees for a function that
   returns several values (20), because the bridge does not grow the stack itself. If the OS gets the event as a list
   instead of a value list, the limit costs nothing.
2. A value is a boolean, an integer, a float or a string. `nil` is never used, because a trailing `nil` in a list of
   arguments can be lost and an empty string says the same thing.
3. An integer is in the range `-2147483648` to `2147483647`. A protocol that needs a counter larger than that must
   treat running out of numbers as an error and end the connection.
4. A float is any double that is not `nan`. A protocol that needs an integer must send an integer, and one that needs
   a fraction must send a float, because the two types are passed on as they are and are not converted into each other.
5. A string only contains bytes from `0x01` to `0x7F`. Anything else must be encoded first (see
   [Payload encoding](#payload-encoding)). The empty string is fine.
6. All the strings in a frame together are at most 4608 characters, and one string at most 4096. The mod does not
   limit this. It is a limit that this specification sets, so that one frame cannot use a large part of a computer's
   memory when 75 of them are waiting in a queue.
7. The first value of every frame is a string that names the protocol (`"rpp"`, `"spp"`, `"dpp"`, `"rcp"`). A receiver that
   does not know it ignores the frame.

The sender checks these rules too. A frame that breaks them is not sent, and the call that wanted to send it fails with
`EINVAL` (or `EMSGSIZE`, if only a size rule is broken). These are the shared NEATO codes of
[errors.md](../common/errors.md).

---

### Payload encoding

A payload is what a program hands to `conn:send` or `dpp.send`: a boolean, a number, a string, or a table of such
values that does not refer to itself. Frames cannot hold tables or arbitrary strings, so a payload is turned into one
string, and sent in a frame as a single string value.

The encoding has two steps. First the payload is walked depth-first and written as bytes, the *raw encoding*.
Second, the raw encoding is turned into Base64 with the standard alphabet and `=` padding (`crypto.Base64`), so that it
is made of bytes from `0x01` to `0x7F` only. The receiver reverses both steps.

Raw encoding

| Value           | Encoding                                                                                                                                                                       |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `true`, `false` | `T`, `F`                                                                                                                                                                       |
| integer         | `i`, decimal digits with a leading `-` if negative, then `;`. A Lua integer outside the 32-bit range is allowed here, because it is not a frame value, but must fit in 64 bits |
| float           | `f`, then the `%.17g` text of the number (or `inf`, `-inf`), then `;`. `nan` is not a valid payload                                                                            |
| string          | `s`, the length in bytes in decimal, `:`, then the bytes. Any byte is allowed                                                                                                  |
| table           | `t`, the number of entries in decimal, `;`, then for every entry the key and then the value, each encoded in turn                                                              |

A payload is invalid, and `send` fails with `EINVAL`, if it nests tables deeper than 32 levels, if it contains itself,
if it holds a function, thread or userdata, or if any key or value is `nan`. Metatables are not sent. The receiver builds
plain tables. A table that is referred to twice is sent twice, as two separate copies, which is the same behaviour that
[local SPP](spp.md#messages) has for copies.

The size limit of a payload is a limit on the raw encoding: 3072 bytes, which after Base64 is 4096 characters, the
longest string a frame may hold. A payload that is larger fails with `EMSGSIZE`. Protocols that need to send more than that
must split it themselves, because none of them fragments.

The decoder must reject anything that is not exactly one valid value with nothing after it. It must check
every length and entry count against the bytes that are left before it allocates anything, so that a short frame cannot
ask for a huge table, and it must stop at 32 levels of nesting. A frame whose payload does not decode is dropped like any
other invalid frame.
