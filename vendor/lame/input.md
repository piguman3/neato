# Lame `input` API Specification

Extension: `x.lame.input`

Version: 1

Requires: `core`

Note: Vendor-specific, can be implemented by any OS but is fully dictated by one vendor (Lame)

---

Any NEATO environment that implements this extension must provide a function at `_G.input` that prompts the user for a text input and returns what they enter. 

The user cannot cancel this function and the function cannot return a failure code. It must always return a string with what the user entered. The first and only argument of this function is an optional string that gets displayed when the user types a character instead of the character the user actually entered. If this string is longer than 1 character then only the first character of it is used and the rest is discarded.

| Name     | Description                | Arguments          | Returns       |
| -------- | -------------------------- | ------------------ | ------------- |
| _G.input | Prompts the user for input | hideChar (string?) | text (string) |


### Example usage

```lua
term.write("Username: ")
local username = input()
term.write("Password: ")
local password = input("*")
print(username.." - "..password)
  -> Username: NEETFan1234
  -> Password: *******
  -> NEETFan1234 - hunter2
```