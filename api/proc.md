# NEATO `proc` API Specification

Extension: `ext.proc`

Version: 1

Requires: `core`, `ext.stdio`, `ext.env`

---

A NEATO compliant operating system that reports this extension must provide a global `proc` API for starting,
watching and stopping other programs. It is what turns a pile of single-purpose programs into a system: a shell
runs a pipeline by starting each step connected to the next one, `xargs` starts a program once per line of input,
and a wrapper such as `nice` or `timeout` is just a program that starts another one.

Every program that this API starts is an ordinary NEATO program: a Lua chunk launched exactly as
[globals.md](globals.md#the-program) describes, with the program model of `core`, the streams of
[`ext.stdio`](stdio.md) and the environment of [`ext.env`](env.md). There is no second kind of program.

| Name        | Description                                                                                    | Arguments                                             | Returns                                                    |
| ----------- | ---------------------------------------------------------------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| proc.getpid | Returns this program's process id.                                                             | none                                                  | pid (int)                                                  |
| proc.getppid| Returns the process id of the program that started this one, or 0 once that program has finished.| none                                                 | pid (int)                                                  |
| proc.exit   | Ends this program immediately with the given exit status. Does not return.                      | status (int?)                                         |                                                            |
| proc.spawn  | Starts a program. See [Starting a program](#starting-a-program).                                | path (string), args (table?), opts (table?)           | pid (int), or nil, code and message                        |
| proc.pipe   | Creates a pipe: two connected ends, what is written to one can be read from the other.          | none                                                  | readEnd (handle), writeEnd (handle)                        |
| proc.wait   | Waits until a child of this program finishes, and returns its exit status.                      | pid (int?)                                            | pid (int), status (int), or nil, code and message          |
| proc.poll   | Like proc.wait, but returns `nil` immediately when nothing has finished.                         | pid (int?)                                            | pid (int), status (int), or nil, or nil, code and message  |
| proc.kill   | Asks another program to stop, or stops it. See [Stopping a program](#stopping-a-program).        | pid (int), how (string?)                              | true, or nil, code and message                             |

A process id, or pid, is an integer that identifies one running program. The operating system chooses pids and may
reuse the pid of a program that has finished and been waited for. A program that has finished keeps its pid, as a
finished child, until a parent waits for it; the parent does not have to.

---

### Starting a program

`proc.spawn(path, args, opts)` starts a program and returns its pid. It returns as soon as the child has been
started, not when the child finishes; the two programs then run at the same time.

`path` says what to start. If it contains `/` or `:` it is a path to the program file, in the
[paths](../common/paths.md) format, relative or full. Otherwise it is a bare name, such as `"grep"`, and the
operating system looks for a file of that name in each directory of the `PATH` variable, in order, using the
convention [env.md](env.md) defines. Nothing found is `ENOENT`.

The child receives `args` as its arguments, through `...`, exactly as [globals.md](globals.md#the-program)
defines. `args` is `nil` or a table; each element must be a string, or a number, which is converted to a string;
anything else raises an error. An argument list that is too long for the operating system fails with `ELIMIT`.

`opts` is `nil` or a table with any of the following fields. Anything the operating system does not recognize is
ignored; an operating system may accept additional fields of its own under names beginning with `x.`.

| Field     | Type            | Meaning                                                                                                                        |
| --------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `cwd`     | string          | The child's [`CWD`](cwd.md). It must be an existing directory when the child starts. The default is this program's `CWD`.        |
| `env`     | table           | The child's environment, from name to value. It replaces the inherited one entirely. The default is a copy of this program's environment. |
| `stdin`   | handle          | The child's standard input. Must be open for reading. The default is this program's standard input.                             |
| `stdout`  | handle          | The child's standard output. Must be open for writing. The default is this program's standard output.                           |
| `stderr`  | handle          | The child's standard error. Must be open for writing. The default is this program's standard error.                             |

The environment and option tables are copied the moment `proc.spawn` is called; changing them afterwards does
nothing to the child.

The three streams are shared, not copied: a file opened through a stream that two programs share has one position,
and moving it in either program moves it for both. No other handle is inherited: the child sees exactly the three
streams it was given, and this program keeps all of its own. A program that hands a pipe end to a child should
close its own copy of it.

The program file must be something the operating system can load and run as a NEATO program. When
[`ext.perms`](perms.md) is reported, its permission bits must allow execution. The child's process limit may refuse
the spawn with `ELIMIT`.

---

### Pipes

`proc.pipe()` returns two handles that are connected to each other: everything written to the second can be read
from the first. They are streams like the ones [`ext.stdio`](stdio.md) describes, and they are what connects the
standard output of one program to the standard input of the next.

Writing to a pipe whose reader still exists copies the data into the pipe, and waits while the pipe is full. When
every reading end has been closed or every reading program has finished, writing fails with `EPIPE`. A pipe holds
at least 4096 bytes before a write has to wait. Reading returns the data in the order it was written, and `nil`
once the pipe is empty and every writing end is gone, which is an ordinary end of file, not a failure. Pipes
cannot be seeked: `handle:seek` fails with `ESPIPE`.

Either end may be given to `proc.spawn` as one of the child's streams, and both ends may be given to children of
the same parent, which is how a pipeline is built.

---

### Waiting

`proc.wait` blocks until one of this program's children has finished, and then returns that child's pid and its
exit status as defined by [globals.md](globals.md#the-program). With no pid, any finished child is returned; with a
pid, that child is. Events that arrive while waiting stay in the event queue, as with
[`sys.sleep`](sys.md).

`proc.poll` does the same without waiting: it returns the pid and status of a finished child, or `nil` when every
child is still running. `nil` with no code is an ordinary answer, not a failure.

Both functions reap the child they return: it is gone afterwards, and waiting for it again fails with `ECHILD`.
Asking about a pid that is not a child of this program, or a child that has already been reaped, fails with
`ECHILD`. A program cannot wait for grandchildren, or for a program someone else started.

When a program finishes, its children keep running; who may wait for them afterwards is up to the operating
system.

---

### Stopping a program

`proc.kill(pid, how)` asks the program with that pid to stop.

| `how`       | Meaning                                                                                                                                              |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"term"`    | The default. Asks politely: the target receives the `terminate` event, exactly as if the user had asked it to stop, and is expected to exit. If it keeps running for too long, an operating system may then end it. |
| `"kill"`    | Ends the target now. It receives no event and cannot clean up.                                                                                          |

A program stopped this way, without exiting on its own, reports a nonzero status of at least 128, chosen by the
operating system. A program may stop itself: `"kill"` on its own pid ends it, and `"term"` on its own pid is the
same as receiving the terminate event.

A program may always stop its own children. Who else it may stop, if anyone, is up to the operating system, and is
typically a matter of being an administrator (see [`ext.user`](user.md)); anything it may not stop fails with
`EACCES`. A pid that names no program fails with `ESRCH`.

---

### Failure codes

| Function   | Condition                                                   | Code           |
| ---------- | ----------------------------------------------------------- | -------------- |
| proc.spawn | the program file does not exist                             | `ENOENT`       |
| proc.spawn | the path is a directory                                     | `EISDIR`       |
| proc.spawn | a component of the path is not a directory                  | `ENOTDIR`      |
| proc.spawn | the program file is not allowed to be run                   | `EACCES`       |
| proc.spawn | the file is not a program this operating system can run     | `ENOEXEC`      |
| proc.spawn | the `cwd` does not exist, or the given stream is open the wrong way | `EINVAL` |
| proc.spawn | the argument list is too long, or a process limit is reached | `ELIMIT`      |
| proc.spawn | the name is too long                                        | `ENAMETOOLONG` |
| proc.spawn | the program cannot be started for another reason            | `EIO`          |
| proc.pipe  | the operating system is out of pipes                        | `ELIMIT`       |
| proc.wait  | the pid is not a child of this program, or was already reaped | `ECHILD`     |
| proc.poll  | the pid is not a child of this program, or was already reaped | `ECHILD`     |
| proc.kill  | no program has that pid                                     | `ESRCH`        |
| proc.kill  | this program may not stop that one                          | `EACCES`       |

---

### Example usage

```lua
-- Run a command given as arguments, like a shell would, and exit with its status.
local args = table.pack(...)
local pid = assert(proc.spawn(args[1], { table.unpack(args, 2, args.n) }))
local _, status = assert(proc.wait(pid))
proc.exit(status)
```

```lua
-- A pipeline: everything the first program writes, the second reads.
-- Usage: pipe a b  -- starts "a" with its output connected to "b"'s input.
local a, b = ...
local r, w = proc.pipe()
local pa = assert(proc.spawn(a, nil, { stdout = w }))
w:close()                       -- this program does not write into the pipe
local pb = assert(proc.spawn(b, nil, { stdin = r }))
r:close()                       -- and does not read from it
assert(proc.wait(pa))
local _, status = assert(proc.wait(pb))
proc.exit(status)
```

```lua
-- Stop a child that takes too long.
local pid = assert(proc.spawn("backup", nil))
if not proc.poll(pid) then
  sys.sleep(30)
  if not proc.poll(pid) then
    proc.kill(pid, "kill")
  end
end
```

---

### Footnotes

On process groups, sessions and job control: a shell can run a pipeline and wait for it with this extension alone,
and the operating system decides which program receives keys and the terminate event, as
[event.md](event.md) allows. Anything finer, such as moving a running job between the foreground and the
background, needs more than this specification gives, and is left to a future extension.
