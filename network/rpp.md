# NEATO `Routed Packet Protocol` Specification

Author: XesHuman

Extension: `ext.rpp`

Version: 1

Requires: `core`

Status: draft. An OS must not report `ext.rpp` until this notice is removed.

---

This specification defines the internet layer of NEATO Network: how a packet gets from one computer to another
computer that is not necessarily next to it. It gives every computer an address, wraps every transport frame in a
small packet header, and defines how computers that can reach more than one neighbour (routers) learn where
to send packets, and pass them on.

RPP is called the Routed Packet Protocol because that is the part of it that was missing. NEET Computers has no
addresses, so every message sent with `io.broadcastLocal` or an Access Point reaches everyone the medium reaches, and
nobody can say who sent it (see [Media](spp-remote.md#media)). RPP builds addressing on top of that, and nothing else
is needed from the medium.

RPP is meant to be small. NEET networks are small (hundreds of computers, tens of hops), the medium throws away messages when
a computer's queue of 75 events fills up, and a routing protocol that talks a lot would make that worse. So there are no
prefixes or subnets, no name service, and only one kind of routing: periodic distance vectors with a hop limit.

An OS that reports `ext.rpp` must provide the `rpp` API described in this file. Programs rarely use it directly. The
transport protocols use RPP to address another computer, and programs reach it through them
(`spp.connect(port, { scope = "network", host = address })`).

---

### Terms

| Term      | Meaning                                                                                                     |
| --------- | ----------------------------------------------------------------------------------------------------------- |
| host      | Any computer that takes part in RPP. Every host has one address.                                            |
| router    | A host that also passes packets on that are not for itself. Every other host is an end host.                |
| interface | One medium that a host is connected to: the network cable graph (`wired`), or an Access Point (`wireless`). |
| neighbour | A host that this host can hear directly on an interface, without a router in between.                       |
| link      | The path between a host and one neighbour.                                                                  |
| frame     | A list of values as defined in [wire.md](wire.md#frames). An RPP packet is a frame.                         |

---

### Addresses

An address is a string of exactly 12 lowercase hexadecimal digits (`0-9`, `a-f`), for example `"7c04e91ab3d2"`. That is
48 bits, like a hardware address.

Two addresses have a special meaning:

- `"000000000000"` is never given to a host.
- `"ffffffffffff"` is the broadcast address. A packet sent to it is received by every host on the same medium
  that can hear the sender, and is never passed on by a router. It is how a host asks a question of everyone nearby
  without knowing who is there.

- Choosing an address. A host generates its address once, from `crypto.SecureRNG`, rejecting the two special values,
and stores it where the OS stores its settings, so that the address stays the same after a restart. The address is
random, because NEATO gives a computer no identity of its own to derive one from. With 48 random bits, two hosts out of
1000 have the same address with a probability of about 2 in a million.

- Duplicate addresses. Copying a disk to another computer copies the address too, so duplicates have to be detected. A
medium never delivers a frame back to the computer that sent it. So if a host receives a `hello` (see below) whose
source is *its own address*, another host has that address. `hello` is never passed on, so this cannot be the host's own
packet coming back around a loop. The host then waits a random 0 to 5 seconds, chooses a new address, stores it, and
sends a `hello` at once. Connections that were using the old address will not survive it. Both hosts see the other's
`hello` and both change address, which is what makes this work without anyone being in charge.

---

### Interfaces

A host has one interface for each medium it can use, at most one of each in this version:

| Interface    | Medium                                                                         | Sends with                           |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------ |
| `"wired"`    | `io.broadcastLocal`: all hosts on the same network cable graph                 | `io.broadcastLocal(...)`             |
| `"wireless"` | an Access Point: all Access Points in range (see [Media](spp-remote.md#media)) | `broadcast(...)` of the Access Point |

An OS implements RPP on one interface or both. An OS with both can be a router between them, which is how a wired
segment is connected to a wireless one. A host that touches two separate cable networks can be a router between them
too, although it has only one interface: its broadcasts reach both cables, but it only passes on what it hears from
either of them if it is a router (see `rpp.setRouter`).

- What the medium gives, and what it does not. These are the properties that RPP is built around:

- A frame goes to everyone in reach. There is no addressing at the link. A host looks at the header and ignores
  everything that is not for it.
- The receiver is not told who sent it, so every packet carries its own source address. That address is not
  checked by anything. Anybody can write anything in it.
- A frame is not delivered to the host that sent it.
- A frame can be lost, if the receiver's queue is full (see [Loss](spp-remote.md#loss)). It can also be late
  by up to a tick, and a wired host that was just connected can be missed for about a second.
- On `wireless`, only the sender's range counts, so a link can work in one direction and not in the other. The
  receiver learns the distance in blocks (see `distance` below), which is useful as a hint and is not an identity.

---

### Packets

An RPP packet is one frame:

```
("rpp", destination, source, ttl, next, <transport frame>...)
```

| Position | Name        | Type    | Meaning                                                                                                                      |
| -------- | ----------- | ------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 1        | protocol    | string  | Always `"rpp"`.                                                                                                              |
| 2        | destination | string  | Address of the host the packet is for, or the broadcast address.                                                             |
| 3        | source      | string  | Address of the host that made the packet. A router leaves it as it is.                                                       |
| 4        | ttl         | integer | Hops the packet may still make, 1 to 256. A router lowers it by one.                                                         |
| 5        | next        | string  | Address of the neighbour that must handle the packet next, or the broadcast address.                                         |
| 6 and on | transport   | values  | One complete frame of a transport protocol. Its first value, at position 6, names the protocol: `"dpp"`, `"spp"` or `"rcp"`. |

The transport frame has at most 11 values, because a frame has at most 16 and the header takes 5. All the rules in
[wire.md](wire.md#frames) hold for the packet as a whole, so, for example, the strings of the whole packet together
are at most 4608 characters.

`destination` and `next` differ when a packet is being passed through a router. In the first hop of a packet to a
neighbour they are equal.

A packet is valid only if the first value is `"rpp"`, `destination`, `source` and `next` are addresses (and `source`
is not the broadcast address), `next` is the broadcast address exactly when `destination` is, `ttl` is an integer from 1
to 256, there is a transport frame, and the frame follows [wire.md](wire.md#frames). A host drops packets that are not
valid without answering them.

---

### Handling a packet

When a host receives a valid packet from an interface:

1. If `source` is the host's own address and the transport frame is not a `hello`: drop it. (A `hello` with the
   host's own address is a [duplicate address](#addresses).)
2. If `next` is neither the host's own address nor the broadcast address: drop it. It was overheard on the way to
   somebody else.
3. If `destination` is the host's own address, or both `destination` and `next` are the broadcast address: deliver
   the transport frame to the protocol that its first value names (`rcp` is handled by RPP itself). Frames of a
   protocol that the OS does not know are dropped.
4. Otherwise the packet is for another host and `next` is this host. If the host is not a router, or `ttl` is 1, or
   the destination is the broadcast address, drop it. If it is a router, look the destination up in the
   [routing table](#routes). If there is no route, drop it. Else send it again on the route's interface with `ttl`
   lowered by 1 and `next` set to the route's next hop.

Dropping a packet is silent. A router does not tell the sender that there was no route, and nobody learns that a
`ttl` ran out. That is deliberate: the transports have their own timeouts, and it keeps a failing network from filling
every queue with error messages. A program can see the cause with `rpp.ping` and `rpp.getRoutes`.

A host that sends a packet to another host looks the destination up. If it is a bidirectional neighbour,
`next` is the destination. If there is a route, `next` is the route's next hop. If neither, the send fails with
`EHOSTUNREACH` at once. A packet to the broadcast address is sent with `next` set to the broadcast address, on every
interface. `ttl` starts at 256, except for control frames to the broadcast address, which are always sent with 1.

A packet to the host's own address is never sent. The routing table does not contain the host's own address, so the send
fails with `EHOSTUNREACH`. Connections between programs on the same computer are `ext.spp` with scope `"local"` and never
use RPP, and `spp.connect` with scope `"network"` never looks on this computer (see [spp.md](spp.md#scope)).

---

### Control frames

RPP uses its own frames, with the protocol name `"rcp"` (the Routing Control Protocol), for neighbours, routes and
echo. They are carried in RPP packets like any other transport frame.

| Frame                           | Values                                       | Meaning                                                                                                                                                                                                                      |
| ------------------------------- | -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `("rcp", "hello", role, heard)` | role: `"h"` or `"r"`<br>heard: string        | A host announces itself. `role` is `"r"` for a router and `"h"` for an end host. `heard` is a comma separated list of the addresses that the host has heard on this interface in the last 120 seconds, at most 300, or `""`. |
| `("rcp", "vector", entries)`    | string                                       | A router tells its neighbours what it can reach. Entries are comma separated, and each is `address:metric:via` (see [Routes](#routes)), at most 136 entries. A longer table is sent in several frames.                       |
| `("rcp", "echo", id)`           | id: string, 1 to 16 characters from `0-9a-f` | Asks the destination to answer.                                                                                                                                                                                              |
| `("rcp", "echoreply", id)`      | the same string                              | The answer to an echo, sent to the address the echo came from.                                                                                                                                                               |

`hello` and `vector` are always sent to the broadcast address with `ttl` 1, and a router never passes them on.
`echo` and `echoreply` are ordinary routed packets.

---

### Neighbours

A host learns about its neighbours from their `hello` packets. On every interface it keeps, for each address it has heard,
the time it last heard it, its `role`, whether it listed this host in `heard`, and (on `wireless`) the `distance` of
the last frame.

- A host sends a `hello` when it starts, after a random 0 to 2 seconds, and then every 30 seconds, each
  time with a random delay of up to 25% either way so that hosts do not all talk at the same moment. When it hears a
  host it has not heard before, it sends an extra `hello` after a random 0 to 1 second, so that a new host does not have to
  wait for the next one.
- A neighbour that has not been heard for 120 seconds is forgotten, and so are the routes through it.
- On `wireless` a neighbour is bidirectional, and so usable as a next hop, only if its latest `hello` lists the
  host's own address. That is how a link that works in one direction is kept out of the routing table. On `wired` a link
  always works in both directions, so a neighbour that has been heard is bidirectional at once.
- A `hello` carries the whole `heard` list as one string, and a frame limits a string to 4096 characters, so at most
  about 300 addresses fit. A host with more neighbours cannot list them all, and more than about 300 hosts in one
  wireless range is not supported by version 1. The practical limit is lower, because a queue of 75 events drops
  messages when a dense range is busy, so a large range should be split by routers into smaller segments.

---

### Routes

A host keeps a table of destinations it can reach: `address`, `next` (the neighbour to hand the packet to), the
interface, `metric` (the number of hops, 1 to 255) and when it was last refreshed.

- Every bidirectional neighbour is a route of metric 1 with itself as the next hop. The host does not need a router to
  reach it.
- Only routers send `vector` frames. They send one every 30 seconds (with the same random delay as `hello`), and
  also, at most once every 5 seconds on each interface, when a route has changed. A vector lists every route in the
  table, each as `address:metric:via`, where `via` is the next hop that the router uses for that destination. It also
  lists, with metric 256, every route that the router has removed since its previous vector. A router's own neighbours
  are listed with `via` equal to the neighbour itself.
- A host that hears a `vector` from a bidirectional neighbour N considers each entry `address:metric:via`:
  - It skips the entry if `address` is its own, or if `via` is its own address. The second rule keeps two hosts from
    sending traffic back and forth to each other for a destination that each thinks the other reaches. This is the
    version of split horizon that works on a medium where everyone hears everyone.
  - The new metric is `metric + 1`. If it is 256 or more, the destination is unreachable through N. If the host's route
    to that destination uses N as next hop, the host removes it, and it is announced in the next vector with metric 256.
  - Otherwise, the route is installed if the host has none, or if the new metric is lower than the old one, or if N is
    already the route's next hop (in which case it is only refreshed, and updated if the metric has changed). In every
    case the route is marked as refreshed now.
- A route that has not been refreshed for 180 seconds is removed, and announced with metric 256. A route is also
  removed when its next hop is forgotten.
- Routes that tie are decided in favour of the one that exists already.

The largest metric is 255, so a network is at most 255 hops across. When a loop is made by something larger than two
hosts the routes count up to 256 and are then thrown away, the usual weakness of distance vectors. With this ceiling a
loop can take many vector rounds to count to infinity, so the 120 and 180 second timeouts, not the metric, are what free
a broken route, and a network should still be kept small.
A simulation of four routers in a chain, with one link cut, had every stale route gone after about three minutes: the 120
seconds until the neighbour was forgotten, plus a few vector rounds to carry the withdrawal along. It also showed
that a one-way wireless link is kept out of every table by the `heard` check.

A default route and summaries are not part of this version. Every host a router can reach has an entry of its own,
which is why a vector has to be sent in pieces when there are more than 136 hosts.

---

### Timers

| Name                       | Value                              |
| -------------------------- | ---------------------------------- |
| Hello interval             | 30 s, +- 25%                       |
| First hello after start    | 0 to 2 s                           |
| Extra hello for a new host | 0 to 1 s                           |
| Neighbour timeout          | 120 s                              |
| Vector interval            | 30 s, +- 25%                       |
| Triggered vector           | at most once per 5 s per interface |
| Route timeout              | 180 s                              |
| Address change delay       | 0 to 5 s                           |
| Largest metric / ttl       | 255 / 256                          |

---

### Transports over RPP

- [SPP](spp.md). With `ext.rpp`, an OS carries the frames of `ext.sppRemote` as the transport frame of RPP packets,
  and `spp.connect` accepts `opts.host` (see [spp.md](spp.md#the-spp-api)). The `syn` goes to `opts.host`, or, if there
  is none, to the broadcast address, which is how a program connects to "whichever host answers first". The `synack`
  goes to the source address of the `syn`, and every later frame to the address that each end recorded then (see [With
  RPP](spp-remote.md#with-rpp)). A connection no longer belongs to one medium: it follows the routes. The `peer` of a
  connection is `{ origin = "remote", host = address }`, where the host is the source address of its packets, and like
  every source address it is not authenticated.
- [DPP](dpp.md). `dpp.md` defines a message as a table, which a frame cannot hold (see [wire.md](wire.md)), so it
  has to be revised before it can be carried at all. It should then be carried the same way as SPP, as the transport
  frame of an RPP packet with the protocol name `"dpp"`, and `dpp.send` should get a destination host. Until then DPP is
  outside RPP.
- A host that implements `ext.rpp` always wraps transport frames in RPP packets, and ignores frames that arrive without
  the wrapper. An OS without `ext.rpp` that sends its `ext.sppRemote` frames directly cannot talk to it.

---

### The `rpp` API

| Name              | Description                                                                                                                                                          | Arguments                           | Returns                                                             |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- | ------------------------------------------------------------------- |
| rpp.getAddress    | The address of this host.                                                                                                                                            | none                                | address (string)                                                    |
| rpp.getInterfaces | The interfaces of this host.                                                                                                                                         | none                                | array of `{name = string, up = boolean}`                            |
| rpp.getNeighbors  | The neighbours that are currently known.                                                                                                                             | none                                | array of `{address, interface, role, bidirectional, age, distance}` |
| rpp.getRoutes     | The routing table.                                                                                                                                                   | none                                | array of `{address, next, interface, metric, age}`                  |
| rpp.isRouter      | Whether this host passes packets on.                                                                                                                                 | none                                | boolean                                                             |
| rpp.setRouter     | Turns forwarding on or off. A host is a router by default if it has two interfaces, and an end host otherwise. An OS may refuse to let a program do this (`EACCES`). | enabled (boolean)                   | true, or nil, code and message                                      |
| rpp.ping          | Sends an echo to `address` and waits for the answer. Yields. Returns the round trip time in seconds.                                                                 | address (string), timeout (number?) | seconds (number), or nil, code and message                          |

`age` is the number of seconds since the entry was last refreshed. `distance` is only present on `wireless` neighbours, in
blocks, and is `nil` otherwise. The routing table does not include the host's own address.

The failure codes are the shared NEATO codes defined in [errors.md](../common/errors.md):

| Code           | Meaning in RPP                                 |
| -------------- | ---------------------------------------------- |
| `EHOSTUNREACH` | There is no route to the address.              |
| `ETIMEDOUT`    | The timeout passed before an answer came.      |
| `EINVAL`       | An address is not valid.                       |
| `EACCES`       | The OS does not allow this program to do this. |

---

### Security considerations

RPP adds addressing and nothing else. In particular:

- Source addresses are claims. Nothing checks them, so any host can send as any other, and a nearby host can answer for
  an address that is not its own. Programs that need to know who they are talking to have to authenticate inside their
  payloads, for example with signatures from the `crypto` API, and never trust `peer.host` by itself.
- Routes can be forged. A host that sends `vector` frames can attract any traffic to itself, or throw it away. Only a
  router that is trusted should be kept connected to a network that matters.
- Amplification. A router passes on packets for free, so a flood sent through it arrives at hosts further away, where
  each can lose 75 events from it. An OS should limit how many packets a router passes on in a second to what it can
  handle, and drop the oldest first.
- Nothing is secret. Every host on a medium hears every frame.
