# NEATO `Local Communication Link` and `Wireless Communication Link` Specification

Written by UsUsStudios

Revision 1 of September 14, 2026

---

This specification defines the two primitive link-layer protocols used to transfer payloads directly between hosts.
They are identical in all functions, except that LCL sends arguments using `io.broadcastLocal(arguments)` and WCL
sends arguments using `access_point.broadcast(arguments)`. Because of this, when received, WCL packets also contain
a distance value added by NEETComputers while LCL packets do not. To refer to both protocols at the same time, the
term CL (Communication Link) will be used in this specification.

A CL packet consists of three arguments passed to either broadcast function: `source`, `target` and `payload`. Each
one is described below.

### source

The first argument is simply a string equal to the UUID of the computer that sent the packet. This may be used for
responding to the packet.

### target

The second argument is either a UUID as a string, or false (the boolean value). If it is a UUID, that packet is known
as targeted, and the payload is only intended for the computer with that exact UUID to read. If the `target` is false,
that packet is known as broadcasted, and any computer that receives this packet may read the payload and do whatever
it wants with it.

### payload

The sixth argument can be any value that a protocol defines. If the payload is a dictionary table, then it SHOULD have
a `protocol` entry with the protocol name/header for programs to check against, and if the payload is an array table,
then its first entry SHOULD be the protocol name/header, but neither of these are required. It is against NEATO
specification to read, access or index the payload if the target of the packet is not this computer and the packet
is not broadcasted.
