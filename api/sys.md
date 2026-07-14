# NEATO `sys` API Specification

Written by UsUsStudios

Revision 2 of July 12, 2026

---

A NEATO compliant operating system must expose a global API in `_G.sys` to retrieve system information about the running operating system and computer, defined below.

|name|description|arguments|returns|
|-|-|-|-|
| sys.getOSName | Returns the name of the active operating system that exposed the `sys` API. | none | The OS name (str) |
| sys.getOSVersion | Returns the version of the active operating system. It is reccomended to use [semvar](https://semver.org/) versioning. | none | The OS version (str) |
| sys.getNEATOCompat | Returns the latest released NEATO version that the OS is fully compatible with, or 0 if the OS is incompatible with any released version.   | none | The NEATO version number (int), the revision number (int)  |

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
print(sys.getNEATOCompat())
  -> 1  3
  (version  revision)
```
