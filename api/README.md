# NEATO API

Program API environment specifications.

The NEATO API consists of the default \_G, events and handlers of the provided NEET Computers API, with changes and additions
defined by the specifications. Which parts a given operating system provides is determined by `core` and the extensions
it reports, as described in the [main README](../README.md).

This does not require the API to explicitly behave the same as stock NEET Computers API, or that it cannot have additions
from an operating system, but rather that the expected parameters and return types are the same across all OSes.

For example, in a GUI based operating system with applications that each get their own window, it may modify the behavior
of the terminal to draw into the application's window, and the graphics API to use a specific layer for the window,
rather than the main global layer, and remain NEATO compliant.

Arguably the most important spec in the NEATO API is [globals](globals.md), as it defines the entirety of the NEATO environment for clarity, including things that might not have explicit API specs for whatever reason.

### Replacements for the default API

NEATO does not expose the raw NEET Computers `event` and `files` APIs, because they give programs direct access to
the "hardware" with no protection. NEATO defines its own [event](event.md) and [fs](fs.md) APIs in `core` instead, and
an OS maps them onto whatever it has underneath. The other raw NEET APIs (`headsup`, `io`, `chip`, `internet`) are
reserved extension names until they are abstracted.

### `core` (always available)

- [Globals (globals.md)](globals.md) - The complete global environment, including the program model: the arguments a
  program receives and the exit status it returns.
- [Terminal emulation (term.md)](term.md) - Functions for a standardized terminal emulator.
- [System info (sys.md)](sys.md) - OS name and version, extension queries, and sleeping.
- [Events (event.md)](event.md) - Input and other events delivered to a program.
- [Files (fs.md)](fs.md) - Files and directories, using the [paths](../common/paths.md) format, and [filesystem points](fs.md#filesystem-points).
- [Current working directory (cwd.md)](cwd.md) - The directory the application was launched in.
- [Cryptography (crypto.md)](crypto.md) - Cryptography functions.
- [Error codes (../common/errors.md)](../common/errors.md) - The shared codes every failing API returns.

### Extensions

- [Standard streams (stdio.md)](stdio.md) - `ext.stdio`
- [Environment (env.md)](env.md) - `ext.env`
- [Time (time.md)](time.md) - `ext.time`
- [Processes (proc.md)](proc.md) - `ext.proc`
- [Users (user.md)](user.md) - `ext.user`
- [System (system.md)](system.md) - `ext.system`
- [Permissions (perms.md)](perms.md) - `ext.perms`
- [Screen (screen.md)](screen.md) - `ext.screen`
- [Direct Payload Protocol (../network/dpp.md)](../network/dpp.md) - `ext.dpp`
- [Sequenced Payload Protocol (../network/spp.md)](../network/spp.md) - `ext.spp`
- [Routed Packet Protocol (../network/rpp.md)](../network/rpp.md) - `ext.rpp` (draft)

### Extensions that extend `fs`

Some extensions define no global table of their own and add functions to the [`fs`](fs.md) API instead. An
operating system defines those functions exactly when it reports the extension, so `sys.hasExtension` and a check
for `nil` both work.

- [Permissions (perms.md)](perms.md) - `ext.perms`, adds `fs.access`, `fs.chmod`, `fs.chown`, `fs.umask`.
- [Symbolic links (symlink.md)](symlink.md) - `ext.symlink` (draft), adds `fs.symlink`, `fs.readlink`.
- [Filesystem usage (statfs.md)](statfs.md) - `ext.statfs` (draft), adds `fs.statfs`.
