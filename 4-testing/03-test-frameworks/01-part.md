# Using a Test Framework

The previous part built a manual harness: a `main()` that calls three
test functions, counts failures, and returns nonzero on failure. That
harness is honest and has zero dependencies — but every new test adds
boilerplate, every assertion looks like `if (...) { printf(...); return 1; }`,
and there is no setup/teardown, no test names in the output, and no way to
skip one test while debugging another.

A test framework is the harness you would eventually write anyway, packaged
once and reused everywhere. This part evolves your manual harness into a
mini-runner with `TEST(name)` registration, rich asserts, fixtures, and
skip support — and shows where the real frameworks (Unity, Check, cmocka)
fit when you outgrow it.

---

## 1. What a Framework Gives You Beyond `main()` + `if`

A manual harness from part 01 looks like this:

```c
/* manual.c — the starting point from 02-unit-tests */
#include <stdio.h>
#include <string.h>

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

static int test_clamp_low(void) {
    if (clamp(-5, 0, 10) != 0) {
        printf("FAIL test_clamp_low\n");
        return 1;
    }
    return 0;
}

static int test_clamp_high(void) {
    if (clamp(99, 0, 10) != 10) {
        printf("FAIL test_clamp_high\n");
        return 1;
    }
    return 0;
}

static int test_clamp_mid(void) {
    if (clamp(4, 0, 10) != 4) {
        printf("FAIL test_clamp_mid\n");
        return 1;
    }
    return 0;
}

int main(void) {
    int failures = 0;

    failures += test_clamp_low();
    failures += test_clamp_high();
    failures += test_clamp_mid();

    if (failures == 0) {
        printf("ALL TESTS PASSED (3 tests)\n");
    } else {
        printf("%d TEST(S) FAILED\n", failures);
    }
    return failures != 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g manual.c -o manual && ./manual
```

```text
ALL TESTS PASSED (3 tests)
```

It works. Now list everything it does *not* do:

| Feature | Manual harness | Mini-runner in this part | Full framework |
| --- | --- | --- | --- |
| Test names in output | no (you hard-code strings) | yes (`TEST(name)` registers `__func__`-style name) | yes, with file:line |
| Rich asserts (`EXPECT_INT_EQ` prints both sides) | no (bare `if`) | yes (macros) | yes, dozens of them |
| Automatic setup/teardown per test | no | yes (fixture hooks) | yes |
| Runner (loops over all tests, counts, summarizes) | hand-written `main` | generic `run_all_tests()` | yes, plus filtering |
| Skip / focus one test | no (comment code out) | yes (`SKIP()`, `TEST_ONLY` env) | yes (`--filter`, `TEST_IGNORE`) |
| Isolation (one crash kills all) | no (single process) | partial (documented limit) | fork-based (Check, cmocka) |

The rule of thumb: when you catch yourself copy-pasting the same
`if (...) { printf("FAIL ..."); return 1; }` for the fifth time, you are
writing a framework badly. Write it once, well, as a tiny header — that is
what this part does.

---

## 2. The Landscape: Unity, Check, cmocka (Concept Table)

You do **not** need to install anything for this chapter. The mini-runner
below teaches the concepts all three share. But you should know what the
real options look like so you can recognize them in other projects:

| Framework | Style | Needs install? | Strengths | Costs / quirks |
| --- | --- | --- | --- | --- |
| [Unity](https://github.com/ThrowTheSwitch/Unity) (ThrowTheSwitch) | single `unity.c` + `unity.h`, `TEST_ASSERT_EQUAL_INT` | no (drop two files in) | tiny, embedded-friendly, huge assert catalog, great docs | you write your own `main` + runner script (Ruby helper optional) |
| [Check](https://libcheck.github.io/check/) | `START_TEST` / `END_TEST`, suites, `srunner` | yes (`apt install check`, `pkg-config`) | process isolation per test (a segfault fails one test, not the suite), timeouts, XML output | heavier dependency, fork-based (slow on some embedded targets) |
| [cmocka](https://cmocka.org/) | `cmocka_unit_test()`, `assert_int_equal` | yes (`apt install libcmocka-dev`) | mocking support, fixtures, JUnit XML, active maintenance | API surface larger than needed for small projects |

Conceptual equivalence — the same test in three dialects:

| Idea | Mini-runner (this chapter) | Unity | Check |
| --- | --- | --- | --- |
| Declare a test | `TEST(clamp_low) { ... }` | `void test_clamp_low(void) { ... }` | `START_TEST(test_clamp_low) { ... } END_TEST` |
| Assert integers equal | `EXPECT_INT_EQ(a, b)` | `TEST_ASSERT_EQUAL_INT(a, b)` | `ck_assert_int_eq(a, b)` |
| Run everything | `return run_all_tests();` | generated runner or hand loop | `srunner_run_all(sr, CK_NORMAL)` |
| Skip | `SKIP("reason")` | `TEST_IGNORE_MESSAGE("reason")` | N/A (remove from suite) |

If you learn the left column, the other two are a one-hour port later.
That is the point: **concepts transfer, syntax does not matter.**

When to actually adopt one (preview of section 6):

```text
  your suite today                    adopt a real framework when...
  ───────────────                    ─────────────────────────────────
  < 50 tests, one .c file             > 100 tests OR need XML for CI
  no mocks needed                     you must mock hardware / syscalls
  single developer                    a segfault in test 12 hides tests 13-40
  no CI integration                   CI needs per-test timing / TAP output
```

Until then, the harness below is the standard for this course.

---

## 3. The Mini-Runner: `TEST(name)` + Registration

Goal: writing a new test is one block, zero runner edits:

```c
TEST(my_new_case) {
    EXPECT_INT_EQ(add(2, 3), 5);
}
```

Design in ASCII:

```text
  test_table[] (generated by TEST macro)
  ┌────────────┬──────────────┐
  │ name       │ func pointer │
  ├────────────┼──────────────┤
  │ "clamp_low"│ test_clamp_low│
  │ "clamp_high"│test_clamp_high│
  │ "clamp_mid"│ test_clamp_mid│
  └────────────┴──────────────┘
          │
          ▼
  run_all_tests(): for each entry → run → print PASS/FAIL → count
          │
          ▼
  exit code: 0 = all passed, 1 = at least one failed
```

Implementation (`mini_test.h`, ~80 lines, C11, no dependencies):

```c
/* mini_test.h — tiny test framework for this course. C11, no dependencies. */
#ifndef MINI_TEST_H
#define MINI_TEST_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* --- per-test state (reset before each test) --- */
static int mt_failures_in_test;
static int mt_skipped_in_test;
static const char *mt_current_name;

/* --- global counters --- */
static int mt_total_passed;
static int mt_total_failed;
static int mt_total_skipped;

/* --- optional fixture hooks (NULL = none) --- */
static void (*mt_setup)(void);
static void (*mt_teardown)(void);

/* --- test table --- */
typedef void (*mt_func_t)(void);

typedef struct {
    const char *name;
    mt_func_t func;
} MtTest;

#define MT_MAX_TESTS 256

static MtTest mt_tests[MT_MAX_TESTS];
static int mt_test_count;

/* Register one test. Called via constructor trick below. */
static void mt_register(const char *name, mt_func_t func) {
    if (mt_test_count < MT_MAX_TESTS) {
        mt_tests[mt_test_count].name = name;
        mt_tests[mt_test_count].func = func;
        mt_test_count++;
    }
}

/*
 * TEST(name): defines test_##name() and auto-registers it.
 * Uses GCC/Clang __attribute__((constructor)). This is a common
 * extension (not ISO C), guarded here because course toolchains
 * are GCC/Clang. On strict ISO-only compilers, call
 * mt_register("name", test_name) manually from main() instead.
 */
#define TEST(name)                                                      \
    static void test_##name(void);                                      \
    static void mt_reg_##name(void) __attribute__((constructor));       \
    static void mt_reg_##name(void) { mt_register(#name, test_##name); } \
    static void test_##name(void)

/* --- asserts: record failure, print once, keep running the test --- */

#define EXPECT_INT_EQ(actual, expected) do {                            \
    long long _a = (long long)(actual);                                 \
    long long _e = (long long)(expected);                               \
    if (_a != _e) {                                                     \
        printf("  FAIL %s: expected %lld, got %lld\n",                  \
               mt_current_name, _e, _a);                                \
        mt_failures_in_test++;                                          \
    }                                                                   \
} while (0)

#define EXPECT_STR_EQ(actual, expected) do {                            \
    const char *_a = (actual);                                          \
    const char *_e = (expected);                                        \
    if (_a == NULL || _e == NULL || strcmp(_a, _e) != 0) {              \
        printf("  FAIL %s: expected \"%s\", got \"%s\"\n",              \
               mt_current_name,                                         \
               _e ? _e : "(null)", _a ? _a : "(null)");                \
        mt_failures_in_test++;                                          \
    }                                                                   \
} while (0)

#define EXPECT_TRUE(cond) do {                                          \
    if (!(cond)) {                                                      \
        printf("  FAIL %s: expected true: %s\n",                        \
               mt_current_name, #cond);                                 \
        mt_failures_in_test++;                                          \
    }                                                                   \
} while (0)

/* ASSERT_* aborts the current test immediately (returns to runner). */
#define ASSERT_INT_EQ(actual, expected) do {                            \
    EXPECT_INT_EQ((actual), (expected));                                \
    if (mt_failures_in_test > 0) {                                      \
        return;                                                         \
    }                                                                   \
} while (0)

/* SKIP("reason"): mark test skipped, stop its body. */
#define SKIP(reason) do {                                               \
    printf("  SKIP %s: %s\n", mt_current_name, (reason));               \
    mt_skipped_in_test = 1;                                             \
    return;                                                             \
} while (0)

/* Run every registered test, honoring TEST_ONLY=name to focus one. */
static int run_all_tests(void) {
    const char *only = getenv("TEST_ONLY"); /* NULL or empty = run all */
    int i;

    for (i = 0; i < mt_test_count; i++) {
        mt_current_name = mt_tests[i].name;

        /* Focused run: silently skip every non-matching test. */
        if (only != NULL && only[0] != '\0' &&
            strcmp(mt_current_name, only) != 0) {
            continue;
        }

        mt_failures_in_test = 0;
        mt_skipped_in_test = 0;

        if (mt_setup != NULL) {
            mt_setup();
        }
        mt_tests[i].func();
        if (mt_teardown != NULL) {
            mt_teardown();
        }

        if (mt_skipped_in_test) {
            mt_total_skipped++;
            printf("[ SKIP ] %s\n", mt_current_name);
        } else if (mt_failures_in_test == 0) {
            mt_total_passed++;
            printf("[ PASS ] %s\n", mt_current_name);
        } else {
            mt_total_failed++;
            printf("[ FAIL ] %s (%d assertion(s) failed)\n",
                   mt_current_name, mt_failures_in_test);
        }
    }

    printf("----\n%d passed, %d failed, %d skipped\n",
           mt_total_passed, mt_total_failed, mt_total_skipped);
    return mt_total_failed != 0;
}

#endif /* MINI_TEST_H */
```

Note the two details that make the runner trustworthy rather than just
short: `getenv("TEST_ONLY")` is read once up front (unset or empty means
"run everything" — section 5c turns this into your debug workflow), and
skips are counted in their own bucket so `1 skipped, 0 failed` still exits
`0`. If you port this header to a compiler without `getenv` (bare metal),
replace that one line with `const char *only = NULL;` and every test runs
unconditionally.

---

## 4. Migration: 3 Manual Tests Become 3 `TEST()` Blocks

Take the manual `manual.c` from section 1 and migrate it mechanically.
Each migration step is one mechanical edit:

| Step | Manual form | Mini-runner form |
| --- | --- | --- |
| 1. Include the header | `#include <stdio.h>` | `#include <stdio.h>` + `#include <stdlib.h>` + `#include "mini_test.h"` |
| 2. Declare a test | `static int test_clamp_low(void) {` | `TEST(clamp_low) {` |
| 3. Assert | `if (clamp(...) != 0) { printf(...); return 1; }` | `EXPECT_INT_EQ(clamp(...), 0);` |
| 4. Runner | hand-written `main` with 3 calls | `int main(void) { return run_all_tests(); }` |

Result (`test_clamp.c`):

```c
/* test_clamp.c — migrated suite. */
#include <stdio.h>
#include <stdlib.h>

#include "mini_test.h"

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

TEST(clamp_low) {
    EXPECT_INT_EQ(clamp(-5, 0, 10), 0);
}

TEST(clamp_high) {
    EXPECT_INT_EQ(clamp(99, 0, 10), 10);
}

TEST(clamp_mid) {
    EXPECT_INT_EQ(clamp(4, 0, 10), 4);
}

int main(void) {
    return run_all_tests();
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g test_clamp.c -o test_clamp && ./test_clamp; echo "exit: $?"
```

```text
[ PASS ] clamp_low
[ PASS ] clamp_high
[ PASS ] clamp_mid
----
3 passed, 0 failed, 0 skipped
exit: 0
```

Now break one on purpose to see the assert quality:

```c
TEST(clamp_mid_broken_demo) {
    EXPECT_INT_EQ(clamp(4, 0, 10), 999); /* wrong on purpose */
}
```

```text
  FAIL clamp_mid_broken_demo: expected 999, got 4
[ FAIL ] clamp_mid_broken_demo (1 assertion(s) failed)
----
3 passed, 1 failed, 0 skipped
```

Compare with the manual version, which would print only `FAIL
test_clamp_mid` — no expected/got values. That missing detail is the
difference between a 10-second diagnosis and a 10-minute one.

Adding a fourth test is now zero-touch on the runner:

```c
TEST(clamp_boundary_lo) {
    EXPECT_INT_EQ(clamp(0, 0, 10), 0);
    EXPECT_INT_EQ(clamp(-1, 0, 10), 0);
}

TEST(clamp_boundary_hi) {
    EXPECT_INT_EQ(clamp(10, 0, 10), 10);
    EXPECT_INT_EQ(clamp(11, 0, 10), 10);
}
```

No `main()` edit. No counter edit. Registration happened at startup via
the constructor. That is the whole payoff.

---

## 5. Setup/Teardown, Skip, and Focus

Three features that turn a test list into a workflow.

### 5a. Fixtures: `mt_setup` / `mt_teardown`

A fixture is code that runs before (`setup`) and after (`teardown`) every
test: allocate a scratch buffer, reset a global, open a temp file, then
release it. Without fixtures, each test repeats the same five lines, and
the fifth copy forgets the cleanup.

```c
/* fixture_demo.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "mini_test.h"

#define SCRATCH_SIZE 64

static char *scratch;

static void my_setup(void) {
    scratch = malloc(SCRATCH_SIZE);
    if (scratch != NULL) {
        memset(scratch, 0, SCRATCH_SIZE);
    }
}

static void my_teardown(void) {
    free(scratch);
    scratch = NULL; /* use-after-teardown now crashes loudly, not silently */
}

TEST(scratch_starts_empty) {
    ASSERT_INT_EQ(scratch != NULL, 1); /* abort test if setup failed */
    EXPECT_INT_EQ((int)scratch[0], 0);
    EXPECT_STR_EQ(scratch, "");
}

TEST(scratch_writes_do_not_leak) {
    ASSERT_INT_EQ(scratch != NULL, 1);
    strcpy(scratch, "hello");
    EXPECT_STR_EQ(scratch, "hello");
    /* no free here — teardown owns it */
}

int main(void) {
    mt_setup = my_setup;
    mt_teardown = my_teardown;
    return run_all_tests();
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g fixture_demo.c -o fixture_demo && ./fixture_demo
```

```text
[ PASS ] scratch_starts_empty
[ PASS ] scratch_writes_do_not_leak
----
2 passed, 0 failed, 0 skipped
```

Flow diagram:

```text
  run_all_tests()
    for each test:
      mt_setup()      → malloc + zero scratch
      test_body()     → use scratch freely
      mt_teardown()   → free + NULL
    next test gets a FRESH scratch (no cross-test leakage)
```

Rules:

1. `setup` must leave the world valid or the test must `ASSERT` it did.
   A `NULL` scratch used blindly is a segfault that looks like a product bug.
2. `teardown` owns cleanup — test bodies never free fixture resources.
3. Keep fixtures tiny. If setup needs 50 lines, the unit under test is too
   coupled; that smell is the fixture telling you to refactor.

### 5b. `SKIP()` for known-broken / platform-specific tests

```c
TEST(json_unicode_not_yet_supported) {
    SKIP("unicode escapes land in milestone 2");
    EXPECT_INT_EQ(0, 1); /* never runs while skipped */
}
```

```text
  SKIP json_unicode_not_yet_supported: unicode escapes land in milestone 2
[ SKIP ] json_unicode_not_yet_supported
----
0 passed, 0 failed, 1 skipped
```

A skipped test is visible in the summary (unlike a commented-out test,
which is invisible). The summary line `0 passed, 0 failed, 1 skipped`
tells CI and teammates: "we know, we tracked it, it is not a pass."

### 5c. Focus with `TEST_ONLY=name` while debugging

When test 37 of 40 fails, you do not want to scroll past 36 passes:

```bash
TEST_ONLY=clamp_high ./test_clamp
```

```text
[ PASS ] clamp_high
----
1 passed, 0 failed, 0 skipped
```

Only the matching test runs. Unset the variable to run everything again.
(The filter lives in `run_all_tests()` itself — section 3 — so every
binary built on this header supports it with no extra wiring.)

| Workflow | Command | When |
| --- | --- | --- |
| Full suite | `./test_clamp` | before every commit |
| One test | `TEST_ONLY=clamp_high ./test_clamp` | debugging a single failure |
| Skip known issue | `SKIP("...")` inside the test | tracked debt, milestone later |
| Fixture reset | `mt_setup` / `mt_teardown` | every test needs the same world |

---

## 6. When to Adopt a Real Framework vs Keep the Harness

The mini-runner is the course default. A real framework is an upgrade you
buy with a dependency. Here is the decision as a table, not vibes:

| Signal | Keep the mini-runner | Adopt Unity / Check / cmocka |
| --- | --- | --- |
| Test count | < ~100, one or two binaries | hundreds, suites need grouping |
| Isolation | a crash failing the whole binary is acceptable | must have per-test processes (Check forks each test) |
| Mocking | hand stubs suffice (2–3 functions) | need generated mocks / call expectations (cmocka) |
| CI output | plain text summary is fine | CI parses JUnit XML / TAP |
| Portability target | desktop Linux/macOS, GCC/Clang | bare metal (Unity shines) or strict POSIX (Check) |
| Team familiarity | everyone reads the 80-line header | team already knows Unity asserts |

Migration cost if you switch later is low *because* the concepts match:
`TEST(x)` → `void test_x(void)`, `EXPECT_INT_EQ` → `TEST_ASSERT_EQUAL_INT`
/ `ck_assert_int_eq`, `run_all_tests()` → Unity runner / `srunner`.
The asserts rename; the test logic does not change.

One hard rule either way: **the harness (mini or real) is the standard,
never ad-hoc `printf` debugging committed as "tests".** If a check is worth
running once, it is worth registering as a `TEST()` with a name, an assert
that prints both sides, and an exit code CI can read. Part 02 makes that
exit code and the `make test` habit concrete.

---

## Bugs You Will Meet

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| 1 | New `TEST()` never runs (total count unchanged) | forgot `#include "mini_test.h"` in that `.c`, or constructor extension unsupported | include the header; on exotic compilers call `mt_register` manually |
| 2 | `FAIL name: expected 10, got 10` (looks equal) | signed/unsigned or `int` vs `size_t` comparison truncated by `(long long)` cast, or float compared with `EXPECT_INT_EQ` | use matching assert (`EXPECT_TRUE(a == b)` for floats with epsilon; cast sizes explicitly) |
| 3 | Tests pass alone (`TEST_ONLY=x`) but fail together | shared global mutated by an earlier test; missing fixture reset | move state into `mt_setup`, NULL it in `mt_teardown` |
| 4 | `SKIP` still counts as failure in CI | runner version without `mt_skipped_in_test` branch | use the fixed `run_all_tests` (section 3) that counts skips separately |
| 5 | Double-free when a test also frees fixture memory | ownership unclear: both body and teardown free | rule: teardown owns fixture memory, bodies never free it |
| 6 | `__attribute__((constructor))` warning on MSVC / TCC | extension is GCC/Clang-only | register explicitly: `mt_register("name", test_name);` in `main` before running |
| 7 | Adopted Unity but tests silently do nothing | forgot `RUN_TEST(test_x)` lines in runner `main` (Unity needs explicit calls) | list every test in `main`, or use the auto-generate script |

---

## Checklist

1. Can you name the four things a framework adds over `main()` + `if` (named tests, rich asserts, fixtures, runner/skip)?
2. Can you place Unity vs Check vs cmocka from the table without installing any of them?
3. Does your `TEST(name)` need zero runner edits (auto-registration)?
4. Do failing asserts print *both* expected and actual values?
5. Is fixture memory owned by `teardown`, never by test bodies?
6. Does `SKIP("reason")` show up in the summary instead of vanishing?
7. Can you focus one test with `TEST_ONLY=name` and then run the full suite unset?
8. Do you know the one-line trigger for adopting a real framework (isolation, mocks, or XML — not test count alone)?

---

#### [Previous](../02-unit-tests/exercises.md) | [Next](./02-part.md)
