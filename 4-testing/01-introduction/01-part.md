# Why Test C

In Python, a bug usually announces itself. An index past the end raises an exception. A wrong type raises another exception. The program stops, points at the line, and waits for a fix.

In C, a bug usually says nothing. The program keeps running, prints a plausible answer, exits with status `0`, and corrupts something you will only notice next week. The difference is not style. It is the execution model: C performs almost no runtime checks, so reading one element past an array, using an uninitialized variable, or leaking a block are all defined by the standard as *your* problem, not the runtime's.

That is why testing in C is not optional polish. It is the only way to see the bugs the language hides from you.

```c
/* Looks harmless. Compiles clean. Runs clean. Still wrong. */
#include <stdio.h>

int main(void) {
    int scores[4] = { 10, 20, 30, 40 };
    int total = 0;
    for (int i = 0; i <= 4; i++) {   /* BUG: <= reads scores[4] */
        total += scores[i];
    }
    printf("total = %d\n", total);
    return 0;
}
```

```text
total = 100
```

On this machine, on this run, `scores[4]` happened to read a `0` sitting next to the array, so the total *looks* right. Recompile with different optimization, run it on another machine, or add one local variable, and the same program prints garbage or crashes. Nothing in the plain output tells you which run to trust. A test plus a sanitizer does.

| Language | Out-of-bounds read | Uninitialized use | Leak | Signed overflow |
| --- | --- | --- | --- | --- |
| Python / Java | exception, program stops | exception / default | GC handles it | exception or big-int |
| C (plain build) | silent garbage, maybe crash later | silent garbage | silent growth | silent wraparound |

The rest of this part makes that table actionable: what "silent" means mechanically, how testing differs from debugging, why every fix starts with a reproducer, and how AddressSanitizer and UndefinedBehaviorSanitizer turn silent bugs into loud, line-numbered reports.

---

## Silent Bugs: The Three Ways C Lies to You

Group every silent C bug into one of three families. The family tells you which tool catches it.

**1. Out-of-bounds access (spatial bug).** You touch memory that does not belong to the object. The read or write succeeds at the hardware level because the address is still mapped — it just belongs to a neighbor.

```c
#include <stdio.h>
#include <string.h>

static void copy_tag(char *dst, size_t dst_cap, const char *src) {
    /* BUG: copies the terminator over the edge when src fills dst. */
    size_t len = strlen(src);
    for (size_t i = 0; i <= len; i++) {   /* <= copies len + 1 bytes */
        dst[i] = src[i];
    }
    (void)dst_cap;   /* capacity ignored: the real bug */
}

int main(void) {
    char tag[8];
    copy_tag(tag, sizeof(tag), "ABCDEFG");   /* 7 chars + terminator = 8, fits */
    printf("tag=%s\n", tag);
    copy_tag(tag, sizeof(tag), "ABCDEFGH");  /* 8 chars + terminator = 9, overflows */
    printf("tag=%s\n", tag);
    return 0;
}
```

```text
tag=ABCDEFG
tag=ABCDEFGH
```

Both lines print exactly what a casual reader expects. The second call wrote one byte past `tag`, into whatever the compiler placed next on the stack. No crash, no warning, exit code `0`. The corruption may surface ten thousand lines later as a wrong return address or a mysteriously changed variable.

**2. Undefined behavior that "works" (temporal / arithmetic bug).** Signed integer overflow, null-pointer arithmetic, use-after-free, data races: the standard says the compiler may assume they never happen, so the optimizer reasons from a false premise and the program takes a path that exists in no source line you wrote.

```c
#include <stdio.h>
#include <limits.h>

static int add_scores(int a, int b) {
    return a + b;   /* BUG when the sum exceeds INT_MAX: signed overflow is UB */
}

int main(void) {
    int total = add_scores(INT_MAX, 1);
    printf("total = %d\n", total);
    if (total < INT_MAX) {
        printf("overflow happened (but the check may be optimized away)\n");
    }
    return 0;
}
```

```text
total = -2147483648
```

The program wrapped to a negative number and kept going. A compiler at `-O2` is allowed to assume `a + b > a` whenever `b > 0`, so the `total < INT_MAX` guard can be folded away entirely. The bug is not the wrong number. The bug is that the program has no meaning after the overflow, and everything downstream — including your safety check — is suspect.

**3. Leaks and resource drift (lifetime bug).** Every `malloc` without a matching `free`, every `fopen` without `fclose`, is invisible in a single run. The program prints the right answer and exits. Run it in a loop, ship it as a server, or call the leaking function a million times, and memory grows until the OOM killer intervenes.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static char *make_greeting(const char *name) {
    char *buf = malloc(64);
    if (buf == NULL) {
        return NULL;
    }
    snprintf(buf, 64, "hello, %s", name);
    return buf;   /* caller must free; nothing enforces it */
}

int main(void) {
    for (int i = 0; i < 3; i++) {
        char *g = make_greeting("ada");
        if (g == NULL) {
            return 1;
        }
        printf("%s\n", g);
        /* BUG: free(g) missing. 64 bytes leak per iteration. */
    }
    return 0;
}
```

```text
hello, ada
hello, ada
hello, ada
```

Correct output, zero warnings, 192 bytes leaked. Scale the loop to a daemon handling requests and the "correct" program becomes the outage.

ASCII summary of the failure timeline:

```text
  line with bug          observable symptom           distance
  ─────────────          ──────────────────           ────────
  dst[i] write OOB  ---> crash in unrelated function   FAR (stack smash detected at return)
  a + b overflow    ---> wrong branch 200 lines later  FAR (optimizer assumed no overflow)
  missing free      ---> OOM after 10M requests        VERY FAR (no single-run signal)
  NULL dereference  ---> immediate SEGV                 NEAR (the easy one)
```

The near bugs need no methodology; the far ones do. Testing exists for the far ones.

---

## Test vs Debugging: Two Different Jobs

Beginners use the words interchangeably. They are opposite directions of travel.

- **Debugging** starts from a *known failure* and works backward to the cause. "The program crashed on input X; which line did it?" It is reactive and unrepeatable unless you write something down.
- **Testing** starts from a *claim* and checks it before anything fails. "Function `f` returns `-1` on empty input." It is proactive and repeatable: a small program that encodes the claim and reports pass or fail every time it runs.

| | Debugging | Testing |
| --- | --- | --- |
| Trigger | something already broke | nothing broke yet (or a repro you wrote) |
| Question | "why did this fail?" | "does this claim still hold?" |
| Artifact | a fix, maybe a guess | a program that re-checks the claim forever |
| Reuse | none (the session evaporates) | every future run re-verifies the fix |
| C-specific role | reads sanitizer reports | runs the code *under* the sanitizer |

The workflow that connects them:

```text
  observe failure
       │
       ▼
  write MINIMAL reproducer ──► run under sanitizers ──► read report ──► fix
       │                              │                                  │
       │                              ▼                                  │
       └──────── KEEP the reproducer as a regression test ◄──────────────┘
```

The loop matters more than any single tool. A debugger answers one question once. A test answers its question on every commit from now on. This module teaches the second habit: every bug you meet becomes a test you keep.

A concrete contrast. Debugging the `copy_tag` overflow above means staring at `tag=ABCDEFGH` (correct-looking output!) and guessing. Testing it means writing this *before* you touch the fix:

```c
#include <assert.h>
#include <stdio.h>
#include <string.h>

/* Function under test (fixed version takes capacity seriously). */
static void copy_tag(char *dst, size_t dst_cap, const char *src) {
    size_t len = strlen(src);
    if (len + 1 > dst_cap) {
        if (dst_cap > 0) {
            dst[0] = '\0';
        }
        return;
    }
    for (size_t i = 0; i <= len; i++) {
        dst[i] = src[i];
    }
}

int main(void) {
    char tag[8];
    copy_tag(tag, sizeof(tag), "ABCDEFG");
    assert(strcmp(tag, "ABCDEFG") == 0);
    copy_tag(tag, sizeof(tag), "ABCDEFGH");   /* must NOT overflow */
    assert(strcmp(tag, "") == 0);             /* overflow -> defined empty */
    printf("all tag tests passed\n");
    return 0;
}
```

```text
all tag tests passed
```

The `assert` lines are the test. They pin the contract ("overlong input yields empty, never an overflow") so the next person who "simplifies" the function gets a failure instead of a silent regression.

---

## The Rule: Reproduce First, Fix Second

Never fix a C bug you cannot trigger on demand. The rule is absolute because C bugs are input- and layout-sensitive: the same source passes on your machine and corrupts on the grader's. A minimal reproducer removes that ambiguity.

**A reproducer is a tiny complete program (usually under 30 lines) that triggers the bug deterministically and prints PASS/FAIL or crashes loudly.** Minimal means: one file, `main` included, no user input, no network, no randomness, smallest input that still fails.

Good reproducer anatomy:

```text
  [ ] single .c file, compiles with the course flags
  [ ] #include only what it uses
  [ ] smallest input that triggers the bug (2-3 elements, not 10000)
  [ ] prints expected vs actual, or asserts
  [ ] returns nonzero on failure (so scripts detect it)
  [ ] deterministic: no rand(), no time(), no pointer printing with %p
```

Example: a real off-by-one reproducer for a `sum` function that reads one past the end.

```c
/* repro_sum.c — minimal reproducer for an off-by-one in sum(). */
#include <stdio.h>

static int sum(const int *a, size_t n) {
    int total = 0;
    /* BUG: <= walks to a[n], one past the last element. */
    for (size_t i = 0; i <= n; i++) {
        total += a[i];
    }
    return total;
}

int main(void) {
    const int a[] = { 1, 2, 3, 4 };   /* stack-local on purpose: see below */
    const size_t n = sizeof(a) / sizeof(a[0]);
    const int got = sum(a, n);
    const int want = 10;
    printf("want=%d got=%d %s\n", want, got, got == want ? "PASS" : "FAIL");
    return got == want ? 0 : 1;
}
```

Plain build — passes by accident (the neighbor happened to be zero):

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g repro_sum.c -o repro_sum
./repro_sum; echo "exit=$?"
```

```text
want=10 got=10 PASS
exit=0
```

The PASS is a lie, and that is precisely the lesson. The reproducer is correct; the *build* is too weak to see the bug. Run the identical file under the sanitizer (next section) and the same binary fails loudly. Keep the file either way: after the fix (`i < n`), it passes under both builds, and it becomes the regression test that stops anyone reintroducing `<=`.

Why "reproduce first" is load-bearing in C specifically:

1. **UB is layout-dependent.** `a[n]` reads whatever the linker placed next. Your stack layout and the CI layout differ, so "works on my machine" is the default, not the exception.
2. **The fix for the wrong bug compiles too.** Without a failing test, you can "fix" the symptom (add a `memset`, change `-O2` to `-O0`) while the overflow remains. The reproducer distinguishes the real fix (report disappears) from cosmetics (report persists).
3. **Reviewers cannot see silence.** A diff that changes `<=` to `<` looks trivial. The reproducer attached to it proves the change was necessary and sufficient.

---

## Sanitizers: An Executable Specification

A sanitizer is a compiler mode that inserts runtime checks around every memory access and every arithmetic operation the standard calls undefined. You do not change the source. You change the build flags, and silent bugs become reports with file names, line numbers, and stack traces.

The course build for this entire module:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer file.c -o file_san
./file_san
```

Flag by flag:

| Flag | Job |
| --- | --- |
| `-std=c11` | pin the language version; no GNU extensions hiding bugs |
| `-Wall -Wextra -pedantic` | turn suspicious code into compile-time warnings |
| `-g` | embed line numbers so reports name lines, not addresses |
| `-fsanitize=address` (ASan) | poison redzones around stack/heap/global objects; abort on OOB, UAF, leak |
| `-fsanitize=undefined` (UBSan) | abort on signed overflow, bad shifts, null arithmetic, bad alignment |
| `-fno-omit-frame-pointer` | keep frame pointers so stack traces are complete |

Run every test three ways and compare. The comparison *is* the diagnosis:

| Build | What it catches | What it misses |
| --- | --- | --- |
| plain (`-g` only) | crashes, wrong output | silent OOB that lands in mapped memory, leaks, signed overflow |
| ASan | OOB, use-after-free, double-free, leaks (`LeakSanitizer`) | signed overflow, bad shifts (needs UBSan) |
| ASan + UBSan | both families above | logic bugs (wrong but in-bounds answer) — that is what assertions test |

### Report 1: heap-buffer-overflow (ASan)

```c
/* asan_demo.c */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *a = malloc(4 * sizeof(int));
    if (a == NULL) {
        return 1;
    }
    for (int i = 0; i < 4; i++) {
        a[i] = i * 10;
    }
    printf("a[4] = %d\n", a[4]);   /* BUG: one past the heap block */
    free(a);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer asan_demo.c -o asan_demo
./asan_demo
```

```text
=================================================================
==44123==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x503000000050 at pc 0x55a1c2e0134a bp 0x7ffd9e2a1a10 sp 0x7ffd9e2a1a00
READ of size 4 at 0x503000000050 thread T0
    #0 0x55a1c2e0134a in main /tmp/asan_demo.c:13
    #1 0x7f3b8c029d8f in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #2 0x7f3b8c029e3f in __libc_start_main_impl ../csu/libc-start.c:360
    #3 0x55a1c2e011c4 in _start (/tmp/asan_demo+0x11c4)

0x503000000050 is located 0 bytes to the right of 16-byte region [0x503000000040,0x503000000050)
allocated by thread T0 here:
    #0 0x7f3b8c4b4887 in __interceptor_malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:145
    #1 0x55a1c2e012e3 in main /tmp/asan_demo.c:6

SUMMARY: AddressSanitizer: heap-buffer-overflow /tmp/asan_demo.c:13 in main
```

How to read it: `READ of size 4` at line 13 is the guilty access; `0 bytes to the right of 16-byte region` says the index walked exactly one element past a 4-`int` block allocated at line 6. Fix the index, not the allocator.

### Report 2: signed integer overflow (UBSan)

```c
/* ubsan_demo.c */
#include <stdio.h>
#include <limits.h>

static int bump(int x) {
    return x + 1;   /* BUG when x == INT_MAX */
}

int main(void) {
    printf("%d\n", bump(INT_MAX));
    return 0;
}
```

```text
/tmp/ubsan_demo.c:6:12: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior /tmp/ubsan_demo.c:6:12 in bump
```

One line, exact column, exact values. The fix is a wider or unsigned type plus a range check — never "it wraps fine on x86".

### Report 3: leak (LeakSanitizer, part of ASan)

```c
/* leak_demo.c */
#include <stdlib.h>

int main(void) {
    int *p = malloc(400 * sizeof(int));
    if (p == NULL) {
        return 1;
    }
    p[0] = 42;
    /* BUG: free(p) missing */
    return 0;
}
```

```text
=================================================================
==44210==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 1600 byte(s) in 1 object(s) allocated from:
    #0 0x7f3b8c4b4887 in __interceptor_malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:145
    #1 0x55a1c2e012e3 in main /tmp/leak_demo.c:5

SUMMARY: AddressSanitizer: 1600 byte(s) leaked in 1 allocation(s).
```

`Direct leak` means no pointer to the block survived at exit — the memory is unreachable, not merely "still allocated". Every `Direct leak` is a missing `free` on some path. (Deliberately one-time startup allocations are the only acceptable exception, and even those should be freed in tests.)

Treat the sanitizer pair as the executable half of the specification: the human half says "returns the sum"; the sanitizer half says "while doing so, touches nothing out of bounds, overflows nothing signed, and leaks nothing". A test passes only when *both* halves agree. Silence under `-fsanitize=address,undefined` is the passing state.

---

## Tests as Documentation

A C function signature hides its contract. `int sum(const int *a, size_t n)` does not say whether `a` may be `NULL` when `n` is `0`, who owns the array, or what happens on overflow. Comments can say it, but comments rot. Tests cannot rot silently: when the behavior drifts, they fail.

Write the contract three times, in three places that check each other:

```c
/* CONTRACT (comment): borrows a[0..n); REQUIRES a != NULL or n == 0;
   RETURNS the sum, or -1 on overflow; never writes through a. */
static int checked_sum(const int *a, size_t n);
```

```c
#include <assert.h>
#include <stdio.h>
#include <limits.h>

/* Header above pins ownership; body enforces it. */
static int checked_sum(const int *a, size_t n) {
    long long acc = 0;
    if (a == NULL && n != 0) {
        return -1;
    }
    for (size_t i = 0; i < n; i++) {
        acc += a[i];
        if (acc > INT_MAX || acc < INT_MIN) {
            return -1;
        }
    }
    return (int)acc;
}

int main(void) {
    static const int v1[] = { 1, 2, 3, 4 };   /* happy path */
    static const int empty[] = { 0 };          /* edge: n == 0, pointer unused */
    static const int big[] = { INT_MAX, 1 };   /* failure: overflow */

    assert(checked_sum(v1, 4) == 10);          /* documents the normal answer */
    assert(checked_sum(empty, 0) == 0);        /* documents the empty contract */
    assert(checked_sum(big, 2) == -1);         /* documents the overflow contract */
    assert(checked_sum(NULL, 0) == 0);         /* documents the NULL-tolerant edge */
    printf("checked_sum contract holds\n");
    return 0;
}
```

```text
checked_sum contract holds
```

Each `assert` is a sentence of documentation that the compiler executes. A new reader learns the edge cases by reading `main`, not by hunting callers. Part 02 turns this sketch into a discipline (naming, seams, determinism); the unit-test module turns it into a runner. The principle stays: **if a behavior matters, a test says it; if no test says it, the behavior is undecided.**

---

## Common Bugs

| Bug | Symptom | Fix |
| --- | --- | --- |
| `<=` instead of `<` on `0..n` loop | passes plain, ASan `heap/stack-buffer-overflow` | use `i < n`; capacity-checked copy |
| Signed `int` accumulator for sums | passes small inputs, UBSan `signed integer overflow` | accumulate in `long long`, range-check, return error code |
| Missing `free` on some path | correct output, LeakSanitizer `Direct leak` | one `free` per `malloc`; free before every `return` |
| Fixing without a reproducer | "fixed" on one machine, fails on CI | write `repro_*.c` first; fix only when its report disappears |
| Reading `%p` or uninitialized memory in tests | flaky PASS/FAIL across runs | never assert on addresses or padding; init every variable |
| Sanitizer flags only on some files | bug hides in the uninstrumented file | put `-fsanitize=address,undefined` on *every* compile and link line |

---

## Checklist

```text
[ ] every claim about behavior has a small program checking it
[ ] every bug fix started as a minimal reproducer (single file, tiny input)
[ ] reproducer kept as a regression test after the fix
[ ] all tests run three ways: plain, ASan, ASan+UBSan; results compared
[ ] sanitizer silence treated as half the pass criteria (output correct AND no report)
[ ] reports read precisely: access kind, line, allocation site, overflow values
[ ] contracts written as comment + assert pairs, not comments alone
[ ] helpers are static; read-only inputs are const-qualified
[ ] builds with -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined
```

---

#### [Previous](../../3-advanced/challenge.md) | [Next](./02-part.md)
