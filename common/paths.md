# NEATO Paths Specification

Written by piguman3

Revision 1 of July 15, 2026

---

NEET computers by default uses "partition:filepath" for most of it's functions, but, to lower the number of arguments and the complexity of many things in NEATO we instead have adopted the format of "disk:partition:filepath".

---

### Examples

- `0:bios:hello.lua` -> file named "hello.lua", at the root of partition "bios" on disk number 0.
- `4:hello:my/epic/file.lua` -> file named "file.lua" inside of directory "epic", inside of "my", which is at the root of partition "hello" on disk number 4.