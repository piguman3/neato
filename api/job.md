# NEATO `job` API Specification

Extension: `ext.job`

Version: 1

Requires: `core`, `ext.proc`

---

A NEATO compliant operating system that reports this extension must provide a global `job` API for grouping programs
into jobs and for choosing which job uses the terminal. [`ext.proc`](proc.md) ends by naming this extension as the
one its footnote waits for. A shell can already run a pipeline and wait for it, but which program receives keys and
the terminate event is, without [`ext.job`](job.md), the operating system's undisclosed decision. This extension adds
the group a pipeline runs as, the answer to who gets the terminal, and a way to end a group as one.

A job is a group of processes. Every process belongs to exactly one job at any moment, from the moment it starts to
the moment it finishes. A process starts in a job of its own that contains only itself, and a program gathers its
children into jobs it creates: the members of a job are the processes a program explicitly put there. Nothing joins
a job by itself, so the members of every job are children of one program, which is the program that has to keep
track of them anyway.

A job id, or jid, is an integer that identifies one job. The operating system chooses jids and may reuse a jid once
the job is gone, as it may a pid. A job is gone when its last member finishes; a job that has never had a member is
gone when the program that created it finishes. Every function of this API that takes a jid fails with `EINVAL` when
no job has it.

```c
job
  create
  join
  current
  getForeground
  setForeground
  setBackground
  kill
```

| Name              | Description                                                                     | Arguments                | Returns                             |
| ----------------- | ------------------------------------------------------------------------------- | ------------------------ | ----------------------------------- |
| job.create        | Creates a job with no members. The calling program becomes its owner.           | none                     | jid (int), or nil, code and message |
| job.join          | Moves one of this program's children into a job this program owns.              | jid (int), pid (int)     | true, or nil, code and message      |
| job.current       | Returns the job this program belongs to.                                        | none                     | jid (int)                           |
| job.getForeground | Returns the jid of the foreground job, or `nil` when no job is foreground.      | none                     | jid (int), or nil                   |
| job.setForeground | Makes a job the foreground job.                                                 | jid (int)                | true, or nil, code and message      |
| job.setBackground | Clears the foreground designation when the job holds it.                        | jid (int)                | true, or nil, code and message      |
| job.kill          | Stops every member of a job. See [Stopping a job](#stopping-a-job).             | jid (int), how (string?) | true, or nil, code and message      |

---

### Jobs and membership

`job.create()` creates an empty job. The calling program is its owner: only the owner may add members to it,
foreground it, background it or stop it as a whole. A program also owns the job it starts in, its initial job, which
contains only itself.

`job.join(jid, pid)` makes the process `pid` a member of the job `jid`, and it stops being a member of whatever job
it was in. A process is a member of exactly one job at a time, and a child may be moved between jobs freely while it
runs; joining it into the job it is already in succeeds and changes nothing. Two rules bound who may join:

- `pid` must name a running child of the calling program. This is the same privilege [`proc.kill`](proc.md) rests
  on: a program may always stop its own children, so a program may always rearrange its own children. Joining the
  caller itself, or anyone else's process, fails with `EACCES`.
- The calling program must own `jid`.

A program that joins its children into its own current job gets a job of itself and everything it started, which is
the shape a wrapper program wants. When a process finishes it stops being a member, and the job may then be gone
under it, with the rule the introduction gives.

---

### Foreground and background

At most one job at a time is the foreground job. Every other job is a background job, and a job is background unless
`job.setForeground` made it the foreground one. Which job, if any, is foreground when the computer starts, and where
input goes while no job is foreground, is up to the operating system.

The rule of this extension: only the programs of the foreground job receive the input events
[event.md](core/event.md) defines, which are `key`, `key_up` and `char`; this is the choice
[event.md](core/event.md) leaves to the operating system, made definable by shrinking the pool to one job. Which of
the foreground job's programs receives each event is still up to the operating system, as [event.md](core/event.md)
allows among programs. This is the decision [`term.readLine`](core/term.md) points at where it says a line is typed
into "the foreground job where the operating system supports job control", and the same rule governs what
[`stdio.readLine`](stdio.md) reads from a terminal.

A background job keeps running: nothing about being background slows it down, and nothing restricts what it may
write to the terminal. It receives no typed input, so a background program reading a terminal keeps waiting, as any
terminal read does; `stdio.readLine` and `term.readLine` included.

Two things the rule does not touch:

- The `terminate` event is not one of the input events, and the foreground rule does not gate it. The operating
  system delivers it to a program's queue whatever job the program is in, which is what
  [stopping a job](#stopping-a-job) relies on.
- There is no handoff beyond the input rule: no event tells a program that its job became foreground or background.
  A program learns it is foreground by receiving input, and a shell simply foregrounds its own job again before it
  reads a prompt, as the example below does.

`job.setForeground(jid)` makes `jid` the foreground job; a job that already is it stays so.
`job.setBackground(jid)` takes the designation away when `jid` holds it, leaving no job foreground until one is set
again; on a job that does not hold it, it succeeds and changes nothing. When the foreground job's last member
finishes, the job is gone and with it the designation: `job.getForeground()` answers `nil` until some job is set
again. `nil` is an ordinary answer here, not a failure, as [errors.md](../common/errors.md) defines for such
answers.

A program may set or clear the designation of a job it owns or of a job it belongs to. Whether any other program may
is up to the operating system; anything it does not allow fails with `EACCES`.

---

### Stopping a job

`job.kill(jid, how)` stops every member of the job, and `how` means what it means for [`proc.kill`](proc.md):

| `how`    | Meaning                                                                                                                                                                                                      |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `"term"` | The default. Every member receives the `terminate` event, exactly as [`proc.kill`](proc.md) delivers it to one program, and is expected to exit. A member that keeps running for too long may then be ended by the operating system. |
| `"kill"` | Ends every member now. They receive no event and cannot clean up.                                                                                                                                            |

A member ended this way, without exiting on its own, reports a nonzero status of at least 128, as
[proc.md](proc.md) defines, and its owner may [`proc.wait`](proc.md) for it as for any child. A member that exits
stops being a member, so the job shrinks as its programs go down and is gone when the last one has.

Only the owner may kill a job, and it needs no permission it does not already have: every member is its child, and
[proc.md](proc.md) lets a program stop its own children, so the owner could do with [`proc.kill`](proc.md) alone
what `job.kill` does to all of them at once. An operating system may let anyone else do it, typically a matter of
being an administrator (see [`ext.user`](user.md)); anything else fails with `EACCES`.

---

### Failure codes

| Function          | Condition                                                                                            | Code      |
| ----------------- | ---------------------------------------------------------------------------------------------------- | --------- |
| job.create        | a limit on jobs is reached                                                                           | `ELIMIT`  |
| job.join          | no job has that jid                                                                                  | `EINVAL`  |
| job.join          | no running program has that pid                                                                      | `ESRCH`   |
| job.join          | the pid is not a child of this program, or the job is not owned by this program                      | `EACCES`  |
| job.join          | the child cannot be moved for another reason                                                         | `EIO`     |
| job.current       |                                                                                                      | (never fails) |
| job.getForeground |                                                                                                      | (never fails) |
| job.setForeground | no job has that jid                                                                                  | `EINVAL`  |
| job.setForeground | the job is neither owned by this program nor a job this program belongs to, and the operating system does not allow it | `EACCES` |
| job.setBackground | no job has that jid                                                                                  | `EINVAL`  |
| job.setBackground | the job is neither owned by this program nor a job this program belongs to, and the operating system does not allow it | `EACCES` |
| job.kill          | no job has that jid                                                                                  | `EINVAL`  |
| job.kill          | the job is not owned by this program, and the operating system does not allow it                     | `EACCES`  |

---

### Example usage

```lua
-- A shell: run a pipeline as one job and hand the terminal to it.
local jid = assert(job.create())
local r, w = proc.pipe()
local pa = assert(proc.spawn(a, nil, { stdout = w }))
w:close()                                 -- this program does not write into the pipe
local pb = assert(proc.spawn(b, nil, { stdin = r }))
r:close()                                 -- and does not read from it
assert(job.join(jid, pa))
assert(job.join(jid, pb))
assert(job.setForeground(jid))            -- the pipeline gets the keys now
assert(proc.wait(pb))                     -- the shell waits, and reads nothing meanwhile
assert(job.setForeground(job.current()))  -- and takes the terminal back for its prompt
```

```lua
-- The user has given up on a background job: end it as one.
assert(job.kill(jid, "kill"))
```
