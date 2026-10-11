# Exercise List 1 - Testing Introduction

Five exercises covering both parts: silent bugs, reproducer discipline, sanitizer trios, seams, happy/edge/failure cases, and invariants-as-asserts. Every program must compile with `gcc -std=c11 -Wall -Wextra -pedantic -g file.c -o file` with **zero** warnings, then also compile and run clean under:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer file.c -o file_san
```

Rules for all five: `static` helpers, `const` on read-only pointer parameters, `size_t` for indices and counts, every `malloc` checked against `NULL` and matched with `free`, no VLAs, deterministic tests only (fixed seeds, no `time(NULL)` in logic, no assertions on `%p` addresses).

---

## Exercise 1: Write a Minimal Reproducer and Paste the ASan Report

Take a known off-by-one bug, shrink it to the smallest program that still triggers it, and capture the sanitizer verdict. This is the "reproduce first" habit from part 01 as muscle memory.

```c
#include <stdio.h>
#include <stddef.h>

/* BUG (given): reads one past the end. Do NOT fix before reproducing. */
static int sum_buggy(const int *a, size_t n) {
    int total = 0;
    for (size_t i = 0; i <= n; i++) {   /* TODO: reproduce first, then fix */
        total += a[i];
    }
    return total;
}

int main(void) {
    /* TODO: smallest input that triggers it (4 elements is enough).
       TODO: print want vs got and return nonzero on mismatch.
       TODO: run plain (expect accidental PASS) and under ASan (expect
       heap/stack-buffer-overflow). Paste the SUMMARY line in a comment.
       TODO: then fix to (i < n), re-run both builds, keep both outputs. */
    const int a[] = { 1, 2, 3, 4 };   /* stack-local on purpose: the plain
       build usually (not guaranteed!) reads a zero neighbour and prints
       an accidental PASS, while ASan reports stack-buffer-overflow */
    const size_t n = sizeof(a) / sizeof(a[0]);
    int got = sum_buggy(a, n);
    printf("want=10 got=%d %s\n", got, got == 10 ? "PASS" : "FAIL");
    return got == 10 ? 0 : 1;
}
```

Expected output (after the fix; before the fix the plain build accidentally prints the same PASS, which is the point):

```text
want=10 got=10 PASS
```

Requirements:

- Keep the reproducer to one file under 30 lines plus comments. No user input, no randomness, no extra helpers. The input `{1, 2, 3, 4}` with `want=10` is mandatory so reports are comparable.
- Run the buggy version twice and record both: plain build output (usually an accidental `PASS` with exit `0` for this stack-local array — record whatever YOUR machine prints, PASS or garbage FAIL) and sanitizer build output (full `ERROR: AddressSanitizer: stack-buffer-overflow` block ending in its `SUMMARY:` line). Paste the `SUMMARY:` line verbatim in a comment at the top of the file.
- Explain in a comment why the plain build passes (reads whatever sits adjacent; on your stack layout it happened to be benign) and why that makes the PASS untrustworthy without the sanitizer run.
- Apply the one-character-class fix (`<=` to `<`), re-run both builds, and confirm both print `PASS` with zero sanitizer diagnostics. Silence is half the grade.
- Keep the fixed reproducer in the file as a regression test: it must return `0` on success and nonzero on failure so a script can gate on it. State in a comment what future change would turn it red again.

---

## Exercise 2: The Sanitizer Trio — Plain vs ASan vs UBSan

Run the same three tiny programs under three builds and build the comparison table yourself. The goal is to feel which tool sees which bug family, instead of memorizing it.

```c
#include <stdio.h>
#include <stdlib.h>
#include <limits.h>

/* TODO: program A: heap over-read. malloc 4 ints, print a[4]. */
/* TODO: program B: signed overflow. bump(INT_MAX) via (x + 1). */
/* TODO: program C: leak. malloc 400 ints, touch p[0], skip free. */

static void prog_oob(void) {
    /* TODO: malloc, fill 0..3, printf a[4], free. */
}

static void prog_overflow(void) {
    /* TODO: int x = INT_MAX; printf("%d\n", x + 1); */
}

static void prog_leak(void) {
    /* TODO: malloc(400 * sizeof(int)), NULL check, p[0] = 42, no free. */
}

int main(void) {
    /* TODO: call all three in order with banner prints.
       TODO: build three binaries (plain / asan-only / asan+ubsan) and
       fill in the 3x3 table below with PASS / REPORT + report kind. */
    prog_oob();
    prog_overflow();
    prog_leak();
    return 0;
}
```

Expected output (plain build — all three look innocent; yours must match this shape before sanitizers are enabled):

```text
--- oob ---
a[4] = 0
--- overflow ---
-2147483648
--- leak ---
leak demo done
```

Requirements:

- Build all three variants from the same source: `plain` (`-g` only), `asan` (`-fsanitize=address`), `full` (`-fsanitize=address,undefined`). Use `-g -fno-omit-frame-pointer` on the sanitizer builds so line numbers appear.
- Fill in this table in a comment from your actual runs (do not copy the example verdicts blindly):

```text
  program      | plain        | asan-only              | asan+ubsan
  -------------+--------------+------------------------+----------------------
  oob (a[4])   | silent PASS? | REPORT (which kind?)   | REPORT (which kind?)
  overflow     | silent wrap? | silent? or REPORT?     | REPORT (which kind?)
  leak         | silent exit? | REPORT (Direct leak?)  | REPORT (Direct leak?)
```

- Paste one `SUMMARY:` line per program (three lines total) verbatim in comments, each labeled with the build that produced it.
- Explain in a comment why the overflow row differs between `asan-only` and `asan+ubsan` (ASan watches addresses; signed overflow needs UBSan), and why the leak row needs no special flag beyond ASan (LeakSanitizer rides along).
- Finish with a one-paragraph conclusion comment: which single build flag set you will use for the rest of the module and why running only the plain build would have certified all three bugs as "working".

---

## Exercise 3: Refactor to Testable — Add the Seam

Take an untestable function (hardcoded `stdout`, direct `malloc`, magic success path) and refactor it so every path is assertable. This is part 02's seam technique end to end.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

/* BEFORE (given, untestable): welded to malloc + stdout, no failure signal. */
static void make_and_print_bad(size_t n) {
    int *a = malloc(n * sizeof(int));
    /* BUG: no NULL check; BUG: prints instead of returning; untestable OOM path */
    for (size_t i = 0; i < n; i++) {
        a[i] = (int)(i * 2);
    }
    for (size_t i = 0; i < n; i++) {
        printf("%d ", a[i]);
    }
    printf("\n");
    free(a);
}

typedef void *(*alloc_fn)(size_t nbytes);

/* TODO: write make_doubles(FILE *out, size_t n, alloc_fn alloc):
   - return -1 on (NULL out), (NULL alloc), or allocation failure
   - return -1 when n > 0 and allocation fails; return 0 and write
     nothing (but succeed) when n == 0
   - on success write "%d " per element + "\n", return 0
   - set *out_arr? No: keep it simple, print path only, but status-checked. */

/* TODO: failing_alloc (returns NULL) and prod_alloc (wraps malloc). */

int main(void) {
    /* TODO: tmpfile seam test: make_doubles(tmp, 4, prod_alloc) == 0,
       rewind, check bytes equal "0 2 4 6 \n".
       TODO: failing_alloc test: returns -1, writes nothing.
       TODO: NULL-stream and n == 0 edge tests. Print one PASS line. */
    make_and_print_bad(4);   /* keep for before/after comparison, then stop calling it */
    return 0;
}
```

Expected output (after the refactor; the first line is the legacy function, the second is the test verdict):

```text
0 2 4 6
seam tests passed
```

Requirements:

- Do not modify `make_and_print_bad`; keep it as the documented "before". The new `make_doubles` must take `(FILE *out, size_t n, alloc_fn alloc)` in exactly that order and return `0`/`-1` only.
- Cover four cases with asserts: happy (`n = 4`, exact bytes `"0 2 4 6 \n"` via `tmpfile` + `rewind` + `fgets`/`fread` + `strcmp`), OOM (`failing_alloc` returns `-1`), null stream (`NULL` returns `-1`), and empty (`n = 0` returns `0` and writes zero bytes — verify with `ftell`).
- Verify the happy-path bytes exactly, including trailing space and newline. Explain in a comment why exact-byte matching is legitimate here (the format is the function's interface) while matching `%p` addresses would not be.
- Run the full file under ASan+UBSan with zero diagnostics, including the legacy call. No `free` is needed for the `tmpfile` path beyond `fclose`; any heap allocation in your solution must be `NULL`-checked and freed.
- End with a comment block answering: what does each seam parameter buy (`FILE *` / `alloc_fn`), and which production call replaces the test call (`make_doubles(stdout, n, malloc)` or your `prod_alloc` wrapper)?

---

## Exercise 4: Three Cases — Happy, Edge, Failure

Given a fixed function, write the three-case test suite that should have shipped with it. No refactoring allowed here: the function is already testable; your job is the suite.

```c
#include <assert.h>
#include <stdio.h>
#include <stddef.h>
#include <limits.h>

/* GIVEN (do not modify): 0 on success, -1 on bad input or overflow. */
static int checked_sum(const int *a, size_t n, int *out) {
    long long acc = 0;
    if (out == NULL) {
        return -1;
    }
    if (a == NULL && n != 0) {
        return -1;
    }
    for (size_t i = 0; i < n; i++) {
        acc += a[i];
        if (acc > INT_MAX || acc < INT_MIN) {
            return -1;
        }
    }
    *out = (int)acc;
    return 0;
}

int main(void) {
    int out = 0;
    /* TODO: HAPPY: {1,2,3,4} -> 10, return 0. */
    /* TODO: HAPPY: {-5, 5} -> 0, return 0. */
    /* TODO: EDGE: (any, 0) -> 0; (NULL, 0) -> 0. */
    /* TODO: EDGE: single element {42} -> 42. */
    /* TODO: FAILURE: NULL out -> -1; (NULL, 3) -> -1. */
    /* TODO: FAILURE: {INT_MAX, 1} -> -1; {INT_MIN, -1} -> -1. */
    (void)out;
    printf("TODO: replace with asserts + PASS line\n");
    return 0;
}
```

Expected output:

```text
happy: 2 passed
edge: 3 passed
failure: 4 passed
all sum tests passed
```

Requirements:

- Implement at least 2 happy, 3 edge, and 4 failure asserts (9 total) matching the expected-output counters. Each group prints its own `happy: N passed`-style line so a reader sees the taxonomy, not just a final PASS.
- Keep the three groups visually separated with `/* HAPPY */`, `/* EDGE */`, `/* FAILURE */` banners. A future reader must learn the contract from the group layout alone.
- For every failure assert, also assert that `out` was *not* clobbered with a success value where the contract demands it (decide and document: does your suite require `out` untouched on error, or merely the `-1` return? Pick one, comment it, test it).
- Include both overflow directions (`INT_MAX + 1` and `INT_MIN - 1`). Explain in a comment why unsigned wraparound would not be caught by UBSan here but signed overflow is — and why the function uses `long long` accumulation to stay defined.
- Run under ASan+UBSan with zero diagnostics and with `-Werror`. A single warning fails the exercise.

---

## Exercise 5: Invariants as Executable Asserts

Turn prose invariants ("sorted", "non-null on success", "length matches") into helper predicates plus asserts that run on every test. Documentation that executes.

```c
#include <assert.h>
#include <stdio.h>
#include <stddef.h>
#include <stdlib.h>

/* TODO: predicate 1: is_sorted_asc(const int *a, size_t n) -> 1/0.
   Must return 1 for n < 2 (vacuous truth); document why. */

/* TODO: predicate 2: all_even(const int *a, size_t n) -> 1/0. */

/* TODO: make_doubles(int **out, size_t n): malloc n doubles-of-index
   (0,2,4,...); on success *out set + return 0; on (n==0) *out=NULL + 0;
   on malloc failure *out=NULL + -1. NULL out param -> -1. */

static void bubble_sort(int *a, size_t n) {
    if (a == NULL || n < 2) {
        return;
    }
    for (size_t end = n; end > 1; end--) {
        for (size_t i = 0; i + 1 < end; i++) {
            if (a[i] > a[i + 1]) {
                int t = a[i];
                a[i] = a[i + 1];
                a[i + 1] = t;
            }
        }
    }
}

int main(void) {
    /* TODO: sort {5,1,4,2,8}, assert is_sorted_asc == 1.
       TODO: sort empty and single-element, assert vacuous 1.
       TODO: make_doubles(&p, 4): assert return 0, p != NULL,
       all_even(p, 4) == 1, then free(p).
       TODO: make_doubles(&p, 0): assert return 0 and p == NULL.
       TODO: print "invariants hold". */
    printf("TODO: replace with predicates + asserts\n");
    return 0;
}
```

Expected output:

```text
sorted: 1 2 4 5 8
doubles: 0 2 4 6
invariants hold
```

Requirements:

- Write both predicates as `static int` functions taking `(const int *a, size_t n)`, handling `n == 0` (and `NULL` with `n == 0`) without crashing. Document the vacuous-truth choice (`n < 2` is sorted) in a comment above `is_sorted_asc`.
- Call `is_sorted_asc` after *every* sort in `main` (random, empty, single-element, already-sorted, reversed) — five calls minimum. A sort without a post-check does not count.
- For `make_doubles`, assert the full invariant set on the happy path: return `0`, pointer non-`NULL`, length implicitly 4, `all_even == 1`, exact contents `{0, 2, 4, 6}`. On the empty path assert return `0` *and* `*out == NULL` (no allocation for zero elements).
- Never assert on pointer values themselves — only on `NULL`-ness, contents, and predicate results. State this rule in a comment and point at one line where a beginner would be tempted to print `%p`.
- Free every successful allocation exactly once, set the pointer to `NULL` after freeing in the test to model use-after-free hygiene, and finish with zero LeakSanitizer diagnostics. Paste your leak-free `OK`-equivalent (no-diagnostic run) as a comment: `leak check: no output from LeakSanitizer`.

---

## Sanitize Everything

Do this for **every** exercise above, and for every exercise from now on:

1. Compile with warnings as errors and with both sanitizers:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -Werror -g \
       -fsanitize=address,undefined -fno-omit-frame-pointer main.c -o main_san
   ```

2. Run it several times, including the empty and one-element inputs. Boundary bugs hide at size 0 and 1.
3. Fix everything the sanitizer reports. It is not a suggestion.
4. Re-run until the output contains no sanitizer diagnostics at all. Silence is the passing state.
5. Then compile the plain version and confirm identical behavior:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
   ```

Report, for each program, in a comment at the top of the file:

```text
[ ] compiles with -Werror, zero warnings
[ ] no sanitizer diagnostics
[ ] reproducer kept (ex 1) / trio table filled (ex 2)
[ ] seams used, no hardcoded stdout/malloc in testable code (ex 3)
[ ] happy/edge/failure groups all present and counted (ex 4)
[ ] every sort/allocation followed by its invariant predicate (ex 5)
[ ] every malloc has a matching free; no use after free
[ ] deterministic: fixed inputs, no rand/time in logic, no %p asserts
```

That checklist is the habit the whole module is trying to build. A test that passes it may still miss a logic case — but it will be repeatable, attributable, and sanitizer-clean, which means the missing case can be added instead of debugged from scratch.

---

#### [Previous](./02-part.md) | [Next](../02-unit-tests/01-part.md)
