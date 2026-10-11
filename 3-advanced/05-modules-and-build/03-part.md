# Build Flags and Project Layout

Parts 1 and 2 got code split and linked. This part makes the build *strict* (warnings become errors, bugs are caught at runtime by sanitizers) and the project *navigable* (every file has a home directory).

---

## The Strict Flag Set

Start every compile with this baseline:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c vec.c -o app
```

| Flag | What it turns on | Example it catches |
| --- | --- | --- |
| `-std=c11` | compile as standard C11 (no GNU extensions by default) | nested functions, `typeof` |
| `-Wall` | the important warning group (despite the name, not "all") | unused variable, missing `return` |
| `-Wextra` | more warnings: sign compare, unused parameter, missing initializers | `if (n < sizeof(buf))` signed/unsigned mix |
| `-pedantic` | strict ISO C: reject extensions | `//` is fine, but `__attribute__` warns |
| `-g` | debug info for `gdb` and sanitizer line numbers | without it, crashes print addresses only |

Add these two when the code is warning-clean:

| Flag | Effect | When |
| --- | --- | --- |
| `-Werror` | every warning becomes a hard error | CI, exercises, before every commit |
| `-O2` | optimize (also enables warnings that need dataflow analysis, like `-Wmaybe-uninitialized`) | release builds, and any `-Werror` run |

Typical progression:

```bash
# 1. develop: full warnings + debug info, no optimization (fast compile, debuggable)
gcc -std=c11 -Wall -Wextra -pedantic -g main.c vec.c -o app -lm

# 2. harden: warnings are errors
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g main.c vec.c -o app -lm

# 3. release: optimize, keep debug info (harmless, helps crash reports)
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -O2 main.c vec.c -o app -lm
```

What `-Werror` feels like:

```c
/* unused.c */
#include <stdio.h>

int main(void) {
    int total = 0;
    int unused_counter = 42;
    printf("total = %d\n", total);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g unused.c -o unused
```

```text
unused.c: In function ‘main’:
unused.c:5:9: warning: unused variable ‘unused_counter’ [-Wunused-variable]
    5 |     int unused_counter = 42;
      |         ^~~~~~~~~~~~~~
```

Compiles (exit 0) with a warning. Now with `-Werror`:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g unused.c -o unused
```

```text
unused.c: In function ‘main’:
unused.c:5:9: error: unused variable ‘unused_counter’ [-Werror=unused-variable]
    5 |     int unused_counter = 42;
      |         ^~~~~~~~~~~~~~
cc1: all warnings being treated as errors
```

No executable is produced. That is the point: an unused variable is often a typo (`total` vs `totla`) or a forgotten computation. Fix the code, do not remove the flag. (Temporarily silence a *specific* known-harmless warning with `-Wno-unused-parameter` if you must — never by dropping `-Wall`.)

`-O2` deserves a warning of its own: it makes the compiler smarter, which makes some warnings *appear* only at `-O2`. Always do your final `-Werror` check with the same `-O` level you ship.

---

## Sanitizers: Bugs That Warnings Cannot See

Warnings catch suspicious code at compile time. Sanitizers catch wrong behavior at runtime by instrumenting the executable:

| Sanitizer | Flag | Catches |
| --- | --- | --- |
| AddressSanitizer (ASan) | `-fsanitize=address` | out-of-bounds read/write, use-after-free, double free, leaks (`LeakSanitizer` included on Linux) |
| UndefinedBehaviorSanitizer (UBSan) | `-fsanitize=undefined` | signed overflow, bad shift, misaligned / null dereference, out-of-bounds on known sizes |

Both together, plus debug info and no optimization (so line numbers match your source):

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -fsanitize=address,undefined buggy.c -o buggy
./buggy
```

Example — heap overflow:

```c
/* buggy.c */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *a = malloc(3 * sizeof *a);
    if (a == NULL) {
        return 1;
    }

    a[0] = 10;
    a[1] = 20;
    a[2] = 30;
    a[3] = 40;   /* BUG: one past the end */

    printf("%d %d %d\n", a[0], a[1], a[2]);
    free(a);
    return 0;
}
```

```text
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x60400000001c at pc 0x55... bp 0x7ff... sp 0x7ff...
WRITE of size 4 at 0x60400000001c thread T0
    #0 0x55... in main buggy.c:13
    #1 0x7f... in __libc_start_main
    ...
0x60400000001c is located 0 bytes to the right of 12-byte region [0x604000000010,0x60400000001c)
allocated by thread T0 here:
    #0 0x55... in malloc ...
    #1 0x55... in main buggy.c:5

SUMMARY: AddressSanitizer: heap-buffer-overflow buggy.c:13 in main
```

How to read it:

1. `heap-buffer-overflow` — the bug class (write past a `malloc` region).
2. `buggy.c:13 in main` — the exact line (`a[3] = 40;`). `-g` made this possible.
3. `12-byte region` — the `malloc(3 * 4)` block; the write landed `0 bytes to the right` of it.
4. Fix: allocate 4 elements, or loop with `i < 3`.

Example — signed overflow (UBSan):

```c
/* overflow.c */
#include <stdio.h>
#include <limits.h>

int main(void) {
    int x = INT_MAX;
    x = x + 1;   /* BUG: signed overflow is undefined behaviour */
    printf("x = %d\n", x);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=undefined overflow.c -o overflow
./overflow
```

```text
overflow.c:7:9: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
x = -2147483648
```

Without UBSan this program prints `-2147483648` silently and you might conclude "it wraps, fine". It does wrap *on this compiler today* — the standard permits anything, including deleting the whole branch. UBSan turns the silent assumption into a loud line-numbered report.

Sanitizer rules:

1. Compile **and** link with the same `-fsanitize` flags (put them on the final link line too; `Makefile` `CFLAGS` handles this automatically if you link with `$(CC) $(CFLAGS)`).
2. Keep `-g` so reports name files and lines.
3. Use `-O1` or no `-O` for sanitizer runs; `-O2` is fine but can inline away line precision.
4. Never ship sanitizer builds to users — 2x slowdown and extra memory. They are for tests and exercises.

---

## `-I` and `-D`: Include Paths and Definitions

### `-I`: where are my headers?

Once headers move out of the current directory, the compiler needs directions:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -Iinclude -c src/main.c -o build/main.o
```

`-Iinclude` means "also search `include/` for `#include`". The lookup order for quoted includes:

```text
#include "phonebook.h" searched in:
  1. directory of the including file (src/phonebook.c's own dir)
  2. every -I directory, left to right (-Iinclude -Ivendor/xyz)
  3. system directories (/usr/include)
```

Multiple `-I` flags stack: `-Iinclude -Itests/helpers`. Keep them in the `Makefile` so every compile uses the same search path.

### `-D`: compile-time switches

`-DNAME` defines a preprocessor macro, as if `#define NAME 1` sat at the top of every file. `-DNAME=value` sets a value.

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -DLOG_LEVEL=2 main.c vec.c -o app
```

```c
#ifndef LOG_LEVEL
#define LOG_LEVEL 1
#endif

#if LOG_LEVEL >= 2
#define LOG_DEBUG(fmt, ...) printf("[debug] " fmt "\n", ##__VA_ARGS__)
#else
#define LOG_DEBUG(fmt, ...) ((void)0)
#endif
```

The most important `-D` is `NDEBUG`: it disables `assert`:

```c
/* demo_assert.c */
#include <assert.h>
#include <stdio.h>

static int divide(int a, int b) {
    assert(b != 0);   /* contract: caller never passes 0 */
    return a / b;
}

int main(void) {
    printf("%d\n", divide(10, 2));
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g demo_assert.c -o demo_debug && ./demo_debug
gcc -std=c11 -Wall -Wextra -pedantic -g -O2 -DNDEBUG demo_assert.c -o demo_release && ./demo_release
```

```text
10
10
```

Same output — because the assertion held. The difference is invisible until a contract breaks: the debug build aborts with `Assertion 'b != 0' failed`, the `-DNDEBUG` build divides by zero (undefined behaviour, possible crash with no message).

| Build | Flags | `assert` | Use |
| --- | --- | --- | --- |
| Debug / dev | `-g`, no `-DNDEBUG` | active, aborts on violation | development, tests |
| Release | `-O2 -DNDEBUG` | compiled out, zero cost | shipped binary |

Convention: `Makefile` debug target keeps asserts; `release`/`CFLAGS_RELEASE` adds `-O2 -DNDEBUG`. Never pass `-DNDEBUG` while developing — asserts are the cheapest tests you own.

---

## Project Layout: `include/` `src/` `tests/`

Flat directories break at ~10 files. The standard small-C layout:

```text
phonebook/
  include/
    phonebook.h      public API (guards, structs, prototypes, externs)
  src/
    phonebook.c      definitions (includes "phonebook.h")
    main.c           the application (includes "phonebook.h")
  tests/
    test_phonebook.c minimal test driver (includes "phonebook.h")
  build/             created by make, never committed (.gitignore it)
  Makefile
```

| Directory | Contains | Installed? |
| --- | --- | --- |
| `include/` | public `.h` files | yes (`make install` copies these) |
| `src/` | `.c` files + private headers (`*_internal.h`, not installed) | no |
| `tests/` | test drivers, expected outputs | no |
| `build/` | `.o`, `.a`, executables | no, generated |

Rules:

1. Public headers include only what they need (`<stddef.h>` for `size_t`, never `<stdio.h>` for a struct-only header).
2. Every `src/*.c` includes its own `include/*.h` **first**, so a missing prototype is caught immediately.
3. Private helpers are `static` in `src/*.c` or in a `src/*_internal.h` included only by sources that need them — never in `include/`.
4. `build/` is in `.gitignore`. Only sources are committed.

---

## Full Example: Phonebook in Three Files

```c
/* include/phonebook.h */
#ifndef PHONEBOOK_H
#define PHONEBOOK_H

#include <stddef.h>

#define PHONEBOOK_MAX 100
#define NAME_LEN 32
#define PHONE_LEN 16

typedef struct {
    char name[NAME_LEN];
    char phone[PHONE_LEN];
} Contact;

typedef struct {
    Contact items[PHONEBOOK_MAX];
    size_t count;
} Phonebook;

void phonebook_init(Phonebook *book);
int phonebook_add(Phonebook *book, const char *name, const char *phone);
int phonebook_find(const Phonebook *book, const char *name);
void phonebook_list(const Phonebook *book);

#endif /* PHONEBOOK_H */
```

```c
/* src/phonebook.c */
#include <stdio.h>
#include <string.h>

#include "phonebook.h"

static void copy_fixed(char *dst, size_t dst_size, const char *src) {
    if (dst_size == 0) {
        return;
    }
    strncpy(dst, src, dst_size - 1);
    dst[dst_size - 1] = '\0';
}

void phonebook_init(Phonebook *book) {
    book->count = 0;
}

int phonebook_add(Phonebook *book, const char *name, const char *phone) {
    if (book->count >= PHONEBOOK_MAX) {
        return -1;
    }
    copy_fixed(book->items[book->count].name, NAME_LEN, name);
    copy_fixed(book->items[book->count].phone, PHONE_LEN, phone);
    book->count++;
    return 0;
}

int phonebook_find(const Phonebook *book, const char *name) {
    for (size_t i = 0; i < book->count; i++) {
        if (strcmp(book->items[i].name, name) == 0) {
            return (int)i;
        }
    }
    return -1;
}

void phonebook_list(const Phonebook *book) {
    for (size_t i = 0; i < book->count; i++) {
        printf("%-10s %s\n", book->items[i].name, book->items[i].phone);
    }
}
```

```c
/* src/main.c */
#include <stdio.h>

#include "phonebook.h"

int main(void) {
    Phonebook book;

    phonebook_init(&book);
    phonebook_add(&book, "Ada", "555-0101");
    phonebook_add(&book, "Grace", "555-0102");

    phonebook_list(&book);
    printf("find Grace: %d\n", phonebook_find(&book, "Grace"));
    printf("find Alan: %d\n", phonebook_find(&book, "Alan"));

    return 0;
}
```

Complete `Makefile`:

```make
CC = gcc
CFLAGS = -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude
LDLIBS =

BUILD = build
TARGET = $(BUILD)/phonebook
OBJS = $(BUILD)/main.o $(BUILD)/phonebook.o

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $(OBJS) $(LDLIBS) -o $(TARGET)

$(BUILD)/main.o: src/main.c include/phonebook.h
	mkdir -p $(BUILD)
	$(CC) $(CFLAGS) -c src/main.c -o $(BUILD)/main.o

$(BUILD)/phonebook.o: src/phonebook.c include/phonebook.h
	mkdir -p $(BUILD)
	$(CC) $(CFLAGS) -c src/phonebook.c -o $(BUILD)/phonebook.o

sanitize: CFLAGS += -fsanitize=address,undefined
sanitize: clean $(TARGET)

clean:
	rm -rf $(BUILD)

.PHONY: all sanitize clean
```

Build and run:

```bash
make
./build/phonebook
make clean
make sanitize && ./build/phonebook
```

```text
Ada        555-0101
Grace      555-0102
find Grace: 1
find Alan: -1
```

Same output from all three builds. The `sanitize` target appends the flags and rebuilds from clean so every object is instrumented (mixing instrumented and plain objects misses bugs). `mkdir -p $(BUILD)` keeps object files out of `src/`.

Why `copy_fixed` is `static`: truncation policy is an implementation detail. Callers see fixed-size fields and a guarantee (always NUL-terminated); how that is achieved can change without touching the header. `phonebook_find` returns `int` (not `size_t`) precisely so `-1` means "not found" — the classic sentinel the header documents by convention.

---

## Bugs You Will Meet

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| 1 | warning treated as error under `-Werror` | real defect (unused var, sign compare) | fix the code, keep `-Werror` |
| 2 | new warnings appear only at `-O2` | dataflow warnings need optimization | always verify `-Werror` at ship `-O` level |
| 3 | sanitizer report with no line numbers | missing `-g` | add `-g` to `CFLAGS` |
| 4 | sanitizer misses a bug | mixed instrumented/plain objects, or forgot flag on link line | `make clean`, keep flags in `CFLAGS`, link with `$(CC) $(CFLAGS)` |
| 5 | `phonebook.h: No such file` | forgot `-Iinclude` | add `-Iinclude` to `CFLAGS` |
| 6 | `assert` fires in dev, silent crash in release | `-DNDEBUG` compiled asserts out (correct) | fix the contract violation; never develop with `-DNDEBUG` |
| 7 | struct layout crash after header edit | stale `build/*.o` | `make clean && make` |

---

## Checklist

1. Does the baseline build use `-std=c11 -Wall -Wextra -pedantic -g` with zero warnings?
2. Is `-Werror` on for exercises and CI, verified at the same `-O` level you ship?
3. Do sanitizer runs use `-fsanitize=address,undefined` on both compile and link, with `-g`?
4. Is `-Iinclude` (not a copied header) how sources find public headers?
5. Are configurable values `-D` flags with sane `#ifndef` defaults in code?
6. Do debug builds keep `assert` (no `-DNDEBUG`) and release builds add `-O2 -DNDEBUG`?
7. Does the layout separate `include/` (public), `src/` (private), `tests/`, and generated `build/` (git-ignored)?

---

#### [Previous](./02-part.md) | [Next](./exercises.md)
