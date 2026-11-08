# NEATO `compat` API Specification

Extension: `ext.compat`

Version: 1

Requires: `core`

---

NEET Computers can run a compatibility layer for ComputerCraft: peripherals and other equipment written for
ComputerCraft are attached to a NEET computer through a wrapper, and the wrapper queues the events that equipment
generates to the computer unchanged, under ComputerCraft's own event names. This extension defines what those events
are called in NEATO and what an operating system that runs the compatibility layer must deliver, so a program can
wait for them through the [event](core/event.md) API of `core` like any other event. Which program a compat event is
delivered to is up to the operating system, as everywhere in `core`.

Compat events exist only while the operating system runs the compatibility layer. An OS that does not report
`ext.compat` must not define any event name this extension defines, so on such a system no `compat_` event ever
arrives, and a program can treat the absence of the extension as the absence of the whole vocabulary. Compat events
come from the compatibility layer's equipment alone: a plain NEET peripheral is not ComputerCraft equipment, and no
event of a plain NEET peripheral is ever a compat event, whether or not a wrapper is present on the machine.

---

### Standard events

Every event the compatibility layer queues is delivered as one `compat_` event:

| Name           | Values                                           | Description                                                     |
| -------------- | ------------------------------------------------ | --------------------------------------------------------------- |
| compat_`event` | the values of the compatibility event, unchanged | The compatibility layer queued the ComputerCraft event `event`. |

The name of the NEATO event is `compat_` followed by the event's ComputerCraft name, byte for byte. The values are
the values of that event, in the same order, the same number of them and the same types, unchanged: NEATO does not
redefine what the values mean, that stays with ComputerCraft's own documentation, coordinate conventions included.
A `monitor_touch` arrives as `compat_monitor_touch`, a `disk` arrives as `compat_disk` with the side as its first
value, a `timer` arrives as `compat_timer` with its token, and so on for every event name the layer produces,
including names this specification does not list.

The prefix is what makes the name. An event of the compatibility layer is never delivered under its raw
ComputerCraft name; the one exception is typed input, which the next section allows the operating system to deliver
as the standard events of `core` instead.

---

### Overlap with the input events of `core`

The compatibility layer also produces events for typed input, under ComputerCraft's names `key`, `key_up`, `char`
and `paste`. Those overlap in meaning with the standard `key`, `key_up` and `char` events `core` defines for the same
physical keystrokes, but they carry ComputerCraft's values, key numbers where `core` uses key names, so delivering
both kinds would hand a program the same keystroke twice in two shapes.

An operating system chooses, per keystroke, which shape it delivers. It may deliver the keystroke as the standard
event of `core`, translated from ComputerCraft's values into what [core/event.md](core/event.md) defines, or deliver
it as `compat_key`, `compat_char` and so on under the naming rule above, but it must not deliver both to the same
program. Everything the compatibility layer produces that `core` does not define an input event for is always passed
through under the naming rule.

---

### Example usage

```lua
while true do
  local name, side, x, y = event.pull()
  if name == "compat_monitor_touch" then
    print("touch on " .. side .. " at " .. x .. ", " .. y)
  elseif name == "terminate" then
    return
  end
end
```

```lua
local name, side = event.pull("compat_disk")
print("disk inserted on " .. side)
```