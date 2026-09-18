# NEATO Network

NEATO specifications for networking protocols.

The NEATO Network module defines specifications for networking protocols between programs running on separate
computers or possibly between programs running on a single computer. These networking protocols are, at the lowest
abstraction layer, designed to be used with the built-in `io.broadcastLocal(arguments)` function, the Access Point
peripheral’s `broadcast(arguments)`, and any other data transfer methods that may be used in the future. The module
defines the protocols used in each Internet Protocol Suite layer. Programs and operating systems are welcome to
implement their own protocols on any layer they like, but these standard protocols are designed so that a) program
developers do not have to reinvent the wheel every time and b) programs and operating systems that use the same
protocol can interoperate. In the future, NEATO API modules should be specified to add abstraction for each protocol
so that developers do not have to reimplement them.

## Protocol specifications:

### Link Layer

The link layer consists of protocols for sending packets directly between computers/routers that are directly physically
or wirelessly connected to.

- [Local Communication Link and Wireless Communication Link (lcl-wcl.md)](lcl-wcl.md) - The two link-layer data transfer
  primitives, using `io.broadcastLocal(arguments)` and `access_point.broadcast(arguments)` respectively.

### Internet Layer

The internet layer consists of protocols for transporting network packets from the originating host to the correct
destination, possibly across different networks.

- [NEET Internet Protocol (nip.md)](nip.md) - This specification defines the primary internet-layer protocol used to
  transport internet packets between computers across networks.

### Transport Layer

The layer consists of protocols for providing communication services between applications, typically by allowing
applications to send packets to and listening for packets on ports. The operating system should handle these protocols
directly, routing the packet's payload to the program that is listening on the specified port, and sending transport-layer
packets based on the data that the program sends and the destination.

- [Direct Payload Protocol (dpp.md)](dpp.md) - A connectionless transport-layer protocol comparable to the real-life
  User Datagram Protocol.

### Application Layer

The application layer consists of abstractions that specify the shared communication protocols used by programs to
communicate for certain purposes.

_No protocols have been developed for the application layer yet. You are welcome to draft your own for a chance for it
to be standardized!_
