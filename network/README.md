# NEATO Network

NEATO specifications for networking protocols.

The NEATO Network module defines specifications for networking protocols between programs running on separate
computers or possibly between programs running on a single computer. These networking protocols are, at the lowest
abstraction layer, designed to be used with the built-in `io.broadcastLocal(arguments)` function, as well as potential
future wireless equivalents. The module defines the protocols used in each Internet Protocol Suite layer. Programs are
welcome to implement their own protocols on any layer they like, but these standard protocols are designed so that a)
program developers do not have to reinvent the wheel every time and b) programs that use the same protocol can
interoperate. Each protocol that has an API for programs to use it is a NEATO extension (for example, `ext.dpp`), so an operating
system that does not implement it simply does not report it.

## Protocol specifications:

### Link Layer

The link layer consists of protocols for sending packets directly between computers/routers that are directly physically
or wirelessly connected to.

The two media that NEET Computers has are a wired one (`io.broadcastLocal`, which reaches every computer on the same
network cable graph) and a wireless one (the Access Point peripheral, which reaches every other Access Point in range).
Both send to *everybody* in reach, lose messages when a receiver's queue is full, and never say who sent a message.
There is no separate link layer protocol. [rpp.md](rpp.md#interfaces) describes how the media are used.

- [Wire format (wire.md)](wire.md) - What a frame can contain on either medium (which values survive the trip, and the
  limits that follow), and the encoding that carries tables and arbitrary strings through it. Shared definitions, not
  an extension.

### Internet Layer

The internet layer consists of protocols for transporting network packets from the originating host to the correct
destination, possibly across different networks.

- [Routed Packet Protocol (rpp.md)](rpp.md) - Host addresses, packet headers, neighbour discovery and distance-vector
  routing, so that a packet can cross routers. It is the draft NEATO extension `ext.rpp`.

### Transport Layer

The transport layer consists of protocols for providing communication services between applications, typically by
allowing applications to send packets to, and listen for packets on, ports. The operating system should handle
these protocols directly, routing the packet's payload to the program that is listening on the specified port,
and sending transport-layer packets based on the data that the program sends and the destination.

- [Direct Payload Protocol (dpp.md)](dpp.md) - A connectionless transport-layer protocol comparable to the real-life
  User Datagram Protocol. It is the NEATO extension `ext.dpp`.
- [Sequenced Payload Protocol (spp.md)](spp.md) - A connection-oriented transport-layer protocol with reliable,
  in-order, message-preserving connections between programs on one computer, per-connection inboxes, pollable
  handles, and peer identity. It is the NEATO extension `ext.spp`. Connections between computers are a draft
  extension, [`ext.sppRemote`](spp-remote.md).

### Application Layer

The application layer consists of abstractions that specify the shared communication protocols used by programs to
communicate for certain purposes.

_No protocols have been developed for the application layer yet. You are welcome to draft your own for a chance for it
to be standardized!_
