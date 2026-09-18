# NEATO `NEET Internet Protocol` Specification

Written by UsUsStudios

Revision 1 of September 17, 2026

---

This specification defines the primary internet-layer protocol used to transport internet packets between computers across
networks. It is based on a network of connected hosts (either wirelessly or locally), each of which contribute to routing
other hosts' packets.

At network setup, a computer generates its network address using the last 16 hex digits of its computer’s UUID, each group
of 4 digits separated by colon (e.g. 1234:5678:90ab:cdef). It also starts and maintains a list of packet UUIDs (explained
later), but it does not need to be maintained across system restarts, while the address does.

## Sending a packet

To send a packet to another host of a specific address (the target), the host must broadcast a WCL packet and an identical
LCL packet with the payload consisting of a table as follows:

```
{
    protocol: string = "nip",
    target: string,
    source: string,
    uuid: string,
    payload: any
}
```

### protocol

The value of the key `protocol` must be a string consisting of exactly `nip`, for the receiving program to confirm that
this is an NIP packet.

### target

The value of the key `target` must be a string conisting of the address of the target computer.

### source

The value of the key `source_port` should be a string consisting of the address of the sending computer, but it is allowed
to instead set it to nil. (This may be for privacy, but keep in mind that that the source address is still not truly private.)

### uuid

The value of the key `uuid` must be a string consisting of a UUID unique to that packet. It doesn't need to be RFC 4122
compliant, but it does need to be robust enough to be entirely unique to that packet. An good implementation can be found
[here](https://github.com/tcjennings/LUA-RFC-4122-UUID-Generator/blob/master/uuid4.lua) under the MIT license, but of course
you should ensure it is properly seeded (in the code `os.time()` is used for the seed, which doesn't exist in NEETComputers).
The purpose of this is explained later.

### payload

The value of the key `payload` can be any value the transport-layer protocol requires. If the payload is a dictionary table,
then it SHOULD have a `protocol` entry with the protocol name/header for programs to check against, and if the payload is
an array table, then its first entry SHOULD be the protocol name/header, but neither of these are required. It is against
NEATO specification to read, access or index the payload if the target of the packet is not this computer.

## Receiving a packet

When a device receives a packet, the first thing it should do is check whether the packet’s UUID is in the device’s list of
packet UUIDs. If it is, the device can completely disregard the packet. If not, it can continue by adding the UUID to its
list of packet UUIDs. Afterwards, it checks whether the target is equal to the device’s address. If so, the operating system
can read the payload and handle the transport-later protocol however it needs to. If not, it must broadcast the packet in
WCL and LCL with exactly the same values as received.
