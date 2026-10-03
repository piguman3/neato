# NEATO `event` API Specification

Extension: `core`

Version: 2

---

A NEATO compliant environment must provide a global `event` API, through which a program receives input and other
events. It replaces the raw NEET Computers `event` API, which is not part of NEATO.

Every running program has its own event queue. The operating system puts events into it, and the program takes them
out in the order they arrived (first in, first out). Which program receives input events such as key presses (for
example, only the one that has focus) is up to the operating system.

An event is a name (a string) followed by zero or more values.

| Name        | Description                                                                                                                                                                                                                        | Arguments                           | Returns                          |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- | -------------------------------- |
| event.pull  | Waits until an event is available and removes it from the queue. If `filter` is given, events with a different name are removed and discarded while waiting. If `timeout` seconds pass first, returns `nil`. Yields while waiting. | filter (string?), timeout (number?) | name (string), ... (any), or nil |
| event.poll  | Removes and returns the next event without waiting, or returns `nil` if the queue is empty.                                                                                                                                        | none                                | name (string), ... (any), or nil |
| event.push  | Adds an event to the end of this program's own queue. Fails with `ENOBUFS` if the queue is full and the event was dropped.                                                                                                         | name (string), ... (any)            | true, or nil, code and message   |
| event.clear | Removes every event from this program's queue.                                                                                                                                                                                     | none                                | nil                              |

`event.push` fails with `ENOBUFS` when the queue is full, returning `nil`, then the code, then a message, as defined in
[errors.md](../common/errors.md). `event.pull` and `event.poll` returning `nil` is an ordinary answer, not a failure.

---

### Standard events

`core` defines the following events. Extensions define their own (for example, `ext.dpp` defines `dpp`). Event names
that are not defined by a specification and do not start with `x.` are reserved.

| Name      | Values                           | Description                                                                                           |
| --------- | -------------------------------- | ----------------------------------------------------------------------------------------------------- |
| key       | key (string), isRepeat (boolean) | A key was pressed, or is being repeated while held down (`isRepeat` is `true`).                       |
| key_up    | key (string)                     | A key was released.                                                                                   |
| char      | character (string)               | A character was typed, after the keyboard layout and modifier keys were applied. One UTF-8 character. |
| terminate | none                             | The user asked for the program to stop. The gesture that causes this is up to the operating system.   |

**Key names.** `key` and `key_up` use the name of the physical key without modifiers applied. Letters are lowercase
(`"a"`), digits and symbol keys are the character on the key without shift (`"1"`, `"-"`), and the following named keys
are defined: `"enter"`, `"backspace"`, `"tab"`, `"space"`, `"escape"`, `"delete"`, `"insert"`, `"home"`, `"end"`,
`"pageUp"`, `"pageDown"`, `"up"`, `"down"`, `"left"`, `"right"`, `"leftShift"`, `"rightShift"`, `"leftCtrl"`,
`"rightCtrl"`, `"leftAlt"`, `"rightAlt"` and `"f1"` to `"f12"`. A key with no name here uses a name that starts with
`x.`, which is defined by the operating system.

---

### Example usage

```lua
local name, c = event.pull("char", 5)
if name == nil then
  print("too slow")
else
  print("you typed " .. c)
end
```

```lua
while true do
  local name, a, b = event.pull()
  if name == "terminate" then break end
  if name == "key" then print("key", a, b) end
end
```

---

### Footnotes

On `event.pull` with a filter: discarding the events that do not match is intentional. It matches how the
ComputerCraft `os.pullEvent` behaves and means a program that only listens for one event cannot be made to run out of
memory by another. A program that needs every event should call `event.pull` without a filter.
