# Table-Driven Tests and Fixtures

One test per input does not scale. Ten inputs for `binary_search` written as ten copy-pasted test functions drift apart within a week: someone fixes case 3 and forgets case 7. A table-driven test stores `{input, expected}` rows in an array and runs the same check in a loop, so adding a case means adding one line, not one function.

Fixtures solve the other half: any test that needs setup (a `malloc` buffer, a temporary file) must also do teardown (`free`, `fclose`), on every path, including the failure path. This part shows both patterns, plus how to test failure inputs (`NULL`, empty, full, out-of-memory) and how to keep suites deterministic.

Compile everything in this chapter with:

```text
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer
```

Same harness as part 1 (counters, `CHECK_EQ`, `CHECK_STR`, `RUN_TEST`). It is repeated in each full listing so every file compiles on its own.

---

## Tables: One Loop Instead of Ten Functions

The shape is always the same:

```text
  table of rows                loop                    check per row
  +----------------+     +-----------------+     +----------------------+
  | input | expect |     | for each row i  |     | CHECK_EQ(run(input), |
  |-------+--------| --> |   got = run()   | --> |   expected) + index  |
  |  10   |   0    |     |   compare       |     | printed on failure   |
  |  99   |  -1    |     +-----------------+     +----------------------+
  +----------------+
   adding a case = adding a row
```

Concrete example with `binary_search` (sorted input, the precondition is part of the contract):

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

static int binary_search(const int v[], int n, int target) {
    if (v == NULL || n <= 0) {
        return -1;
    }
    int lo = 0;
    int hi = n - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;   /* no (lo + hi) overflow */
        if (v[mid] == target) {
            return mid;
        } else if (v[mid] < target) {
            lo = mid + 1;
        } else {
            hi = mid - 1;
        }
    }
    return -1;
}

static void test_binary_search_table(void) {
    static const int data[] = {10, 20, 30, 40, 50};
    static const struct {
        int input;
        int expected;
    } cases[] = {
        { 10,  0 },   /* first element */
        { 20,  1 },
        { 30,  2 },   /* middle */
        { 40,  3 },
        { 50,  4 },   /* last element */
        {  5, -1 },   /* below range */
        { 99, -1 },   /* above range */
        { 25, -1 },   /* gap between elements */
    };

    for (size_t i = 0; i < sizeof(cases) / sizeof(cases[0]); i++) {
        int got = binary_search(data, 5, cases[i].input);
        if (got != cases[i].expected) {
            g_fail++;
            fprintf(stderr, "  FAIL %s:%d: case %zu: input %d -> got %d, want %d\n",
                    __FILE__, __LINE__, i, cases[i].input, got, cases[i].expected);
        } else {
            g_pass++;
        }
    }
}

int main(void) {
    RUN_TEST(test_binary_search_table);
    printf("----\n%d passed, %d failed\n", g_pass, g_fail);
    return g_fail != 0 ? 1 : 0;
}
```

```text
[ RUN  ] test_binary_search_table
[ PASS ] test_binary_search_table
----
8 passed, 0 failed
```

Two details make tables debuggable:

1. **Print the row index `i` on failure.** Eight identical `got -1, want 2` lines are useless; `case 2: input 30 -> got -1, want 2` tells you exactly which row broke.
2. **Keep the table `static const`.** The rows are read-only test data, never modified by the loop. `const` lets the compiler place them in read-only memory and warn if a test accidentally writes through them.

When a row fails, fix the code or the row — but decide consciously. A wrong `expected` value is a test bug, not a code bug; re-derive it by hand (for `binary_search`, trace `lo/mid/hi`) rather than copying the program's output into the table, which just blesses whatever the code does.

| Table style | Good for | Avoid when |
| --- | --- | --- |
| `{input, expected}` + loop | pure functions (`search`, comparators, parsers) | each case needs different setup |
| one function per case | cases with different fixtures or error paths | more than ~5 near-identical cases |
| generated rows (`for` loop building inputs) | ranges, boundaries (`0..255`) | readability beats brevity — prefer explicit rows for tricky cases |

---

## Fixtures: Setup and Teardown Per Case

A fixture is the code that builds the world a test needs and then tears it down. The rule is absolute: **every setup must be undone on every path**, pass or fail, or the suite leaks and later cases inherit garbage.

### Fixture 1: `malloc` + `free` Per Case

```text
  per iteration:
  +---------+    +--------+    +--------+
  | malloc  | -> |  test  | -> |  free  |
  +---------+    +--------+    +--------+
       |                           ^
       +-- on NULL: fail row, -----+
           skip test, no free
           (nothing allocated)
```

```c
static void test_reverse_table_heap(void) {
    static const struct {
        const char *input;
        const char *expected;
    } cases[] = {
        { "abc",   "cba" },
        { "hello", "olleh" },
        { "",      "" },
        { "a",     "a" },
    };

    for (size_t i = 0; i < sizeof(cases) / sizeof(cases[0]); i++) {
        size_t n = strlen(cases[i].input);
        char *buf = malloc(n + 1);
        if (buf == NULL) {
            g_fail++;
            fprintf(stderr, "  FAIL %s:%d: case %zu: malloc failed\n",
                    __FILE__, __LINE__, i);
            continue;
        }
        for (size_t j = 0; j < n; j++) {
            buf[j] = cases[i].input[n - 1 - j];
        }
        buf[n] = '\0';

        if (strcmp(buf, cases[i].expected) != 0) {
            g_fail++;
            fprintf(stderr, "  FAIL %s:%d: case %zu: reverse(\"%s\") -> \"%s\", want \"%s\"\n",
                    __FILE__, __LINE__, i, cases[i].input, buf, cases[i].expected);
        } else {
            g_pass++;
        }
        free(buf);   /* teardown: exactly one free per malloc, fail or pass */
    }
}
```

`free` sits after the check, outside any `if/else`, so both branches reach it. The only `continue` is the `malloc == NULL` path, where there is nothing to free. Under ASan/LSan this loop is silent; move `free(buf)` inside the `else` and LSan reports 4 leaks on the first red run.

### Fixture 2: Temporary Files with `tmpfile`

File tests must never touch the real filesystem layout (`/tmp/mytest.txt` collides with parallel runs and lingers after crashes). `tmpfile()` creates an anonymous temporary file, already opened for reading and writing, automatically deleted on `fclose`:

```c
#include <stdio.h>
#include <string.h>

/* Writes lines to f, rewinds, reads them back, compares. 0 ok, -1 mismatch. */
static int round_trip_lines(FILE *f, const char *const *lines, size_t nlines) {
    if (f == NULL || (nlines > 0 && lines == NULL)) {
        return -1;
    }
    for (size_t i = 0; i < nlines; i++) {
        if (fputs(lines[i], f) == EOF) {
            return -1;
        }
        if (fputc('\n', f) == EOF) {
            return -1;
        }
    }
    rewind(f);
    char buf[256];
    for (size_t i = 0; i < nlines; i++) {
        if (fgets(buf, sizeof(buf), f) == NULL) {
            return -1;
        }
        buf[strcspn(buf, "\n")] = '\0';   /* strip one trailing newline */
        if (strcmp(buf, lines[i]) != 0) {
            return -1;
        }
    }
    return 0;
}

static void test_tmpfile_round_trip(void) {
    static const char *lines[] = { "alpha", "beta", "gamma" };
    FILE *f = tmpfile();
    if (f == NULL) {
        g_fail++;
        fprintf(stderr, "  FAIL %s:%d: tmpfile() returned NULL\n",
                __FILE__, __LINE__);
        return;
    }
    if (round_trip_lines(f, lines, 3) != 0) {
        g_fail++;
        fprintf(stderr, "  FAIL %s:%d: round-trip mismatch\n",
                __FILE__, __LINE__);
    } else {
        g_pass++;
    }
    fclose(f);   /* teardown: file vanishes, no cleanup script needed */
}
```

Real output:

```text
[ RUN  ] test_tmpfile_round_trip
[ PASS ] test_tmpfile_round_trip
----
1 passed, 0 failed
```

If you need a *named* temporary file (to pass a path to the function under test), use `mkstemp` instead: it creates the file with `0600` permissions and returns a file descriptor. Always `close` it and `unlink` the path when done. Never use `tmpnam` or fixed names — both are race conditions.

---

## Testing Failure Inputs: NULL, Empty, Full, OOM

Happy-path tables prove the function works when used correctly. Failure tests prove it does not crash, hang, or corrupt memory when used incorrectly. Every container and every allocator-facing function needs all four:

| Failure class | Example check | Expected behavior |
| --- | --- | --- |
| `NULL` pointer | `binary_search(NULL, 5, 10)` | return error (`-1`), no dereference |
| empty (`n == 0`, `""`) | `linear_search(v, 0, 10)` | return error, no read at all |
| full / boundary | push into a capacity-4 buffer, 5th push grows or returns `-1` | documented growth or clean error, never overflow |
| out-of-memory | allocation fails | return `-1`, old state intact, nothing leaked |

`NULL` and empty are trivially testable — just call the function. OOM needs a seam: a place where the test can inject failure. The simplest seam in C is a function pointer (or a counter) standing in for `malloc`:

```c
#include <stdlib.h>

/* Seam: tests flip g_alloc_fail_at to force the next malloc to fail. */
static int g_alloc_fail_at = -1;   /* -1 = never fail */
static int g_alloc_count = 0;

static void *test_malloc(size_t n) {
    if (g_alloc_fail_at >= 0 && g_alloc_count == g_alloc_fail_at) {
        g_alloc_count++;
        return NULL;   /* simulated OOM */
    }
    g_alloc_count++;
    return malloc(n);
}

static int push_with_alloc(int **data, size_t *cap, int value) {
    size_t new_cap = (*cap == 0) ? 4 : (*cap * 2);
    int *tmp = test_malloc(new_cap * sizeof(*tmp));
    if (tmp == NULL) {
        return -1;
    }
    /* (copy old contents here in a real container) */
    free(*data);
    *data = tmp;
    *cap = new_cap;
    (*data)[0] = value;
    return 0;
}

static void test_oom_seam(void) {
    int *data = NULL;
    size_t cap = 0;

    g_alloc_count = 0;
    g_alloc_fail_at = 0;                    /* first allocation fails */
    CHECK_EQ(push_with_alloc(&data, &cap, 42), -1);
    CHECK_EQ(data == NULL, 1);              /* unchanged: still NULL */
    CHECK_EQ((long long)cap, 0);

    g_alloc_fail_at = -1;                   /* seam off: real malloc */
    CHECK_EQ(push_with_alloc(&data, &cap, 42), 0);
    CHECK_EQ((long long)cap, 4);
    free(data);
}
```

Production code calls `malloc` through the seam (a `#define malloc test_malloc` in the test build, or a function pointer field in a config struct). The test flips the failure on, asserts the error path, flips it off, and asserts recovery. Without a seam, OOM is untestable short of exhausting real memory — which is slow, flaky, and rude to the rest of the machine.

---

## Testing Strings: Always `strcmp`, Never `==`

From part 1, but worth repeating with the failure modes tables expose:

```c
static void test_strings_table(void) {
    static const struct {
        const char *got;
        const char *want;
    } cases[] = {
        { "hello",  "hello" },
        { "",       "" },
        { NULL,     NULL },      /* both absent: equal */
        { "Hello",  "hello" },   /* case differs: NOT equal (this row fails on purpose below) */
    };

    for (size_t i = 0; i < 3; i++) {   /* only the first three should pass */
        const char *a = cases[i].got;
        const char *e = cases[i].want;
        int ok = (a == NULL && e == NULL) || (a && e && strcmp(a, e) == 0);
        if (!ok) {
            g_fail++;
            fprintf(stderr, "  FAIL %s:%d: case %zu: got \"%s\", want \"%s\"\n",
                    __FILE__, __LINE__, i,
                    a ? a : "(null)", e ? e : "(null)");
        } else {
            g_pass++;
        }
    }
}
```

String pitfalls that break naive comparisons:

- **Trailing newline from `fgets`.** `"hello\n"` vs `"hello"` fails `strcmp`. Strip with `buf[strcspn(buf, "\n")] = '\0'` before comparing, exactly as the `tmpfile` fixture does.
- **Case and whitespace.** `"Hello"` vs `"hello"` differ; `"a "` vs `"a"` differ. If the spec says comparison is case-insensitive, write a `strcasecmp`-style helper and test *that* — do not lowercase inside the assertion.
- **`NULL` vs `""`.** An absent string is not an empty string. `CHECK_STR` treats `NULL == NULL` as pass and `NULL` vs `""` as fail; decide the same in your tables and document which one your functions return.

---

## Flaky Tests and How to Avoid Them

A flaky test passes nine times and fails the tenth with no code change. Every flake in C comes from depending on something the test does not control:

| Flaky source | Example | Deterministic replacement |
| --- | --- | --- |
| wall-clock time | `assert(elapsed < 100ms)` | test logic, not speed; measure benchmarks separately |
| `rand()` without seed | random inputs each run | fixed seed (`srand(42)`) or hardcoded table rows |
| addresses / ASLR | `assert(ptr == 0x...)` or hashing pointers | compare contents, never addresses |
| ports / network | bind port 8080 | `tmpfile` / pipes, or skip network in unit tests |
| absolute paths | open `/home/user/data.txt` | relative paths or `tmpfile`/`mkstemp` |
| uninitialized memory | read `buf[i]` that was never written | always init; run under MSan/ASan to catch it |
| test order dependence | test B assumes test A ran first | each test builds its own fixture from scratch |

Rules for this course: no test may depend on the time of day, the machine's address layout, an open port, or the current directory's contents. If you need randomness (fuzzing a parser), seed it: `srand(1234)` makes the "random" sequence identical on every run and every machine, so a failure reproduces with the same seed printed in the failure message.

Isolation completes the picture: each table row and each fixture must be independent. The `malloc`-per-case loop above frees inside the iteration; the `tmpfile` test closes its file before returning. No globals leak from case `i` into case `i + 1` except the `g_pass` / `g_fail` counters — and those are the scoreboard, not state.

---

## Bugs You Will Write (and What They Look Like)

### Bug 1: Shared Buffer Across Rows

```c
char *buf = malloc(64);              /* BUG: one buffer, reused for all rows */
for (size_t i = 0; i < n; i++) {
    fill(buf, cases[i].input);       /* leftover bytes from row i-1 leak into row i */
    CHECK_STR(buf, cases[i].expected);
}
free(buf);
```

Row 2 passes only because row 1 left the right tail in the buffer. Allocate (and free) inside the loop, or `memset` the buffer each iteration. ASan will not catch this — it is a logic bug, not a memory error — only independent fixtures catch it.

### Bug 2: Forgetting `rewind` Before Reading Back

```c
fputs("hello\n", f);
/* BUG: missing rewind(f); */
char buf[16];
fgets(buf, sizeof(buf), f);   /* reads EOF immediately: buf untouched */
```

`fgets` returns `NULL`, and if the test ignores the return value it compares stale stack contents — sometimes passing, sometimes failing, depending on what was on the stack. Always check `fgets`/`fread` return values, and always `rewind` (or `fseek(f, 0, SEEK_SET)`) between the write phase and the read phase.

### Bug 3: Blessing Output Into the Table

```c
{ 25, 1 },   /* BUG: copied program output instead of hand-derived answer */
```

`binary_search({10,20,30,40,50}, 5, 25)` returns `-1` (25 is absent), but someone pasted the buggy program's `1` into the table to make it green. Tables must be derived from the spec by hand. When code and table disagree, trace the algorithm on paper before touching either.

### Bug 4: Time-Dependent "Performance" Test

```c
clock_t t0 = clock();
sort(big, 1000000);
CHECK_EQ(clock() - t0 < CLOCKS_PER_SEC / 10, 1);  /* BUG: fails on loaded CI */
```

Passes on a laptop, fails on a busy CI runner. Timing is not correctness. Test that the output is sorted (a loop with `CHECK_EQ(out[i] <= out[i+1], 1)`); measure speed in a separate benchmark that never fails the build.

| Bug | Symptom | Fix |
| --- | --- | --- |
| shared buffer across rows | row passes only in sequence, fails alone | alloc/free (or memset) per iteration |
| missing `rewind` | `fgets` returns `NULL`, stale compare | `rewind` + check I/O return values |
| blessed output in table | suite green on wrong code | hand-derive every `expected` |
| timing assertion | flaky red on slow machines | assert sortedness, not milliseconds |
| fixed path / port | fails in parallel or second run | `tmpfile` / `mkstemp`, no fixed names |
| `==` on strings | passes by interning luck | `strcmp` via `CHECK_STR` |

---

## Checklist

- [ ] I can convert five copy-pasted tests into one `{input, expected}` table with a loop and a row index in every failure message.
- [ ] Each fixture allocates and frees (or opens and closes) inside the same iteration, on pass and fail paths; LSan is clean.
- [ ] File tests use `tmpfile` (or `mkstemp` + `close` + `unlink`), check every I/O return, and `rewind` between write and read phases.
- [ ] I test `NULL`, empty, full/boundary, and OOM (via a `malloc` seam) — not just the happy path.
- [ ] String comparisons go through `strcmp` with `NULL` handled and newlines stripped; `==` never touches strings.
- [ ] No test reads the clock, the network, an absolute path, an address, or unseeded `rand`; tables are hand-derived, not pasted from program output.

---

#### [Previous](./01-part.md) | [Next](./exercises.md)
