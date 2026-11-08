# NEATO `loadfile` API Specification

Extension: `core`

Version: 4

---

A NEATO compliant environment must provide a function under the name `_G.loadfile`, directly equivalent to the base Lua
implementation of `loadfile`.

## `loadfile(filename, mode, env) --> chunk, err`

This function should be equivalent to running [`fs.open`](fs.md) on the exact `filename` parameter passed to the function
with the mode `"r"`, reading the entire file with `handle:read("a")`, and passing the resulting string to `load` along with
the filename, `mode` and `env` (ei, `load(string, filename, mode, env)`). The returned values should be the same as those
returned by that load function call.
