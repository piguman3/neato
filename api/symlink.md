# NEATO `symlink` API Specification

Extension: `ext.symlink`

Version: 1

Requires: `core`

Status: draft. An OS must not report `ext.symlink` until this notice is removed.

---

A NEATO compliant operating system that reports this extension must provide symbolic links in its
[`fs`](fs.md) filesystem: a link is a small object that names another path, and every path in every `fs` function
goes through it to whatever it names. Links are what let one name follow another when the target moves, and let a
package present its files under several points.

This extension adds two functions to the `fs` API. An operating system defines them exactly when it reports this
extension, as [fs.md](fs.md) describes.

| Name       | Description                                                                       | Arguments                          | Returns                                    |
| ---------- | --------------------------------------------------------------------------------- | ---------------------------------- | ------------------------------------------ |
| fs.symlink | Creates a link at a path, naming a target path.                                    | target (string), linkPath (string) | true, or nil, code and message             |
| fs.readlink| Returns the target path a link names.                                              | path (string)                      | target (string), or nil, code and message  |

`fs.symlink(target, linkPath)` creates a link at `linkPath` that holds `target`, a path in the
[paths](../common/paths.md) format exactly as given. The target is checked for being a valid path, and for nothing
else: it does not have to exist, and it is looked up again every time the link is used, so a link to a path that
exists later works, and a link to a path that stops existing dangles. Creating the link fails with `EEXIST` when
something is already at `linkPath`.

`fs.readlink(path)` returns the target of the link at `path`. A path that is not a link fails with `EINVAL`, and
nothing that exists at the path fails with `ENOENT`.

Links are followed transparently. Every `fs` function that takes a path resolves the links it meets on the way,
and the object a path names is whatever its final link names: [`fs.stat`](fs.md#file-information) reports
`"link"` as the `type` only for the link itself, [`fs.open`](fs.md#opening-files) on a link opens what it points
to, and [`fs.delete`](fs.md) on a link removes the link and leaves the target alone. A path that runs through more
links than the operating system allows, or through a ring of links, cannot be resolved and fails with `EIO`.

An operating system that reports this extension reports it for its whole `fs` filesystem. Where the underlying
storage of a partition cannot hold links, the operating system refuses to create one there; paths on that partition
simply never name links.

---

### Failure codes

| Function    | Condition                                               | Code           |
| ----------- | ------------------------------------------------------- | -------------- |
| fs.symlink  | something already exists at the link path               | `EEXIST`       |
| fs.symlink  | a component of a path is not a directory                | `ENOTDIR`      |
| fs.symlink  | a parent of the link path does not exist                | `ENOENT`       |
| fs.symlink  | the link is not allowed to be created                   | `EACCES`       |
| fs.symlink  | the filesystem is read-only                             | `EROFS`        |
| fs.symlink  | the name is too long                                    | `ENAMETOOLONG` |
| fs.symlink  | the link cannot be created for another reason           | `EIO`          |
| fs.readlink | nothing exists at the path                              | `ENOENT`       |
| fs.readlink | the path is not a link                                  | `EINVAL`       |
| fs.readlink | the link is not allowed to be read                      | `EACCES`       |
| fs.readlink | the target cannot be read for another reason            | `EIO`          |

---

### Example usage

```lua
assert(fs.symlink("0:system:/bin/", "0:system:/usr/bin/"))
print(fs.readlink("0:system:/usr/bin/"))
  -> 0:system:/bin/

-- fs functions follow it:
print(fs.isFile("0:system:/usr/bin/sh"))
  -> true
```

---

### Footnotes

The draft notice comes off when the resolution rules have survived an implementation or two. In particular these
are still open: whether a dedicated code replaces `EIO` for rings of links, whether links may point across
partitions, and whether [`fs.stat`](fs.md#file-information) gains a field for the target of the final link. There
is deliberately no function that changes a link's target in place; use `fs.delete` and `fs.symlink`.
