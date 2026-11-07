# NEATO `internet` API Specification

Extension: `ext.internet`

Version: 1

Requires: `core`

---

An environment that reports `ext.internet` must provide the global `internet` table with the same functions,
parameters and return types as the NEET Computers API. It is the machine's escape hatch to the real-world
internet: HTTP requests and WebSocket connections to servers outside the game, made from a computer inside it.
It keeps its raw NEET names so that a program written for NEET Computers works unchanged on an operating system
that reports this extension, and it is a raw passthrough extension in the way [ext.screen](screen.md) and
[ext.chip](chip.md) are: the operating system carries it as it is and does not re-model it.

`ext.internet` has nothing to do with the internet layer of [NEATO Network](../network/README.md), which is
`ext.rpp`. The protocols of NEATO Network run between NEET computers, inside the game's world; `internet`
reaches whatever real-world network the machine running the game is attached to. The two share a subject and
nothing else.

`internet` keeps the NEET Computers API unchanged, including how it reports its own failures: a failure raised
at call time is a Lua error with a plain string message, and there is nothing for an operating system to
redirect, so it does not use the codes in [errors.md](../common/errors.md). Very little fails at call time,
though. The three network calls return at once with an id, and the answer - a response, a message, or the news
that a message was dropped - arrives later as an [event](core/event.md). The events are defined below. An
operating system that does not report this extension must not define them, like any other name an extension
carries.

```c
internet
  GET
  POST
  CreateWebsocket
  hasAccess
  isReady
```

| Name                     | Description                                                                                                | Arguments                                                       | Returns           |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | ----------------- |
| internet.GET             | Begins an HTTP GET request. Returns an id at once; the answer arrives as an `HttpResponse` event.            | URL (string), headers (table, optional)                         | id (int)          |
| internet.POST            | Begins an HTTP POST request, with an optional body. Returns an id at once; the answer arrives as an `HttpResponse` event. | URL (string), headers (table, optional), body (bytes, optional) | id (int)          |
| internet.CreateWebsocket | Begins opening a WebSocket connection. Returns an id at once; the socket arrives as a `WebsocketOpened` event. | URL (string), headers (table, optional)                         | id (int)          |
| internet.hasAccess       | Returns whether internet access is switched on for this machine.                                            | none                                                            | allowed (boolean) |
| internet.isReady         | Returns whether the machine can accept another outgoing request right now.                                  | none                                                            | ready (boolean)   |

---

### Ids

Every network call returns an id: a non-negative integer, chosen by the implementation, that names the request
or the socket the call began. The id is opaque, a program compares ids for equality and does nothing else with
them, and request ids and socket ids come out of the same supply, so a program keeps them apart by what it
asked for, not by their values. An id is a reservation, not a promise: `internet.GET` and `internet.POST` return
one even when access is switched off or the outgoing buffer is full, and the refusal arrives as the event
instead. `internet.CreateWebsocket` is the one exception, and raises instead when it is refused up front, as
[Requests](#requests) describes.

---

### Requests

`URL` is a string: `http` or `https` for the request calls, `ws` or `wss` for `internet.CreateWebsocket`. A URL
that cannot be parsed raises the error `Invalid URL`.

`headers` is a table whose keys and values are both strings: a header name and its value. Anything else raises
the error `Invalid Header (All values must be strings)`.

`body` in `internet.POST` is the bytes of the request body, given as a bytes value or as a string; leaving it
out or passing `nil` sends no body.

Nothing about the network raises in `internet.GET` or `internet.POST`. Only the two errors above fail the call;
a request that cannot be served returns its id like any other and answers by event, whatever went wrong.

`internet.CreateWebsocket` can be refused before there is a socket to report about, and then it raises:
`Internet access disabled` when access is switched off for the machine, `Limit on active sockets reached` when
the implementation is already holding its limit of open sockets, `Invalid Request` and `Insecure operation` for
a request it will not act on, `Handshake failed` and `Timed out` for a connection attempt that fails up front.
Since the attempt happens off the computer, which of these names a given refusal is the implementation's
business; the program's contract is that a refused connection raises and a begun one returns an id.

---

### Responses and websocket events

The events are ordinary events of [core](core/event.md), delivered to a program's queue in the order the
implementation reported them. A response is reported when it completes, so answers to two requests may arrive in
either order whatever order the requests were made in, and messages, responses and send results interleave by
arrival: a program matches its answers by `id`, never by order. Which program receives which of these events is
up to the operating system, as [core/event.md](core/event.md) says for events generally, usually the program
that made the call, and a program that was not delivered an event never sees it.

| Name                 | Values                                                                          | Description                                                          |
| -------------------- | ------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| HttpResponse         | id (int), code (int), message (string), headers (table), body (bytes, optional)  | The answer to a request made with `internet.GET` or `internet.POST`. |
| WebsocketOpened      | id (int), send (function), close (function)                                     | The connection `internet.CreateWebsocket` began has opened.          |
| WebsocketMessage     | id (int), body (bytes), binary (boolean)                                        | A message has arrived on the socket.                                 |
| WebsocketSendSuccess | id (int)                                                                        | A message queued with `send` has been sent.                          |
| WebsocketSendFailure | id (int), message (string) - or message (string) alone                          | A message queued with `send` has been dropped and will not be sent.  |
| WebsocketClosed      | id (int), code (int), reason (string)                                           | The socket has closed. Its id is spent.                              |

#### HttpResponse

Delivered once for each request, carrying the request's id. `headers` is a table of the response's header names
and values; a name the response repeats is one entry whose value joins the repeats with `", "`. `body` is the
response's bytes, present as a fifth value only when the response carried one - an event with no body has four
values, and this is not a missing value but the shape the API uses for an empty body.

When a server answered, `code` is the status code of its answer and `message` its text as the implementation
knows it (`OK`, `NOT FOUND`, `I'M A TEAPOT`, and `UNKNOWN STATUS CODE` for one it does not know). When the
request failed short of a server, the event keeps its one shape and is filled with a synthetic code instead.
The current implementations use these:

| Code | Message                  | When                                                               |
| ---- | ------------------------ | ------------------------------------------------------------------ |
| 105  | `ENOBUFSNO`              | the outgoing buffer was full when the call was made                |
| 110  | `ETIMEDOUT`              | the request ran past the implementation's timeout (three seconds)  |
| 111  | `ECONNREFUSED`           | internet access is switched off for the machine                    |
| 400  | `BAD REQUEST`            | the request could not be issued                                    |
| 404  | `NOT FOUND`              | the host could not be found, or the connection could not be made   |
| 409  | `CONFLICT`               | the request was interrupted                                        |
| 0    | `UNIDENTIFIED EXCEPTION` | the implementation failed in a way it cannot name                  |

The list is informative, not exhaustive: an implementation may answer other codes, and nothing marks a code as
synthetic - 400 and 404 are also perfectly ordinary answers from a server. A program that must tell the two
apart cannot, and that is accepted.

#### WebsocketOpened

Delivered when the connection opens, after the handshake, carrying the socket's id. Its second and third values
are the socket's whole surface - the API has no table or handle for a socket anywhere; its methods arrive by
mail, in the event that opens it.

`send` takes the message as bytes - a string is accepted - and a boolean: `false` sends a text message, `true`
a binary one. It queues the message and returns before anything has gone out; the outcome arrives later as
`WebsocketSendSuccess` or `WebsocketSendFailure` with the socket's id. `close` begins a normal close of the
socket and raises the error `Already closed` when the socket has closed already.

#### WebsocketMessage

Delivered once per message as it arrives, carrying the socket's id. `body` is the message's bytes and `binary`
says which kind arrived: `false` for a text message, `true` for a binary one.

#### WebsocketSendSuccess

Delivered when a message queued with `send` has gone out, carrying the socket's id. Messages leave in the order
they were queued.

#### WebsocketSendFailure

Delivered when a message queued with `send` has been dropped and will not be sent. It carries the socket's id
and a message - except that a send refused because the outgoing buffer was full is reported as the message
alone, without the id. The messages the implementations use are `Outgoing buffer full, try again later`,
`Line closed` and `Line busy, try again later`.

#### WebsocketClosed

Delivered when the socket closes, carrying the socket's id, a WebSocket close code and a reason string - close
codes, not HTTP ones. The socket's id is spent: no further messages arrive on it, and `send` can no longer
succeed. An abnormal end is reported as code `1006` with reason `Abnormal Closure`, and a reset of the computer
closes every live socket with code `1001` and reason `Going Away`.

If the handshake fails, the connection never opens and no event reports it: the id simply never answers. A
program that must not wait forever gives its `event.pull` a timeout, as [core/event.md](core/event.md) allows.

---

### Access

`internet.hasAccess` and `internet.isReady` are the hardware's gates, not the operating system's. `hasAccess`
answers whether internet access is switched on for this machine: in NEET Computers it is a setting of the server
the computer belongs to, decided outside the computer and outside any operating system. It answers for the
machine, not for any program, and an operating system must not answer on its own behalf - even for a program it
has itself cut off, `hasAccess` keeps reporting what the hardware reports. `isReady` answers whether the machine
can accept another outgoing request right now: in the current implementations, whether the outgoing buffer has
room, and `hasAccess` besides. It is a hint, not a promise: between the check and the call, other traffic can
fill the buffer.

On top of the hardware's gate, an operating system MAY deny internet access itself, to every program or only to
some - why, and which programs, is entirely its policy and is not defined here, any more than
[chip.md](chip.md) defines why an operating system blocks `shutdown`. A denied call must have no effect: no
request may leave the machine, no socket may open, and no `HttpResponse` or websocket event may be produced for
it. It must also fail in a way the program can detect: it raises a Lua error, carrying a plain string message
the operating system chooses, exactly as [chip.md](chip.md) blocks destructive calls. The distinction this draws
is deliberate: an error means this program was refused by this operating system, while an id means the machine
itself took the call.

The hardware's own denial behaves differently, and must stay that way. When internet access is switched off for
the machine, `internet.GET` and `internet.POST` still return an id and answer it with an `HttpResponse` of code
111, and `internet.CreateWebsocket` raises NEET's own `Internet access disabled`. That is NEET Computers
behavior, and an operating system that permits everything passes it through unchanged.

`internet.hasAccess` and `internet.isReady` must always work where this extension is reported; they reach
nothing, and an operating system must not block them. An operating system running on a machine whose internet
access is switched off may still report this extension: the hardware's answers are the honest report, and it
does not withhold `ext.internet` over a machine setting it does not control.

---

### Example usage

```lua
local id = internet.GET("https://example.com/index.html", { ["User-Agent"] = "neeto-fetch/1.0" })
local code, message, headers, body
while true do
  local name, rid, c, m, h, b = event.pull(nil, 10)
  if name == nil then
    error("no response within ten seconds")
  elseif name == "HttpResponse" and rid == id then
    code, message, headers, body = c, m, h, b
    break
  end
end
print(("got HTTP %d %s"):format(code, message))
if body then print(#body .. " bytes") end
```

```lua
local id = internet.CreateWebsocket("wss://echo.example.org")
local _, sid, send, close = event.pull("WebsocketOpened", 5)
if sid == nil then error("connection did not open") end
send("hello", false)
while true do
  local name, rid, a, b = event.pull()
  if name == "terminate" then
    close()
    return
  elseif name == "WebsocketClosed" and rid == id then
    print(("closed (%d %s)"):format(a, b))
    return
  elseif name == "WebsocketMessage" and rid == id then
    print(("echo: %s"):format(tostring(a)))
    send(a, b)
  end
end
```
