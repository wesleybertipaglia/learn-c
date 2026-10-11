# Final Challenge of Module 4 — and of the Course

You have finished the whole course: fundamentals, data structures, advanced C, and testing. This last project does not teach a new feature. It asks you to prove that everything you built actually works — with a test suite strict enough that you would trust it on someone else's machine.

---

## Objective

Write a **fully tested integer vector library** in C (`IntVec`: dynamic array with doubling growth), shipped as a multi-file project with a `Makefile`, a hand-rolled test suite, sanitizer runs, and a coverage report.

The suite must pass with zero warnings, zero sanitizer diagnostics, and no uncovered failure branches:

```bash
make
make test
make sanitize
make coverage
```

`make test` printing `all tests passed` is the passing state. Anything else — a warning, a leak, an untested branch — is a failure, even if the demo output looks right.

---

## Features

The library under test (`include/vec.h`, `src/vec.c`):

```c
typedef struct {
    int   *data;
    size_t size;
    size_t capacity;
} IntVec;

void vec_init(IntVec *v);
void vec_free(IntVec *v);
int  vec_push(IntVec *v, int value);              /* 0 ok, -1 on NOMEM */
int  vec_get(const IntVec *v, size_t i, int *out); /* 0 ok, -1 on bounds */
int  vec_pop(IntVec *v, int *out);                 /* 0 ok, -1 on empty */
size_t vec_size(const IntVec *v);
```

The test binary (`tests/test_vec.c`) covers, at minimum:

```text
  push/get ............ 8 cases (table-driven, incl. growth 1-2-4-8)
  pop ................. 4 cases (incl. empty -> STATUS, pop preserves order)
  bounds .............. 5 cases (get past end, get on empty, pop on empty)
  persistence ......... 2 cases (save to tmpfile, load back, identical)
  regression .......... 2 cases (one per bug below: realloc-loss, size/capacity swap)
  sanitize ............ clean under ASan+UBSan
  coverage ............ every failure branch executed at least once
```

| Suite | What it proves |
| --- | --- |
| push/get | growth works, order preserved, `size`/`capacity` consistent after every push |
| pop/bounds | underflow returns a status instead of crashing or reading garbage |
| persistence | `vec_save`/`vec_load` round-trip through a `tmpfile` byte-identically |
| regression | the two historical bugs stay fixed (see below) |
| sanitize | no leak, no overflow, no use-after-free in library **or** tests |
| coverage | `gcov -b` shows the `== NULL` and `>= size` branches were all taken |

---

## Requirements

### Project layout

```text
intvec/
  Makefile
  include/
    vec.h        /* public API with guards; no definitions */
    test.h       /* TEST()/CHECK_* mini-runner */
  src/
    vec.c        /* the library; every helper static */
    persist.c    /* vec_save/vec_load via FILE* (seam) */
  tests/
    test_vec.c   /* the suite; main returns 0/1 */
```

The `Makefile` has at least these targets:

```make
CC = gcc
CFLAGS = -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude
COVFLAGS = -fprofile-arcs -ftest-coverage

all:       build the demo
test:      build tests/test_vec and run it (fails the build on any failure)
sanitize:  rebuild with -fsanitize=address,undefined -fno-omit-frame-pointer and run
coverage:  rebuild with $(COVFLAGS), run, then gcov -b
clean:     remove objects, binaries and coverage data
```

### The two regression tests (mandatory)

These are not invented. They are the two bugs everyone writes once:

**Bug A — `realloc` into the live pointer.** The first version of `vec_push` did `v->data = realloc(v->data, ...)`. When `realloc` fails it returns `NULL` **without freeing the old block** — so the assignment destroys the only pointer to live data: a leak plus data loss in one line. The fix routes through a temporary:

```c
int *tmp = realloc(v->data, new_cap * sizeof(int));
if (tmp == NULL) {
    return -1;   /* v->data untouched, size/capacity untouched */
}
v->data = tmp;
```

Your regression test forces the failure path (allocation seam: inject a failing allocator via function pointer, or test the invariant "on `-1`, `size`, `capacity` and old contents are unchanged") and fails against the one-liner version. Prove it: temporarily restore the bug, watch the test go red, paste the output.

**Bug B — `size`/`capacity` swapped after growth.** A version that sets `v->size = new_cap` instead of `v->capacity = new_cap` passes every test that only pushes and reads — until a `pop` or a second growth reads garbage. Your regression test asserts `vec_size` **and** capacity behavior after each push in the 1-2-4-8 growth sequence (`CHECK_EQ(v.size, ...)` plus a `CHECK` that `capacity >= size`), so the swap turns the suite red. Prove it the same way.

### Testing discipline

1. **Table-driven where there is a table.** Push/get cases are `{input, expected}` rows with the row index in every failure message — never six copy-pasted blocks.
2. **Fixture per test.** Each test builds its own `IntVec`, and each one frees it — even on failure paths (the `goto cleanup` pattern inside tests too). The sanitizer run covers the tests directory, not just the library.
3. **Deterministic only.** No `rand()` without a fixed seed, no wall-clock, no network, no hardcoded `/tmp` paths — `tmpfile()` for persistence tests.
4. **Red-green proven.** For each regression test: check in the output of the suite against the *buggy* library (red) and against the fixed one (green). Both transcripts go in a comment at the top of `tests/test_vec.c`.
5. **Coverage of failure branches.** `gcov -b` must show the `malloc == NULL`, `index >= size`, and empty-pop branches as taken. An `if` your suite never exercises is an `if` you did not test.

### Style (same course rules, now enforced by `make test`)

1. Guards in every header, `static` helpers, `const` on read-only parameters, `size_t` for sizes, one declaration per line.
2. `realloc`/`malloc` always through a temporary + `NULL` check; `free(NULL)`-safe cleanup.
3. Errors to `stderr`, test progress to `stdout`, nonzero exit on any failure (so `make test` actually gates).
4. No test asserts on pointer formatting (`%p`), addresses, or timing.

---

## Example of Operation

```text
$ make test
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude -c src/vec.c -o src/vec.o
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude -c tests/test_vec.c -o tests/test_vec.o
gcc src/vec.o tests/test_vec.o -o tests/test_vec
./tests/test_vec
[PASS] push_get:8 (table rows 0..7, capacity 1-2-4-8-16)
[PASS] pop_order:4
[PASS] bounds:5 (empty get, past-end get, empty pop, ...)
[PASS] persistence:2 (save 10 ints to tmpfile, load back identical)
[PASS] regression_realloc:1 (failure path keeps data, red on one-liner)
[PASS] regression_sizecap:1 (size/capacity consistent, red on swap)
6 groups, 21 cases, 21 passed, 0 failed
all tests passed

$ make sanitize
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude -fsanitize=address,undefined -fno-omit-frame-pointer ... 
./tests/test_vec_san
6 groups, 21 cases, 21 passed, 0 failed
all tests passed

$ make coverage
./tests/test_vec_cov
6 groups, 21 cases, 21 passed, 0 failed
gcov -b src/vec.c
File 'src/vec.c'
Lines executed:100.00% of 42
Branches executed:100.00% of 18
```

And the red proof, kept in the test file header comment:

```text
# against buggy vec_push (direct realloc assignment), injected OOM on 3rd push:
[FAIL] regression_realloc tests/test_vec.c:88: vec_size kept (expected 2, got 3)
1 failed -> suite exit 1, make test stops. Bug reproduced, then fixed.
```

Notes:

- `make test` failing the build is the point: a red suite must stop the pipeline, not scroll past.
- 100% on this tiny library is achievable and therefore required. On a real codebase the target would be "every failure branch", not a number — the habit transfers, the threshold does not.

---

## Tips

### The seam that makes OOM testable

```c
/* vec.c: the only place that allocates, behind a pointer the tests can swap */
static void *(*alloc_fn)(size_t) = malloc;

void vec_set_allocator(void *(*fn)(size_t)) { alloc_fn = (fn != NULL) ? fn : malloc; }
```

Production never calls it. Tests install a function that fails on the Nth call, assert the library returns `-1` with data intact, then restore `malloc`. One seam, and the "untestable" failure path becomes a row in a table.

### `tmpfile` keeps persistence tests hermetic

```c
FILE *fp = tmpfile();
CHECK(fp != NULL);
vec_save(&v, fp);
rewind(fp);
CHECK_EQ(vec_load(&w, fp), 0);
CHECK_VEC_EQ(&v, &w);
fclose(fp);
```

No paths, no leftovers, no ordering dependence between tests. If `vec_save` takes a `FILE*` instead of a filename, this test is three lines — which is the test telling you the API is right.

### gcov shows what you forgot

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude \
    -fprofile-arcs -ftest-coverage src/vec.c tests/test_vec.c -o tests/test_vec_cov
./tests/test_vec_cov
gcov -b src/vec.c
```

Read the `branch` lines, not just the percentage. An `if (tmp == NULL)` marked `never taken` means your OOM test does not run — fix the test, not the metric.

---

## Extra challenge

If `make test`, `make sanitize` and `make coverage` are all green and the red-green proofs are pasted, pick one:

1. **Fuzz the parser.** Feed `vec_load` 100 malformed buffers (truncated, swapped fields, huge counts) from a table; every one must return an error, none may crash under ASan.
2. **Shrink API.** Add `vec_reserve`/`vec_shrink_to_fit` with tests proving contents survive both directions, including `shrink` to zero.
3. **Performance regression.** Assert the 1-2-4-8 growth sequence explicitly (capacity log per push) so a future "grow by one" change fails the suite on complexity, not just correctness.
4. **Test the tests.** Mutation-test by hand: flip one `<` to `<=` in `vec.c`, confirm exactly one test goes red. Revert. Repeat for two more operators.
5. **Port the suite.** Run the same `make test` under `clang` (or `-O2 -fanalyzer`) and document every new diagnostic — or the clean bill, with versions pasted.

---

## Grading Checklist

### Build

```text
[ ] make builds with -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude, zero warnings
[ ] make test builds the suite and runs it; any failure stops the build (nonzero exit)
[ ] make sanitize rebuilds with ASan+UBSan and passes silently
[ ] make coverage produces gcov -b output; failure branches all taken
[ ] make clean && make test rebuilds from scratch
```

### Library

```text
[ ] vec_push grows 1-2-4-8 via realloc-through-temporary; OOM returns -1, data intact
[ ] vec_get/vec_pop validate bounds/empty and return status, never abort on bad input
[ ] size/capacity consistent after every operation (suite asserts both, not just size)
[ ] vec_save/vec_load round-trip byte-identically via FILE* seam (tmpfile-tested)
[ ] Guards, static helpers, const readers, size_t sizes, one declaration per line
```

### Suite

```text
[ ] Table-driven push/get with row indices in failures; fixture malloc/free per test
[ ] Both regression tests present with red (buggy) and green (fixed) transcripts pasted
[ ] OOM path tested through allocator seam, not by exhausting real memory
[ ] Deterministic: fixed seeds only, tmpfile only, no addresses/ports/clocks asserted
[ ] No test ignored, no test order-dependent (each builds and frees its own vector)
```

### Discipline

```text
[ ] Every failure branch in vec.c executed at least once (gcov -b proof pasted)
[ ] Sanitizer output empty for library AND tests (paste the silent run)
[ ] Each bug fix started from a failing test, not from the fix (red-green order kept)
[ ] Comments explain why each test exists (which bug, which branch), not what it does
```

A submission that ships the vector, the suite, the `Makefile`, the sanitizer silence and the coverage proof has finished the course. One that also mutation-tests its own suite — and watches it catch every planted bug — is thinking like a C engineer now.

Congratulations. Go break something else.

---

#### [Previous](./03-test-frameworks/exercises.md)
