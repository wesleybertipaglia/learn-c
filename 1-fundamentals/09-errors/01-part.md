# The C Error Model

C has no exceptions. If you learned C#, Java, JavaScript or Python first, the biggest adjustment of all is here: when something goes wrong in C, the program does not jump. It returns a value, and the caller decides what to do about it.

---

## No `throw`, No `try`, No `catch`

In C#:

```text
try {
    Save(path);
} catch (IOException e) {
    Log(e.Message);
}
```

In C those keywords do not exist. There is no keyword at all — the language simply does not have the concept. So how does a C function tell you it failed?

It returns a value that means "I failed", and it is **your job to look**.

---

## The Rule

> A function that can fail returns a status. The caller checks the status.

That is the entire model. Everything in this chapter is a variation on it.

| Language | How a failure travels |
| --- | --- |
| C# / Java | an **exception object** is thrown and unwinds the stack until something catches it |
| Go | an extra `error` value is returned and checked |
| **C** | **an `int`, an enum or a pointer value is returned, and it is checked** |

C is closer to Go than to C# here. C# hides the error path unless you look at the signature; in C the error path is always written out in plain sight, on the same screen as the happy path.

---

## Returning `int` as a Status

The simplest convention: `0` means success, anything else means failure.

```c
#include <stdio.h>

int divide(int a, int b) {
    if (b == 0) {
        return -1;
    }
    return a / b;
}
```

There is a real problem hiding in that version. A valid result can be negative, so `-1` is ambiguous. Two standard fixes:

1. Return the status through a **separate variable** (via pointer), and keep the real value as the return.
2. Return an **enum** with named values, so nothing is ambiguous.

The version that scales uses both — a named status returned, and the real value written through a pointer:

```c
typedef enum {
    STATUS_OK = 0,
    STATUS_NOT_FOUND,
    STATUS_NO_MEMORY,
    STATUS_INVALID
} status_t;

static const char *status_text(status_t s) {
    switch (s) {
        case STATUS_OK:        return "ok";
        case STATUS_NOT_FOUND: return "not found";
        case STATUS_NO_MEMORY: return "out of memory";
        case STATUS_INVALID:   return "invalid argument";
    }
    return "unknown status";
}

static status_t divide_checked(int a, int b, int *out) {
    if (out == NULL || b == 0) {
        return STATUS_INVALID;
    }
    *out = a / b;
    return STATUS_OK;
}
```

The call site looks like this, every single time:

```c
int quotient = 0;
status_t rc;

rc = divide_checked(10, 2, &quotient);
if (rc != STATUS_OK) {
    fprintf(stderr, "error: %s\n", status_text(rc));
    return 1;
}
printf("10 / 2 = %d\n", quotient);
```

```text
10 / 2 = 5
```

`status_text` is not decoration. It exists so the human-readable message lives in exactly one place and the branch that handles the error stays short enough to read at a glance.

That verbosity is not a flaw in C. It is the feature. In C# you write `try`/`catch` and the compiler guarantees someone handles it. In C, the guarantee is that **you saw the code**. When a function returns a status and you ignore it, the compiler stays silent and the bug is yours — which is why the next section exists.

---

## The Golden Rule: Never Ignore a Return Value

```c
malloc(100);            /* WRONG: memory could be NULL, you never looked */
printf("%d\n", 42);     /* WRONG: could have failed, you never looked */
fclose(f);              /* WRONG: a flush error is silently discarded */
```

Every one of those compiles without a warning. There is no `[[nodiscard]]` in C11, nothing forces you. The habit has to be manual, and it is the single most important habit in this chapter:

> A function that returns something is telling you something. Read it, check it, act on it.

For functions that report errors through `errno` instead of a return value (like `printf`), you at least check for a sentinel: `printf` returns the number of characters written, or a **negative value** on error. `fflush(NULL)` returns `0` on success and `EOF` on failure.

---

## `<errno.h>`: What Went Wrong

Most of the C standard library cannot return a useful status for every failure, so it uses a separate global variable: **`errno`**, declared in `<errno.h>`.

The rules:

1. A function that fails sets `errno` to a nonzero code describing the failure.
2. You must read `errno` **immediately** after the call, before any other library call. Any function may overwrite it.
3. `errno` has no meaning when the call succeeded. Do not read it "just in case".
4. `errno` is never set to zero for you. Set it to `0` yourself before the call if you want a clean check.

The common codes you will meet:

| `errno` | Macro | Meaning |
| --- | --- | --- |
| 2 | `ENOENT` | No such file or directory |
| 13 | `EACCES` | Permission denied |
| 12 | `ENOMEM` | Cannot allocate memory |
| 22 | `EINVAL` | Invalid argument |
| 9 | `EBADF` | Bad file descriptor |

### `perror` and `strerror`

You do not have to memorize the numbers. `<string.h>` gives you `strerror(errno)`, which returns a human-readable sentence, and `<stdio.h>` gives you `perror(label)`, which prints the sentence to **stderr** for you:

```c
#include <stdio.h>
#include <errno.h>
#include <string.h>

int main(void) {
    FILE *f;

    printf("checking for the file...\n");

    errno = 0;
    f = fopen("missing.txt", "r");

    if (f == NULL) {
        perror("open");
        fprintf(stderr, "errno = %d, strerror says: %s\n",
                errno, strerror(errno));
    } else {
        printf("file opened\n");
        fclose(f);
    }

    return 0;
}
```

```text
open: No such file or directory
errno = 2, strerror says: No such file or directory
checking for the file...
```

`perror("open")` printed `open: No such file or directory`. The argument is your own label describing **what you were trying to do** — `"open"`, `"malloc"`, `"fopen config.txt"`. That label is what makes the message useful, because `errno` only knows the symptom, not the call site.

Look at the order. The two error lines appear **before** `checking for the file...`, even though the `printf` ran first. `stderr` is unbuffered and `stdout` is, so on a terminal the errors show up immediately while the `printf` waits for the buffer to flush. That is not a bug — it is exactly why diagnostics belong on `stderr`: redirect stdout to a file and the errors still reach your terminal, in real time, while the results are collected cleanly.

---

## A Complete Example

Everything from this part, compiled and run:

```c
#include <stdio.h>
#include <errno.h>
#include <string.h>

typedef enum {
    STATUS_OK = 0,
    STATUS_NOT_FOUND,
    STATUS_NO_MEMORY,
    STATUS_INVALID
} status_t;

static const char *status_text(status_t s) {
    switch (s) {
        case STATUS_OK:        return "ok";
        case STATUS_NOT_FOUND: return "not found";
        case STATUS_NO_MEMORY: return "out of memory";
        case STATUS_INVALID:   return "invalid argument";
    }
    return "unknown status";
}

static status_t divide_checked(int a, int b, int *out) {
    if (out == NULL || b == 0) {
        return STATUS_INVALID;
    }
    *out = a / b;
    return STATUS_OK;
}

int main(void) {
    FILE    *f;
    int      quotient = 0;
    status_t rc;

    rc = divide_checked(10, 2, &quotient);
    if (rc != STATUS_OK) {
        fprintf(stderr, "error: %s\n", status_text(rc));
        return 1;
    }
    printf("10 / 2 = %d\n", quotient);

    rc = divide_checked(10, 0, &quotient);
    printf("10 / 0 returned %s\n", status_text(rc));

    errno = 0;
    f = fopen("missing.txt", "r");
    if (f == NULL) {
        perror("open");
        fprintf(stderr, "errno = %d, strerror says: %s\n",
                errno, strerror(errno));
    } else {
        printf("file opened\n");
        fclose(f);
    }

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g errors.c -o errors
./errors
```

```text
open: No such file or directory
errno = 2, strerror says: No such file or directory
10 / 2 = 5
10 / 0 returned invalid argument
```

---

## Where You Handle Errors

The one big difference in philosophy:

- In C#, an exception can travel a long way before it is caught. The throw site and the catch site can be in completely unrelated files.
- In C, the error is handled **where it happens**, in the caller, one line after the call. There is nowhere else for it to go.

```text
  C#                            C

  Save();                       rc = save(path);
      throws  ------------->     if (rc != STATUS_OK) {
  ...                                  handle_it_here();
  Log(e.Message);               }
  ...catch...
```

This means C error handling is more repetitive and much harder to forget. There is no mechanism that carries the failure somewhere else for you, so the code reads top to bottom exactly as it runs.

---

## Checklist

```text
[ ] Does this function return a status? Am I checking it?
[ ] If I read errno, am I reading it immediately after the failing call?
[ ] Do error messages go to stderr, not stdout?
[ ] Does every perror() call have a label that says what I was doing?
[ ] Is the status distinguishable from a legitimate result value?
```

---

#### [Previous](../08-pointers/05-part.md) | [Next](./02-part.md)