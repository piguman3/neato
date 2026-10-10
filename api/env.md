# NEATO `env` API Specification

Extension: `ext.env`

Version: 1

Requires: `core`

---

A NEATO compliant operating system that reports this extension must provide a global `env` API giving each program
an environment: a map from names to string values that the operating system keeps for the program. Configuration
that a user or a parent program wants to hand down without editing a file lives here, such as `PATH`, `HOME` or a
preferred editor.

A program starts with a copy of the environment of the program that started it (see
[`ext.proc`](proc.md#starting-a-program) for the one exception). Changing the environment changes only this
program's own copy, never the parent's.

| Name   | Description                                                                                    | Arguments                        | Returns                                   |
| ------ | ---------------------------------------------------------------------------------------------- | -------------------------------- | ----------------------------------------- |
| env.get| Returns the value of a variable, or `nil` if there is none.                                     | name (string)                    | value (string), or nil                    |
| env.set| Creates a variable, changes its value, or removes it when the value is `nil`.                   | name (string), value (string?)   | true, or nil, code and message            |
| env.all| Returns a new table with every variable, from name to value. Changing it does nothing.          | none                             | variables (table from str to str)         |

A name is any non-empty string that does not contain `=`; a value is any string, including the empty one. The
convention is names of uppercase letters, digits and underscores. A name that breaks these rules, or a value that is
not a string or `nil`, fails with `EINVAL`; anything else about the environment that the operating system will not
store, such as a value that is too large or too many variables, fails with `ELIMIT`. Passing anything other than a
string as `name`, or anything other than a string or `nil` as `value`, raises an error.

`env.get` and `env.all` never fail.

---

### The `PATH` convention

The variable `PATH`, when it is set, holds a list of directories that a program launcher searches in order when it
is asked to run a program by its bare name. Because `:` appears inside full NEATO paths, the separator is `;`:

```lua
env.set("PATH", "0:system:/bin/;0:system:/usr/bin/")
```

An entry may be any path in the [paths](../common/paths.md) format, full or relative, and should name a directory
and end in `/`. [`ext.proc`](proc.md) uses this exact convention when it resolves a bare name.

---

### Failure codes

| Function | Condition                                    | Code     |
| -------- | -------------------------------------------- | -------- |
| env.set  | the name is empty or contains `=`            | `EINVAL` |
| env.set  | the value is not a string                    | `EINVAL` |
| env.set  | the operating system will not store it       | `ELIMIT` |

---

### Example usage

```lua
print(env.get("PATH"))
  -> 0:system:/bin/;0:system:/usr/bin/

env.set("EDITOR", "ee")
print(env.get("EDITOR"))
  -> ee

env.set("EDITOR", nil)
print(env.get("EDITOR"))
  -> nil

for name, value in pairs(env.all()) do
  print(name .. "=" .. value)
end
```
