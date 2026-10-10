# NEATO `user` API Specification

Extension: `ext.user`

Version: 1

Requires: `core`, `ext.perms`

---

A NEATO compliant operating system that reports this extension must provide a global `user` API with the user
accounts it has and the account the calling program runs as. It answers who the program is and who else exists, so
that programs such as `whoami`, `id` and `ls` can name users, and so that an operating system can ground its
"may this program do that" decisions in an identity instead of in one flag for the whole computer.

Nothing in this specification is about logging in. Asking for a password and starting a program as another user is
deliberately not part of version 1.

A user is described by a table:

| Field    | Type             | Meaning                                                                                                  |
| -------- | ---------------- | -------------------------------------------------------------------------------------------------------- |
| `id`     | int              | The user's id, chosen by the operating system. There is no reserved id: whether a user is an administrator is the `admin` field, never the id. |
| `name`   | string           | The login name.                                                                                           |
| `admin`  | boolean          | Whether the user is an administrator. An operating system may have any number of administrators, or none.  |
| `home`   | string, optional | A full [path](../common/paths.md), ending in `/`, to the user's home directory. It may not exist. `nil` when the operating system has no home directories. |
| `groups` | table, optional  | The ids of the groups the user belongs to, when the operating system has groups.                           |

| Name        | Description                                                            | Arguments       | Returns                              |
| ----------- | ---------------------------------------------------------------------- | --------------- | ------------------------------------ |
| user.current| Returns the account this program runs as.                              | none            | user (table)                         |
| user.byName | Returns the account with the given name, or `nil` if there is none.    | name (string)   | user (table), or nil                 |
| user.byId   | Returns the account with the given id, or `nil` if there is none.      | id (int)        | user (table), or nil                 |
| user.list   | Returns every account the program may see, in no particular order.     | none            | users (table of user tables)         |

A single-user operating system reports this extension too: it has one user, and every function answers from that
one account. `user.list` may hold fewer accounts than exist when the operating system hides some from this program;
it never fails for that reason.

Every returned table is a snapshot, and changing it does nothing. None of these functions fail: a user that does
not exist is `nil`, and a mistake of use raises an error, as described in [errors.md](../common/errors.md).

---

### Example usage

```lua
local me = user.current()
print(me.name, me.admin and "(admin)" or "")

local u = user.byName("root")
if u then print("root has id " .. u.id) end

local names = {}
for _, u in ipairs(user.list()) do names[#names + 1] = u.name end
table.sort(names)
print(table.concat(names, ", "))
  -> alice, root
```

---

### Footnotes

On passwords and switching users: an operating system that can authenticate a name and a password, and start a
program as the matching user, is expected to define that in a future extension (`ext.auth` is reserved for it).
Programs that need to know only whether they may do something should ask the operating system by trying, and read
the `EACCES`, instead of deciding from `admin` themselves; `admin` is for display and for choosing what to offer.
