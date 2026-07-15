# NEATO Bootloader Specification

Written by SpSf

Revision 2 of July 12, 2026

---

A NEATO compatible bootloader must obtain the `boot.lua` file from `any:system:/boot/cfg/boot.lua`.

A NEATO compatible boot.lua might appear like this:

```lua
return {
  ["Config"] = {
    ["DefaultEntry"] = 1,
    ["Autoboot"] = "none",
  },
  ["Bootlist"] = {
    {
      ["OS Name"] = "NeetOS",
      ["OS Version"] = "0.2.4",
      ["OS Description"] = "The NeetComputers Operating System",
      ["OS Args"] = {"-v", "-f"},
      ["OS Boot Path"] = "0:bios:/boot.lua",
      ["OS Environment Variable Definition"] = {
        ["EXAMPLEVAR"] = 0,
        ["VAR2"] = "Hi",
      },
    },
    {
      ...
    },
  },
}
```

### Bootlist

NEATO compatible bootloaders must support ALL fields defined, as well as supporting multiple operating systems defined in `boot.lua`.

The `OS Args` field must take in either a string, e.g. "EXAMPLEARG=1" or a table (as shown).

Arguments must be passed into the boot path as provided, so a table must pass into the boot path as ("-v", "-f") for example, or as a raw string.

The `OS Environment Variable Definition` field must create environment variables for the boot path, as provided. This can be done different ways, but the boot path MUST be able to see defined varaibles in _ENV. So, for the provided example, `0:bios:/boot.lua` would be able to see `_ENV.EXAMPLEVAR` as `0`, and `VAR2` as "Hi".

---

### Config

NEATO compatible bootloaders must respect the Config field.

`Config.DefaultEntry` is 1-indexed as how Lua tables are indexed. `Config.DefaultEntry` controls which boot entry is first shown and selected initially when the bootloader is opened.

`Config.Autoboot` is 1-indexed as how Lua tables are indexed. `Config.Autoboot` if set to `"none"` or any value other than a valid index into `Bootlist` disables autoboot. If set to an index of `Bootlist` then automatically boots into it. NEATO does not define how long it must take, if any time, for `Config.Autoboot` to complete or confirm.
