# A Hand-Rolled Test Harness

A unit test checks one small function against known inputs and expected outputs. No user input, no network, no manual inspection: the test program decides by itself whether everything passed, prints a summary, and returns an exit code the shell (and your CI) can trust.

You do not need a framework to do that in C. About thirty lines of macros give you named tests, file-and-line failure reports, pass/fail counters, and a `main` that returns `0` on success and `1` on failure. This part builds that harness from scratch and uses it to test functions you already know from `01-arrays` and `02-strings`, including `linear_search`.

Compile everything in this chapter with:

```text
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer
```

The sanitizers matter here more than usual: a test that passes without ASan/UBSan may only be passing by accident (out-of-bounds read that happened to find the right byte). Run every suite under sanitizers, always.

---

## What a Harness Does

A test harness has exactly three jobs:

```text
  +------------------+      +------------------+      +------------------+
  | code under test  |      | test functions   |      | main + counters  |
  |                  |      |                  |      |                  |
  | linear_search()  |--<---| test_search_...  |--->--| g_pass / g_fail  |
  | count_occurrences|--<---| CHECK_EQ /       |      | summary +        |
  | string_length()  |      | CHECK_STR        |      | return 0 or 1    |
  +------------------+      +------------------+      +------------------+
         ^                          |                          |
         |              __FILE__ + __LINE__              exit code for CI
     pure functions      on every failure             0 = all green
     (no exit inside)    printed to stderr             1 = something red
```

Rules that keep the diagram honest:

| Rule | Why |
| --- | --- |
| Code under test returns values, never calls `exit` | Otherwise the first failure kills the whole suite |
| Each test function returns `void` and records failures in globals | `main` decides the exit code once, at the end |
| Failures print `file:line` plus `got` vs `want` | Without a location, a failing suite is a guessing game |
| The suite keeps running after a failure | One broken test must not hide the other five |

---

## The Whole Harness in 30 Lines

Here is the complete harness. Copy it verbatim to the top of every test file in this chapter. It has zero dependencies beyond `<stdio.h>` and `<string.h>`:

```c
#include <stdio.h>
#include <string.h>

/* Global counters. Every CHECK_* updates exactly one of them. */
static int g_pass = 0;
static int g_fail = 0;

#define CHECK_EQ(actual, expected) do {                                  \
    long long a_ = (long long)(actual);                                  \
    long long e_ = (long long)(expected);                                \
    if (a_ == e_) {                                                      \
        g_pass++;                                                        \
    } else {                                                             \
        g_fail++;                                                        \
        fprintf(stderr, "  FAIL %s:%d: CHECK_EQ(%s, %s) -> got %lld, want %lld\n", \
                __FILE__, __LINE__, #actual, #expected, a_, e_);         \
    }                                                                    \
} while (0)

#define CHECK_STR(actual, expected) do {                                 \
    const char *a_ = (actual);                                           \
    const char *e_ = (expected);                                         \
    int ok_ = 0;                                                         \
    if (a_ == NULL && e_ == NULL) {                                      \
        ok_ = 1;                                                         \
    } else if (a_ != NULL && e_ != NULL && strcmp(a_, e_) == 0) {         \
        ok_ = 1;                                                         \
    }                                                                    \
    if (ok_) {                                                           \
        g_pass++;                                                        \
    } else {                                                             \
        g_fail++;                                                        \
        fprintf(stderr, "  FAIL %s:%d: CHECK_STR(%s, %s) -> got \"%s\", want \"%s\"\n", \
                __FILE__, __LINE__, #actual, #expected,                  \
                a_ ? a_ : "(null)", e_ ? e_ : "(null)");                 \
    }                                                                    \
} while (0)

#define RUN_TEST(fn) do {                                                \
    int fails_before_ = g_fail;                                          \
    printf("[ RUN  ] %s\n", #fn);                                        \
    fn();                                                                \
    if (g_fail == fails_before_) {                                       \
        printf("[ PASS ] %s\n", #fn);                                    \
    } else {                                                             \
        printf("[ FAIL ] %s (%d new failure(s))\n",                      \
               #fn, g_fail - fails_before_);                             \
    }                                                                    \
} while (0)
```

And the `main` that every suite ends with:

```c
int main(void) {
    RUN_TEST(test_search_found);
    RUN_TEST(test_search_missing);
    RUN_TEST(test_search_empty);

    printf("----\n%d passed, %d failed\n", g_pass, g_fail);
    return g_fail != 0 ? 1 : 0;
}
```

Three preprocessor tricks do all the work:

1. **`__FILE__` and `__LINE__`** expand at the call site, not inside the macro definition. That is why the failure message points at your test line, not at the harness header. Without them you would know *that* something failed but not *where*.
2. **`#actual` (stringification)** turns the expression text into a string literal. `CHECK_EQ(linear_search(v, 4, 30), 2)` prints the literal text `linear_search(v, 4, 30)` in the failure line, so you can see which call broke without opening the file.
3. **`do { ... } while (0)`** makes the macro behave like a single statement. A bare `{ ... }` breaks `if (x) CHECK_EQ(...); else ...` because the trailing semicolon creates an empty statement between the block and the `else`. The `do/while` wrapper absorbs that semicolon.

Each `CHECK_*` evaluates its arguments exactly once (into `a_` / `e_`). That matters: `CHECK_EQ(stack_pop(&s, &v), 0)` must not pop twice. Never write a check macro that mentions `actual` two times in the body.

---

## Counters and Exit Codes: the CI Contract

The shell only sees one byte from your program: the exit code. The convention this course uses everywhere:

| Exit code | Meaning | What CI does |
| --- | --- | --- |
| `0` | all checks passed | green, continue pipeline |
| `1` | at least one check failed | red, stop pipeline |
| `2` (optional) | harness usage error (bad argument) | red, but points at the suite itself |

`main` implements it in one line: `return g_fail != 0 ? 1 : 0;`. Nothing else in the program may call `exit` or `abort` on a test failure — the library function returns an error code, the check macro counts it, `main` translates the count into the exit code at the very end.

In a shell (and in CI) it looks like this:

```text
$ gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer test_search.c -o test_search
$ ./test_search; echo "exit=$?"
[ RUN  ] test_search_found
[ PASS ] test_search_found
...
exit=0
```

```text
$ ./test_search_broken; echo "exit=$?"
[ RUN  ] test_search_found
[ PASS ] test_search_found
[ RUN  ] test_search_missing
  FAIL test_search_broken.c:71: CHECK_EQ(linear_search(v, 4, 99), 5) -> got -1, want 5
[ FAIL ] test_search_missing (1 new failure(s))
----
2 passed, 1 failed
exit=1
```

A minimal CI step is just "build with warnings as errors, run, trust the exit code":

```sh
#!/bin/sh
set -eu
cc -std=c11 -Wall -Wextra -pedantic -Werror -g \
   -fsanitize=address,undefined -fno-omit-frame-pointer \
   test_search.c -o test_search
./test_search
```

If the binary returns nonzero, `set -e` fails the job. No output parsing needed.

> Never `return 0` unconditionally from a test `main`. A suite that always exits zero is decoration: CI stays green while the code is broken.

---

## Example: Testing `linear_search` and Friends

These are the exact functions from `2-data-structures/01-arrays` and `02-strings`, copied into the test file so the suite is self-contained:

```c
#include <stdio.h>
#include <string.h>

static int g_pass = 0;
static int g_fail = 0;

#define CHECK_EQ(actual, expected) do {                                  \
    long long a_ = (long long)(actual);                                  \
    long long e_ = (long long)(expected);                                \
    if (a_ == e_) {                                                      \
        g_pass++;                                                        \
    } else {                                                             \
        g_fail++;                                                        \
        fprintf(stderr, "  FAIL %s:%d: CHECK_EQ(%s, %s) -> got %lld, want %lld\n", \
                __FILE__, __LINE__, #actual, #expected, a_, e_);         \
    }                                                                    \
} while (0)

#define CHECK_STR(actual, expected) do {                                 \
    const char *a_ = (actual);                                           \
    const char *e_ = (expected);                                         \
    int ok_ = 0;                                                         \
    if (a_ == NULL && e_ == NULL) {                                      \
        ok_ = 1;                                                         \
    } else if (a_ != NULL && e_ != NULL && strcmp(a_, e_) == 0) {         \
        ok_ = 1;                                                         \
    }                                                                    \
    if (ok_) {                                                           \
        g_pass++;                                                        \
    } else {                                                             \
        g_fail++;                                                        \
        fprintf(stderr, "  FAIL %s:%d: CHECK_STR(%s, %s) -> got \"%s\", want \"%s\"\n", \
                __FILE__, __LINE__, #actual, #expected,                  \
                a_ ? a_ : "(null)", e_ ? e_ : "(null)");                 \
    }                                                                    \
} while (0)

#define RUN_TEST(fn) do {                                                \
    int fails_before_ = g_fail;                                          \
    printf("[ RUN  ] %s\n", #fn);                                        \
    fn();                                                                \
    if (g_fail == fails_before_) {                                       \
        printf("[ PASS ] %s\n", #fn);                                    \
    } else {                                                             \
        printf("[ FAIL ] %s (%d new failure(s))\n",                      \
               #fn, g_fail - fails_before_);                             \
    }                                                                    \
} while (0)

/* ---- code under test (from 01-arrays / 02-strings) ---- */

static int linear_search(const int values[], int n, int target) {
    if (values == NULL || n <= 0) {
        return -1;
    }
    for (int i = 0; i < n; i++) {
        if (values[i] == target) {
            return i;
        }
    }
    return -1;
}

static int count_occurrences(const int values[], int n, int target) {
    if (values == NULL || n <= 0) {
        return 0;
    }
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (values[i] == target) {
            count++;
        }
    }
    return count;
}

/* Returns the length of a NUL-terminated string, 0 for NULL. */
static size_t string_length(const char *s) {
    if (s == NULL) {
        return 0;
    }
    size_t n = 0;
    while (s[n] != '\0') {
        n++;
    }
    return n;
}

/* ---- tests ---- */

static void test_search_found(void) {
    int v[] = {10, 20, 30, 40};
    CHECK_EQ(linear_search(v, 4, 10), 0);   /* first element */
    CHECK_EQ(linear_search(v, 4, 30), 2);   /* middle element */
    CHECK_EQ(linear_search(v, 4, 40), 3);   /* last element */
}

static void test_search_missing(void) {
    int v[] = {10, 20, 30, 40};
    CHECK_EQ(linear_search(v, 4, 99), -1);
    CHECK_EQ(linear_search(NULL, 4, 10), -1);  /* NULL input, not a crash */
    CHECK_EQ(linear_search(v, 0, 10), -1);     /* empty range */
}

static void test_count_and_length(void) {
    int v[] = {5, 7, 5, 5, 9};
    CHECK_EQ(count_occurrences(v, 5, 5), 3);
    CHECK_EQ(count_occurrences(v, 5, 7), 1);
    CHECK_EQ(count_occurrences(v, 5, 0), 0);
    CHECK_EQ((long long)string_length("hello"), 5);
    CHECK_EQ((long long)string_length(""), 0);
    CHECK_STR("hello", "hello");
}

int main(void) {
    RUN_TEST(test_search_found);
    RUN_TEST(test_search_missing);
    RUN_TEST(test_count_and_length);

    printf("----\n%d passed, %d failed\n", g_pass, g_fail);
    return g_fail != 0 ? 1 : 0;
}
```

Real output (all green):

```text
[ RUN  ] test_search_found
[ PASS ] test_search_found
[ RUN  ] test_search_missing
[ PASS ] test_search_missing
[ RUN  ] test_count_and_length
[ PASS ] test_count_and_length
----
12 passed, 0 failed
```

Count the checks: 3 + 3 + 6 = 12, matching the summary line. If the summary ever disagrees with your manual count, one of your tests returned early (a stray `return` inside a test function skips the remaining checks) — grep for `return` inside `test_*` functions and remove it.

---

## A Failing Test on Purpose

Change one expectation to something wrong — expecting index `5` for a value that is absent:

```c
static void test_search_missing_broken(void) {
    int v[] = {10, 20, 30, 40};
    CHECK_EQ(linear_search(v, 4, 99), 5);   /* WRONG on purpose */
}
```

Real output (compiled with the same flags, run, then `echo $?`):

```text
[ RUN  ] test_search_found
[ PASS ] test_search_found
[ RUN  ] test_search_missing_broken
  FAIL test_search.c:88: CHECK_EQ(linear_search(v, 4, 99), 5) -> got -1, want 5
[ FAIL ] test_search_missing_broken (1 new failure(s))
----
2 passed, 1 failed
```

```text
$ echo $?
1
```

Read the failure line left to right: file `test_search.c`, line `88`, the exact expression text, then `got -1, want 5`. That is the whole debugging session for a pure function: the contract says "absent means `-1`", the test demanded `5`, so the test is wrong, not the function. Fix the expectation back to `-1` and the suite goes green again.

Note where each stream goes: `[ RUN ]` / `[ PASS ]` / summary on **stdout**, `FAIL` details on **stderr**. That split lets CI show the summary while still capturing precise failures: `./test_search >results.txt 2>failures.txt` keeps both.

---

## Bugs You Will Write (and What They Look Like)

### Bug 1: `assert` Inside the Suite

```c
#include <assert.h>
static void test_search_abort(void) {
    int v[] = {10, 20, 30};
    assert(linear_search(v, 3, 99) == 5);  /* BUG: aborts on first failure */
    CHECK_EQ(linear_search(v, 3, 10), 0);  /* never runs */
}
```

```text
test_search: test_search.c:4: test_search_abort: Assertion `linear_search(v, 3, 99) == 5' failed.
Aborted (core dumped)
```

The process dies, remaining tests never run, and the exit code is `134` (SIGABRT), not `1`. `assert` is for impossible internal invariants, never for test expectations. Check macros count the failure and continue.

### Bug 2: `printf`-Only Tests With No Counters

```c
static void test_looks_fine(void) {
    int v[] = {10, 20, 30};
    printf("search 20 -> %d\n", linear_search(v, 3, 20));  /* BUG: human must read it */
}
```

The output `search 20 -> 1` looks right, but the program returns `0` even when the function is broken — nothing compares against an expected value. Every test must end in a `CHECK_*` that moves a counter. If you cannot state the expected value, you do not have a test yet.

### Bug 3: Forgetting to Register the Test in `main`

```c
static void test_count_empty(void) {
    CHECK_EQ(count_occurrences(NULL, 0, 5), 0);
}
/* BUG: main never calls RUN_TEST(test_count_empty), so it never runs. */
```

The suite is green and the coverage is zero for that function. Keep `main`'s `RUN_TEST` list alphabetical and grep for `static void test_` vs `RUN_TEST(` — every definition must appear exactly once in `main`.

### Bug 4: Comparing Strings With `==`

```c
CHECK_EQ(greeting == "hello", 1);       /* BUG: compares pointers, not contents */
CHECK_STR(greeting, "hello");           /* correct: strcmp inside */
```

Two identical string literals may or may not share an address; two separately built buffers never do. Pointer comparison passes one day and fails the next. Strings always go through `CHECK_STR`.

| Bug | Symptom | Fix |
| --- | --- | --- |
| `assert` for expectations | abort, exit 134, suite stops early | `CHECK_*` + continue |
| `printf` without checks | green exit code on broken code | every test ends in `CHECK_*` |
| test missing from `main` | green suite, untested function | one `RUN_TEST` per `test_*` |
| `==` on strings | flaky pass/fail on identical text | `CHECK_STR` (`strcmp`) |
| `return 0` unconditionally | CI green on failure | `return g_fail != 0 ? 1 : 0` |
| `exit()` inside library code | suite dies on first error input | status return, harness counts it |

---

## Checklist

- [ ] I can paste the 30-line harness from memory: two counters, `CHECK_EQ`, `CHECK_STR`, `RUN_TEST`.
- [ ] I know what `__FILE__`, `__LINE__`, and `#expr` expand to, and why `do { } while (0)` is required.
- [ ] Each `CHECK_*` evaluates its arguments once and prints `got` vs `want` to stderr.
- [ ] `main` runs every `test_*` via `RUN_TEST`, prints `N passed, M failed`, and returns `1` when `M > 0`.
- [ ] My tests cover found / missing / NULL / empty for `linear_search`, and I have seen one deliberately failing test produce a `FAIL file:line` line plus exit code `1`.
- [ ] The suite compiles clean under `-Wall -Wextra -pedantic` with ASan/UBSan and uses no external framework.
- [ ] No test depends on wall-clock time, network ports, absolute paths, or random addresses.

---

#### [Previous](../01-introduction/exercises.md) | [Next](./02-part.md)
