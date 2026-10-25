# Lame `os` API Specification

Extension: `x.lame.os`

Version: 1

Requires: `core`, `env`

Note: Vendor-specific, can be implemented by any OS but is fully dictated by one vendor (Lame)

---

Any NEATO environment that implements this extension must provide this NEET-specific reimplementation of the Lua `os` library.

Some of the library is the same as is described in these Lua manuals (https://www.lua.org/pil/22.1.html and https://www.lua.org/manual/5.5/) but a non-zero amount of it has been changed to be more similar to how Computercraft implements it.

Implementations that are different, have special requirements, or are not in the vanilla lua library are described below.

| Name        | Description                                                                                                                                  | Arguments                       | Returns                    |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- | -------------------------- |
| os.version  | Identical to NEET `chip.version`, returns the current NEET Computers version                                                                 | none                            | none                       |
| os.getenv   | Returns the environment variable `key`, must use the same global environment as the `env` extension and follow the same rules as `env.get`.  | key (string)                    | value (string?)            |
| os.setenv   | Sets the environment variable `key`, must use the same global environment as the `env` extension and follow the same rules as `env.set`.     | key (string), value (string)    | none                       |
| os.listenv  | Lists all currently set environment variables, must use the same global environment as the `env` extension                                   | none                            | table [string -> string]   |
| os.exit     | Exits the currently running program immedietly. If the OS supports exit codes then it may take an option exit code argument.                 | exit (number?)                  | none                       |
| os.execute  | More adjacent to CC's `os.run` than the vanilla Lua function. Launches the program at the exact path `path` with the args `args...`          | path (string), args... (vararg) | none                       |
| os.shutdown | Shuts the computer down.                                                                                                                     | none                            | none                       |
| os.reboot   | Reboots the computer.                                                                                                                        | none                            | none                       |

The following functions are unchanged from vanilla Lua. Please see the manuals linked for exact specifications.

- `os.time(unixTimestamp?)`
- `os.date(format, unixTimestamp?)` 
- `os.remove(file)`
- `os.rename(from, to)`
- `os.tmpname()`

For consistency, you should also implement the NEATO `ext.time` extension as it shares a large part of its code with `os.time` and `os.date`