# NEATO Bootloader Specification

Written by SpSf July 9, 2026

Revision 1

---

A NEATO compatible bootloader must obtain the `boot.lua` file from `any:system:/boot/cfg/boot.lua`.

A NEATO compatible boot.lua might appear like this:

```lua
return {
  1 = {
    ["OS Name"] = "NeetOS",
    ["OS Version"] = "0.2.4",
    ["OS Description"] = "The NeetComputers Operating System",
    ["OS Args"] = {"-v", "-f"}, --optional
    ["OS Boot Path"] = "0:bios:/boot.lua",
    ["OS Environment Variable Definition"] = { --optional
      ["EXAMPLEVAR"] = 0,
      ["VAR2"] = "Hi",
    },
  },
  2 = {
    ...
  },
}
```

NEATO compatible bootloaders must support ALL fields defined, as well as supporting multiple operating systems defined in `boot.lua`.

The `OS Args` field must take in either a string, or a table (as shown).

Arguments must be passed into the boot path as provided, so a table must pass into the boot path as ("-v", "-f") for example, or as a string, must pass in as "EXAMPLEARG=1".

The `OS Environment Variable Definition` field must create environment variables for the boot path, as provided. This can be done different ways, but the boot path MUST be able to see defined varaibles in _ENV. So, for the provided example, `0:bios:/boot.lua` would be able to see `_ENV.EXAMPLEVAR` as `0`, and `VAR2` as "Hi".
