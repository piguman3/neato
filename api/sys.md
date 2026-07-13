# NEATO `Sys` API Specification

Written by UsUsStudios

Revision 2 of July 12, 2026

---

A NEATO compliant operating system must expose a global API in `_G.Sys` to retrieve system information about the running operating system and computer, defined below.

|name|description|arguments|returns|
|-|-|-|-|
| GetOSName | Returns the name of the active operating system that exposed the `Sys` API. | none | The OS name (str) |
| GetOSVersion | Returns the version of the active operating system. It is reccomended to use [semvar](https://semver.org/) versioning. | none | The OS version (str) |
| GetNEATOCompat | Returns the latest released NEATO version that the OS is fully compatible with, or 0 if the OS is incompatible with any released version.   | none | The NEATO version number (int), the revision number (int)  |

---

### Example usage

```lua
print(Sys.GetOSName())
  -> "OS Name"
```
```lua
print(Sys.GetOSName()) 
  -> "v1.5.4"
```
```lua
print(Sys.GetNEATOCompat())
  -> 1  3
  (version  revision)
```
