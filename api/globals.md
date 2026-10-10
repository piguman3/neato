# NEATO globals API Specification

Extension: `core`

Version: 3

---

This file is a general specification of the entire global environment that a NEATO compatible application should
receive. Some APIs might only be defined here and nowhere else, as they might be really close to either default NEET or
Lua functions. Otherwise, the relevant API spec will be referenced in a comment (`// like this`).

*Note: since NEATO is still in development, this page will definitely receive a lot of changes in the future*

The environment is what `core` provides plus, for every extension the operating system reports through
`sys.getExtensions`, the globals that extension defines. An environment must not contain a global that belongs to an
extension it does not report. This applies to the raw NEET Computers APIs as well: `files`, `headsup`, `io`, `chip`
and `internet` are not part of a NEATO environment. Their names are only available through the extensions that will
replace them (the reserved names in the [main README](../README.md)). The name `event` is used by NEATO, but it is the
NEATO [event](event.md) API and not the NEET Computers one.

A global function that can fail reports the failure as `nil`, an error code, then a message, as defined in
[errors.md](../common/errors.md). A wrong argument type raises an error instead. Each API specification lists the exact
codes its functions must use.

### NEATO `core` globals

```c
// Defined in api/cwd.md
CWD

// Defined in api/sys.md
sys
  getOSName
  getOSVersion
  getExtensions
  hasExtension
  sleep

// Defined in api/event.md
event
  pull
  poll
  push
  clear

// Defined in api/fs.md
fs
  exists
  isFile
  isDir
  list
  makeDir
  delete
  open
  resolve
  rename
  stat
  getDisks
  getPartitions
  getPoint
  getPoints

// Defined in api/crypto.md
crypto

// Defined in api/term.md
term
  clear
  write
  getSize
  clearLine
  setCursorPos
  getCursorPos
  setBackgroundColor
  getBackgroundColor
  setTextColor
  getTextColor
  scroll
print
```

### The program

A NEATO program is a Lua chunk that the operating system loads and runs in the environment described in this file.
`core` defines how the program is launched and how it ends.

- **Arguments.** The chunk receives the program's arguments as its vararg expression `...`, one string per argument,
in the order they were given. The program's own name is not included.
- **Exit status.** When the chunk returns, its first result is the program's exit status: an integer from 0 to 255
that the operating system may report onward, for example to a shell. Any other result, including no result at all,
is a status of 0, and an integer outside the range is taken modulo 256. A status of 0 means success and any other
status means the program chose to report a failure.
- **Uncaught errors.** An error that is not caught ends the program with a nonzero status of the operating system's
choice. If [`ext.stdio`](stdio.md) is reported, the operating system may write a diagnostic to `stdio.stderr`.
- **Termination.** The `terminate` event (see [event.md](event.md)) asks the program to stop. The operating system
must deliver the event first and let the program react; a program that keeps running may afterwards be ended by the
operating system, and its status is then an OS-defined value of at least 128.

### Extension globals

```c
// ext.stdio, defined in api/stdio.md
stdio
  stdin
  stdout
  stderr
  isatty
  readLine

// ext.env, defined in api/env.md
env
  get
  set
  all

// ext.time, defined in api/time.md
time
  monotonic
  wall
  date

// ext.proc, defined in api/proc.md
proc
  getpid
  getppid
  exit
  spawn
  wait
  poll
  kill
  pipe

// ext.user, defined in api/user.md
user
  current
  byName
  byId
  list

// ext.system, defined in api/system.md
system
  hostname
  setHostname
  machine
  uptime

// ext.screen, defined in api/screen.md
screen

// ext.dpp, defined in network/dpp.md
dpp
  listen
  unlisten
  send

// ext.spp, defined in network/spp.md
spp
  listen
  connect
  poll
  getLimits

// ext.rpp, defined in network/rpp.md
rpp
  getAddress
  getInterfaces
  getNeighbors
  getRoutes
  isRouter
  setRouter
  ping
```

The extensions [`ext.perms`](perms.md), [`ext.symlink`](symlink.md) and [`ext.statfs`](statfs.md) define no global
table of their own: they add functions to the `fs` API instead, as [fs.md](fs.md) describes.

### Reserved core names (not yet specified)

`require` and `loadfile` are moved from the default Lua environment and are still to be defined, and `requestNeato` is
also still to be defined. Until they are, an environment must not define these names as anything other than what a
future specification says.

```c
require
loadfile
requestNeato
```

### Default Lua global variables (mostly unchanged)

Some things have been moved (require, loadfile, print) due to the change in functionality, or because NEET doesn't include them, but most of these should be OK as they are basically just backend things that don't interact with the "hardware" of the computer.

```c
rawget
rawequal
rawlen
rawset
setmetatable
getmetatable
tostring
tonumber
pcall
xpcall
type
load
pairs
ipairs
next
select
error
assert

_G
_VERSION Lua 5.5 string

utf8
  charpattern
  offset
  codepoint
  codes
  len
  char

table
  sort
  concat
  unpack
  remove
  pack
  move
  create
  insert

math
  acos
  atan
  huge inf number
  mininteger -9223372036854775808 number
  log
  rad
  maxinteger 9223372036854775807 number
  ult
  random
  randomseed
  type
  frexp
  asin
  sqrt
  ceil
  min
  tan
  cos
  tointeger
  abs
  sin
  floor
  modf
  max
  ldexp
  pi 3.1415926535897931 number
  exp
  fmod
  deg

bit32
  extract
  lrotate
  lshift
  arshift
  band
  btest
  rshift
  rrotate
  replace
  bxor
  bor
  bnot

string
  find
  pack
  dump
  byte
  char
  match
  reverse
  packsize
  format
  unpack
  upper
  sub
  rep
  lower
  gsub
  len
  gmatch

coroutine
  resume
  memoryused
  running
  yield
  close
  fork
  wrap
  isyieldable
  create
  status
```
