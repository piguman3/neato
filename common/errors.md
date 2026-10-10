# NEATO Error Code Specification

Extension: `core`

Version: 3

---

Every NEATO program API that can fail reports the failure the same way, so that a program sees the same exact error on
every operating system. This file defines the codes and what they mean. No specification, `core` included, defines a
failure code of its own: a code that is needed is added here.

---

### Reporting a failure

A function that can fail returns `nil`, then a code (a string from the table below), then a message (a string for
humans). A program compares the code, and must not compare the message; the message is free for an operating system to
word however it likes.

A function that returns `false`, or `nil` without a code, as an ordinary answer is not failing and does not use a code.
For example, `fs.exists` answers `false` when nothing is there, `event.poll` answers `nil` when the queue is empty, and
[`fs.getPoint`](../api/fs.md#filesystem-points) answers `nil` for a point the operating system does not provide. A
program tells a failure from such an answer by the code: a failure is exactly a `nil` first result that is followed by a
code.

Passing a value of the wrong type, calling a method on a closed handle, a path that is not valid, and every other
mistake in how a program uses an API raises an error instead, as each specification says. A raised error is not one of
these codes; it means the program is wrong, not that the environment refused something.

Every specification states, for each function that can fail, the conditions it must report and the exact code it must
use. An operating system must use that code for that condition. If an operating system fails for a reason a
specification does not list, it uses the code from the table that fits best, or `EIO` if none does. An operating system
must not return a code that is not in this table, and a program must be ready to receive any code in it.

---

### Codes

| Code           | Meaning                                                                                   |
| -------------- | ----------------------------------------------------------------------------------------- |
| `EACCES`       | The program is not allowed to access the object, because of the permissions on it.        |
| `EADDRINUSE`   | The port is already in use.                                                               |
| `EBADF`        | The handle is not open for that operation.                                                |
| `EBUSY`        | The object is in use and cannot be changed right now.                                     |
| `ECHILD`       | The process is not a child of the calling program, or has already been waited for.        |
| `ECONNREFUSED` | Nothing accepted the connection.                                                          |
| `ECONNRESET`   | The connection ended abnormally.                                                          |
| `EEXIST`       | The file or directory already exists.                                                     |
| `EFBIG`        | The file is too large.                                                                    |
| `EHOSTUNREACH` | There is no route to the host.                                                            |
| `EINTR`        | The operation was stopped before it finished, by a terminate request.                     |
| `EINVAL`       | An argument value is not valid.                                                           |
| `EIO`          | An input/output error happened, and no other code fits.                                   |
| `EISDIR`       | The path is a directory where a file was expected.                                        |
| `ELIMIT`       | A limit of the specification or the operating system was reached.                         |
| `EMSGSIZE`     | A message or payload is too large.                                                        |
| `ENAMETOOLONG` | A name or a path is too long.                                                             |
| `ENOBUFS`      | No buffer or queue space is available.                                                    |
| `ENODEV`       | The disk or device does not exist.                                                        |
| `ENOENT`       | The file or directory does not exist.                                                     |
| `ENOEXEC`      | The file is not a program the operating system can run.                                   |
| `ENOSPC`       | There is no space left on the filesystem.                                                 |
| `ENOTDIR`      | A path component is not a directory, or a directory was expected and the path is not one. |
| `ENOTEMPTY`    | The directory is not empty.                                                               |
| `ENOTSUP`      | The operation, option or scope is not supported.                                          |
| `EPERM`        | The operation is not permitted on that object at all.                                     |
| `EPIPE`        | The connection was closed at the other end.                                               |
| `ERANGE`       | A value is outside the range the operation allows.                                        |
| `EROFS`        | The filesystem is read-only.                                                              |
| `ESPIPE`       | The handle does not support seeking.                                                      |
| `ESRCH`        | The process does not exist.                                                               |
| `ETIMEDOUT`    | The operation did not finish in time.                                                     |
| `EXDEV`        | The operation would have to move the object between filesystems.                          |

---

### Adding a code

A code is added here, not in the specification that needs it, so that the meaning stays global. It uses the same style:
`E` followed by a short uppercase word or phrase with no spaces (for example `ENAMETOOLONG`), and its meaning is stated
in one sentence without reference to a particular function. A code is only added for a condition that is neither a
wrong-type mistake (which raises) nor an ordinary negative answer (which needs no code), and that no existing code
already covers.
