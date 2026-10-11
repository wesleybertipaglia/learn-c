# Exercise List 5 - Modules and Build

Six exercises on headers, static linkage, archives, Makefiles, strict flags with sanitizers, and project layout. Work inside an empty directory per exercise so stale `.o` files cannot hide mistakes.

Base compile line (strict from the start):

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g main.c vec.c -o app
```

Keep `-Werror` on for the whole list. If a warning appears, fix the code — never drop the flag to make it compile.

---

## Exercise 1 — Split a Program Into Two `.c` Files Plus a Header

Take this single-file program and split it into `vec.h`, `vec.c`, and `main.c` without changing behavior.

Starting point (`single.c`):

```c
/* single.c — everything in one file */
#include <stdio.h>

typedef struct {
    double x;
    double y;
} Vec2;

Vec2 vec_make(double x, double y) {
    Vec2 v = { x, y };
    return v;
}

Vec2 vec_add(Vec2 a, Vec2 b) {
    Vec2 r = { a.x + b.x, a.y + b.y };
    return r;
}

int main(void) {
    Vec2 a = vec_make(1.0, 2.0);
    Vec2 b = vec_make(3.0, 4.0);
    Vec2 s = vec_add(a, b);
    printf("sum = (%.1f, %.1f)\n", s.x, s.y);
    return 0;
}
```

Target layout:

```c
/* vec.h */
#ifndef VEC_H
#define VEC_H

typedef struct {
    double x;
    double y;
} Vec2;

Vec2 vec_make(double x, double y);
Vec2 vec_add(Vec2 a, Vec2 b);

#endif /* VEC_H */
```

```c
/* vec.c */
#include "vec.h"

Vec2 vec_make(double x, double y) {
    Vec2 v = { x, y };
    return v;
}

Vec2 vec_add(Vec2 a, Vec2 b) {
    Vec2 r = { a.x + b.x, a.y + b.y };
    return r;
}
```

```c
/* main.c */
#include <stdio.h>

#include "vec.h"

int main(void) {
    Vec2 a = vec_make(1.0, 2.0);
    Vec2 b = vec_make(3.0, 4.0);
    Vec2 s = vec_add(a, b);
    printf("sum = (%.1f, %.1f)\n", s.x, s.y);
    return 0;
}
```

Build and run:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g main.c vec.c -o app
./app
```

```text
sum = (4.0, 6.0)
```

Also verify the split-object path gives the same binary behavior:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -c main.c -o main.o
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -c vec.c -o vec.o
gcc main.o vec.o -o app2
./app2
```

```text
sum = (4.0, 6.0)
```

### Requirements

1. `vec.h` uses `#ifndef VEC_H` / `#define VEC_H` / `#endif` guards.
2. `vec.h` contains only the `struct` definition, `typedef`, and prototypes — no function bodies, no variable definitions.
3. `vec.c` includes `"vec.h"` first (quoted include, not `<vec.h>`).
4. Both build lines compile with zero warnings under `-Werror`.
5. Deliberately break it once: paste a function body into `vec.h`, rebuild with both `.c` files, and confirm you get `multiple definition`. Then revert.

---

## Exercise 2 — Make a Helper `static` and Prove It Vanishes

Add a private helper to `vec.c`, mark it `static`, and prove no other file can link against it.

`vec.h` (add one public function that uses the helper internally):

```c
/* vec.h */
#ifndef VEC_H
#define VEC_H

typedef struct {
    double x;
    double y;
} Vec2;

Vec2 vec_make(double x, double y);
Vec2 vec_add(Vec2 a, Vec2 b);
Vec2 vec_scale(Vec2 v, double k);

#endif /* VEC_H */
```

`vec.c`:

```c
/* vec.c */
#include "vec.h"

static double mul(double x, double k) {
    return x * k;
}

Vec2 vec_make(double x, double y) {
    Vec2 v = { x, y };
    return v;
}

Vec2 vec_add(Vec2 a, Vec2 b) {
    Vec2 r = { a.x + b.x, a.y + b.y };
    return r;
}

Vec2 vec_scale(Vec2 v, double k) {
    Vec2 r = { mul(v.x, k), mul(v.y, k) };
    return r;
}
```

`main.c`:

```c
/* main.c */
#include <stdio.h>

#include "vec.h"

int main(void) {
    Vec2 v = vec_make(2.0, 3.0);
    Vec2 s = vec_scale(v, 2.0);
    printf("scaled = (%.1f, %.1f)\n", s.x, s.y);
    return 0;
}
```

Normal build works:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g main.c vec.c -o app
./app
```

```text
scaled = (4.0, 6.0)
```

Now prove `mul` is file-private. Method A — inspect symbols:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -c vec.c -o vec.o
nm vec.o | grep -E "mul|vec_scale"
```

```text
0000000000000000 t mul
0000000000000050 T vec_scale
```

Lowercase `t` = local symbol (static, invisible to the linker). Uppercase `T` = global symbol (linkable). That one letter is the whole difference.

Method B — try to call it from another file (`cheat.c`):

```c
/* cheat.c — tries to reach vec.c's private helper */
#include <stdio.h>

double mul(double x, double k);

int main(void) {
    printf("%f\n", mul(2.0, 3.0));
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g cheat.c vec.o -o cheat
```

```text
/usr/bin/x86_64-linux-gnu-ld.bfd: cheat.o: in function `main':
cheat.c:(.text+0x1c): undefined reference to `mul'
collect2: error: ld returned 1 exit status
```

The compiler accepted the prototype, but the linker found no global `mul` — `static` kept it inside `vec.o`.

### Requirements

1. `mul` is `static` in `vec.c` and is not declared in `vec.h`.
2. Paste the `nm` output showing `t mul` vs `T vec_scale`.
3. Paste the `undefined reference to 'mul'` link error from `cheat.c`.
4. Comment in `vec.c` (one line above `mul`) why it is `static`: private helper, avoids collisions, not part of the public API.
5. Remove `static` once, rebuild `cheat.c`, and confirm it links — then put `static` back.

---

## Exercise 3 — Create a Static Library and Link It

Bundle `vec.o` into `libvec.a` with `ar`, then link `main.c` against it two ways (direct object vs `-L`/`-l`).

Reuse the three files from Exercise 1 (`vec.h`, `vec.c`, `main.c` with `vec_make` + `vec_add`).

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -c vec.c -o vec.o
ar rcs libvec.a vec.o
ar t libvec.a
```

```text
vec.o
```

```bash
nm libvec.a | grep " T "
```

```text
0000000000000056 T vec_make
0000000000000025 T vec_add
```

Link method A — objects directly:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g main.c vec.o -o app_direct
./app_direct
```

```text
sum = (4.0, 6.0)
```

Link method B — library flags:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g main.c -L. -lvec -o app_lib
./app_lib
```

```text
sum = (4.0, 6.0)
```

Now demonstrate link order. This fails on GNU `ld`:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -L. -lvec main.c -o app_wrong
```

```text
/usr/bin/x86_64-linux-gnu-ld.bfd: /tmp/ccXXXXXX.o: in function `main':
main.c:(.text+0x2b): undefined reference to `vec_add'
main.c:(.text+0x52): undefined reference to `vec_make'
collect2: error: ld returned 1 exit status
```

Same program, same library — only the order changed. The linker scans left to right and discards `libvec.a` before `main.c` has told it what is missing.

### Requirements

1. `libvec.a` built with exactly `ar rcs libvec.a vec.o`.
2. `ar t` output pasted (must list `vec.o`).
3. Both `app_direct` and `app_lib` run and print identical output.
4. The wrong-order command and its `undefined reference` output pasted, with a one-paragraph comment explaining why order matters.
5. Library file named `libvec.a` (not `vec.a`) so `-lvec` resolves.

---

## Exercise 4 — Write a Makefile With `clean` and Rebuild

Write a minimal `Makefile` for `main.c` + `vec.c` + `vec.h` that builds `app`, rebuilds incrementally, and cleans.

```make
CC = gcc
CFLAGS = -std=c11 -Wall -Wextra -Werror -pedantic -g
LDLIBS =

app: main.o vec.o
	$(CC) $(CFLAGS) main.o vec.o $(LDLIBS) -o app

main.o: main.c vec.h
	$(CC) $(CFLAGS) -c main.c -o main.o

vec.o: vec.c vec.h
	$(CC) $(CFLAGS) -c vec.c -o vec.o

clean:
	rm -f *.o app

.PHONY: clean
```

(Every recipe line starts with a TAB. Spaces give `missing separator. Stop.`)

Full session:

```bash
make clean
make
./app
touch vec.c
make
make clean
ls
```

```text
rm -f *.o app
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -c main.c -o main.o
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -c vec.c -o vec.o
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g main.o vec.o  -o app
sum = (4.0, 6.0)
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -c vec.c -o vec.o
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g main.o vec.o  -o app
rm -f *.o app
main.c  Makefile  vec.c  vec.h
```

Notice the second `make` (after `touch vec.c`) recompiled only `vec.o` and relinked — `main.o` was untouched. That is incremental rebuild: `make` compared timestamps and rebuilt the smallest chain possible.

Also verify headers are prerequisites: `touch vec.h && make` must rebuild *both* `.o` files, because both list `vec.h` after the colon.

### Requirements

1. Variables named exactly `CC`, `CFLAGS`, `LDLIBS`.
2. `vec.h` listed as a prerequisite of both `main.o` and `vec.o`.
3. `clean` removes `*.o` and `app`; `.PHONY: clean` present.
4. Transcript pasted showing: fresh `make`, `touch vec.c` + `make` rebuilding only `vec.o` + relink, `make clean` leaving only sources.
5. `make` with zero warnings under `-Werror`.

---

## Exercise 5 — Strict Build With Sanitizers and a Pasted Diagnostic

Compile a buggy program with `-Werror` plus AddressSanitizer/UBSan, paste the diagnostic, fix the bug, and show the clean rerun.

Buggy program (`buggy.c`):

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

Build with sanitizers (same flags on compile and link):

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -fsanitize=address,undefined buggy.c -o buggy
./buggy
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

(PIDs and addresses differ per run; the bug class, file, and line do not.)

Fix (allocate what you use, loop with `< 3`):

```c
/* fixed.c */
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

    printf("%d %d %d\n", a[0], a[1], a[2]);
    free(a);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -fsanitize=address,undefined fixed.c -o fixed
./fixed
```

```text
10 20 30
```

Silent, exit code 0, no sanitizer report. Also confirm `-Werror` discipline with an unused-variable probe:

```bash
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -fsanitize=address,undefined -c fixed.c -o /dev/null
echo "exit: $?"
```

```text
exit: 0
```

### Requirements

1. Full sanitizer build line uses `-fsanitize=address,undefined` together with `-Werror` and `-g`.
2. Complete ASan `heap-buffer-overflow` report pasted, naming `buggy.c:13`.
3. One paragraph explaining the report in your own words: which line, which allocation, why `0 bytes to the right of 12-byte region` pinpoints `a[3]`.
4. Fixed program rerun pasted (`10 20 30`, no report, exit 0).
5. Never ship the sanitizer binary; note in a comment why (2x slowdown, extra memory, test-only).

---

## Exercise 6 — Organize a Mini-Project in `include/` and `src/` With `-I`

Reorganize the phonebook program into the standard layout and build it with `-Iinclude`.

Layout to create:

```text
phonebook/
  include/phonebook.h
  src/phonebook.c
  src/main.c
  Makefile
```

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

#endif /* PHONEBOOK_H */
```

```c
/* src/phonebook.c */
#include <string.h>

#include "phonebook.h"

void phonebook_init(Phonebook *book) {
    book->count = 0;
}

int phonebook_add(Phonebook *book, const char *name, const char *phone) {
    if (book->count >= PHONEBOOK_MAX) {
        return -1;
    }
    strncpy(book->items[book->count].name, name, NAME_LEN - 1);
    book->items[book->count].name[NAME_LEN - 1] = '\0';
    strncpy(book->items[book->count].phone, phone, PHONE_LEN - 1);
    book->items[book->count].phone[PHONE_LEN - 1] = '\0';
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

    printf("count = %zu\n", book.count);
    printf("find Grace: %d\n", phonebook_find(&book, "Grace"));
    printf("find Alan: %d\n", phonebook_find(&book, "Alan"));

    return 0;
}
```

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

clean:
	rm -rf $(BUILD)

.PHONY: all clean
```

Build and run:

```bash
make
./build/phonebook
```

```text
count = 2
find Grace: 1
find Alan: -1
```

Prove `-I` is load-bearing: remove `-Iinclude` from `CFLAGS`, run `make clean && make`, and confirm `fatal error: phonebook.h: No such file or directory`. Then restore it.

```text
src/main.c:3:10: fatal error: phonebook.h: No such file or directory
    3 | #include "phonebook.h"
      |          ^~~~~~~~~~~~~
compilation terminated.
```

### Requirements

1. Exact directory layout (`include/phonebook.h`, `src/*.c`, `Makefile`, generated `build/`).
2. `CFLAGS` contains `-Iinclude`; both `.c` files use `#include "phonebook.h"` (quoted).
3. Header guards `#ifndef PHONEBOOK_H`; no function bodies or variable definitions in the header.
4. `make` builds with zero warnings; program output pasted.
5. Failed build without `-Iinclude` pasted, with a one-line explanation of the search path.
6. `build/` is git-ignored (show the `.gitignore` line).

---

## Checklist Before You Run

| Check | Why |
| --- | --- |
| Every header has `#ifndef X_H` guards | double inclusion is otherwise a redefinition error |
| No non-`static` function or variable defined in any header | duplicates cause `multiple definition` at link |
| Helpers are `static` unless a second file needs them | proves with `nm` (`t` vs `T`) and link errors |
| Archive built with `ar rcs`, linked with `-L`/`-l` after sources | wrong name or order gives `cannot find -l` / `undefined reference` |
| `Makefile` uses `CC`/`CFLAGS`, TAB recipes, `.PHONY: clean` | spaces break make, missing `.PHONY` breaks `clean` |
| Strict flags `-Werror` + `-fsanitize=address,undefined` + `-g` on exercise builds | warnings are defects, sanitizers prove runtime safety |
| Layout is `include/` + `src/` + `build/` with `-Iinclude` | flat dirs and copied headers do not scale |

---

#### [Previous](./03-part.md) | [Next](../challenge.md)
