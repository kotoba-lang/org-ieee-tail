# kotoba-lang/org-ieee-tail — POSIX `tail`, as a Kotoba command binary

`tail` from IEEE Std 1003.1 for one operand, written in `.kotoba` and
compiled to a standalone native executable.

```sh
./tail FILE          # the last 10 lines
./tail -n 2 FILE     # the last 2
```

## Two places `tail` and `head` differ, both measured

They look like twins and are not:

| | `head` | `tail` |
|---|---|---|
| `-n 0` | `head: illegal line count -- 0`, exit **1** | nothing, exit **0** |
| a last line without a newline | adds nothing | adds nothing |

`grep`, `sort` and `uniq` all **do** add one in the same situation. Six
commands in this family, two contracts, and running them is the only way to
know which is which — `x\ny` with `-n 1` is **one byte**, `y`.

## The model

```
lines = newlines, or newlines + 1 when the text does not end in one
skip  = lines - N, floored at 0
emit  = the bytes from just after the skip-th newline to the end
```

Emitting a **span of the original bytes** is what makes "adds nothing" fall
out rather than being a special case — and `-n 0` falls out too: skip reaches
the last newline and the span is empty.

## The boundary test this does *not* use

"Does the text end in a newline" is **not**
`(string-substring text (- n 1) n)`. A substring offset must be a code-point
boundary and `n-1` is not one when the last character is multi-byte — the
same trap [`org-ieee-ls`](https://github.com/kotoba-lang/org-ieee-ls) hit,
where it listed every ASCII entry and then trapped `SIGILL` on `é`.

Here the question is answered from the newline walk instead: the text ends in
a newline exactly when the offset just after the **last** newline is the
length.

## Measured against the system utility

Sixteen cases, all identical on stdout, stderr and exit status. Boundaries
are deliberate: `-n 20` is the file's exact line count and `-n 25` is past
it; `-n 0` on a file and on an empty file; blank lines, which are lines
(`-n 2` on `a\n\n\nb\n` answers `"\nb\n"`, not `"b\n"`).

Verified to fail as well as pass: counting a trailing newline as starting
another line fails three cases, adding the newline `grep` adds fails three,
and refusing `-n 0` the way `head` does fails two — **with identical stdout
on both sides**, the difference being the exit status alone.

## Capabilities

`:cli/args` (38), `:fs/app-data` (35), `:io/write` (37), `:io/write-error`
(39). A missing operand is reported byte-for-byte
(`tail: PATH: No such file or directory`, exit 1) using wire 35's `EXISTS`
form — the read form traps, and a trap cannot be caught.

## What this is not

One operand. No `-c`, `-f`, `-r`, no `+N` form, no reading standard input.
