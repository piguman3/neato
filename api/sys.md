# NEATO `sys` API Specification

Written by UsUsStudios

Extension: `core`

Version: 2

---

A NEATO compliant operating system must expose a global API in `_G.sys` to retrieve system information about the
running operating system and computer, defined below.

| name              | description                                                                                                                                                           | arguments                           | returns                                      |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- | -------------------------------------------- |
| sys.getOSName     | Returns the name of the active operating system that exposed the `sys` API.                                                                                           | none                                | The OS name (str)                            |
| sys.getOSVersion  | Returns the version of the active operating system. It is recommended to use [semver](https://semver.org/) versioning.                                                | none                                | The OS version (str)                         |
| sys.getExtensions | Returns every NEATO extension that the OS supports, as a table from the extension's name to its version. `core` is always in it. See the [main README](../README.md). | none                                | Supported extensions (table from str to int) |
| sys.hasExtension  | Returns whether the OS supports an extension. If `version` is given, it is only `true` if the OS supports exactly that version of it.                                 | name (str), version (int, optional) | boolean                                      |
| sys.sleep         | Repeatedly yields until a given number of seconds has passed. Events that arrive while sleeping stay in the event queue.                                              | time (number)                       | nil, or nil, code and message                |

`sys.sleep` fails with `EINVAL` if `time` is negative. On success it returns `nil`; on failure it returns `nil`, then
the code, then a message, as defined in [errors.md](../common/errors.md). The other `sys` functions cannot fail.

---

### Example usage

```lua
print(sys.getOSName())
  -> "OS Name"
```

```lua
print(sys.getOSVersion())
  -> "v1.5.4"
```

```lua
print(sys.hasExtension("ext.dpp"))
  -> false
```

```lua
print(sys.hasExtension("core", 2))
  -> true
```

```lua
for name, version in pairs(sys.getExtensions()) do
  print(name, version)
end
  -> core  2
     ext.screen  1
```
