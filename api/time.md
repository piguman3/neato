# NEATO `time` API Specification

Extension: `ext.time`

Version: 1

Requires: `core`

---

A NEATO compliant operating system that reports this extension must provide a global `time` API with the two kinds
of clock a program needs: a monotonic clock for measuring how long something took, and a wall clock for naming the
moment something happened. All times are in seconds, and every calendar value is in UTC: local time is a display
concern for the program, not something the clocks know about.

The two clocks are independent. The monotonic clock never goes backwards and never jumps; the wall clock may be
adjusted, and on a computer without a battery or a network time source it may not be set at all.

| Name           | Description                                                                                                                              | Arguments       | Returns                                          |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | --------------- | ------------------------------------------------ |
| time.monotonic | Returns seconds on a clock that only moves forward, from an arbitrary origin such as the start of the operating system.                    | none            | seconds (number)                                 |
| time.wall      | Returns the current wall clock time as seconds since `1970-01-01T00:00:00Z`, or `nil` on a computer with no wall clock.                    | none            | seconds (int), or nil                            |
| time.date      | Splits a wall clock time into a calendar table. See below.                                                                                | secs (int?)     | date (table), or nil                             |

`time.monotonic` keeps whatever precision the computer has and should be precise enough to measure intervals well
under a second. Its origin is arbitrary: a program must compare it only with other `time.monotonic` results, never
with a `time.wall` result or a file's `mtime`.

`time.wall` returns an integer, or `nil` when the computer has no way to know the current date. `nil` is an ordinary
answer, not a failure.

`time.date` splits a wall clock time into a table of integers:

| Field   | Range            | Meaning                                   |
| ------- | ---------------- | ----------------------------------------- |
| `year`  |                  | The year, such as `2026`.                  |
| `month` | 1 to 12          | The month.                                 |
| `day`   | 1 to 31          | The day of the month.                      |
| `hour`  | 0 to 23          | The hour.                                  |
| `min`   | 0 to 59          | The minute.                                |
| `sec`   | 0 to 59          | The second.                                |
| `wday`  | 0 to 6           | The day of the week, with 0 for Sunday.    |
| `yday`  | 1 to 366         | The day of the year.                       |

When `secs` is not given, `time.date` uses the current `time.wall`. When the computer has no wall clock,
`time.date` returns `nil`. Passing a value that is not an integer raises an error.

No function in this API fails, and none of them yields.

---

### Failure codes

None. A missing wall clock is answered with `nil`, and every mistake of use raises an error, as described in
[errors.md](../common/errors.md).

---

### Example usage

```lua
local t0 = time.monotonic()
doHardWork()
print(("took %.3f s"):format(time.monotonic() - t0))

if time.wall() then
  local d = assert(time.date())
  print(("%04d-%02d-%02d %02d:%02d:%02d"):format(d.year, d.month, d.day, d.hour, d.min, d.sec))
    -> 2026-10-09 21:44:10
else
  print("no clock on this computer")
end
```

```lua
-- The mtime of fs.stat is on the same wall clock.
local s = assert(fs.stat("notes.txt"))
if s.mtime and time.wall() - s.mtime > 60 * 60 * 24 then
  print("notes.txt is older than a day")
end
```
