# Termination and Defensive Programming

Errors happen in the middle of a function. Resources have already been acquired. The last piece of the error-handling model is: **how do you get out, and how do you leave nothing behind?**

---

## `exit` vs `return`

From `<stdlib.h>`, `exit(code)` ends the program immediately with `code` as its exit status. It works from anywhere, including from deep inside a library function:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    printf("something went wrong\n");
    exit(1);          /* stops right here */
    printf("never printed\n");
}
```

`return` from `main` does the same thing, more politely. Both run the handlers registered with `atexit` and both flush the stdio buffers:

```c
return 0;   /* 0 = success, anything else = failure */
exit(0);    /* identical effect when called from main */
```

| | `return` from `main` | `exit(code)` |
| --- | --- | --- |
| exit status | `return 0` / `return 1` | the `code` argument |
| runs `atexit` handlers | yes | yes |
| flushes `stdout` | yes | yes |
| callable from | `main` only | anywhere |

The convention is simple and worth following: **`return` from `main`; `exit` from anywhere else.** Use `exit` when a deep function cannot meaningfully continue and the program as a whole has nothing left to do. Use a status return when the caller has somewhere better to go — which, per the previous parts, is nearly always.

### See It for Yourself

```c
#include <stdio.h>
#include <stdlib.h>

static void farewell(void) {
    printf("atexit handler ran\n");
}

int main(void) {
    int code;

    atexit(farewell);
    printf("main is ending\n");

    code = 3;
    exit(code);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g bye.c -o bye
./bye
echo "exit status: $?"
```

```text
main is ending
atexit handler ran
exit status: 3
```

The `atexit` handler ran even though `exit` was called, and the shell saw status `3`.

### `_Exit` Skips Everything

`_Exit` terminates immediately: no `atexit` handlers, no buffer flushing. It exists for cases where the process state is already too damaged to trust (a crashed child, a handler that must not run again). You will rarely write it, but you should know it is there — and never use it as a lazy `exit`:

```c
#include <stdio.h>
#include <stdlib.h>

static void farewell(void) {
    printf("atexit handler ran\n");
}

int main(void) {
    atexit(farewell);
    printf("about to call _Exit\n");
    fflush(stdout);
    _Exit(2);
    return 0;
}
```

```text
about to call _Exit
exit status: 2
```

The handler did not run. Note the explicit `fflush(stdout)` — without it, the `printf` output would have been lost, because `_Exit` does not flush.

---

## `abort()`

`abort()` raises `SIGABRT` and stops the program without running `atexit` handlers or flushing anything. That is what a failed `assert` calls.

```text
Aborted (core dumped)
```

Use it when the program is in a state it cannot reason about — an internal invariant is broken and continuing would corrupt more data. Never use it for ordinary failures: the user mistyped a number, so `abort()` is a terrible answer. Return a status and print a message instead.

---

## Cleanup on Error: The `goto` That Is Not a Sin

Here is a real problem. A function acquires resources one after another, and every later step can fail:

```text
1. acquire A   -> ok
2. acquire B   -> ok
3. acquire C   -> FAILED, must undo 1 and 2
4. acquire D   -> FAILED, must undo 1, 2 and 3
```

The naive fix repeats the cleanup at every failure point:

```c
left = malloc(...);
if (left == NULL) return -1;          /* nothing to clean */

right = malloc(...);
if (right == NULL) {
    free(left);                        /* cleanup written once */
    return -1;
}

if (parse(...) != 0) {
    free(left);                        /* cleanup written again */
    free(right);
    return -1;
}

if (parse(...) != 0) {
    free(left);                        /* and again */
    free(right);
    return -1;
}
```

This is how leaks happen. Four acquisition points means five places to remember, and adding a fifth allocation means editing all five. One of them will be forgotten, and the bug only appears under memory pressure.

The C idiom is a single cleanup label:

```c
static int build_pair(const char *left_text, const char *right_text,
                      int **left_out, int **right_out) {
    int *left  = NULL;
    int *right = NULL;

    left = malloc(4 * sizeof(int));
    if (left == NULL) {
        fprintf(stderr, "error: out of memory for the left buffer\n");
        return -1;
    }

    right = malloc(4 * sizeof(int));
    if (right == NULL) {
        fprintf(stderr, "error: out of memory for the right buffer\n");
        goto cleanup;
    }

    if (parse_pair(left_text, &left[0], &left[1]) != 0 ||
        parse_pair(right_text, &right[0], &right[1]) != 0) {
        goto cleanup;
    }

    *left_out  = left;
    *right_out = right;
    return 0;

cleanup:
    free(left);
    free(right);
    return -1;
}
```

with the parser it calls:

```c
#include <stdlib.h>
#include <string.h>

static int parse_pair(const char *text, int *left, int *right) {
    char *end = NULL;
    long  a, b;

    a = strtol(text, &end, 10);
    if (end == text || end[0] != ',') {
        fprintf(stderr, "error: '%s' is not a pair of numbers\n", text);
        return -1;
    }
    b = strtol(end + 1, &end, 10);
    if (end[0] != '\0') {
        fprintf(stderr, "error: trailing junk in '%s'\n", text);
        return -1;
    }
    *left  = (int)a;
    *right = (int)b;
    return 0;
}
```

and `main`:

```c
#include <stdio.h>

int main(void) {
    int *a = NULL;
    int *b = NULL;

    if (build_pair("12,7", "34,9", &a, &b) != 0) {
        fprintf(stderr, "could not build the pair\n");
        return 1;
    }

    printf("a = [%d, %d], b = [%d, %d]\n", a[0], a[1], b[0], b[1]);

    free(a);
    free(b);
    a = NULL;
    b = NULL;

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g pair.c -o pair
./pair
```

```text
a = [12, 7], b = [34, 9]
```

Now the failing path, with the second argument changed to `"oops"`:

```text
error: 'oops' is not a pair of numbers
could not build the pair
exit status: 1
```

Both buffers were freed by the one `cleanup` block, and `main` never touched `a` or `b` because `build_pair` returned `-1` without writing to the out-parameters.

### Why `goto` Is the Right Answer Here

| Repeated cleanup | Single cleanup label |
| --- | --- |
| 5 allocations, 5 copies of the `free` list | 5 allocations, 1 copy |
| adding a resource means editing every error path | adding a resource means one `if` and one `free` |
| easy to miss one | impossible to miss, the list is written once |
| more lines | fewer lines |

`goto cleanup;` is not jumping into the middle of an expression or skipping an initialization. It is a forward jump to the end of the function that exists for exactly this purpose. Every serious C codebase uses it. You will meet it in the kernel, in `git`, in SQLite.

The discipline that keeps it sane:

1. Always jump **forward**, to a label at the end of the function.
2. The label must be the last thing in the function.
3. Every resource is initialized to `NULL` at declaration, so `cleanup` can free it whether or not it was acquired.
4. On success, `return 0` before the label. On failure, `goto cleanup` then return the failure status from there.

Point 3 is what makes this work: `free(NULL)` is defined to do nothing, so the cleanup block is correct no matter how far the function got.

---

## Return-Value Conventions, Summarized

| Return type | Success | Failure | Caller checks |
| --- | --- | --- | --- |
| `int` status | `0` | `-1` or any nonzero | `if (rc != 0)` |
| `int` status enum | `STATUS_OK` | any other enumerator | `if (rc != STATUS_OK)` |
| value, with status out-param | value written to the pointer | status returned | `if (rc != STATUS_OK)` and use the out-param |
| `FILE *` | non-`NULL` | `NULL` | `if (fp == NULL)` |
| `void *` from `malloc` | non-`NULL` | `NULL` | `if (p == NULL)` |
| `size_t` (like `fread`) | items read | may equal fewer than asked, or on error with `ferror` | compare to the requested count |
| `char *` from a search | the match | `NULL` | `if (result != NULL)` |
| `char *` for an error string | the message | `NULL` (with `errno` set) | `if (msg == NULL)` |

Note that nothing in this table says "throw" and nothing says "ignore". That is the point.

---

## The Five Mistakes Beginners Make

### 1. Ignoring return values

```c
malloc(100);                  /* if it returns NULL, the next line crashes */
scanf("%d", &n);              /* if the user typed letters, n is untouched */
printf("%d\n", x);            /* if the write failed, you do not know */
```

Cost: a crash three functions away, with no useful message. Fix: check every one.

### 2. Using `assert` for user input

```c
scanf("%d", &count);
assert(count > 0);            /* aborts in debug, no check at all in release */
```

Cost: a crash for a typo, and a different behaviour per build. Fix: `if` plus a status return.

### 3. Freeing the same pointer twice

```c
free(p);
free(p);
```

gcc catches this one, and its warning is more useful than the crash:

```text
df.c: In function ‘main’:
df.c:7:5: warning: pointer ‘p’ used after ‘free’ [-Wuse-after-free]
    7 |     free(p);
      |     ^~~~~~
df.c:6:5: note: call to ‘free’ here
    6 |     free(p);
      |     ^~~~~~~
free(): double free detected in tcache 2
Aborted (core dumped)
```

Note that glibc detected the corruption at runtime. On another allocator, or with a different program layout, the second `free` can succeed silently and corrupt the heap instead — which is worse, not better.

Fix: after `free`, set the pointer to `NULL`. `free(NULL)` is defined to do nothing, so a double free becomes a silent no-op and the bug degrades from corruption into a harmless accident.

```c
free(p);
p = NULL;
free(p);          /* fine: free(NULL) is a no-op */
```

### 4. Not checking `malloc` for `NULL`

```c
int *data = malloc(n * sizeof(int));
data[0] = 1;                  /* writing to address 0 -> segfault */
```

Cost: a segfault with no explanation. Fix:

```c
int *data = malloc(n * sizeof(int));
if (data == NULL) {
    fprintf(stderr, "error: cannot allocate %zu bytes\n", n * sizeof(int));
    return -1;
}
```

### 5. Reporting errors to stdout

```c
printf("error: file not found\n");     /* WRONG */
fprintf(stderr, "error: file not found\n");   /* RIGHT */
```

Cost: `./app > output.txt` captures your error messages as if they were results, and in the order the buffers happened to flush. Fix: results to `stdout`, diagnostics to `stderr`.

---

## `setjmp` and `longjmp`

`<setjmp.h>` gives you `setjmp`, `longjmp` and the `jmp_buf` type, which together jump from one function to another without unwinding. It is the closest thing C has to an exception.

You should know it exists and then almost never use it:

- it skips every `free` between the jump and the target, which defeats the cleanup discipline above;
- local variables that changed between `setjmp` and `longjmp` have indeterminate values unless declared `volatile`;
- error-returning functions are clearer, and the cleanup-label pattern handles the hard cases.

If you find yourself reaching for it, you probably want a status return and a `goto cleanup`.

---

## Checklist

```text
[ ] Does every resource get cleaned up on every failure path?
[ ] Is there a single cleanup label instead of repeated free blocks?
[ ] Do I free() and then set the pointer to NULL?
[ ] After a double free, would the program corrupt or abort?
[ ] Do I exit(0) on success and exit(nonzero) on failure?
[ ] Is abort() reserved for broken invariants, not bad input?
```

---

#### [Previous](./02-part.md) | [Next](./exercises.md)