# Exercise List 2 - Unit Tests

Six exercises, from a minimal harness to a full suite for a container you already built. Keep one test program per exercise (`ex1.c` … `ex6.c`), compile with:

```text
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer
```

Rules for all six (enforced by the harness, not by honor):

- No external dependencies: only `<stdio.h>`, `<stdlib.h>`, `<string.h>` and your own code. No framework, no network, no packages.
- Every suite returns nonzero on failure: `return g_fail != 0 ? 1 : 0;` in `main`. A suite that always returns `0` fails the exercise even if the output looks right.
- No test depends on wall-clock time, open ports, absolute paths, or addresses. Fixed seeds and `tmpfile` only.
- Every `FAIL` line prints `file:line` plus `got` vs `want`. `printf`-only "tests" do not count.

## How to Work These Exercises

- Do them in order: 1 gives you the harness, 2–5 drill one pattern each, 6 assembles everything on a real container.
- Paste the 30-line harness from part 1 into each file (counters, `CHECK_EQ`, `CHECK_STR`, `RUN_TEST`). Do not share it via copy-paste of a possibly buggy version — retype it and diff.
- After every exercise run the happy path and at least one deliberately failing expectation, confirm the `FAIL file:line` message and exit code `1`, then fix it back and confirm exit code `0` under ASan/UBSan/LSan.
- If `gcc` warns, fix the warning before testing: `-Wall -Wextra -pedantic` catches the `size_t` vs `int` and missing-`const` mistakes these exercises are designed to surface.

---

## Exercise 1: Minimal Harness + 3 Tests of an Existing Function

Bring up the part-1 harness and point it at `linear_search` from `2-data-structures/01-arrays` (or any function you already wrote and trust).

```c
#include <stdio.h>
#include <string.h>

static int g_pass = 0;
static int g_fail = 0;

/* TODO: paste CHECK_EQ, CHECK_STR, RUN_TEST here (do/while(0), __FILE__/__LINE__). */

static int linear_search(const int values[], int n, int target) {
    /* TODO: copy your implementation (NULL -> -1, empty -> -1). */
    (void)values; (void)n; (void)target;
    return -1;
}

static void test_found(void) {
    /* TODO: at least one CHECK_EQ for first/middle/last element. */
}

static void test_missing(void) {
    /* TODO: absent value -> -1. */
}

static void test_null_empty(void) {
    /* TODO: NULL input and n == 0 both -> -1, no crash. */
}

int main(void) {
    /* TODO: RUN_TEST all three, print "N passed, M failed", return 1 on failure. */
    return 0;
}
```

```text
# output (all green)
[ RUN  ] test_found
[ PASS ] test_found
[ RUN  ] test_missing
[ PASS ] test_missing
[ RUN  ] test_null_empty
[ PASS ] test_null_empty
----
5 passed, 0 failed
```

```text
# output (one expectation broken on purpose, e.g. want 5 for absent value)
[ RUN  ] test_missing
  FAIL ex1.c:42: CHECK_EQ(linear_search(v, 4, 99), 5) -> got -1, want 5
[ FAIL ] test_missing (1 new failure(s))
----
4 passed, 1 failed
$ echo $?
1
```

Requirements:

- Harness has zero external dependencies; `CHECK_EQ` evaluates each argument once and prints `got`/`want` with `__FILE__`/`__LINE__` to stderr.
- At least 3 test functions, at least 5 checks total, covering found / missing / NULL / empty.
- `main` returns `0` when green and `1` when red; demonstrate both exit codes (break one expectation, show `exit=1`, fix it, show `exit=0`).
- Clean under `-Wall -Wextra -pedantic` with ASan/UBSan; no `assert`, no `exit` inside tests.

---

## Exercise 2: Table-Driven Binary Search or Comparator

Replace five near-identical test functions with one `{input, expected}` table and a loop. Pick `binary_search` (sorted array) or an integer/string comparator.

```c
#include <stdio.h>
#include <string.h>

static int g_pass = 0;
static int g_fail = 0;
/* TODO: harness macros. */

static int binary_search(const int v[], int n, int target) {
    /* TODO: lo/hi/mid with lo + (hi - lo) / 2, NULL/empty -> -1. */
    (void)v; (void)n; (void)target;
    return -1;
}

static void test_binary_table(void) {
    /* TODO: static const int data[] = {10, 20, 30, 40, 50}; */
    /* TODO: static const struct { int input; int expected; } cases[] with
       first / middle / last / below-range / above-range / gap rows (8+ rows). */
    /* TODO: loop, compare, print case index i on failure, count pass/fail. */
}

int main(void) {
    /* TODO: RUN_TEST + summary + nonzero-on-failure return. */
    return 0;
}
```

```text
# output
[ RUN  ] test_binary_table
[ PASS ] test_binary_table
----
8 passed, 0 failed
```

```text
# output (row 5 broken on purpose)
[ RUN  ] test_binary_table
  FAIL ex2.c:52: case 5: input 99 -> got -1, want 0
[ FAIL ] test_binary_table (1 new failure(s))
----
7 passed, 1 failed
```

Requirements:

- At least 8 rows: first, middle, last, below range, above range, gap between elements, plus `NULL` and empty handled (either as rows with a flag or as separate checks with justification in a comment).
- Failure message includes the row index, the input, `got`, and `want`.
- Every `expected` value hand-derived from the spec (trace `lo/mid/hi` on paper); no pasting program output into the table.
- Comparator alternative (if chosen instead): table of `{a, b, expected_sign}` with negative / zero / positive rows and a comment stating the ordering contract.

---

## Exercise 3: Fixture with `malloc`/`free` per Test (Prove with Sanitizer)

Test a heap-owning operation (string reverse into a fresh buffer, array duplicate, or vector push) where each case allocates and frees. Prove the teardown with sanitizers.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static int g_pass = 0;
static int g_fail = 0;
/* TODO: harness macros. */

/* TODO: function under test, e.g.:
   char *reverse_new(const char *s);  // malloc(strlen(s)+1), NULL -> NULL */

static void test_reverse_heap(void) {
    /* TODO: table {input, expected} with "", "a", "abc", "hello" rows. */
    /* TODO: per row: malloc (or call reverse_new), CHECK_STR, free.
       On malloc == NULL: count failure, continue (nothing to free). */
}

int main(void) {
    /* TODO: RUN_TEST + summary + nonzero-on-failure return. */
    return 0;
}
```

```text
# output
[ RUN  ] test_reverse_heap
[ PASS ] test_reverse_heap
----
4 passed, 0 failed
```

```text
# sanitizer proof (run after commenting out ONE free on purpose, then restore)
==12345==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 6 byte(s) in 1 object(s) allocated from:
    #0 0x... in malloc ...
    #1 0x... in test_reverse_heap ex3.c:30

SUMMARY: AddressSanitizer: 6 byte(s) leaked in 1 allocation(s).
```

Requirements:

- `malloc` and `free` live in the same iteration: exactly one `free` per successful allocation, reached on both pass and fail branches.
- Table has at least 4 rows including `""` and a single-character string.
- Demonstrate the leak check: comment out one `free`, capture the LeakSanitizer report above, then restore the `free` and show a clean run. Paste both runs in a comment at the top of the file.
- `NULL` input case documented: state whether the function returns `NULL` or `""` and test that contract.

---

## Exercise 4: File Round-Trip with `tmpfile`

Test a file-writing and file-reading pair without touching the real directory tree: write lines, `rewind`, read back, compare.

```c
#include <stdio.h>
#include <string.h>

static int g_pass = 0;
static int g_fail = 0;
/* TODO: harness macros. */

/* TODO: helpers under test, e.g.:
   int write_lines(FILE *f, const char *const *lines, size_t n);
   int read_lines(FILE *f, char out[][256], size_t n);  // 0 ok, -1 error */

static void test_round_trip(void) {
    /* TODO: tmpfile(); on NULL count failure and return. */
    /* TODO: write {"alpha", "beta", "gamma"}, rewind, read back, CHECK_STR each. */
    /* TODO: fclose (file vanishes). */
}

static void test_empty_file(void) {
    /* TODO: tmpfile, rewind immediately, fgets -> NULL is the expected result. */
    /* TODO: fclose. */
}

int main(void) {
    /* TODO: RUN_TEST both, summary, nonzero-on-failure. */
    return 0;
}
```

```text
# output
[ RUN  ] test_round_trip
[ PASS ] test_round_trip
[ RUN  ] test_empty_file
[ PASS ] test_empty_file
----
4 passed, 0 failed
```

Requirements:

- Uses `tmpfile()` (or `mkstemp` + `close` + `unlink` with a comment explaining why the name is needed); no fixed paths like `/tmp/test.txt`.
- Checks every I/O return (`fputs`/`fputc`/`fgets`/`fread`); `rewind` (or `fseek(f, 0, SEEK_SET)`) between write and read phases.
- Strips at most one trailing `\n` with `strcspn` before `CHECK_STR`; documents the `NULL` vs `""` decision for empty files.
- Empty-file case included; `fclose` on every path including the write-failure path.

---

## Exercise 5: Removal from a List or Vector, Including Underflow

Test removal (`pop`, `list_remove`, `vector_pop`) where the interesting cases are the boundaries: last element, single element, and empty (underflow must return an error, never abort).

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static int g_pass = 0;
static int g_fail = 0;
/* TODO: harness macros. */

/* TODO: minimal container under test: array stack or int vector with
   push + pop(&out) returning 0 ok / -1 underflow (and -1 on OOM). */

static void test_pop_lifo_order(void) {
    /* TODO: push 10, 20, 30; pop -> 30, 20, 10 in order; CHECK_EQ each value and return. */
}

static void test_pop_single_then_empty(void) {
    /* TODO: push one value, pop it (size 0), pop again -> -1, container still valid. */
}

static void test_pop_empty_never_aborts(void) {
    /* TODO: fresh empty container, pop -> -1, peek -> -1, size still 0; */
    /* TODO: push after underflow still works (state not corrupted). */
}

int main(void) {
    /* TODO: RUN_TEST all three, summary, nonzero-on-failure. */
    return 0;
}
```

```text
# output
[ RUN  ] test_pop_lifo_order
[ PASS ] test_pop_lifo_order
[ RUN  ] test_pop_single_then_empty
[ PASS ] test_pop_single_then_empty
[ RUN  ] test_pop_empty_never_aborts
[ PASS ] test_pop_empty_never_aborts
----
10 passed, 0 failed
```

Requirements:

- Status-return convention: `0` success, `-1` underflow/OOM; no `abort`/`exit`/`assert` in container code on input-driven underflow.
- LIFO order asserted value-by-value, not just "size decreased".
- Underflow leaves the container usable: a `push` after a failed `pop` succeeds and the elements come back in order.
- ASan/LSan clean; exactly one destroy/free per container; `realloc` (if array-backed) goes through a `tmp` pointer.

---

## Exercise 6: Turn an Old `exercises.md` into a Suite (e.g. Stacks)

Pick one exercise list you already completed — `06-stacks` is ideal — and convert its manual scripts into an automated suite: every `# output` block becomes `CHECK_*` rows.

```c
/* ex6.c — stack suite converted from 2-data-structures/06-stacks/exercises.md */
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <stddef.h>

static int g_pass = 0;
static int g_fail = 0;
/* TODO: harness macros (CHECK_EQ + RUN_TEST at minimum). */

/* TODO: paste your array-stack implementation:
   stack_init / stack_destroy / stack_push / stack_pop / stack_peek / stack_is_empty. */

static void test_script_ex1(void) {
    /* TODO: convert the ex1 script: push 10/20/30, peek 30, pop 30/20/10,
       final over-pop -> -1. One CHECK per step (value AND return code). */
}

static void test_growth(void) {
    /* TODO: push 5+ elements, CHECK capacity sequence 4 -> 8 (and size each step). */
}

static void test_balanced_or_calc(void) {
    /* TODO: convert one application: balanced("a+(b*[c-{d}])") == true,
       balanced("([)]") == false, OR eval_postfix("3 4 + 2 *") == 14. */
}

static void test_underflow_never_aborts(void) {
    /* TODO: pop/peek on empty -> -1; min/get on empty (if min-stack) -> -1. */
}

int main(void) {
    /* TODO: RUN_TEST all four, summary, nonzero-on-failure. */
    return 0;
}
```

```text
# output
[ RUN  ] test_script_ex1
[ PASS ] test_script_ex1
[ RUN  ] test_growth
[ PASS ] test_growth
[ RUN  ] test_balanced_or_calc
[ PASS ] test_balanced_or_calc
[ RUN  ] test_underflow_never_aborts
[ PASS ] test_underflow_never_aborts
----
18 passed, 0 failed
```

Requirements:

- At least 4 test functions, at least 15 checks total; every manual `# output` line from the original exercise maps to at least one `CHECK_*` (note the mapping in a comment per test).
- Growth behavior asserted (capacity `4 -> 8`, or one-`malloc`-per-push for the linked version with LSan proof).
- At least one application test (balanced brackets, postfix `14`, or reverse) driven through tables where inputs vary.
- Underflow suite included; suite returns `1` on any failure; state at the top of the file which original `exercises.md` was converted and which exercise numbers map to which `test_*` functions.

---

#### [Previous](./02-part.md) | [Next](../03-test-frameworks/01-part.md)
