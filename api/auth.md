# NEATO `auth` API Specification

Extension: `ext.auth`

Version: 1

Requires: `core`, `ext.user`

---

A NEATO compliant operating system that reports this extension must provide a global `auth` API that verifies a
password for an account and switches the calling program to run as another. It answers the two questions
[`ext.user`](user.md) deliberately leaves open: whether this is the password for that user, and, from now on,
whether this program is that user. It is what `login` and `su` are written from.

An operating system defines the `auth` global exactly when it reports this extension, as [globals.md](globals.md)
describes, and a program tests for it with [`sys.hasExtension`](core/sys.md) or by checking for `nil`.

```c
auth
  verify
  switch
```

| Name        | Description                                                                                                                     | Arguments                        | Returns                           |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- | --------------------------------- |
| auth.verify | Returns whether the password is the correct one for the named user.                                                              | name (string), password (string) | boolean, or nil, code and message |
| auth.switch | Checks the pair and, when it is correct, makes the named user the user this program runs as, from the moment the call succeeds. | name (string), password (string) | true, or nil, code and message    |

`name` is a login name, as [ext.user](user.md) defines them, and `password` is the password as the user would type it.
Both must be strings, and anything else raises an error. A name that no account has is not a mistake of use: it is
answered exactly like a wrong password, as the two sections below define.

A single-user operating system may report this extension too. It has the one account, `auth.verify` checks that
account's one password, and switching to it succeeds and changes nothing, but demands the password all the same.
Whether any account has a password at all is the operating system's business, as is how accounts and their passwords
come to exist: [ext.user](user.md) does not define how accounts are created either.

---

### Verifying

`auth.verify(name, password)` answers one question: is `password` the password of the account named `name`? The answer
is `true` or `false`, and a wrong answer is never a failure; a wrong password and a name that no account has are the
same ordinary `false`, with nothing to tell them apart. It is a question in the sense of [`fs.access`](perms.md),
not an operation. Asking changes nothing.

The one way it fails is by being refused. An operating system may refuse to let this program verify a user at all,
under the one rule of [Who may call what](#who-may-call-what), and a refused call fails with `EACCES`. Any other
reason the check cannot be made is `EIO`, as [Failure codes](#failure-codes) lists.

### Switching

`auth.switch(name, password)` makes the check `auth.verify` makes and, when the pair is correct, makes the account
named `name` the account this program runs as. It returns `true` when it has, and `nil`, a code and a message when it
has not, and a switch that fails changes nothing: the program stays the user it was.

Switching to the user the program already runs as is allowed and ordinary: the password is checked all the same, and a
successful switch of it changes nothing.

From the moment a switch succeeds:

- [`user.current`](user.md) answers the new account. A table taken before the switch keeps the old account, as any
  snapshot does.
- Every "may this program do that" decision from then on is made as the new user: what it may read, write, change and
  run ([`ext.perms`](perms.md)), which programs it may stop ([`proc.kill`](proc.md)), and what
  [`proc.spawn`](proc.md) asks of it before it starts anything.
- Anything the program does from then on, it does as the new user, and a program it starts afterwards is started by
  the new user and runs as that user. Nothing else in NEATO chooses a user for a new program:
  [`proc.spawn`](proc.md) takes none, so a child is the user its parent was at the moment it was started, and a child
  started before the switch keeps the user it was started as.

What a switch does not change is handles opened before the switch keep the access they were opened with, and the
switch closes nothing and widens nothing; programs already running keep their users; and the program's own
surroundings stay as they are. Its environment ([`ext.env`](env.md)), its three streams
([`ext.stdio`](stdio.md)), its [`CWD`](core/cwd.md) and its umask ([`fs.umask`](perms.md)) do not become the new
user's, and a program that wants the new user's home resolves [`user.current().home`](user.md) itself.

Switching back is an ordinary switch: it needs the previous user's password again, and no function of this API
remembers one.

### Who may call what

When an operating system restricts nothing, every program may verify every name and switch to every account whose
password it presents. That is the default this specification states.

An operating system MAY restrict the calls that reach across users, and MAY tie the restriction to being an
administrator, as [ext.user](user.md) defines the word. Only an administrator may verify a user other than the one
this program runs as, or switch to one. Whatever it restricts, it must leave this program the two calls on the user it
runs as at the moment, or no lock screen could ever ask for the password back. A refused call fails with `EACCES`, as
[`proc.kill`](proc.md) fails for a program this one may not stop.

A restriction must depend only on who is asking and whom the call names, never on whether that name exists or whether
the password would have been right. A call refused for one user is refused for every name, and the refusal comes
before any password is looked at. A program that is answered `EACCES` has learned nothing about the account it named.

### Secrecy

A failed authentication must not reveal whether the name exists. Wrong name and wrong password have the same answer,
`false` from `auth.verify`, `EAUTH` from `auth.switch`, and the message must not say which of the two it was, because
a message that did would let a program prove an account exists by telling the answers apart. It is the rule
[ext.peripherals](peripherals.md) states for a hidden peripheral, that a hidden thing is answered exactly like a
missing one, applied to accounts.

Passwords are write-only knowledge. No function of this API returns a password, confirms one, or says anything about
one without being given it to check. How an operating system stores, hashes or checks its passwords is entirely its
business, and nothing here requires any particular way of it, [`crypto`](core/crypto.md) included.

`auth.verify` and `auth.switch` SHOULD take a moment to answer, about half a second, right or wrong alike. A
program that is answered instantly can try passwords far faster than any human could type them. An operating
system MAY also slow down or lock out repeated failures. That is policy, not API: this specification defines no code
and no function for it, and a refused attempt is answered like a failed one, the same `false` or the same `EAUTH` as
any other, so the program that asked cannot tell the two apart and is not meant to. What the human gets that the
program does not is the operating system's own voice, a warning it puts up on its own, and the wait itself, which a
rate-limited answer makes conspicuously longer.

### Failure codes

| Function    | Condition                                                            | Code     |
| ----------- | -------------------------------------------------------------------- | -------- |
| auth.verify | the operating system does not allow this program to verify that user | `EACCES` |
| auth.verify | the check cannot be made for another reason                          | `EIO`    |
| auth.switch | the name and the password do not match, or no account has the name   | `EAUTH`  |
| auth.switch | the operating system does not allow this program to switch to that user | `EACCES` |
| auth.switch | the switch cannot be made for another reason                         | `EIO`    |

`auth.verify` has no row for a wrong name or password, because that is not a failure but the ordinary answer `false`.
`EAUTH` is used for that one condition and nothing else, by every function of this API that can report it.

### Example usage

```lua
while true do
  stdio.stdout:write("name: ")
  local name = stdio.readLine()
  if not name then return end
  stdio.stdout:write("password: ")
  local password = stdio.readLine({ noecho = true })
  if not password then return end
  print()

  if auth.verify(name, password) then
    assert(auth.switch(name, password))
    print("welcome, " .. user.current().name)
    return
  end
  print("login incorrect")
end
```

```lua
local name = ...
stdio.stdout:write("password: ")
local password = stdio.readLine({ noecho = true })
print()
local ok, code = auth.switch(name, password)
if not ok then
  print("su: " .. (code == "EAUTH" and "authentication failed" or "not allowed"))
  proc.exit(1)
end
```
