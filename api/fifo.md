# NEATO `fifo` API Specification

Extension: `ext.fifo`

Version: 1

Requires: `core`

---

A NEATO compliant operating system that reports this extension must provide named pipes in its [`fs`](core/fs.md)
filesystem: `fs.mkfifo` creates one at a path, and programs open it with [`fs.open`](core/fs.md#opening-files) like
any other object at a path. It is what lets two programs connect a data stream when they share nothing else. The
ends of an anonymous pipe travel only where [`ext.proc`](proc.md) hands them, through `proc.spawn` to a child; a
named pipe needs no common parent, because both sides find each other through a name in the filesystem instead.

A named pipe is a pipe with a name. Nothing is stored on the disk: the object at the path is a queue in the
operating system's memory. The name exists from `fs.mkfifo` until [`fs.delete`](core/fs.md) removes it or the
computer stops, and it is not kept across reboots, like the data in the `storage.runtime`
[point](core/fs.md#filesystem-points): no named pipe exists when the computer starts again, and `fs.mkfifo` is the
only way one comes to be.

This extension adds one function to the `fs` API. An operating system defines it exactly when it reports this
extension, as [fs.md](core/fs.md) describes.

| Name      | Description                       | Arguments     | Returns                        |
| --------- | --------------------------------- | ------------- | ------------------------------ |
| fs.mkfifo | Creates a named pipe at a path.   | path (string) | true, or nil, code and message |

`fs.mkfifo(path)` creates a named pipe at `path`, in the [paths](../common/paths.md) format. The parent directory
must already exist, and nothing may already be at the path. Creating the pipe writes nothing to storage, so there is
no space condition to fail on.

Until it is deleted, the name behaves like the name of any other object: `fs.exists` answers `true` for it,
`fs.isFile` and `fs.isDir` answer `false`, `fs.list` includes it, and [`fs.stat`](core/fs.md#file-information)
reports it with the `type` `"other"`, which [fs.md](core/fs.md#file-information) uses for everything that is not a
file or a directory, and with no `size`. [`fs.rename`](core/fs.md#renaming) moves the name as it moves any object,
and [`fs.delete`](core/fs.md) removes it; programs that have the pipe open keep it through either, the way a
renamed file stays the same file its open handles see.

---

### Opening a fifo

[`fs.open`](core/fs.md#opening-files) opens a named pipe like a file: the path names the fifo, `mode` is one of the
modes fs.md defines, and the trailing `b` is accepted and has no effect. When the object at the path is a named
pipe, the modes mean:

| Mode | Meaning                                                                         |
| ---- | ------------------------------------------------------------------------------- |
| "r"  | Opens the reading end, and waits until some program opens the same fifo for writing. |
| "w"  | Opens the writing end, and waits until some program opens the same fifo for reading. |
| "a"  | Accepted and behaves exactly like `"w"`: a pipe has no position to append after, so every write goes to the end of the data in the pipe anyway. |

The first program to open waits, and when the matching end is opened, both `fs.open` calls return their handles. The
waiting is not a failure and has no timeout. `fs.open` yields while it waits, and events that arrive stay in the
event queue, as they do while [`proc.wait`](proc.md#waiting) waits; the `EINTR` rule of
[`stdio.readLine`](stdio.md#reading-a-line) is special to that function and does not apply here. If the fifo stops
existing while a program waits, its `fs.open` fails with `ENOENT`.

When the object at the path is not a named pipe, `fs.open` does what [fs.md](core/fs.md) says, and nothing in this
extension changes it: `"r"` on a path with nothing there fails with `ENOENT`, and `"w"` and `"a"` create a regular
file, which is not a fifo. A program that means to open a fifo creates it with `fs.mkfifo` first.

---

### Reading and writing

A fifo end is a handle with the same five methods, called with `:`, as the handles [`fs.open`](core/fs.md#opening-files)
returns, which is how [stdio.md](stdio.md) describes the standard streams, and it follows the same rules: a wrong
argument type raises an error, a method on a closed handle raises an error, and a refusal returns `nil`, then a code
from [errors.md](../common/errors.md), then a message. What the connection between the ends does is what a pipe of
[proc.md](proc.md#pipes) does:

- Writing copies the data into the pipe, and waits while the pipe is full. A fifo holds at least 4096 bytes before a
  write has to wait, like a pipe.
- Reading returns the data in the order it was written, and `nil` once the pipe is empty and every writing end is
  gone, which is an ordinary end of file, not a failure.
- Writing fails with `EPIPE` once every reading end has been closed or every reading program has finished.
- `handle:seek` fails with `ESPIPE`: a pipe has no position to move to.

More than one program may hold each kind of end. The readers share what is written: every byte goes to exactly one
of them, handed out in order to whichever reader reads next, and what one reader reads the others do not. Closing is
what ends the connection: closing the last writing end gives the readers an ordinary end of file once they have read
the rest, and closing the last reading end makes every further write fail with `EPIPE`. Handles still open when a
program ends are closed by the operating system, as [fs.md](core/fs.md) says, which is what "every reading program
has finished" counts. When every end is closed, whatever was still unread is dropped, and the next two programs to
open the name start with an empty pipe.

`handle:flush` succeeds without doing anything: nothing is held back that storage has to take.

---

### Failure codes

| Function     | Condition                                                  | Code           |
| ------------ | ---------------------------------------------------------- | -------------- |
| fs.mkfifo    | something already exists at the path                       | `EEXIST`       |
| fs.mkfifo    | a parent of the path does not exist                        | `ENOENT`       |
| fs.mkfifo    | a component of the path is not a directory                 | `ENOTDIR`      |
| fs.mkfifo    | the fifo is not allowed to be created                      | `EACCES`       |
| fs.mkfifo    | the filesystem is read-only                                | `EROFS`        |
| fs.mkfifo    | the operating system is out of pipes                       | `ELIMIT`       |
| fs.mkfifo    | the name is too long                                       | `ENAMETOOLONG` |
| fs.mkfifo    | the fifo cannot be created for another reason              | `EIO`          |
| fs.open      | the fifo stops existing while the open waits for the other end | `ENOENT`   |
| handle:write | every reading end is closed or gone                        | `EPIPE`        |
| handle:seek  | the handle is a fifo, which cannot seek                    | `ESPIPE`       |

---

### Example usage

```lua
-- the writer side; the fifo must exist before either side opens it
assert(fs.mkfifo("feed"))
local w = assert(fs.open("feed", "w"))    -- waits until a reader arrives
w:write("hello\n")
w:close()                                 -- the reader sees end of file after this
```

```lua
-- the reader side; in another program with nothing in common with the writer:
local r = assert(fs.open("feed", "r"))    -- waits until a writer arrives
print(r:read("l"))
  -> hello
r:close()
```
