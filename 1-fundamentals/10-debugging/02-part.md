# `gdb`: Watching the Program Run

A debugger does not guess. It runs your program and lets you stop it at any point, inspect every variable, and step forward one line at a time. For the bugs the compiler misses — wrong loop bound, `NULL` where a valid pointer should be — this is the tool.

---

## Install and Compile

```bash
# Ubuntu / Debian
sudo apt install gdb

# Fedora
sudo dnf install gdb

# macOS
xcode-select --install      # the Apple debugger, with the same commands
brew install gdb
```

Then compile with `-g`. Without debug symbols `gdb` can only show you assembly, and it cannot name your variables at all.

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g avg.c -o avg
```

Start it:

```bash
gdb ./avg
```

You get a `(gdb)` prompt. Everything happens there.

---

## The Commands You Need

| Command | Short | What it does |
| --- | --- | --- |
| `break file.c:15` | `b` | set a breakpoint at a line |
| `break function_name` | `b` | set a breakpoint at the first line of a function |
| `break 15 if i == 3` | | set a breakpoint that only fires under a condition |
| `run` | `r` | start the program (pass arguments after it: `run data.txt`) |
| `next` | `n` | execute the current line, step **over** function calls |
| `step` | `s` | execute the current line, step **into** the function it calls |
| `continue` | `c` | run until the next breakpoint or the end |
| `print x` | `p` | print the value of a variable or expression |
| `backtrace` | `bt` | print the call stack: who called whom |
| `list` | `l` | show the source around the current line |
| `info locals` | | print every local variable in the current frame |
| `watch x` | | stop every time `x` changes |
| `finish` | `fi` | run until the current function returns |
| `quit` | `q` | leave gdb |

The distinction between `next` and `step` is the one that matters most:

- `next` runs the whole line. If the line calls `fgetc()`, `fgetc` runs to completion and you land on the next line of *your* code.
- `step` enters `fgetc`, and you now stand inside the C library. Use `finish` to come back out.

When you are hunting a bug in *your* logic, use `next`. Step into a function when you need to know what it did to its arguments.

---

## A Real Session: Finding an Off-By-One

The program:

```c
#include <stdio.h>

static double average(int *values, int count) {
    int i;
    int total = 0;

    for (i = 1; i < count; i++) {
        total += values[i];
    }
    return (double) total / count;
}

int main(void) {
    int data[4] = {10, 20, 30, 40};

    printf("average = %.2f\n", average(data, 4));
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g avg.c -o avg
./avg
```

```text
average = 22.50
```

That is wrong. The average of 10, 20, 30 and 40 is `25.00`. The compiler is silent — the code is perfectly legal C. `22.50` is `(20 + 30 + 40) / 4`, so the first element is being skipped.

Now the debugger session. Every line below is real output from `gdb ./avg`:

```text
Reading symbols from ./avg...
(gdb) break average
(gdb) run
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, average (values=0x7fffffffd8b0, count=4) at avg.c:5
5	    int total = 0;
(gdb) print count
$1 = 4
(gdb) print values[0]
$2 = 10
(gdb) print values[1]
$3 = 20
(gdb) print values[2]
$4 = 30
(gdb) print values[3]
$5 = 40
(gdb) list
1	#include <stdio.h>
2	
3	static double average(int *values, int count) {
4	    int i;
5	    int total = 0;
6	
7	    for (i = 1; i < count; i++) {
8	        total += values[i];
9	    }
10	    return (double) total / count;
(gdb) break 8
(gdb) continue
(gdb) print i
$6 = 1
(gdb) print total
$7 = 0
(gdb) continue
(gdb) print i
$8 = 2
(gdb) print total
$9 = 20
(gdb) continue
(gdb) print i
$10 = 3
(gdb) print total
$11 = 50
```

Read the evidence:

- The input is right: `values[0]` through `values[3]` are 10, 20, 30, 40.
- The loop visits `i = 1`, `2`, `3`. It never visits `i = 0`.
- The last `total` gdb showed before the final addition was `50`. One more `values[i]` (40) makes `90`, and `90 / 4` is the `22.50` you saw. The skipped element is `values[0]` — the `10`.

`list` gives the reason in one character: **line 7 starts at `i = 1`, not `i = 0`.**

Fix, and check:

```c
for (i = 0; i < count; i++) {
```

```text
average = 25.00
```

What gdb did here is worth naming: it turned "the number is wrong" into "`i` never equals 0", which is a fact about one character of source code. That is the whole value of a debugger.

---

## What a Segfault Looks Like

When a program crashes, the shell tells you so and nothing more:

```bash
./crash data.txt
```

```text
open: No such file or directory
Segmentation fault (core dumped)
```

"Segmentation fault" is the operating system saying the program touched memory it is not allowed to. It does not say which pointer was wrong. `gdb` does:

```bash
gdb ./crash
```

```text
Reading symbols from ./crash...
(gdb) run data.txt
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/x86_64-linux-gnu/libthread_db.so.1".
open: No such file or directory

Program received signal SIGSEGV, Segmentation fault.
_IO_getc (fp=0x0) at ./libio/getc.c:37
warning: 37	./libio/getc.c: No such file or directory
(gdb) bt
#0  _IO_getc (fp=0x0) at ./libio/getc.c:37
#1  0x0000555555555218 in count_lines (stream=0x0) at crash.c:8
#2  0x00005555555552ae in main (argc=2, argv=0x7fffffffd9f8) at crash.c:30
(gdb) frame 1
#1  0x0000555555555218 in count_lines (stream=0x0) at crash.c:8
8	    while ((c = fgetc(stream)) != EOF) {
(gdb) info locals
lines = 0
c = 0
```

Read it from the bottom:

| Line | Meaning |
| --- | --- |
| `Program received signal SIGSEGV, Segmentation fault.` | the kernel reported the fault |
| `_IO_getc (fp=0x0)` | the crash is inside `fgetc`, and its argument is `0x0` — a **NULL** `FILE *` |
| `warning: 37 ./libio/getc.c: No such file or directory` | gdb has no source for the C library, so it shows the file name from the debug info and warns that it cannot read it. This is normal, not a problem |
| `#0`, `#1`, `#2` | the call stack, innermost first. `main` called `count_lines`, which called `fgetc` |

The bug, in full:

```c
#include <stdio.h>

static int count_lines(FILE *stream) {
    int lines = 0;
    int c;

    while ((c = fgetc(stream)) != EOF) {
        if (c == '\n') {
            lines++;
        }
    }
    return lines;
}

int main(int argc, char **argv) {
    FILE *f;
    int lines;

    if (argc < 2) {
        fprintf(stderr, "usage: %s <file>\n", argv[0]);
        return 1;
    }

    f = fopen(argv[1], "r");
    if (f == NULL) {
        perror("open");
    }

    lines = count_lines(f);
    printf("%d lines\n", lines);

    fclose(f);
    return 0;
}
```

The `perror` printed the problem, but the code kept going and passed `NULL` to `count_lines`. The fix is the missing `return`:

```c
    if (f == NULL) {
        perror("open");
        return 1;
    }
```

Note what `bt` did for you: `stream=0x0` in frame `#1` says the *parameter* was NULL, and `main` at line 30 says who sent it. Without gdb you would be reading both functions line by line.

> The "No such file or directory" in that warning refers to gdb's **own source file lookup**, not to your program's missing input file. Two different things using the same words.

### A Crash You Cause Deliberately

```c
#include <stdio.h>

static int sum(int *values, int count) {
    int i;
    int total = 0;

    for (i = 0; i <= count; i++) {   /* BUG: should be i < count */
        total += values[i];
    }
    return total;
}

int main(void) {
    int data[4] = {1, 2, 3, 4};
    int *empty = NULL;
    int result;

    result = sum(data, 4);
    printf("sum = %d\n", result);

    result = sum(empty, 0);          /* reads through NULL */
    printf("sum = %d\n", result);
    return 0;
}
```

```bash
./sums
```

```text
Segmentation fault (core dumped)
```

In gdb:

```text
(gdb) run
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/x86_64-linux-gnu/libthread_db.so.1".

Program received signal SIGSEGV, Segmentation fault.
0x00005555555551bc in sum (values=0x0, count=0) at sums.c:8
8	        total += values[i];
(gdb) bt
#0  0x00005555555551bc in sum (values=0x0, count=0) at sums.c:8
#1  0x000055555555525e in main () at sums.c:22
(gdb) print i
$1 = 0
(gdb) print values
$2 = (int *) 0x0
(gdb) print values[i]
Cannot access memory at address 0x0
```

`print values` returning `0x0` is the diagnosis. `print values[i]` then confirms it: `Cannot access memory at address 0x0`.

There are **two** bugs here and gdb only showed you the second:

1. `i <= count` reads one element past the end of the array. Undefined behaviour, but usually survivable because the adjacent stack memory happens to be mapped.
2. `values` is `NULL`, so the read is at address `0`, which is never mapped.

The first bug would have crashed on a different machine, or not at all. That is the nature of undefined behaviour.

---

## Conditional Breakpoints

Stopping on every iteration of a 1000-element loop is not useful. Stop on the interesting one:

```text
break 15 if i == 3
```

Using the program above:

```c
#include <stdio.h>

static int first_negative(int *values, int count) {
    int i;

    for (i = 0; i < count; i++) {
        if (values[i] < 0) {
            return values[i];
        }
    }
    return 0;
}

int main(void) {
    int data[6] = {4, 8, 15, -3, 22, 9};

    printf("first negative = %d\n", first_negative(data, 6));
    return 0;
}
```

```text
(gdb) break 7 if values[i] < 0
Breakpoint 1 at 0x1181: file first.c, line 7.
(gdb) run
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, first_negative (values=0x7fffffffd8b0, count=6) at first.c:7
7	        if (values[i] < 0) {
(gdb) print i
$1 = 3
(gdb) print values[i]
$2 = -3
```

The loop ran six times but stopped once, at `i = 3`, the first negative element. The same idea works for anything:

```text
break 20 if balance < 0
break 12 if strcmp(name, "admin") == 0
run 1000
```

`run 1000` passes command-line arguments to `argc`/`argv`, which is how you drive a program that reads input.

---

## Watching a Variable

When you want to know **who** changes a value, not just what it is now:

```text
watch total
```

gdb sets a hardware watchpoint and stops every time the value changes:

```text
Hardware watchpoint 2: total

Old value = 0
New value = 2
```

`watch` on an expression works too, which is how you find the iteration that corrupts an array:

```text
(gdb) break 6
(gdb) run
Breakpoint 1, first_negative (values=0x7fffffffd8b0, count=6) at first.c:6
6	    for (i = 0; i < count; i++) {
(gdb) watch values[3]
Hardware watchpoint 2: values[3]
(gdb) continue
Watchpoint 2 deleted because the program has left the block in
which its expression is valid.
0x000055555555521c in main () at first.c:17
17	    printf("first negative = %d\n", first_negative(data, 6));
```

`values[3]` never changes — the array is set up once in `main` — so the watchpoint never fires, and gdb removes it when the block it belongs to goes out of scope. Watching something nobody writes is a common way to waste an afternoon. Watch the accumulator instead: that is the one that changes, and its sequence of changes tells you the loop's story.

Other useful relatives: `rwatch x` stops on writes, `awatch x` stops on reads or writes.

---

## A Few More Commands Worth Knowing

| Command | What it does |
| --- | --- |
| `break` with no argument | list the breakpoints, and `break` after a number deletes one |
| `info breakpoints` | same, with hit counts |
| `condition 1 i == 3` | add a condition to breakpoint 1 after the fact |
| `tbreak 15` | a breakpoint that deletes itself after one hit |
| `up` / `down` | move one frame up or down the call stack |
| `call printf("x = %d\n", x)` | call a function while stopped, as if it were your own code |
| `x/4d values` | print 4 ints starting at `values` as raw memory |
| `p *values` | dereference, when `values` is `int *` |
| `p &x` | the address of `x` |
| `set pagination off` | stop pausing with `--Type <RET> to continue--` |

Two habits that make sessions readable: `set pagination off` at the start, and `list` whenever you land somewhere unfamiliar.

---

## Checklist

```text
[ ] Did I compile with -g?
[ ] break the function, run, then step with next
[ ] print the inputs before blaming the logic
[ ] print the loop variable at each iteration, looking for the value it never takes
[ ] After a crash: bt first, then frame 1, then info locals
[ ] Is the crashing pointer 0x0 or garbage? NULL and uninitialized look different
```

---

#### [Previous](./01-part.md) | [Next](./03-part.md)