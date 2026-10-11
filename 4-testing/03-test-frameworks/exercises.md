# Exercise List 3 - Test Frameworks

Six exercises on the mini-runner from parts 01–02: migrate a manual suite
to `TEST()`, add global setup/teardown, wire `make test` with sanitizers,
measure coverage with `gcov -b` and close the gap, use skip/focus while
debugging, and lock a real bug from modules 1–3 behind a regression test.

Base flags (strict from the start, C11, no external dependencies):

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g test_x.c clamp.c -o test_x
```

The course `mini_test.h` (fixed runner: `TEST_ONLY` filter + separate skip
count, from 01-part section 3) is the standard header for all six. Real
frameworks (Unity/Check/cmocka) are optional background reading — nothing
below requires installing them. Work in an empty directory per exercise so
stale `build/` artifacts cannot hide mistakes. Keep every suite
deterministic: fixed seeds, no `sleep()`, no wall-clock asserts.

---

## Exercise 1 — Migrate a Manual Suite to Mini-Runner `TEST()`

Starting point: three hand-rolled tests with bare `if` asserts.

```c
/* manual_clamp.c — migrate this */
#include <stdio.h>

int clamp(int v, int lo, int hi);

int clamp(int v, int lo, int hi) {
    if (v < lo) {
        return lo;
    }
    if (v > hi) {
        return hi;
    }
    return v;
}

static int test_low(void) {
    if (clamp(-5, 0, 10) != 0) {
        printf("FAIL test_low\n");
        return 1;
    }
    return 0;
}

static int test_high(void) {
    if (clamp(99, 0, 10) != 10) {
        printf("FAIL test_high\n");
        return 1;
    }
    return 0;
}

static int test_mid(void) {
    if (clamp(4, 0, 10) != 4) {
        printf("FAIL test_mid\n");
        return 1;
    }
    return 0;
}

int main(void) {
    int failures = 0;
    failures += test_low();
    failures += test_high();
    failures += test_mid();
    if (failures == 0) {
        printf("ALL TESTS PASSED (3 tests)\n");
    } else {
        printf("%d TEST(S) FAILED\n", failures);
    }
    return failures != 0;
}
```

Target skeleton (fill in the three bodies, keep the runner untouched):

```c
/* test_clamp.c — your migration target */
#include <stdio.h>
#include <stdlib.h>

#include "mini_test.h"
#include "clamp.h"   /* int clamp(int v, int lo, int hi); */

TEST(clamp_low) {
    /* TODO: EXPECT_INT_EQ(clamp(-5, 0, 10), 0); */
}

TEST(clamp_high) {
    /* TODO */
}

TEST(clamp_mid) {
    /* TODO */
}

int main(void) {
    return run_all_tests();
}
```

Expected output after migration:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g test_clamp.c clamp.c -o test_clamp
./test_clamp; echo "exit: $?"
```

```text
[ PASS ] clamp_low
[ PASS ] clamp_high
[ PASS ] clamp_mid
----
3 passed, 0 failed, 0 skipped
exit: 0
```

Then prove the asserts are richer: temporarily change one expectation to a
wrong value, rebuild, and confirm both sides print:

```text
  FAIL clamp_high: expected 999, got 10
[ FAIL ] clamp_high (1 assertion(s) failed)
----
2 passed, 1 failed, 0 skipped
```

Revert before continuing.

### Requirements

1. Each manual `static int test_*` becomes one `TEST(name)` block; no test logic left in `main()`.
2. Every bare `if (... != ...) { printf; return 1; }` becomes `EXPECT_INT_EQ` (or `EXPECT_STR_EQ` / `EXPECT_TRUE` where fitting).
3. `main()` is exactly `return run_all_tests();` — adding a fourth `TEST()` later needs zero runner edits (prove it by adding one boundary test).
4. Full output pasted: all-pass run with `exit: 0` plus the sabotaged run showing `expected X, got Y`.
5. Zero warnings under `-Werror`; no external framework installed.

---

## Exercise 2 — Add Global Setup/Teardown

Starting point: two tests share a scratch buffer but each manages it by
hand (and one of them forgets cleanup — the bug you will fix with a
fixture).

```c
/* leaky.c — starting point, do not ship this pattern */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "mini_test.h"

static char *scratch; /* shared world, managed ad hoc */

TEST(writes_hello) {
    scratch = malloc(64);
    memset(scratch, 0, 64);
    strcpy(scratch, "hello");
    EXPECT_STR_EQ(scratch, "hello");
    free(scratch); /* remembered here... */
}

TEST(starts_empty) {
    scratch = malloc(64);
    memset(scratch, 0, 64);
    EXPECT_INT_EQ((int)scratch[0], 0);
    /* ...forgotten here: leak, plus next test inherits garbage */
}

int main(void) {
    return run_all_tests();
}
```

Target skeleton — move ownership into the fixture:

```c
/* test_fixture.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "mini_test.h"

#define SCRATCH_SIZE 64

static char *scratch;

static void my_setup(void) {
    /* TODO: malloc SCRATCH_SIZE, memset 0. Handle NULL. */
}

static void my_teardown(void) {
    /* TODO: free(scratch); scratch = NULL; */
}

TEST(scratch_starts_empty) {
    /* TODO: ASSERT scratch != NULL, then EXPECT first byte 0 and STR_EQ "". */
}

TEST(scratch_writes_do_not_leak) {
    /* TODO: strcpy "hello", EXPECT_STR_EQ. No free here. */
}

int main(void) {
    mt_setup = my_setup;
    mt_teardown = my_teardown;
    return run_all_tests();
}
```

Expected output:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g test_fixture.c -o test_fixture
./test_fixture; echo "exit: $?"
```

```text
[ PASS ] scratch_starts_empty
[ PASS ] scratch_writes_do_not_leak
----
2 passed, 0 failed, 0 skipped
exit: 0
```

Prove the fixture owns the memory — run under sanitizers and show clean:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -fsanitize=address,undefined test_fixture.c -o test_fixture_asan
./test_fixture_asan; echo "exit: $?"
```

```text
[ PASS ] scratch_starts_empty
[ PASS ] scratch_writes_do_not_leak
----
2 passed, 0 failed, 0 skipped
exit: 0
```

(No ASan report, exit 0. If you free inside a test body *and* in teardown,
ASan prints `heap-use-after-free` / `double-free` — that error is the
lesson; remove the body-side `free`.)

### Requirements

1. `my_setup` allocates + zeroes; `my_teardown` frees + NULLs; bodies never `malloc`/`free` the fixture.
2. A failing setup is caught with `ASSERT_INT_EQ(scratch != NULL, 1)` (aborts the test, still runs teardown).
3. Both plain and `-fsanitize=address,undefined` runs pasted, exit 0, no sanitizer report.
4. One paragraph explaining the ownership rule: why teardown (not bodies) frees, and what breaks if both do.
5. Deliberately add a body-side `free(scratch);` once, paste the ASan error, then remove it.

---

## Exercise 3 — Create `make test` With Sanitizer

Starting point: you compile and run by hand every time (and sometimes
forget the sanitizer run).

```bash
# the ad-hoc loop you are replacing:
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g test_clamp.c clamp.c -o test_clamp
./test_clamp
# ...forgot: gcc ... -fsanitize=address,undefined ... && ./test_clamp_asan
```

Target skeleton — fill the `Makefile` (TAB-indented recipes only):

```make
# Makefile — TODO: complete the four rules below (each recipe line: TAB)
CC = gcc
CFLAGS = -std=c11 -Wall -Wextra -Werror -pedantic -g
SANFLAGS = -fsanitize=address,undefined
BUILD = build

TEST_BIN = $(BUILD)/test_clamp
TEST_ASAN_BIN = $(BUILD)/test_clamp_asan

all: $(TEST_BIN)

$(BUILD):
	mkdir -p $(BUILD)

$(TEST_BIN): test_clamp.c clamp.c clamp.h mini_test.h | $(BUILD)
# TODO recipe: $(CC) $(CFLAGS) test_clamp.c clamp.c -o $(TEST_BIN)

$(TEST_ASAN_BIN): test_clamp.c clamp.c clamp.h mini_test.h | $(BUILD)
# TODO recipe: $(CC) $(CFLAGS) $(SANFLAGS) test_clamp.c clamp.c -o $(TEST_ASAN_BIN)

test: $(TEST_BIN) $(TEST_ASAN_BIN)
# TODO recipes: ./$(TEST_BIN) then ./$(TEST_ASAN_BIN)

clean:
# TODO recipe: rm -rf $(BUILD)

.PHONY: all test clean
```

Expected session:

```bash
make test; echo "exit: $?"
```

```text
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g test_clamp.c clamp.c -o build/test_clamp
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -fsanitize=address,undefined test_clamp.c clamp.c -o build/test_clamp_asan
./build/test_clamp
[ PASS ] clamp_low
[ PASS ] clamp_high
[ PASS ] clamp_mid
[ PASS ] clamp_boundary
----
4 passed, 0 failed, 0 skipped
./build/test_clamp_asan
[ PASS ] clamp_low
[ PASS ] clamp_high
[ PASS ] clamp_mid
[ PASS ] clamp_boundary
----
4 passed, 0 failed, 0 skipped
exit: 0
```

Then prove failure propagates: sabotage one expectation, run `make test`,
confirm `make` stops with `Error 1` and never reaches the ASan binary:

```text
./build/test_clamp
  FAIL clamp_high: expected 999, got 10
[ FAIL ] clamp_high (1 assertion(s) failed)
----
3 passed, 1 failed, 0 skipped
make: *** [Makefile:20: test] Error 1
```

Revert the sabotage afterwards.

### Requirements

1. `make test` builds plain + ASan/UBSan binaries and runs both, in order, from a clean checkout (`make clean && make test` pasted).
2. Recipes TAB-indented; `.PHONY: all test clean` present; headers listed as prerequisites.
3. `make test` exit `0` on green (pasted `exit: 0`); sabotaged run pasted showing `Error 1` and the ASan binary *not* running.
4. `build/` git-ignored (show the `.gitignore` line); no sanitizer binary shipped as the deliverable.
5. One paragraph: why two binaries (speed/isolation tradeoff) instead of compiling everything with sanitizers always on.

---

## Exercise 4 — Measure Coverage With `gcov` and Cover the Missing Branch

Starting point: a green but lazy suite — only two tests, one branch never
taken.

```c
/* test_lazy.c — green and incomplete */
#include <stdio.h>
#include <stdlib.h>

#include "mini_test.h"
#include "clamp.h"

TEST(clamp_low) {
    EXPECT_INT_EQ(clamp(-5, 0, 10), 0);
}

TEST(clamp_mid) {
    EXPECT_INT_EQ(clamp(4, 0, 10), 4);
}

int main(void) {
    return run_all_tests();
}
```

```c
/* clamp.c — the unit under measurement */
#include "clamp.h"

int clamp(int v, int lo, int hi) {
    if (v < lo) {
        return lo;
    }
    if (v > hi) {
        return hi;
    }
    return v;
}
```

Step 1 — measure (paste this report; numbers measured with GCC 15 —
older GCC counts lines/branches slightly differently, but `#####` and
`taken 0%` mean the same everywhere):

```bash
rm -f *.gcda *.gcno *.gcov
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g --coverage -c clamp.c
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g --coverage -c test_lazy.c
gcc --coverage clamp.o test_lazy.o -o test_lazy
./test_lazy
gcov -b clamp.c
```

```text
[ PASS ] clamp_low
[ PASS ] clamp_mid
----
2 passed, 0 failed, 0 skipped
File 'clamp.c'
Lines executed:83.33% of 6
Branches executed:100.00% of 4
Taken at least once:75.00% of 4
No calls
Creating 'clamp.c.gcov'
```

Annotated `clamp.c.gcov` (the smoking gun):

```text
        -:    0:Source:clamp.c
        -:    0:Graph:clamp.gcno
        -:    0:Data:clamp.gcda
        -:    0:Runs:1
        -:    1:/* clamp.c */
        -:    2:#include "clamp.h"
        -:    3:
function clamp called 2 returned 100% blocks executed 83%
        2:    4:int clamp(int v, int lo, int hi) {
        2:    5:    if (v < lo) {
branch  0 taken 50% (fallthrough)
branch  1 taken 50%
        1:    6:        return lo;
        -:    7:    }
        1:    8:    if (v > hi) {
branch  0 taken 0% (fallthrough)
branch  1 taken 100%
    #####:    9:        return hi;
        -:   10:    }
        1:   11:    return v;
        -:   12:}
```

Step 2 — skeleton: add the test that clears the `#####`:

```c
/* TODO: append to test_lazy.c */
TEST(clamp_high_covers_branch) {
    /* TODO: EXPECT_INT_EQ(clamp(99, 0, 10), 10); — executes line 8 */
}
```

Step 3 — re-measure with the same `-c` recipe (paste this report too):

```text
File 'clamp.c'
Lines executed:100.00% of 6
Branches executed:100.00% of 4
Taken at least once:100.00% of 4
```

### Requirements

1. Both `gcov -b` reports pasted: before (83.33% lines, `Taken at least once` 75%, `#####` on `return hi`, `branch 0 taken 0%`) and after (100% / 100% / 100%).
2. The `#####` line quoted and explained in one paragraph: which input class was missing (values above `hi`), why `Branches executed: 100%` alongside `Taken at least once: 75%` is not a contradiction (reached vs taken), and why pass/fail alone could not see it.
3. New `TEST()` is a behavior test (`clamp(99, 0, 10) == 10`), not a coverage-token test — it would catch a real `return lo` typo on that line.
4. Coverage rebuilt from scratch (`rm -f *.gcda *.gcno *.gcov` before each measure); `*.gcda *.gcno *.gcov` git-ignored.
5. `--coverage` kept off the default `CFLAGS` (separate build line / target); note why in one line.

---

## Exercise 5 — Skip/Only (Focus) for Debug

Starting point: five tests, one failure buried in passing noise. You will
focus the failure, skip a known-unrelated broken test, fix, then un-focus.

```c
/* test_focus.c — starting point */
#include <stdio.h>
#include <stdlib.h>

#include "mini_test.h"
#include "clamp.h"

TEST(clamp_low) {
    EXPECT_INT_EQ(clamp(-5, 0, 10), 0);
}

TEST(clamp_high) {
    EXPECT_INT_EQ(clamp(99, 0, 10), 10);
}

TEST(clamp_mid) {
    EXPECT_INT_EQ(clamp(4, 0, 10), 999); /* BUG IN TEST: wrong expectation, fix to 4 */
}

TEST(clamp_boundary_lo) {
    EXPECT_INT_EQ(clamp(0, 0, 10), 0);
}

TEST(clamp_unicode_future) {
    SKIP("unicode clamping lands in milestone 2");
    EXPECT_INT_EQ(0, 1); /* never runs while skipped */
}

int main(void) {
    return run_all_tests();
}
```

Full run (the noise):

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g test_focus.c clamp.c -o test_focus
./test_focus; echo "exit: $?"
```

```text
[ PASS ] clamp_low
[ PASS ] clamp_high
  FAIL clamp_mid: expected 999, got 4
[ FAIL ] clamp_mid (1 assertion(s) failed)
[ PASS ] clamp_boundary_lo
  SKIP clamp_unicode_future: unicode clamping lands in milestone 2
[ SKIP ] clamp_unicode_future
----
3 passed, 1 failed, 1 skipped
exit: 1
```

Focus the failure (skeleton command — run it and paste output):

```bash
TEST_ONLY=clamp_mid ./test_focus; echo "exit: $?"
```

```text
  FAIL clamp_mid: expected 999, got 4
[ FAIL ] clamp_mid (1 assertion(s) failed)
----
0 passed, 1 failed, 0 skipped
exit: 1
```

Fix the expectation (`999` → `4`), re-run focused (now green), then full:

```text
[ PASS ] clamp_mid
----
1 passed, 0 failed, 0 skipped
exit: 0
```

```text
[ PASS ] clamp_low
[ PASS ] clamp_high
[ PASS ] clamp_mid
[ PASS ] clamp_boundary_lo
  SKIP clamp_unicode_future: unicode clamping lands in milestone 2
[ SKIP ] clamp_unicode_future
----
4 passed, 0 failed, 1 skipped
exit: 0
```

### Requirements

1. All four outputs pasted: full-red, focused-red, focused-green, full-green-with-skip.
2. `SKIP("...")` reason names the milestone/issue (not `"todo"`); skipped test exits the suite `0` (skips are not failures).
3. Proof `TEST_ONLY` is unset for the final run (`env | grep TEST_ONLY` empty, or new shell) — focused runs never ship as "green CI".
4. One paragraph: skip vs comment-out vs delete — when each is correct (tracked debt / dead experiment / wrong test).
5. The fixed expectation diff shown (`999` → `4`); no product code changed in this exercise.

---

## Exercise 6 — Write a Regression Test for a Real Bug From Modules 1–3

Pick one genuine bug class you met in modules 1–3 (not a synthetic typo).
Three approved options — choose one:

| # | Source module | Bug | Failing input | Correct behavior |
| --- | --- | --- | --- | --- |
| A | 2-strings | `strncpy` without `'\0'` terminator | name of exactly `NAME_LEN` chars | truncated string still `'\0'`-terminated |
| B | 3-advanced file-io | `fopen` result unchecked (`NULL` deref) | missing filename | clean error return, no crash |
| C | 1-fundamentals loops | off-by-one `<=` overruns array | `n`-element array, loop `i <= n` | loop uses `< n`, last element exact |

Skeleton for option A (adapt the pattern for B/C):

```c
/* test_regression.c — option A: strncpy termination */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "mini_test.h"

#define NAME_LEN 8

/* Copy helper under test. The BUGGY body was a single line:
 *     strncpy(dst, src, NAME_LEN);   // no terminator on exact fit
 * To replay the bug and watch this test go red, swap the fixed body
 * below back to that one line, rebuild, and rerun. */
static void copy_name(char *dst, const char *src) {
    strncpy(dst, src, NAME_LEN - 1);
    dst[NAME_LEN - 1] = '\0';
}

TEST(strncpy_terminates_on_exact_fit_regression) {
    /* Bug: 2-strings phonebook lost terminator on 8-char names. */
    char buf[NAME_LEN + 1];
    memset(buf, 'X', sizeof buf);
    buf[sizeof buf - 1] = '\0';
    copy_name(buf, "12345678"); /* swap body to buggy line to see red */
    EXPECT_INT_EQ((int)buf[NAME_LEN - 1], 0);
    EXPECT_STR_EQ(buf, "1234567");
}

int main(void) {
    return run_all_tests();
}
```

Red-green proof (paste both):

```text
# with the buggy one-line body (strncpy(dst, src, NAME_LEN);):
  FAIL strncpy_terminates_on_exact_fit_regression: expected 0, got 56
  FAIL strncpy_terminates_on_exact_fit_regression: expected "1234567", got "12345678"
[ FAIL ] strncpy_terminates_on_exact_fit_regression (2 assertion(s) failed)
----
0 passed, 1 failed, 0 skipped
exit: 1
```

(`56` is `'X'` — the sentinel byte the missing terminator failed to clear.)

```text
# with the fixed body:
[ PASS ] strncpy_terminates_on_exact_fit_regression
----
1 passed, 0 failed, 0 skipped
exit: 0
```

For option B, the regression test feeds a missing path and expects a
nonzero error return instead of a crash (run the crashing variant under a
guard or just assert the `NULL` check exists via the fixed function). For
option C, the test fills `{1, 2, 3}`, sums with the fixed `< n` loop, and
expects `6` — then shows the `<= n` variant reading garbage/ASan
`stack-buffer-overflow` under `-fsanitize=address,undefined`.

### Requirements

1. One option (A/B/C) implemented; the test comment names the source module and the symptom in one line.
2. Red-green flip pasted: same `TEST()` fails on buggy code (`exit: 1`) and passes on fixed code (`exit: 0`).
3. For option C (or any out-of-bounds variant): ASan run pasted showing the report class (`stack-buffer-overflow` / `heap-buffer-overflow`) on the buggy side.
4. Suite deterministic: fixed inputs, no randomness, no filesystem paths outside `/tmp` with `mkstemp`-style uniqueness if files are used.
5. Full suite green after the fix (`make test` or direct run pasted); test committed alongside the fix, never as a follow-up.

---

## Checklist Before You Run

| Check | Why |
| --- | --- |
| Every new test is `TEST(name)`, zero `main()` edits | runner edits per test do not scale past five tests |
| Fixture memory owned by teardown, bodies never free it | double ownership is a double-free waiting for ASan to find |
| `make test` runs plain + ASan/UBSan, `.PHONY`, TAB recipes | one command or the suite rots; spaces break make silently |
| Exit code seen both ways (`exit: 0` and `exit: 1`) | a suite you never saw fail is a suite you cannot trust |
| `gcov -b` before/after pasted, `#####` explained and cleared | green without coverage is green by omission |
| `SKIP` reasons tracked, `TEST_ONLY` unset for final runs | focus is a debug tool, never a CI state |
| Every fix paired with a regression `TEST()` naming the bug | the bug either lives in the suite or comes back |

---

#### [Previous](./02-part.md) | [Next](../challenge.md)
