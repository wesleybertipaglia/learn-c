# Writing Testable Code

Part 01 argued that C bugs are silent and that tests plus sanitizers make them loud. This part asks the prior question: *why is some C code easy to test and some nearly impossible?* The answer is rarely the test framework. It is the shape of the code under test.

Testable code has four properties: it computes through **pure functions** where possible, it reports problems through **checkable return values**, it reaches the outside world through **seams** you can substitute in tests, and it behaves **deterministically** for fixed inputs. Miss any one and the test becomes flaky, slow, or impossible to write.

---

## Pure Functions: Input In, Answer Out

A **pure function** reads only its arguments (and constants) and affects nothing except its return value. No globals, no I/O, no `static` counters hidden inside, no `malloc` without a matching contract. Given the same arguments it returns the same answer, every time, on every machine.

```c
#include <stddef.h>

/* PURE: result depends only on (a, n). No globals, no I/O. */
static int sum_pure(const int *a, size_t n) {
    int total = 0;
    for (size_t i = 0; i < n; i++) {
        total += a[i];
    }
    return total;
}
```

Testing it is one line per case:

```c
#include <assert.h>

int main(void) {
    static const int v[] = { 1, 2, 3, 4 };
    assert(sum_pure(v, 4) == 10);
    assert(sum_pure(v, 0) == 0);   /* length 0: reads nothing, returns 0 */
    return 0;
}
```

Compare the impure twin, which does the same arithmetic but drags the outside world in:

```c
#include <stdio.h>
#include <stddef.h>

static int g_total = 0;   /* hidden state: second call sees first call's sum */

/* IMPURE: reads/writes a global, prints, returns nothing checkable. */
static void sum_impure(const int *a, size_t n) {
    for (size_t i = 0; i < n; i++) {
        g_total += a[i];
    }
    printf("total=%d\n", g_total);   /* output is a side effect, not a value */
}
```

To test `sum_impure` you must reset `g_total`, capture `stdout`, and pray no earlier test polluted the global. To test `sum_pure` you call it. That gap widens with every test you add, which is why the discipline is: **push logic into pure functions; leave I/O and globals at the edges.** Heap allocation does not break the rule — an explicit ownership contract (`/* Caller owns the result, must free */`) plus a `free` in the test keeps it testable. What breaks it is *hidden* allocation the caller never learns about (a `static` buffer reused across calls, a leaked block).

| | Pure function | Impure function |
| --- | --- | --- |
| Depends on | arguments only | globals, files, time, `rand()` |
| Affects | return value only | globals, files, screen, heap ownership |
| Test needs | arguments + `assert` | fixtures, output capture, reset protocol |
| Repeatable | always | only with careful setup/teardown |
| Examples | `sum`, `is_sorted`, `parse_int` | `printf`-inside-logic, global accumulators |

Practical rules:

1. **Mark read-only inputs `const`.** `const int *a` tells the caller (and the test) the function borrows without mutating. If the function must mutate, say so: `/* MUTATES a in place */`.
2. **Keep helpers `static`.** A `static` helper is owned by its file; tests in the same file (or via `#include "module.c"` in tiny projects) reach it without polluting the global namespace.
3. **Return the answer; print it elsewhere.** `main` prints. Logic returns. The day logic prints, tests must parse text instead of comparing integers.

---

## Checkable Returns: Status Codes You Can Assert

A function that cannot report failure cannot be tested for failure. In C there are no exceptions, so the return value (plus output parameters) *is* the error channel. Design it before you implement.

The three standard shapes:

```c
#include <stddef.h>

/* Shape 1: 0 on success, -1 on failure. Classic for mutating operations. */
static int fill_sequence(int *out, size_t n) {
    if (out == NULL && n != 0) {
        return -1;
    }
    for (size_t i = 0; i < n; i++) {
        out[i] = (int)(i * 2);
    }
    return 0;
}

/* Shape 2: pointer-or-NULL. NULL means allocation (or lookup) failed. */
static const int *find_int(const int *a, size_t n, int target) {
    for (size_t i = 0; i < n; i++) {
        if (a[i] == target) {
            return &a[i];
        }
    }
    return NULL;
}
```

Each shape gives the test a *value* to assert — no output scraping:

```c
#include <assert.h>
#include <stdio.h>

int main(void) {
    int buf[4];
    assert(fill_sequence(buf, 4) == 0);
    assert(buf[3] == 6);
    assert(fill_sequence(NULL, 2) == -1);   /* failure path is assertable */
    assert(fill_sequence(NULL, 0) == 0);    /* empty contract is pinned */

    static const int v[] = { 10, 20, 30 };
    assert(find_int(v, 3, 20) == &v[1]);
    assert(find_int(v, 3, 99) == NULL);

    printf("status-code tests passed\n");
    return 0;
}
```

```text
status-code tests passed
```

Guidelines that keep status codes testable:

- **One meaning per value.** `0` = success everywhere in your module, `-1` = failure everywhere. Never return `-1` as both "empty" and "I/O error" — tests cannot distinguish them.
- **Document `NULL`/zero-length handling.** Every pointer function answers: is `(NULL, 0)` legal? The answer goes in the comment *and* in an assert.
- **Never hide the error in a global alone.** `errno` is fine as *extra* detail, but the return value must still signal failure. Tests assert the return, then optionally inspect `errno`.
- **Range-check before narrowing.** Accumulate in `long long`, compare against `INT_MAX`/`INT_MIN`, then cast. The check is the testable half of the arithmetic.

---

## Seams: Where Tests Plug In

A **seam** is a place where you can change behavior without editing the function — typically a function parameter that stands in for the outside world. The three seams every C beginner needs:

### Seam 1: inject `FILE *` instead of hardcoding `stdout` / a filename

The untestable version calls `printf` directly, so tests must capture the console. The testable version takes the stream from the caller — production passes `stdout`, tests pass a `tmpfile`:

```c
#include <stdio.h>
#include <stddef.h>

/* TESTABLE: caller picks the stream. Production passes stdout,
   tests pass a tmpfile or a memory stream. */
static int report_good(FILE *out, const int *a, size_t n) {
    if (out == NULL) {
        return -1;
    }
    int total = 0;
    for (size_t i = 0; i < n; i++) {
        total += a[i];
    }
    fprintf(out, "total=%d\n", total);
    return 0;
}
```

Test without touching the screen:

```c
#include <assert.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    static const int v[] = { 1, 2, 3 };
    FILE *tmp = tmpfile();
    assert(tmp != NULL);
    assert(report_good(tmp, v, 3) == 0);
    rewind(tmp);
    char line[64];
    assert(fgets(line, sizeof(line), tmp) != NULL);
    assert(strcmp(line, "total=6\n") == 0);
    fclose(tmp);
    printf("FILE-seam test passed\n");
    return 0;
}
```

```text
FILE-seam test passed
```

### Seam 2: inject the comparator (function pointer)

Sorting or searching with a hardcoded `<` can only be tested on one ordering. A comparator parameter opens every ordering — and lets tests inject a counting comparator that records how many comparisons ran.

```c
#include <stddef.h>

typedef int (*cmp_fn)(int a, int b);

static int cmp_asc(int a, int b) {
    return (a > b) - (a < b);
}

static int cmp_desc(int a, int b) {
    return (a < b) - (a > b);
}

/* OWNERSHIP: MUTATES a in place using cmp for ordering. */
static void bubble_cmp(int *a, size_t n, cmp_fn cmp) {
    if (a == NULL || cmp == NULL || n < 2) {
        return;
    }
    for (size_t end = n; end > 1; end--) {
        for (size_t i = 0; i + 1 < end; i++) {
            if (cmp(a[i], a[i + 1]) > 0) {
                int t = a[i];
                a[i] = a[i + 1];
                a[i + 1] = t;
            }
        }
    }
}
```

```c
#include <assert.h>
#include <stdio.h>

int main(void) {
    int a[] = { 3, 1, 2 };
    bubble_cmp(a, 3, cmp_asc);
    assert(a[0] == 1 && a[1] == 2 && a[2] == 3);
    int d[] = { 1, 2, 3 };
    bubble_cmp(d, 3, cmp_desc);
    assert(d[0] == 3 && d[2] == 1);
    printf("comparator-seam tests passed\n");
    return 0;
}
```

```text
comparator-seam tests passed
```

### Seam 3: inject the allocator (function pointer)

Code that calls `malloc` directly cannot simulate allocation failure — and allocation failure is the path most in need of a test. Accept the allocator as a parameter (defaulting to `malloc` in production):

```c
#include <stddef.h>
#include <stdlib.h>

typedef void *(*alloc_fn)(size_t nbytes);

static void *prod_alloc(size_t nbytes) {
    return malloc(nbytes);
}

static void *failing_alloc(size_t nbytes) {
    (void)nbytes;
    return NULL;   /* simulates OOM on demand */
}

/* Returns 0 on success, -1 when allocation fails. Never leaks. */
static int make_doubles_alloc(int **out, size_t n, alloc_fn alloc) {
    if (out == NULL || alloc == NULL) {
        return -1;
    }
    if (n == 0) {
        *out = NULL;
        return 0;
    }
    int *p = alloc(n * sizeof(int));
    if (p == NULL) {
        *out = NULL;
        return -1;
    }
    for (size_t i = 0; i < n; i++) {
        p[i] = (int)(i * 2);
    }
    *out = p;
    return 0;
}
```

```c
#include <assert.h>
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *p = NULL;
    assert(make_doubles_alloc(&p, 4, prod_alloc) == 0);
    assert(p != NULL && p[3] == 6);
    free(p);

    int *q = (int *)0x1;   /* poison: must be reset to NULL on failure */
    assert(make_doubles_alloc(&q, 4, failing_alloc) == -1);
    assert(q == NULL);
    printf("alloc-seam tests passed\n");
    return 0;
}
```

```text
alloc-seam tests passed
```

Without the seam, the failure test would require exhausting real memory. With it, OOM is one function pointer away, deterministic and instant.

---

## Determinism: Same Input, Same Output, Every Run

A test that passes on Monday and fails on Tuesday without a code change is worse than no test: it teaches the team to ignore failures. In C, nondeterminism sneaks in through four doors. Bolt all four.

| Source | Why it flaps | Testable replacement |
| --- | --- | --- |
| `rand()` without fixed seed | different sequence per run | `srand(42)` in tests, or pass values as arrays |
| `time(NULL)` / wall clock | output depends on when it runs | inject the timestamp as a parameter |
| Uninitialized memory / padding | garbage differs per build | `= {0}`, `memset`, never read padding |
| Pointer values (`%p`), addresses | ASLR changes them every run | assert on contents/offsets, never on addresses |

The untestable version calls `srand(time(NULL))` and `rand()` inside the logic itself, so no two runs agree. The testable version takes the state from the caller — a tiny counter-based generator with no globals:
static unsigned int roll_seeded(unsigned int *state) {
    /* tiny LCG: deterministic given *state, no globals touched */
    *state = *state * 1664525u + 1013904223u;
    return (*state >> 8) % 100u;
}
```

```c
#include <assert.h>
#include <stdio.h>

int main(void) {
    unsigned int s1 = 42;
    unsigned int s2 = 42;
    assert(roll_seeded(&s1) == roll_seeded(&s2));   /* same seed, same roll */
    printf("deterministic roll=%u\n", roll_seeded(&s1));
    return 0;
}
```

```text
deterministic roll=92
```

Same rule for time: never call `time(NULL)` inside logic. Take a `time_t now` parameter; production passes the real clock, tests pass fixed stamps (midnight, leap second, epoch `0`).

---

## What NOT to Test

More tests are not better. Tests pin behavior, and pinning the *wrong* thing makes refactoring impossible — every cleanup breaks tests that asserted on accidents.

Do NOT test:

1. **Internal implementation.** If `sort` uses bubble sort today and mergesort tomorrow, tests asserting comparison counts or swap order break for no user-visible reason. Assert the *output* (sorted, stable) instead.
2. **Exact addresses (`%p`).** ASLR and allocator behavior change them run to run. Assert `&v[1] - &v[0] == 1` (layout) or contents, never `0x7ffd...`.
3. **Exact prose of log messages.** `total=6` as a contract is fine when the format *is* the interface (machine-parsed output). Free-form human logs ("done-ish!!") churn; assert the status code, not the sentence.
4. **Third-party library internals.** Test that *your* wrapper handles the library's documented errors, not that `printf` prints. Do not re-test the C standard library.
5. **Performance cliffs as correctness.** "Sorts 10M ints in under a second" belongs in a benchmark harness with warmup and statistics, not in `assert`. Keep timing out of pass/fail unit tests.

In short: pin return values, output contents, error paths, and stream *contents* via seams — never addresses, internal step counts, or prose that is not the interface.

---

## Full Refactor: From Untestable to Testable

One function with every disease at once — then the cure. The starting point reads the clock, rolls unseeded dice, prints instead of returning, and hardcodes `stdout`:

```c
/* BEFORE: untestable. Try writing a repeatable test for this. */
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

static void play_round_bad(void) {
    srand((unsigned)time(NULL));
    int roll = rand() % 6 + 1;
    printf("rolled %d\n", roll);
    if (roll >= 4) {
        printf("win\n");
    } else {
        printf("lose\n");
    }
}
```

What is wrong, precisely: nondeterministic seed, nondeterministic output, no return value to assert, welded to `stdout`, win threshold buried in the body. Every test would be flaky or require console capture of random text.

```c
/* AFTER: testable. Pure decision + injected stream + injected roll. */
#include <stdio.h>
#include <stddef.h>

/* PURE: win means roll in [4, 6]. No I/O, no globals. */
static int is_win(int roll) {
    return roll >= 4 && roll <= 6;
}

/* SEAM: writes the verdict to any stream; status code is assertable. */
static int report_round(FILE *out, int roll) {
    if (out == NULL) {
        return -1;
    }
    if (roll < 1 || roll > 6) {
        return -1;
    }
    fprintf(out, "rolled %d\n%s\n", roll, is_win(roll) ? "win" : "lose");
    return is_win(roll) ? 1 : 0;
}
```

```c
#include <assert.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    /* pure decision: every face pinned, no randomness involved */
    assert(is_win(1) == 0 && is_win(3) == 0);
    assert(is_win(4) == 1 && is_win(6) == 1);

    /* seam: exact bytes checked via tmpfile, return value checked too */
    FILE *tmp = tmpfile();
    assert(tmp != NULL);
    assert(report_round(tmp, 5) == 1);
    assert(report_round(tmp, 2) == 0);
    assert(report_round(tmp, 0) == -1);   /* invalid roll rejected */
    assert(report_round(NULL, 5) == -1);  /* null stream rejected */
    rewind(tmp);
    char buf[128] = { 0 };
    size_t n = fread(buf, 1, sizeof(buf) - 1, tmp);
    (void)n;
    assert(strstr(buf, "rolled 5\nwin\n") != NULL);
    assert(strstr(buf, "rolled 2\nlose\n") != NULL);
    fclose(tmp);

    printf("refactor tests passed\n");
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer refactor.c -o refactor
./refactor
```

```text
refactor tests passed
```

The refactor checklist used, applicable to any function you inherit:

```text
  [ ] decision extracted into a pure function (is_win)
  [ ] I/O moved to a FILE* seam (report_round)
  [ ] magic inputs (rand/time) moved to parameters
  [ ] every failure returns a checkable code (-1), success returns meaning (1/0)
  [ ] invalid inputs rejected first, before any output
  [ ] test covers happy (win), edge (roll 1/6, NULL), failure (roll 0)
```

Production `main` then becomes three lines — seed once, roll once, call `report_round(stdout, roll)` — and the untested surface shrinks to the wiring, which is exactly where it belongs.

---

## Common Bugs

| Bug | Symptom | Fix |
| --- | --- | --- |
| Logic prints instead of returning | tests must scrape `stdout`; formatting change breaks them | return values; print only in `main`/seam |
| Hidden global accumulator | order-dependent PASS/FAIL | pass state as parameters; pure functions |
| `malloc` called directly in logic | OOM path untestable | inject `alloc_fn`; test with `failing_alloc` |
| `rand()`/`time()` inside logic | flaky tests | inject seed/timestamp; fixed values in tests |
| Asserting on `%p` addresses | passes once, fails on rerun (ASLR) | assert contents, offsets, status codes |
| Pinning swap counts / internals | refactor breaks green tests | assert observable output (sortedness, stability) |
| `(NULL, 0)` contract undocumented | crash on empty input in one caller | decide, comment, and assert the edge |

---

## Checklist

```text
[ ] hot logic lives in pure functions (args in, answer out, const inputs)
[ ] every function documents ownership: borrows vs owns, who frees
[ ] failures surface as checkable returns (0/-1, count-or--1, pointer-or-NULL)
[ ] (NULL, 0) and empty-input behavior decided, commented, and asserted
[ ] output goes through FILE* seams; comparators/allocators injected as pointers
[ ] no rand/time/uninitialized reads inside logic; fixed seeds in tests
[ ] tests assert values and contents, never addresses or internal step counts
[ ] helpers are static; read-only pointers are const
[ ] builds clean with -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined
```

---

#### [Previous](./01-part.md) | [Next](./exercises.md)
