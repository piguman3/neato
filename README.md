# NEATO

(Norms Expressed As a Tool for Operating (S)ystems)

NEATO is a standard that defines APIs and standard conventions for NEET Computers operating systems and their method
of booting.

The standards are not meant to be a large, strict ruleset, but rather a shim for general program compatibility over
the already existing functions that NEET Computers offers, and to make API changes in NEET Computers an issue for OS
maintainers to resolve, rather than the programs themselves. NEATO aims to include basic APIs which most applications
would need, such as for terminal emulation or standards for how program arguments should work.

An OS may comply with these body of standards however the maintainer should desire to. In order to maintain NEATO
compatibility, an operating system must provide NEATO software an environment which complies with NEATO specifications.

---

### Reasoning

NEET Computers encourages users to write their own operating systems, which means that application compatibility
may not be guaranteed from one operating system to the next. It also raises a problem of writing the code itself,
as developers may need to ship different versions of software for every operating system which the software maintainer
would like to deploy it on. NEATO aims to solve this by providing a body of standards for an operating system to
provide to a program, or, how the operating system itself may boot, for compatibility.

---

### Structure: core and extensions

NEATO is modeled on the way POSIX and RISC-V are organized. There is one small mandatory part, and everything else is
an optional, separately named extension. An operating system picks which extensions it implements and tells
programs which ones those are, so an OS never has to implement a large feature just to be compatible, and a program
never has to guess what it is running on.

There are three kinds of name:

| Name                | Meaning                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------- |
| `core`              | The mandatory base. Every NEATO environment provides it.                                    |
| `ext.<name>`        | A standard extension, defined by a specification in this repository. Example: `ext.screen`. |
| `x.<vendor>.<name>` | A non-standard extension defined by an OS or a third party. Example: `x.neetos.windows`.    |

Names are case sensitive and every segment consists only of letters and digits.

Querying. A program finds out what it is running on with the `sys` API (see [sys.md](api/sys.md)):

```lua
if sys.hasExtension("ext.screen", 1) then
  -- safe to use the screen API
end
```

`sys.getExtensions()` returns every supported extension together with its version. `core` is always present.

An extension that an OS does not implement must not appear to be half there. The OS must
report it as unsupported (`sys.hasExtension` returns `false`), and must not define any global, function or event name
that the extension specification defines. A program can therefore also test for a feature by checking for `nil`.

Every extension, `core` included, has one integer version, starting at 1. The version only changes for a
breaking change. Anything that an OS could reasonably decline to implement is never added to an existing extension, it
becomes a new extension instead. An OS implements exactly one version of each extension it reports, and a program
should check for the exact version it was written for. There is no version for NEATO as a whole.

An extension may require other extensions. An OS that reports an extension must also report
everything it requires.

Names in the `ext.` namespace that are reserved have no specification yet (see [Reserved](#reserved)), and names of
draft extensions (see [Drafts](#drafts)) have one that is not final. An OS must not report either as supported. This
keeps the names free until a specification is written and final.

---

### Extension registry

| Name         | Version | Requires | Specification                                                                                   | Contents                                                                                 |
| ------------ | ------- | -------- | ----------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `core`       | 2       |          | [api/](api/README.md), [common/paths.md](common/paths.md), [common/errors.md](common/errors.md) | Lua environment, `sys`, `event`, `term`, `fs`, `print`, `CWD`, paths, error codes        |
| `ext.screen` | 1       | `core`   | [api/screen.md](api/screen.md)                                                                  | NEET Computers `screen` API, possibly redirected to a window or layer.                   |
| `ext.dpp`    | 2       | `core`   | [network/dpp.md](network/dpp.md)                                                                | Direct Payload Protocol.                                                                 |
| `ext.spp`    | 2       | `core`   | [network/spp.md](network/spp.md)                                                                | Sequenced Payload Protocol: reliable local connections, pollable handles, peer identity. |

#### Drafts

A draft extension has a specification, but it is not final yet. Its specification file carries a draft notice. An OS
must not report a draft extension as supported until that notice is removed, so until then the name is reserved in the
same way as the names below.

| Name            | Version | Requires  | Specification                                  | Contents                                                              |
| --------------- | ------- | --------- | ---------------------------------------------- | --------------------------------------------------------------------- |
| `ext.rpp`       | 1       | `core`    | [network/rpp.md](network/rpp.md)               | Routed Packet Protocol: host addresses and routing between computers. |
| `ext.sppRemote` | 1       | `ext.spp` | [network/spp-remote.md](network/spp-remote.md) | SPP connections between computers (`scope = "network"`).              |

#### Reserved

Reserved: `ext.headsup`, `ext.peripherals`, `ext.chip`, `ext.internet`, and `ext.partitions`. `ext.internet` is the name
for the raw NEET Computers `internet` API (HTTP and WebSocket) and has nothing to do with the internet layer of NEATO
Network, which is `ext.rpp`.

Bootloaders are not part of an operating system's program environment, so they are not extensions. They are covered
by their own specification in [boot/](boot/README.md), which carries its own version number.

---

NEATO program API definitions, like functions or events relating to the NEATO body of standards, are placed in the
`api` folder.

NEATO definitions about the NEATO compliant boot process are placed in the `boot` folder.

Common formats that are shared between NEATO program API definitions and the NEATO boot definitions are defined in
the `common` folder.

NEATO specifications for networking protocols are defined in the `network` folder.

Every specification file states, at its top, which extension it belongs to and that extension's version.
