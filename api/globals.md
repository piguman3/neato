# NEATO globals API Specification

Written by piguman3

Revision 1 of September 18, 2026

---

This file is a general specification of the entire global environment that a NEATO compatible application should receive. Some APIs might only be defined here and nowhere else, as they might be really close to either default NEET or Lua functions. Otherwise, the relevant API spec will be referenced in a comment (`// like this`).

*Note: since NEATO is still in development, this page will definitely receive a lot of changes in the future*

### NEET specific (ones modified or removed by NEATO aren't on this list)

Everything marked as "Unfinished" still needs to be thought out more, as we need to allow OSes to build abstractions and privilige layers, but most of these functions access the "hardware" directly with no protections.

```c
// Unfinished
event
  getQueue
  queueEvent
  getFirst
  clear

// Unfinished
files
  getPartition
  isFile
  open
  getDiskID
  delete
  getPartitions
  makeDir
  setPartitionReadOnly
  isDir
  getDisks
  getChildren
  removeDisk
  setPartitionHidden
  getNumberOfDisks
  exists
  deletePartition
  getBootPath
  createPartition
  setBoot

// Unfinished
headsup
  drawLine
  clear
  drawRec
  draw
  getSize
  drawPixel

// Unfinished
io
  getPeripherals
  isCompatibility
  broadcastLocal
  getType
  queryTag
  setTag
  wrapPeripheral
  getTag
  callFunction
  queryType

// Backend only functionality, probably fine to include in NEATO envs by default
crypto
  AES
    Encrypt
    GenerateKeyFromPassword
    GenerateSalt
    GenerateKey
    GenerateIv
    Decrypt
  Base64
    Decode
    Encode
  SecureRNG
    GetRandomFromMin
    GetRandomUpTo
    GetRandom
    GetRandomBetween
  RSA
    Encrypt
    Decrypt
    Verify
    Sign
    GenerateKeyPair
  Hash
    MD5
    SHA256

// Unfinished
chip
  version
  getMachine
  shutdown
  getUnixTime
  getTime
  getUUID
  crash
  reboot
  getLunarTime

// Unfinished
internet
  POST
  isReady
  hasAccess
  GET
  CreateWebsocket

// API stays the same, doesn't necessarily have to access the main screen though
screen
  readData
  draw
  clone
  writePixel
  writeLine
  readPixel
  writeData
  createLayer
  substitute
  fill
  getSize
  set
```

### NEATO additions/modifications

Pretty much there, except for "TBD" (To be defined) APIs, should be pretty simple to write specs for, though.

```c
// Defined in api/cwd.md
CWD

// Defined in api/sys.md
sys
  getOSName
  getOSVersion
  getNEATOCompat
  sleep

// Defined in api/term.md
term
  clear
  write
  getSize
  clearLine
  setCursorPos
  setBackgroundColor
  setTextColor
  setCursorPos
  setBackgroundColor
  setTextColor
  scroll
print

// TBD
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