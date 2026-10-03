# NEATO `Sequenced Payload Protocol over the network` Specification

Extension: `ext.sppRemote`

Version: 1

Requires: `ext.spp`

Status: draft. `ext.sppRemote` is a reserved name (see the [main README](../README.md)). An operating system must
not report it as supported until this notice is removed. Nothing in `ext.spp` depends on it.

---

This specification describes how [SPP](spp.md) connections are made between programs on different computers. It
adds nothing to the `spp` API. It makes `scope = "network"` work on `spp.listen` and `spp.connect`, which without this
extension fail with `ENOTSUP`. Programs written for `ext.spp` need no changes.

SPP frames can travel in two ways. Where the OS has [`ext.rpp`](rpp.md), every frame is carried in an RPP packet, so
that it can be addressed to one host and routed across several (see [With RPP](#with-rpp)). Where it has not, frames are
sent directly on the medium and reach every host that can hear them (see [Without RPP](#without-rpp)).

---

### When this is worth building

Both media below can lose messages (see [Loss](#loss)). So a
reliable remote transport is useful, and not only for the connection semantics that DPP cannot give on any medium:

- One session with one other program, in which each side finds out when the
  other disappears (`EPIPE`, `ECONNRESET`). DPP messages reach every computer that hears them and have no notion of a peer.
- Event queues are small and drop their oldest entries when they overflow, so a burst of traffic can
  silently lose messages. SPP retransmits what was lost.
- A sender that must not overrun a slow receiver, with no application-level acknowledgement.

An application that only sends occasional, self-contained notifications between computers should still use DPP.

---

### Media

NEET Computers has two ways for one computer to put a message in front of another. An operating system implements
SPP over one of them or both. They behave differently.

Both carry a frame, a short list of primitive values, and nothing else. [wire.md](wire.md) says exactly which values
survive the trip: strings must only contain bytes from `0x01` to `0x7F`, integers must fit in 32 bits, a frame has at most 16
values, and tables are not possible at all.

`io.broadcastLocal(...)` hands the values to every network receiver that is on the same *network cable*
graph as the sending computer. The sender itself is not among the receivers. The set of receivers is refreshed about
once a second, so a computer that was just connected or disconnected can be missed or still receive for a moment. The
receiver sees the raw event `networkMessage` with exactly the values that were sent. A computer that is switched off
receives nothing, and the message is gone.

The Access Point peripheral (`neetcomputers:access_point`, attached to a computer by a peripheral
cable) has a function `broadcast(...)` with the same restriction on its arguments. The message goes to every *other*
access point that is in the same dimension and within `range` blocks of the sender. Only the sender's range matters.
The default and the largest range is 500, and `setRange` can lower it. A receiving access point puts a raw event named
after its peripheral type onto the event queue of every computer attached to it, with the values
`(peripheral uuid, "received", distance, ...)`, where `distance` is the number of blocks between the two access
points and `...` is exactly what was sent. The delivery happens when the sending access point next updates, so it is up
to about one game tick (50 ms) late. An access point that is not loaded does not receive.

Neither medium tells the receiver who sent a message. The wireless medium adds the distance, which is not an
identity.

#### Loss

Every event queue of a computer holds at most 75 events per label, and when a new event arrives at a full queue the
oldest event is thrown away. Wired messages arrive in the `NETWORK` queue, which is shared with all internet
(HTTP and WebSocket) events, and wireless messages arrive in the `PERIPHERAL` queue, which is shared with every other
peripheral's events. So both media lose messages when more than 75 events pile up before the operating system reads
them, and a flood on one connection, or an unrelated websocket, can push out the segments of another.

Consequences for SPP:

- An operating system must read the raw queue often and move frames out of it at once, into SPP's own buffers.
- A sender must not have more than 8 unacknowledged messages in flight over one medium (all connections together,
  where it can), which leaves room for acknowledgements and other traffic.
- Recovery from loss is the retransmission described below, not something a program has to deal with.

---

### Segments

An SPP segment is a frame. Every frame follows [wire.md](wire.md#frames). The segment kinds and their values are:

| Position | `syn`             | `synack`     | `data`                    | `ack`        | `fin`     | `rst`     |
| -------- | ----------------- | ------------ | ------------------------- | ------------ | --------- | --------- |
| 1        | `"spp"`           | `"spp"`      | `"spp"`                   | `"spp"`      | `"spp"`   | `"spp"`   |
| 2        | `"syn"`           | `"synack"`   | `"data"`                  | `"ack"`      | `"fin"`   | `"rst"`   |
| 3        | source id         | source id    | source id                 | source id    | source id | source id |
| 4        | `""`              | target id    | target id                 | target id    | target id | target id |
| 5        | target port (int) | window (int) | seq (int)                 | ack (int)    | seq (int) |           |
| 6        | source port (int) |              | payload (string, encoded) | window (int) |           |           |
| 7        | window (int)      |              |                           |              |           |           |

- source id and target id identify the two ends of a connection. An id is 16 lowercase hexadecimal digits
  made from `crypto.SecureRNG`, so it is safe to put in a frame. An end must not use an id that belongs to another of
  its live connections.
- target port and source port are the SPP port being connected to and the port the connecting side shows as
  `peerPort`. They are only sent in `syn`.
- seq counts messages, not bytes, starting at 1 in each direction. ack is the highest `seq` that the sender
  of the `ack` has delivered in order, and 0 if none. window is the number of free places in the sender's inbox.
  A counter has to stay within 2^31 - 1 because a frame cannot hold more, so a connection that reaches 2^30 messages in
  one direction is reset (`ECONNRESET`).
- payload is the message in the [payload encoding](wire.md#payload-encoding): the raw encoding of the value, made
  into Base64. A payload that is longer than 3072 bytes raw cannot be sent over the network (`EMSGSIZE`), because it
  would not fit in a frame. `conn.maxPayload` of a network connection says so (see [spp.md](spp.md)), and is the
  smaller of this and `maxPayload` from `spp.getLimits`. SPP does not fragment.

A receiver validates every segment completely (the number of values, the type of each, ranges, the length and characters
of ids) and drops segments that are not valid without answering. A valid segment other than `syn` whose target id
matches no connection (and no half-open connection) on the receiving computer is answered with `rst`, except when it is
itself an `rst`. A `syn` has no target id and is answered as described under [Connecting](#connecting).

Neither medium returns a computer's own frames to it, so SPP does not need to filter them. Connections between programs
on the same computer are `ext.spp`'s business and never use a medium.

---

### Without RPP

An OS that does not have `ext.rpp` sends each segment as one frame on the medium of the connection, and accepts the
segments that arrive from it. A connection uses exactly one medium, the one its first `synack` arrived on. Segments of a
connection that arrive from the other medium are ignored. A server sends its `synack`, and every later segment, on the
medium that the `syn` arrived on.

A `syn` reaches every computer the medium reaches, so `opts.host` of `spp.connect` must be `nil` (anything else
fails with `ENOTSUP`), which means "whichever computer answers first". The `peer` of both ends is `{ origin =
"remote", medium = "wired" | "wireless", distance = number? }`. `distance` is only present on the wireless medium and is
the value from the last frame received. It tells how far away the peer's access point is, and is not an identity.

On the wireless medium only the sender's range counts. A `syn` can arrive while the `synack` cannot get back,
if the server's access point has a shorter range than the distance between them. The client sees this as `ETIMEDOUT`,
the same as a missing server. An operating system that implements the wireless medium should leave its access points at
the largest range, so that links work in both directions.

---

### With RPP

An OS that has `ext.rpp` carries every segment as the transport frame of an [RPP packet](rpp.md#packets). It never
sends a bare segment on a medium, and ignores bare segments that arrive.

- `opts.host` of `spp.connect` is the address of a host. The `syn` is sent to it and routed there. If `opts.host` is
  `nil` or the broadcast address, the `syn` goes to the broadcast address, which reaches every host on the same medium
  segment and is never passed on by a router. That is "whichever host on this network answers first".
- A listener with scope `"network"` answers a `syn` with a `synack` sent to the source address of the packet that
  carried it. Each end records the source address of the first packet it received from the other end (the `syn` for the
  server, the accepted `synack` for the client) as the address of its peer. Every later segment of the connection goes
  to that address, and a segment of the connection that arrives from any other source address is dropped. That keeps
  `peer.host` true for the whole connection, as [spp.md](spp.md#peer-identity) requires, and means that a forged packet
  cannot redirect a connection.
- If there is no route to `opts.host`, `connect` fails with `EHOSTUNREACH` at once.
- A connection is not tied to a medium. Segments follow the routes, which may change during a connection. Loss and
  delay from a change are handled by retransmission like any other.
- The `peer` of both ends is `{ origin = "remote", host = address }`. The host is the source address in the packets, and
  like every source address it has not been authenticated.
- The segment of a connection is at most 7 values, so it fits in an RPP packet with room to spare. The payload limit is
  the same as without RPP.

---

### Connecting

1. The client picks an id `A` and a free local port, and sends `syn` (without RPP on the medium or media it tries, with
   RPP as described above).
2. Every computer that has an SPP listener on `target port` with scope `"network"` and room in its backlog answers with
   `synack` (`source id` = `B`, `target id` = `A`) and holds a half-open connection for 10 seconds after the latest
   `syn` it answered. A computer with no such listener sends nothing, not even `rst`, so it does not reveal which ports
   are in use. A repeated `syn` with the same source id is answered with the same `synack`.
3. The client takes the first `synack`, sends `ack` (`ack` = 0) and `connect` returns. It answers every later `synack`
   for `A` with `rst`. The connection exists on the server once it receives the `ack`, and is then added to the
   listener's backlog. An `ack` is not retransmitted, so a `data` segment for a half-open connection counts as the `ack`
   as well: it completes the connection and is then handled as data.
4. If no `synack` arrives, the client repeats the `syn` up to 4 more times, one second apart (two seconds with RPP, as
   for retransmission, see [Sending data](#sending-data)), and then fails with `ETIMEDOUT`. If `opts.timeout` of
   `spp.connect` passes earlier, it fails with `ETIMEDOUT` at that point. A longer `opts.timeout` does not make the client
   keep trying after the last repeat. If an OS without RPP implements both media, the client sends the `syn` on both,
   and the first `synack` decides the medium.

---

### Sending data

Each direction counts its own messages from 1. A sender sends `data` segments in order and keeps each of them until it
is acknowledged. A receiver delivers a segment only if its `seq` is the next one it expects and its inbox has room. A
segment it has already delivered is not delivered again, and a segment further ahead, or one that arrives while the
inbox is full, is discarded. Either way it answers with an `ack` for the highest message delivered in order (go-back-N),
so a receiver never buffers anything out of order.

- Every `ack` carries `window`. The sender never has more unacknowledged messages than that, never more than 8 on one
  connection (see [Loss](#loss) for the limit over all connections together), and an operating system may set its own
  limit lower, down to 1.
- When `window` is 0, the sender sends its oldest unsent message once a second as a probe. A probe that is answered does
  not count as a retransmission, so a slow reader does not cause a `reset`.
- If an acknowledgement does not arrive within the retransmission timeout, the sender retransmits all unacknowledged
  messages, oldest first. The timeout starts at 1 second and doubles each time, up to 8 seconds. After 5 rounds of
  retransmission in a row without any progress the connection is `reset` and the sender sends `rst`. With RPP the
  timeout starts at 2 seconds instead, because a packet can wait for a few ticks at every router.
- Because delivery can be up to a tick late and an operating system may only look at its queue now and then, a timeout
  shorter than 1 second would cause needless retransmission.

`conn:send` waits while the window to the peer is exhausted, as in local SPP.

---

### Closing

`conn:close()` sends `fin` with the next sequence number once all earlier messages have been sent. It is delivered and
acknowledged like data. After `conn:close()` returns, the operating system keeps the state of the connection, and goes
on retransmitting, until the `fin` has been acknowledged or the retransmissions have run out, even if the program has
ended. That is what makes messages that were already sent still arrive, as [spp.md](spp.md#closing) promises. When a
receiver has delivered a `fin` in order, `recv` returns `EPIPE` once the inbox is empty. `rst` ends a connection at
once in both ends with `ECONNRESET`, and is sent when a program is killed, a limit is hit, retransmissions have run out, or
the listener of a connection that has not been accepted yet is closed (a half-open connection of that listener is simply
dropped). A `rst` that is lost is not retransmitted, and the other end finds out when its own retransmissions run out.

---

### Security considerations

Both media reach computers that are not part of the connection, and neither names the sender.

- Nothing is secret. Every computer on the cable graph, and every access point in range, can read every frame.
  Programs that need confidentiality use the `crypto` API inside their payloads.
- Nothing is authenticated. Ids are only unguessable to computers that did not hear them, so a nearby computer can
  inject segments or reset a connection. `peer.origin = "remote"` means "not on this computer", not "trusted". With RPP,
  `peer.host` is only a claim as well (see [RPP](rpp.md#security-considerations)).
- Flooding. A computer that sends many `syn` segments can fill the half-open table and the backlog, and can fill a
  victim's event queue, which holds only 75 events (see [Loss](#loss)). An operating system should limit half-open
  connections both overall and per listener, drop the oldest first, and drop frames that cannot be valid cheaply and
  early.
