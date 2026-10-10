# NEATO `stdio` API Specification

Extension: `ext.stdio`

Version: 1

Requires: `core`

---

A NEATO compliant operating system that reports this extension must provide a global `stdio` API with the three
standard streams every program is started with: standard input, standard output and standard error. They are what
makes a program usable as a filter and as a step in a pipeline: a program reads its input from standard input
without caring whether it was given a terminal, a file or another program's output, and writes its result to
standard output the same way.

The streams are handles with the same five methods, called with `:`, as the handles [`fs.open`](fs.md#opening-files)
returns, and they follow the same rules: a wrong argument type raises an error, calling a method on a closed stream
raises an error, and a refusal returns `nil`, then a code from [errors.md](../common/errors.md), then a message.

| Name          | Description                                                                                                                                             | Arguments        | Returns                  |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- | ------------------------ |
| stdio.stdin   | The standard input, open for reading only.                                                                                                              |                  | handle (table)           |
| stdio.stdout  | The standard output, open for writing only.                                                                                                             |                  | handle (table)           |
| stdio.stderr  | The standard error, open for writing only. Meant for diagnostics; a program's result belongs on standard output.                                        |                  | handle (table)           |
| stdio.isatty  | Returns whether the stream is connected to a terminal.                                                                                                  | stream (handle)  | boolean                  |
| stdio.readLine| Reads one line of input, with editing and echo when the input is a terminal. See [Reading a line](#reading-a-line).                                      | options (table?) | line (string), or nil, or nil, code and message |

All three streams always exist. A program that is started without anywhere obvious to put them, such as one started
from a graphical launcher, still gets them: the operating system may connect them to a log, a console window it
creates, or a place where everything written is thrown away. They are never `nil`.

A program has all three streams open from the moment it starts, and does not need to close them: the operating
system closes whatever is still open when the program ends, exactly as it does for `fs` handles.

---

### Reading and writing

`stdio.stdin` is open for reading and `stdio.stdout` and `stdio.stderr` for writing, and the rules of
[fs.md](fs.md#opening-files) apply: reading from an output stream or writing to the input stream fails with `EBADF`.

Reading from a stream that is connected to a terminal reads what the user types. The operating system chooses the
editing keys and the gesture that means end of file (for example, an empty line entered with a particular key).
`"l"` returns one line of input without its newline, `"a"` reads until end of file, and a number `n` reads up to
`n` characters across as many lines as it takes.

A stream that is connected to a terminal or a pipe cannot seek: `handle:seek` fails with `ESPIPE`. A stream
connected to a file can. When a stream was inherited from a parent program, or handed to a child by
[`ext.proc`](proc.md), its position on the file is shared, as [proc.md](proc.md#starting-a-program) describes.

Output to a terminal is written through immediately. Output to anything else may be buffered: the operating system
may hold it and write it later, `handle:flush` makes it write everything held so far, and `stdio.stderr` is never
buffered. The operating system flushes all three streams when the program ends.

If writing to `stdio.stdout` fails with `EPIPE`, because whatever was reading the other end is gone, the operating
system ends this program with a status of at least 128, the way Unix ends a process that receives `SIGPIPE`. This
is what lets a program that only writes, such as one that repeats a line forever, be stopped by the program reading
it closing its end. Any other failed write to `stdio.stdout` is dropped: `print` and `handle:write` do not raise,
and a program that needs the error writes through `stdio.stdout` itself and handles the result.

---

### What `print` does

When this extension is reported, `print` writes its converted arguments, separated by tab characters and followed
by a newline, to `stdio.stdout`. When `stdio.isatty(stdio.stdout)` is `true`, it does so with the full terminal
behavior [term.md](term.md) defines: wrapping at the edge, scrolling, the current colors. When it is `false`, the
same characters are written as plain bytes, with no wrapping, no scrolling and no screen. The rules about a failed
write to `stdio.stdout` above apply.

This is the only way this extension changes a `core` API. Every `term` function except `print` always acts on the
terminal itself, whether or not the standard output is redirected, which is how a program keeps talking to the user
while its output goes down a pipe.

---

### Reading a line

`stdio.readLine(options)` returns the next line of input from `stdio.stdin` without its newline.

When the input is a terminal, the operating system lets the user edit the line while it is being typed: at minimum
typing characters inserts them, backspace erases, and the typed text is drawn as it goes (echoed). The accepted
line is what the user finished with the key that ends a line.

| Option    | Type            | Meaning                                                                                              |
| --------- | --------------- | ---------------------------------------------------------------------------------------------------- |
| `noecho`  | boolean         | Do not draw what the user types, for reading a password. The line is finished the same way.           |
| `history` | table of string | Previous lines. The accepted line is appended to this table, and keys that recall entries, such as the up and down arrows, move through it. |

`stdio.readLine` returns the line (a string), or `nil` at end of file. While the user is typing, other events stay
in the event queue. If the terminate event arrives, the line being typed is abandoned, the terminate event is
consumed by this call, and `stdio.readLine` fails with `EINTR`: a shell, for example, abandons the line, shows a
fresh prompt, and keeps running.

When the input is not a terminal, `stdio.readLine` reads one line from `stdio.stdin`, as `stdio.stdin:read("l")`
would, with no editing, no echo and no history; the options are accepted and ignored. End of file returns `nil`.

Passing anything other than a table, or an option of the wrong type, raises an error.

---

### Failure codes

| Function       | Condition                                                        | Code     |
| -------------- | ---------------------------------------------------------------- | -------- |
| handle:read    | the handle was opened for writing                                | `EBADF`  |
| handle:read    | the input cannot be read                                         | `EIO`    |
| handle:write   | the handle was opened for reading                                | `EBADF`  |
| handle:write   | the other end of a pipe or terminal is gone                      | `EPIPE`  |
| handle:write   | there is no space left                                           | `ENOSPC` |
| handle:write   | the data cannot be written for another reason                    | `EIO`    |
| handle:seek    | the stream cannot seek                                           | `ESPIPE` |
| handle:seek    | the position is out of range                                     | `ERANGE` |
| handle:flush   | the data cannot be written                                       | `EIO`    |
| stdio.readLine | the terminate event arrived while the user was typing            | `EINTR`  |
| stdio.readLine | the input cannot be read                                         | `EIO`    |

---

### Example usage

```lua
-- A filter: reads lines from stdin, writes the matching ones to stdout.
local pattern = ...
while true do
  local line = stdio.readLine()
  if not line then break end
  if line:find(pattern, 1, true) then
    stdio.stdout:write(line, "\n")
  end
end
```

```lua
-- Talking to the user while the output goes somewhere else.
stdio.stderr:write("working...\n")
local ok, err = stdio.stdout:write("result\n")
```

---

### Footnotes

The names follow Unix, and so do the semantics: a program written against this extension is a filter, and two such
programs can be connected by giving one a pipe as standard output and the other the same pipe as standard input.
Which pipes exist and how a program is started with its streams connected to them is defined by
[`ext.proc`](proc.md).
