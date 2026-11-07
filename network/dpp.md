# NEATO `Direct Payload Protocol` Specification

Written by UsUsStudios

Extension: `ext.dpp`

Version: 3

Requires: `core`

---

This specification defines a connectionless transport-layer protocol comparable to the real-life User Datagram Protocol,
called the Direct Payload Protocol, or DPP. It is a protocol without handshaking which in real networking would be
referred to as unreliable due to its lack of protection from data loss, and because the NEET Computers media can lose a
message when the receiving computer's event queue is full (see [Loss](spp-remote.md#loss)), it does not guarantee
delivery in practice either. A program that needs delivery to be guaranteed uses [SPP](spp.md).

An operating system that reports `ext.dpp` must provide the `dpp` API described at the end of this file, and must
handle DPP messages as described here.

A DPP payload is encoded as described in [payload encoding](wire.md#payload-encoding), with the same rules for a legal
payload. A DPP message is sent with the following header to another computer through [RPP](rpp.md), addressed to the
address that the program requested.

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
directed at. The receiving computer's operating system facility that handles networking events should only route the
payload of the packet to the program that is currently listening on this port, and must drop the message if there is none.

### source_port

The `source_port` field should either be an integer from 1 to 65535 that corresponds to the port that the sending
computer is listening to for replies to this packet, or `-1` if the sending computer is not listening for replies.
This information should be passed down to the program that receives the packet, along with the payload, so that the
program knows what destination to target should it want to formulate a reply.

### payload

The `payload` field can be any valid payload according to [the payload encoding rules](wire.md#payload-encoding).
The program that sent the DPP message passes that payload as the argument to `dpp.send`, which raises a few errors if
it is invalid. If the payload is a dictionary table, then it SHOULD have a `protocol` entry with the application-layer
protocol name/header for programs to check against, and if the payload is an array table, then its first entry SHOULD be
the protocol name/header, but neither of these are required.

---

### The `dpp` API

| Name         | Description                                                                                            | Arguments                                                                      | Returns                        |
| ------------ | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ | ------------------------------ |
| dpp.listen   | Starts receiving messages for a port in this program. Only one program can listen on a port at a time. | port (int)                                                                     | true, or nil, code and message |
| dpp.unlisten | Stops receiving messages for a port. Ports are also released when the program ends.                    | port (int)                                                                     | true, or nil, code and message |
| dpp.send     | Sends a message. `source_port` defaults to `-1`. Fails if the ports or the payload are not valid.      | target_addresss (string), target_port (int), payload (any), source_port (int?) | true, or nil, code and message |

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

`dpp` (payload, source_address, source_port)

through [`event`](../api/core/event.md).

---

### Example usage

```lua
dpp.listen(80)
local _, payload, source_address, source_port = event.pull("dpp")
if from ~= -1 then
  dpp.send(source_address, source_port, {"reply", payload[2]}, 80)
end
```
