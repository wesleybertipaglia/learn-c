# Make Test and Coverage Habits

Part 01 gave you a mini-runner worth keeping. This part makes it a habit:
one command (`make test`) that compiles the harness, runs it, and re-runs
it under sanitizers; an exit-code convention CI can trust; coverage with
`gcov -b` so you can see which lines and branches never ran; and the
regression discipline that keeps the suite honest over time.

No new dependencies. Everything here is `make`, `gcc`, and `gcov`.

---

## 1. The `make test` Target: One Command, Three Jobs

A test suite that needs three commands and a README paragraph does not get
run. The target is: fresh clone → `make test` → green or red in seconds.

The three jobs, in order:

```text
  make test
    │
    ├─ 1. build tests (strict flags, -Werror)
    │     test_clamp.c + mini_test.h → build/test_clamp
    │
    ├─ 2. run tests (plain)
    │     ./build/test_clamp → exit 0 = pass
    │
    └─ 3. run tests (sanitizers)
          ./build/test_clamp_asan → exit 0 = memory-clean
```

Concrete project layout:

```text
  clamp_proj/
    Makefile
    mini_test.h        (from 01-part.md, fixed runner version)
    clamp.h
    clamp.c
    test_clamp.c
    build/             (generated, git-ignored)
```

Source files:

```c
/* clamp.h */
#ifndef CLAMP_H
#define CLAMP_H

int clamp(int v, int lo, int hi);

#endif /* CLAMP_H */
```

```c
/* clamp.c */
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

```c
/* test_clamp.c */
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
    EXPECT_INT_EQ(clamp(4, 0, 10), 4);
}

TEST(clamp_boundary) {
    EXPECT_INT_EQ(clamp(0, 0, 10), 0);
    EXPECT_INT_EQ(clamp(10, 0, 10), 10);
}

int main(void) {
    return run_all_tests();
}
```

(The `run_all_tests()` here is the fixed version from 01-part section 3,
with `getenv("TEST_ONLY")` filtering and separate skip counting.)

The `Makefile` (TAB-indented recipes — spaces give `missing separator`):

```make
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
	$(CC) $(CFLAGS) test_clamp.c clamp.c -o $(TEST_BIN)

$(TEST_ASAN_BIN): test_clamp.c clamp.c clamp.h mini_test.h | $(BUILD)
	$(CC) $(CFLAGS) $(SANFLAGS) test_clamp.c clamp.c -o $(TEST_ASAN_BIN)

test: $(TEST_BIN) $(TEST_ASAN_BIN)
	./$(TEST_BIN)
	./$(TEST_ASAN_BIN)

clean:
	rm -rf $(BUILD)

.PHONY: all test clean
```

Session:

```bash
make test
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
```

Why two binaries instead of one? Sanitizer builds are ~2x slower and use
more memory — fine for tests, unacceptable for shipped binaries. Building
both from the same sources proves the code is warning-clean *and*
memory-clean with one command, while `make all` still produces the plain
binary.

| Target | Builds | Runs | Use |
| --- | --- | --- | --- |
| `make all` | plain binary only | nothing | daily dev loop |
| `make test` | plain + ASan/UBSan binaries | both, in order | before every commit |
| `make clean` | — (deletes `build/`) | — | reset after flag/layout changes |

Note the `.PHONY: all test clean` line: `test` is not a file, so without
`.PHONY` a stray file named `test` would make `make test` silently do
nothing — the exact class of false-green this chapter exists to prevent.

---

## 2. Exit-Code Convention: The Contract With CI (and Yourself)

Every runner in this course — manual, mini, or real framework — honors one
contract:

| Exit code | Meaning | Runner behavior |
| --- | --- | --- |
| `0` | all tests passed (skips allowed) | `return 0` / `return mt_total_failed != 0` with zero failures |
| nonzero (`1`) | at least one test failed | `return 1`, summary line shows `N failed` |
| crash / signal (e.g. `139` = SIGSEGV) | suite itself is broken | no summary line at all — worse than a failure |

Why it matters: `make` stops at the first nonzero exit. In the `test`
target above, if `./build/test_clamp` fails, `make` never runs the ASan
binary — the failure is loud and early. CI systems (GitHub Actions,
GitLab, Jenkins) do the same: nonzero means red.

Verify the convention explicitly:

```bash
./build/test_clamp; echo "exit: $?"
```

```text
[ PASS ] clamp_low
[ PASS ] clamp_high
[ PASS ] clamp_mid
[ PASS ] clamp_boundary
----
4 passed, 0 failed, 0 skipped
exit: 0
```

Now inject a failure and watch the code flip:

```bash
# temporary sabotage: change one expectation to a wrong value, rebuild, run
sed 's/clamp(99, 0, 10), 10/clamp(99, 0, 10), 999/' test_clamp.c > test_bad.c
gcc -std=c11 -Wall -Wextra -pedantic -g test_bad.c clamp.c -o /tmp/test_bad
/tmp/test_bad; echo "exit: $?"
```

```text
[ PASS ] clamp_low
  FAIL clamp_high: expected 999, got 10
[ FAIL ] clamp_high (1 assertion(s) failed)
[ PASS ] clamp_mid
[ PASS ] clamp_boundary
----
3 passed, 1 failed, 0 skipped
exit: 1
```

And in the Makefile, a failing suite halts `make test`:

```text
./build/test_clamp
  FAIL clamp_high: expected 999, got 10
[ FAIL ] clamp_high (1 assertion(s) failed)
----
3 passed, 1 failed, 0 skipped
make: *** [Makefile:20: test] Error 1
```

That `Error 1` is the feature. Rules:

1. `main()` returns the runner result directly: `return run_all_tests();`
   — never `return 0;` unconditionally.
2. Skips do **not** fail the build. `1 skipped, 0 failed` exits `0`.
3. A crash (no summary line) is investigated before any failure — it means
   the harness or fixture is broken, not just one assertion.
4. Never wrap the test run in a way that masks the code (`./tests || true`
   in CI turns every failure green — a classic false-confidence bug).

---

## 3. Coverage With `gcov -b`: Lines and Branches

Passing tests prove the code works *for the inputs you tried*. Coverage
answers the complementary question: which lines and branches did the suite
never execute? Untested code is where the next bug lives.

### 3a. Building for coverage

Add `--coverage` (shorthand for `-fprofile-arcs -ftest-coverage`) to both
compile and link, run the suite, then read the report with `gcov -b`
(`-b` = branch probabilities). Compile each translation unit with `-c`
first: a single `gcc a.c b.c -o app` line makes modern GCC prefix the
notes files (`app-a.gcno`), which `gcov -b a.c` then cannot find.
Separate `-c` steps keep the classic `clamp.gcno` name:

```bash
rm -f *.gcda *.gcno *.gcov
gcc -std=c11 -Wall -Wextra -pedantic -g --coverage -c clamp.c
gcc -std=c11 -Wall -Wextra -pedantic -g --coverage -c test_clamp.c
gcc --coverage clamp.o test_clamp.o -o test_cov
./test_cov
gcov -b clamp.c
```

```text
[ PASS ] clamp_low
[ PASS ] clamp_high
[ PASS ] clamp_mid
[ PASS ] clamp_boundary
----
4 passed, 0 failed, 0 skipped
File 'clamp.c'
Lines executed:100.00% of 6
Branches executed:100.00% of 4
Taken at least once:100.00% of 4
No calls
Creating 'clamp.c.gcov'
```

(Exact totals — `of 6`, `of 4` — vary by GCC version; older GCC counts 5
lines and 6 branches here. What never changes: `#####` means "never
executed" and `taken 0%` means "this direction never taken". The numbers
below were measured with GCC 15.)

The annotated `clamp.c.gcov` shows per-line counts plus per-branch
directions:

```text
        -:    0:Source:clamp.c
        -:    0:Graph:clamp.gcno
        -:    0:Data:clamp.gcda
        -:    0:Runs:1
        -:    1:/* clamp.c */
        -:    2:#include "clamp.h"
        -:    3:
function clamp called 5 returned 100% blocks executed 100%
        5:    4:int clamp(int v, int lo, int hi) {
        5:    5:    if (v < lo) {
branch  0 taken 20% (fallthrough)
branch  1 taken 80%
        1:    6:        return lo;
        -:    7:    }
        4:    8:    if (v > hi) {
branch  0 taken 25% (fallthrough)
branch  1 taken 75%
        1:    9:        return hi;
        -:   10:    }
        3:   11:    return v;
        -:   12:}
```

Reading it: the left number is executions (`5:` = ran 5 times). `-` means
non-code. Each `branch 0/1 taken N%` line tells you both directions of the
`if` above it were tried (20/80, 25/75) — no direction starved.

| `gcov -b` line | Meaning |
| --- | --- |
| `Lines executed:100.00% of 6` | every executable line ran |
| `Branches executed:100.00% of 4` | every branch instruction was reached |
| `Taken at least once:100.00% of 4` | every direction (`taken 20%` / `taken 80%`) tried at least once |
| `No calls` | `clamp` calls no other function (a `Calls executed` line appears when it does) |
| `#####` (in other reports) | line never executed — the signal to act on |
| `branch 0 taken 0%` (in other reports) | direction never taken — the subtler signal (section 3b) |

### 3b. A branch not covered (the interesting case)

Delete the high-side tests to simulate a lazy suite:

```c
/* test_lazy.c — only exercises the low clamp and the middle */
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

```bash
rm -f *.gcda *.gcno *.gcov
gcc -std=c11 -Wall -Wextra -pedantic -g --coverage -c clamp.c
gcc -std=c11 -Wall -Wextra -pedantic -g --coverage -c test_lazy.c
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

The suite is green — and lying by omission. Two signals agree: `#####` on
line 9 says `return hi` never ran, and `branch 0 taken 0%` on line 8 says
the `v > hi` direction was never taken. No test pushed a value above `hi`.
Note the trap: `Branches executed` still reads 100% (the branch was
*reached*), while `Taken at least once` drops to 75% — reaching a branch
is not covering it. Branch coverage caught what pass/fail could not.

The fix is a test, not a flag:

```c
TEST(clamp_high_covers_branch) {
    EXPECT_INT_EQ(clamp(99, 0, 10), 10); /* executes line 9 */
}
```

Rebuild, rerun, re-measure: lines back to 100%, branches back to 100%.

Coverage ASCII mental model:

```text
  test suite ──executes──▶ code
       │                      │
       ▼                      ▼
   PASS/FAIL              gcov -b report
   (did tried            (what was NEVER
    inputs work?)         tried at all?)
       │                      │
       └──── BOTH green ──────┘
              = done
```

Targets for this course (not dogma):

| Metric | Target | Rationale |
| --- | --- | --- |
| Line coverage on new code | 100% | new code with untested lines ships unknown behavior |
| Branch coverage on new code | 100% | every `if`/`?:`/`&&` needs both directions tried |
| Legacy code you touch | cover what you touch | full retro-coverage is a separate project |
| `#####` lines | zero before commit | each one becomes a new `TEST()` |

Coverage hygiene: `*.gcno`, `*.gcda`, `*.gcov` are generated — git-ignore
them (`*.gcno`, `*.gcda`, `*.gcov` in `.gitignore`) and `rm` them in `make
clean`. Never commit coverage artifacts.

---

## 4. Regression Discipline: Every Bug Earns a Failing Test

A regression test is a test that **fails without the fix and passes with
it**. The workflow is always the same three steps:

```text
  1. REPRODUCE   write TEST() that fails on current code
                    │
  2. FIX         change product code (not the test)
                    │
  3. VERIFY      same TEST() now passes; full suite still green
```

Real example: `clamp` with swapped bounds. Suppose a caller passes
`clamp(15, 10, 0)` (`lo > hi` — programmer error, but the function should
not return garbage silently). Original code returns `0` (`v > hi` with
`hi = 0`) — arguably wrong; the sane contract is "if bounds are swapped,
treat them as given" or document it. Pick the documented contract used in
this course: **precondition `lo <= hi`; violation returns `lo`**
(so misuse is at least deterministic and testable):

```c
/* clamp.c — v2 with documented precondition handling */
#include "clamp.h"

int clamp(int v, int lo, int hi) {
    if (lo > hi) {
        return lo; /* invalid bounds: deterministic, tested */
    }
    if (v < lo) {
        return lo;
    }
    if (v > hi) {
        return hi;
    }
    return v;
}
```

Regression test (written *before* the fix, observed failing):

```c
TEST(clamp_swapped_bounds_regression) {
    /* Bug #7: clamp(15, 10, 0) returned 0 (fell into v > hi).
       Contract: invalid bounds return lo deterministically. */
    EXPECT_INT_EQ(clamp(15, 10, 0), 10);
}
```

Before fix:

```text
  FAIL clamp_swapped_bounds_regression: expected 10, got 0
[ FAIL ] clamp_swapped_bounds_regression (1 assertion(s) failed)
----
4 passed, 1 failed, 0 skipped
exit: 1
```

After fix:

```text
[ PASS ] clamp_low
[ PASS ] clamp_high
[ PASS ] clamp_mid
[ PASS ] clamp_boundary
[ PASS ] clamp_swapped_bounds_regression
----
5 passed, 0 failed, 0 skipped
exit: 0
```

Rules that make this stick:

1. The test lands in the **same commit** as the fix (or the commit before
   — never after "when I have time").
2. The test comment names the bug (`Bug #7`, issue link, or one-line
   symptom) so future readers know *why* this odd case exists.
3. The test fails for the right reason — verify by stashing the fix and
   watching it go red (`git stash` → `make test` → red → `git stash pop`).
4. One bug, one test. A regression test that checks five things will be
   trimmed later; a focused one survives.

| Without regression test | With regression test |
| --- | --- |
| fix works today, breaks silently in 3 months | refactor that reintroduces the bug goes red immediately |
| bug report lives in chat history | bug lives in the suite as executable documentation |
| "did we fix that?" = archaeology | `grep regression test_clamp.c` = answer |

---

## 5. Keep the Suite Fast (or It Will Not Be Run)

A suite that takes 5 minutes gets run before lunch. A suite that takes 5
seconds gets run before every commit. Speed is a feature, and these are
the levers:

| Technique | What | Example |
| --- | --- | --- |
| Separate fast/slow | `make test` = unit only; `make test-slow` = I/O, fuzz, stress | file-system tests behind a second target |
| No `sleep()` in tests | replace timing waits with deterministic injection | pass a fake clock value instead of `sleep(1)` |
| Small inputs | 3-element arrays, not 3-million | boundary cases beat volume for unit tests |
| Sanitizers in `test`, not in `all` | ASan ~2x slowdown fenced inside `make test` | plain `make` stays instant |
| Deterministic seeds | fixed `srand(42)`, no `time(NULL)`, no ASLR-dependent asserts | same order, same output, every run |

Determinism checklist (nondeterministic tests are worse than no tests —
they teach the team to ignore red):

- No `rand()` without a fixed seed. `srand(42)` at the top or avoid it.
- No wall-clock asserts (`time(NULL)`, `gettimeofday`) — inject the time.
- No temp-file name collisions — `snprintf` with PID or use `mkstemp`.
- No test-order dependence — fixtures reset state (part 01, section 5a).
- No network, no absolute paths, no locale-dependent string compares.

A 40-test C suite built like this one runs in milliseconds:

```bash
time ./build/test_clamp
```

```text
[ PASS ] clamp_low
[ PASS ] clamp_high
[ PASS ] clamp_mid
[ PASS ] clamp_boundary
----
4 passed, 0 failed, 0 skipped

real    0m0.002s
user    0m0.001s
sys     0m0.001s
```

Two milliseconds. There is no excuse left not to run it.

---

## Bugs You Will Meet

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| 1 | `make test` says `Nothing to be done` after editing a test | header (`mini_test.h`, `clamp.h`) missing from prerequisite list | list all headers after the colon: `$(TEST_BIN): test_clamp.c clamp.c clamp.h mini_test.h` |
| 2 | `make test` passes but CI is red | local `test` file shadows the `.PHONY` target, or CI runs a different command | add `.PHONY: test`; make CI run exactly `make test` |
| 3 | `./tests || true` in a script, suite always "green" | exit code masked | remove `\|\| true`; check `echo $?` after a forced failure |
| 4 | `gcov: no .gcda files, did you run the program?` | built with `--coverage` but never executed the binary, or ran a different binary | run `./build/test_cov` first, then `gcov -b` in the same directory |
| 5 | `gcov` reports stale numbers after editing | old `.gcda` from a previous run merged with the new one | `rm -f *.gcda *.gcno *.gcov` (or `make clean`) before re-measuring |
| 6 | `--coverage` flags committed into `CFLAGS` permanently | every build now slow + littered with `.gcda` | keep coverage flags on a separate `test_cov` target / variable, never in default `CFLAGS` |
| 7 | Regression test passes even with the fix reverted | test asserts the buggy behavior, or tests the wrong function | stash the fix, watch red, unstash — the red-green flip is the proof |
| 8 | ASan binary reports leak in fixture | `teardown` missing or test `return`s before cleanup via raw `return` | use `SKIP`/assert macros (which cooperate with teardown); teardown always runs in `run_all_tests` |

---

## Checklist

1. Does `make test` build (strict flags) + run plain + run ASan/UBSan in one command?
2. Is `test` (and `clean`) marked `.PHONY`, with TAB-indented recipes?
3. Does `main()` return the runner result so exit code is `0` / `1` (never masked)?
4. Have you seen both `exit: 0` and `exit: 1` (forced failure) with your own eyes?
5. Can you build with `--coverage`, run, and read `gcov -b` (lines + branches)?
6. Can you point at a `#####` line and turn it into a new `TEST()` that clears it?
7. Is every fixed bug paired with a regression `TEST()` naming the bug, committed together?
8. Is the suite fast (< 1s) and deterministic (fixed seeds, no `sleep`, no wall clock)?
9. Are generated files (`build/`, `*.gcda`, `*.gcno`, `*.gcov`, sanitizer binaries) git-ignored?

---

#### [Previous](./01-part.md) | [Next](./exercises.md)
