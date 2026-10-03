# NEATO `Direct Payload Protocol` Specification

Written by UsUsStudios

Extension: `ext.dpp`

Version: 2

Requires: `core`

---

This specification defines a connectionless transport-layer protocol comparable to the real-life User Datagram Protocol,
called the Direct Payload Protocol, or DPP. It is a protocol without handshaking which in real networking would be
referred to as unreliable due to its lack of protection from data loss, and because the NEET Computers media can lose a
message when the receiving computer's event queue is full (see [Loss](spp-remote.md#loss)), it does not guarantee
delivery in practice either. A program that needs delivery to be guaranteed uses [SPP](spp.md).

An operating system that reports `ext.dpp` must provide the `dpp` API described at the end of this file, and must
handle DPP messages as described here.

A DPP message is sent as a table with the following entries:

```
{
    protocol: string = "dpp",
    target_port: integer,
    source_port: integer,
    payload: any
}
```

### protocol

The `protocol` field must be a string consisting of exactly `dpp`, for the receiving program to confirm that this is
a DPP message. (Currently this is not useful, as DPP is the only transport-layer protocol, but in the future other
protocols may be created, some of which may not even be NEATO-defined.)

### target_port

The `target_port` field must be an integer from 1 to 65535 that corresponds to the port number that this message is
directed at. The computer's operating system facility that handles networking events should only route the payload of
the packet to the program that is currently listening on this port, and must drop the message if there is none.

### source_port

The `source_port` field should either be an integer from 1 to 65535 that corresponds to the port that the sending
computer is listening to for replies to this packet, or `-1` if the sending computer is not listening for replies.
This information should be passed down to the program that receives the packet, along with the payload, so that the
program knows what destination to target should it want to formulate a reply.

### payload

The `payload` field can be whatever the application-layer protocol requires, as long as it can be sent over the
network. A payload is either a boolean, a number, a string, or a table whose keys and values are themselves such
values (tables may be nested, but must not refer to themselves). Functions, threads and userdata cannot be sent, and
metatables are not sent. The largest payload that can be sent is up to the operating system.

The payload can be a single value, or a table (either an array or a dictionary table), as long as it is defined as such
by the application-layer protocol. If the payload is a dictionary table, then it SHOULD have a `protocol` entry with
the protocol name/header for programs to check against, and if the payload is an array table, then its first entry
SHOULD be the protocol name/header, but neither of these are required.

### Delivery

DPP does not yet have any addressing of computers, because the link and internet layers have not been developed. Until
they are, a message reaches every computer that receives the underlying broadcast. Each operating system must deliver
it only to the program on that computer which is listening on `target_port`, and it is against NEATO specification to
allow a program to read, access or index the payload of a message that is not directed at a port it is listening on.
An operating system must silently drop received tables that are not valid DPP messages.

---

### The `dpp` API

| Name         | Description                                                                                            | Arguments                                            | Returns                        |
| ------------ | ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- | ------------------------------ |
| dpp.listen   | Starts receiving messages for a port in this program. Only one program can listen on a port at a time. | port (int)                                           | true, or nil, code and message |
| dpp.unlisten | Stops receiving messages for a port. Ports are also released when the program ends.                    | port (int)                                           | true, or nil, code and message |
| dpp.send     | Sends a message. `source_port` defaults to `-1`. Fails if the ports or the payload are not valid.      | target_port (int), payload (any), source_port (int?) | true, or nil, code and message |

A failing function returns `nil`, then the code, then a message, as defined in
[errors.md](../common/errors.md). The required codes are:

| Function     | Condition                                              | Code           |
| ------------ | ------------------------------------------------------ | -------------- |
| dpp.listen   | the port is already in use                             | `EADDRINUSE`   |
| dpp.listen   | the port is not from 1 to 65535                        | `EINVAL`       |
| dpp.listen   | the program is not allowed to listen on the port       | `EACCES`       |
| dpp.listen   | listening cannot be started for another reason         | `EIO`          |
| dpp.unlisten | the port is not from 1 to 65535                        | `EINVAL`       |
| dpp.unlisten | the program is not allowed to stop listening           | `EACCES`       |
| dpp.send     | `target_port` is not from 1 to 65535                   | `EINVAL`       |
| dpp.send     | `source_port` is neither `-1` nor from 1 to 65535      | `EINVAL`       |
| dpp.send     | the payload is not a value that can be sent            | `EINVAL`       |
| dpp.send     | the payload is larger than the operating system allows | `EMSGSIZE`     |
| dpp.send     | the program is not allowed to send                     | `EACCES`       |
| dpp.send     | the destination cannot be reached                      | `EHOSTUNREACH` |
| dpp.send     | the message cannot be sent for another reason          | `EIO`          |

When a message arrives for a port that a program is listening on, that program receives the event

`dpp` (payload, source_port, target_port)

through [`event`](../api/event.md).

---

### Example usage

```lua
dpp.listen(80)
local _, payload, from = event.pull("dpp")
if from ~= -1 then
  dpp.send(from, {"reply", payload[2]}, 80)
end
```
