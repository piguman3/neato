# NEATO `Sequenced Payload Protocol` Specification

Author: XesHuman

Extension: `ext.spp`

Version: 2

Requires: `core`

---

This specification defines a connection-oriented transport-layer protocol called the Sequenced Payload Protocol, or
SPP. Where the [Direct Payload Protocol](dpp.md) sends independent datagrams to a port, SPP gives two programs a
connection: messages arrive exactly once and in the order they were sent, message boundaries are kept (one `send` is
one `recv`), and either side finds out when the other one is gone. It is comparable to a Unix `SOCK_SEQPACKET`
socket, or to TCP with the message framing already done.

SPP does not change DPP. DPP stays a connectionless, best-effort datagram protocol, and the two have separate port
spaces: listening on DPP port 80 has no effect on SPP port 80. An OS may report `ext.spp` without `ext.dpp`, and the
other way around.

`ext.spp` covers connections between programs on the same computer. Connections to other computers are a separate,
larger piece of work and are specified by [`ext.sppRemote`](spp-remote.md), which is a draft. Everything in this
file is written so that a program does not change when remote connections are added: where something is local or
remote, the program says so explicitly (see [Scope](#scope)).

An operating system that reports `ext.spp` must provide the `spp` API described in this file.

---

### What SPP adds compared to DPP

- Every connection says whether its peer is on this computer or on another one, and who the peer is (see
  [Peer identity](#peer-identity)). The operating system fills this in. A program cannot choose it.
- A program decides whether a listener can be reached from other computers. The default is that it cannot (see
  [Scope](#scope)). Local traffic never touches a network medium.
- Every connection has its own inbox. SPP does not use the [event queue](../api/event.md) at all, so it can neither
  fill it up nor lose data to `event.pull` filters. A full inbox slows the sender down, it never drops a message.
- Listeners and connections can be waited on together, and together with the event queue (see [Polling](#polling)).

---

### Terms

| Term       | Meaning                                                                                                    |
| ---------- | ---------------------------------------------------------------------------------------------------------- |
| listener   | A handle that owns an SPP port and accepts incoming connections on it.                                     |
| connection | A handle for one end of an established connection. Both ends are equal, there is no client or server role. |
| peer       | The program at the other end of a connection.                                                              |
| message    | One payload, sent with `conn:send` and received as one unit with `conn:recv`.                              |
| inbox      | The bounded queue of received, not yet read messages that belongs to one connection.                       |

Listeners and connections are handles in the same sense as the handles returned by [`fs.open`](../api/fs.md): tables
with methods that are called with `:`. Calling a method on a closed handle raises an error. Handles that are still open
when the program ends are closed by the operating system, as if the program had called `close`.

---

### Ports

SPP ports are integers from 1 to 65535. A listener owns one port, and only one listener can own a port at a time,
whatever its scope. A connecting program does not choose its own port. The operating system gives each connection a
free local port, which the program can read as `conn.port`.

An operating system may refuse to let a program listen on certain ports (for example 1 to 1023 for programs that are
not administrative), in which case `spp.listen` fails with `EACCES`. The ports are released when the listener is
closed or the program ends.

---

### Scope

A *scope* says which computers are involved. There are two:

| Scope       | Meaning                                                                                    |
| ----------- | ------------------------------------------------------------------------------------------ |
| `"local"`   | Only programs on this computer. No network traffic is ever produced or accepted.           |
| `"network"` | Programs on other computers. Requires `ext.sppRemote`. Without it, the scope is `ENOTSUP`. |

- `spp.listen` takes a scope that decides who may connect: `"local"` accepts connections from programs on this
  computer, `"network"` accepts them from this computer and from other computers. The default is `"local"`.
  A program therefore has to ask to be reachable from outside, and an operating system never exposes a port by
  accident.
- `spp.connect` takes a scope that decides where to connect: `"local"` only looks for listeners on this computer,
  `"network"` only looks on other computers and never on this one. The default is `"local"`. A program that does not
  care where its service runs tries one and then the other.
- A connection attempt that no listener on the chosen scope admits (nothing listens on the port, or the listener's
  scope is `"local"` and the attempt comes from another computer) fails with `ECONNREFUSED` if it is local. Remote
  attempts are not answered at all, so a computer does not reveal which ports it uses (see `ext.sppRemote`).

---

### Peer identity

Every connection has a `peer` table, which the operating system fills in when the connection is made and which does not
change afterwards. Both ends have one, describing the other end.

```
{
    origin: string,     -- "local" or "remote"
    user: string?,      -- name of the user account the peer runs as
    id: integer?        -- operating system defined identifier of the peer program
}
```

- `origin` is `"local"` if the peer is a program on this computer and `"remote"` if it is on another one. The
  operating system decides it from the path the connection actually took, never from anything the peer sent. Every
  connection that `ext.spp` alone makes is `"local"`.
- `user` is the user account the peer program runs as, as far as the operating system has such a notion. NEATO does not
  define what a user is, so a program must treat the string as opaque and only compare it for equality. It is `nil` if
  the operating system has no user accounts, and it is always `nil` for a remote peer, because another computer's
  accounts cannot be checked. A program that makes a decision based on `user` must treat `nil` as "not allowed".
- An extension may add fields to `peer`. [`ext.sppRemote`](spp-remote.md) adds `medium` and `distance`, or, with [`ext.rpp`](rpp.md), `host`. A program must
  ignore fields it does not know.
- `id` identifies the peer program on this computer while it is running (an operating system with process IDs would
  use those). It means nothing on another computer and may be reused after the peer ends. It is `nil` if the operating
  system has none, and always `nil` for a remote peer.

Because a listener cannot see who is connecting before it accepts, a server checks `conn.peer` after `accept` and
closes the connection if it does not like what it sees.

---

### Messages

A message is a payload: a boolean, a number, a string, or a table whose keys and values are payloads. Tables may be
nested at most 32 levels deep and must not refer to themselves, and `nan` cannot be a key or a value. Functions, threads
and userdata cannot be sent, and metatables are not sent. These are the same rules as the [payload
encoding](wire.md#payload-encoding) that connections between computers use, so a program that works over a local
connection sends exactly the payloads that work over a network one, and an operating system that reports `ext.spp`
without `ext.dpp` does not depend on DPP to know what a payload is. The operating system copies the payload when it is
sent, so the sender and the receiver never share a table. A table that is referred to twice inside one payload arrives
as two separate copies.

Within one connection every message that is sent is received exactly once and in the order it was sent, for as long
as the connection is not `reset`. There is no loss, duplication or reordering to handle. A single payload has a size
limit (see `spp.getLimits` and `conn.maxPayload`), and `conn:send` fails with `EMSGSIZE` if it is exceeded.

Each connection has an inbox on the receiving side, with room for at least 16 messages
(operating systems may allow more, and report it with `spp.getLimits`). When the inbox of the peer is full, `conn:send`
waits (yields) until there is room or its timeout passes. Messages are never dropped to make room.

---

### Closing

`conn:close()` ends the connection in an orderly way. Messages the program already sent are still delivered. The peer
reads them and then gets `EPIPE` from `conn:recv`. Messages that were waiting in the closing program's own inbox are
discarded. `EPIPE` is also what `conn:send` returns when the peer has closed.

A connection can also end badly, which is called a reset (its code is `ECONNRESET`): the peer program was killed in a way that stopped it from
closing, the operating system had to drop the connection (for example, a resource limit), or, with `ext.sppRemote`,
the remote computer stopped answering. Messages that were not yet delivered may be lost. Every later call on that
connection returns `ECONNRESET`.

`listener:close()` releases the port. Connections that were waiting to be accepted are reset, and connections that were
already accepted are not affected.

---

### Polling

`spp.poll` waits until one or more handles are ready, so that one program can serve many connections without blocking
on any one of them, and can wait for the keyboard at the same time.

A handle counts as readable when the matching call would not have to wait:

| Handle               | Readable when                                                                                         |
| -------------------- | ----------------------------------------------------------------------------------------------------- |
| listener             | A connection is waiting for `accept`.                                                                 |
| connection           | `conn:recv` has something to return: a message is in the inbox, or the connection is closed or reset. |
| the string `"event"` | The program's [event queue](../api/event.md) has at least one event. Nothing is removed from it.      |

A connection counts as writable when `conn:send` would not have to wait: the peer's inbox has room, or the
connection is closed or reset (in which case `send` fails immediately).

Polling only reports readiness. It never consumes a message, an event or a pending connection. The program then calls
`recv`, `accept` or `event.pull` itself.

---

### The `spp` API

| Name          | Description                                                                                                                                                                                                                                                                                                                                                                                                          | Arguments                                        | Returns                                                                |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------- |
| spp.listen    | Starts listening on a port. `opts.scope` is `"local"` (default) or `"network"`. `opts.backlog` is how many connections may wait to be accepted (at least 1, limited by the operating system).                                                                                                                                                                                                                        | port (int), opts (table?)                        | listener, or nil, code and message                                     |
| spp.connect   | Connects to a listener. Yields until the connection is made, refused, or `opts.timeout` seconds pass (the default is up to the operating system). `opts.scope` is `"local"` (default) or `"network"`. `opts.host` is the address of a host (a string). It needs `ext.rpp` and `scope = "network"`: without `ext.rpp` a host that is not `nil` fails with `ENOTSUP`, and with scope `"local"` it fails with `EINVAL`. | port (int), opts (table?)                        | connection, or nil, code and message                                   |
| spp.poll      | Waits until a handle in `read` is readable or a handle in `write` is writable, or until `timeout` seconds have passed. `read` may also contain the string `"event"`. Yields while waiting.                                                                                                                                                                                                                           | read (table?), timeout (number?), write (table?) | readable (table), writable (table), or nil if the timeout passed first |
| spp.getLimits | Returns the limits of this operating system.                                                                                                                                                                                                                                                                                                                                                                         | none                                             | `{maxPayload = int, inbox = int, backlog = int, maxConnections = int}` |

`maxPayload` is the largest payload that can be sent, as the operating system measures it in bytes. For a local
connection it may be treated as approximate. For a connection to another computer it is exact: the size of a payload is
the length of its raw encoding in bytes (see [wire.md](wire.md#payload-encoding)). `inbox` is the number of messages
every connection's inbox holds, `backlog` is the largest backlog that `spp.listen` accepts, and `maxConnections` is how
many handles one program may hold at the same time.

Listener (`kind = "listener"`, with read-only fields `port` and `scope`):

| Name            | Description                                                                                                 | Arguments         | Returns                              |
| --------------- | ----------------------------------------------------------------------------------------------------------- | ----------------- | ------------------------------------ |
| listener:accept | Takes the oldest waiting connection. Waits for one to arrive, for at most `timeout` seconds if it is given. | timeout (number?) | connection, or nil, code and message |
| listener:close  | Closes the listener. See [Closing](#closing).                                                               | none              | true                                 |

Connection (`kind = "connection"`, with read-only fields `port`, `peerPort`, `peer` and `maxPayload`):

| Name       | Description                                                                                                               | Arguments                        | Returns                                 |
| ---------- | ------------------------------------------------------------------------------------------------------------------------- | -------------------------------- | --------------------------------------- |
| conn:send  | Sends one message. Waits while the peer's inbox is full, for at most `timeout` seconds if it is given.                    | payload (any), timeout (number?) | true, or nil, code and message          |
| conn:recv  | Takes the next message out of the inbox. Waits for one to arrive, for at most `timeout` seconds if it is given.           | timeout (number?)                | payload (any), or nil, code and message |
| conn:close | Closes the connection. See [Closing](#closing). Calling it again on a closed handle raises an error, as for every handle. | none                             | true                                    |

`conn.maxPayload` is the largest payload that `conn:send` accepts on this connection. It is at most `maxPayload` from
`spp.getLimits`, and is smaller for a connection to another computer (see [`ext.sppRemote`](spp-remote.md)).
`conn.port` is the local port of the connection. `conn.peerPort` is the port of the peer: for a connection made with
`spp.connect` it is the port that was connected to, for an accepted connection it is the port the operating system
gave the connecting program.

A `timeout` of `nil` means to wait without limit, and `0` means never to wait: the call returns at once with
`ETIMEDOUT` if it would have had to wait. A `payload` of `false` is a valid message, so a program must check the first
result against `nil` and not for truthiness.

---

### Failure codes

Functions that can fail return `nil`, then a code, then a human readable message, as defined in
[errors.md](../common/errors.md). The code is part of the NEATO error registry and a program may compare it; the message
is only for humans. The codes SPP uses, and the condition each one reports, are:

| Code           | Meaning in SPP                                                                                                 |
| -------------- | -------------------------------------------------------------------------------------------------------------- |
| `ETIMEDOUT`    | The timeout passed before the call could finish.                                                               |
| `EPIPE`        | The connection was closed in an orderly way. For `recv`, all messages that were sent before it have been read. |
| `ECONNRESET`   | The connection ended abnormally. See [Closing](#closing).                                                      |
| `ECONNREFUSED` | Nothing admitted the connection: no listener, or its scope does not allow this attempt.                        |
| `EADDRINUSE`   | Another listener already owns the port.                                                                        |
| `EACCES`       | The operating system does not allow this program to do this (for example, a privileged port).                  |
| `EINVAL`       | A port, option or payload is not valid.                                                                        |
| `EMSGSIZE`     | The payload is larger than `maxPayload`.                                                                       |
| `ENOTSUP`      | The operating system does not support this scope or option (for example `"network"` without `ext.sppRemote`).  |
| `EHOSTUNREACH` | There is no route to `opts.host`. Only with `ext.rpp`.                                                         |
| `ELIMIT`       | A limit was reached, such as `maxConnections`, or no free local port.                                          |

Passing a value of the wrong type (a port that is not a number, a `timeout` that is not a number) raises an error
instead, like the rest of NEATO.

---

### Example usage

A server that handles many clients and the keyboard in one loop:

```lua
local srv = assert(spp.listen(7000))
local conns = {}

while true do
  local read = { srv, "event" }
  for _, c in ipairs(conns) do read[#read + 1] = c end

  local ready = spp.poll(read)
  for _, h in ipairs(ready) do
    if h == "event" then
      if event.poll() == "terminate" then return end
    elseif h.kind == "listener" then
      local c = h:accept(0)
      if c then
        if c.peer.user == "root" then
          conns[#conns + 1] = c
        else
          c:close()
        end
      end
    else
      local msg, code = h:recv(0)
      if msg ~= nil then
        h:send({ "echo", msg }, 0)
      elseif code == "EPIPE" or code == "ECONNRESET" then
        h:close()
        for i, c in ipairs(conns) do if c == h then table.remove(conns, i) break end end
      end
    end
  end
end
```

A client:

```lua
local c, code = spp.connect(7000, { timeout = 2 })
if not c then error("could not connect: " .. code) end

c:send({ "hello", 42 })
local reply = c:recv(5)
print(reply[1], reply[2][2])
c:close()
```

---

### Footnotes

On `peer.user` and trust: the operating system vouches for `user` only for programs it runs itself. SPP never carries
credentials inside a message, since anything a program can write into a message, a program can forge.

On not using the event queue: DPP delivers into the event queue, so a program that calls `event.pull("char")` throws
away any DPP message that arrives meanwhile, and a full queue silently loses messages. SPP keeps its data out of the
queue for that reason. A program that sits in a blocking `recv` or `accept` does not see `terminate` events while it
waits, so a program that must stay responsive uses a timeout or `spp.poll` with `"event"`.

On `scope`: a `"network"` listener also accepts local connections, and these are made inside the operating system
without any network traffic. There is no scope for "only other computers" on the listening side: a program that serves
remote clients can serve local ones as well, and a server that wants to treat them differently looks at
`conn.peer.origin`.
