# Headers and Multiple Files

A single `.c` file works until your program passes a few hundred lines. Then every change recompiles everything, every helper is visible to every function, and two programmers cannot work on the same file without conflicts.

The fix is to split the program into modules: a header (`.h`) that says *what exists*, and source files (`.c`) that say *how it works* (`vec.h` declares `Vec2`/`vec_add`, `vec.c` defines them, `main.c` uses them).

The compiler checks each `.c` file on its own. The linker then joins the results into one executable. This part explains the contract between them.

---

## Declarations vs Definitions

C draws a hard line between saying something exists and creating it:

| Term | Meaning | Example | How many times allowed |
| --- | --- | --- | --- |
| **Declaration** | "this exists somewhere, here is its type" | `int vec_len(Vec2 v);` | many (in every file that uses it) |
| **Definition** | "here it is, reserve storage / emit code" | `int vec_len(Vec2 v) { ... }` | exactly once in the whole program |
| **Variable declaration** | type plus `extern`, no storage | `extern int log_level;` | many |
| **Variable definition** | storage is reserved | `int log_level = 1;` | exactly once |

A function prototype is a declaration. A function body is a definition. A `struct` definition is a special case: it creates no storage by itself, so it can (and must) appear in every file that uses the struct.

```c
/* declaration: no body, ends with ; */
typedef struct {
    double x;
    double y;
} Vec2;

double vec_len(Vec2 v);

/* definition: has a body, no ; after the signature */
double vec_len(Vec2 v) {
    return v.x * v.x + v.y * v.y;  /* squared length, sqrt later */
}
```

Rule of thumb: headers contain declarations. Source files contain definitions. Every time you break that rule, the linker will tell you — loudly.

---

## What a Header Contains

A header is a shopping list pasted into each `.c` file by `#include`. It must be safe to paste many times. That means only things that *can* be repeated:

| Belongs in `.h` | Does NOT belong in `.h` |
| --- | --- |
| Function prototypes | Function bodies (non-`static inline`) |
| `struct` / `union` / `enum` definitions | Non-`extern` variable definitions |
| `typedef` names | `int counter = 0;` at file scope |
| `#define` constants | Executable statements |
| `extern` variable declarations | `static` variable definitions shared across files |
| Include guards | `#include <stdio.h>` you do not need (only include what the header itself uses) |

Correct header:

```c
/* vec.h — public interface of the vec module */
#ifndef VEC_H
#define VEC_H

typedef struct {
    double x;
    double y;
} Vec2;

Vec2 vec_make(double x, double y);
Vec2 vec_add(Vec2 a, Vec2 b);
double vec_len(Vec2 v);

#endif /* VEC_H */
```

Wrong header (do not do this):

```c
/* BAD: never ship this */
#ifndef VEC_H
#define VEC_H

typedef struct {
    double x;
    double y;
} Vec2;

Vec2 vec_add(Vec2 a, Vec2 b) {  /* BAD: definition in header */
    Vec2 r = { a.x + b.x, a.y + b.y };
    return r;
}

int call_count = 0;             /* BAD: definition in header */

#endif
```

Why the second one is broken is the subject of the "multiple definition" section below. For now, memorize: **never define a non-`static` function or variable in a header.**

---

## Include Guards

`#include` is literal copy-paste done by the preprocessor. If `main.c` includes `vec.h` twice — directly and indirectly through another header — the compiler sees every declaration twice, and `struct` redefinitions are an error.

Include guards make the second paste a no-op:

```c
#ifndef VEC_H      /* "if not yet defined..." */
#define VEC_H      /* "...mark it defined, and read the body..." */

/* ... header body ... */

#endif /* VEC_H */ /* "...otherwise skip to here." */
```

The macro name must be unique across the whole project: file name in upper case plus `_H` (`vec.h` -> `VEC_H`, `phonebook.h` -> `PHONEBOOK_H`).

A second guard style exists:

```c
#pragma once
```

It does the same job in one line and almost every compiler supports it, but it is not standard C11. This course uses `#ifndef` guards everywhere because they work on every compiler, including strict `-pedantic` builds. You may see `#pragma once` in other projects; know what it does, but write guards.

What happens without guards:

```c
/* vec.h WITHOUT guards */
typedef struct { double x; double y; } Vec2;
```

```c
/* main.c */
#include "vec.h"
#include "vec.h"   /* pulled in twice, e.g. via another header */

int main(void) {
    Vec2 v = { 1.0, 2.0 };
    (void)v;
    return 0;
}
```

```text
vec.h:1:16: error: redefinition of ‘struct <anonymous>’
    1 | typedef struct { double x; double y; } Vec2;
      |                ^
```

With guards, the same program compiles cleanly. Guards cost three lines and prevent an entire class of errors. Always write them.

---

## `extern` Globals: How They Work, Why You Avoid Them

Sometimes a module needs shared state — a log level, a configuration flag. C spells that with `extern`:

```c
/* config.h */
#ifndef CONFIG_H
#define CONFIG_H

extern int log_level;   /* declaration: "an int called log_level exists" */

void log_set_level(int level);
int log_get_level(void);

#endif /* CONFIG_H */
```

```c
/* config.c */
#include "config.h"

int log_level = 1;      /* definition: storage lives HERE, exactly once */

void log_set_level(int level) {
    log_level = level;
}

int log_get_level(void) {
    return log_level;
}
```

```c
/* main.c */
#include <stdio.h>

#include "config.h"

int main(void) {
    printf("level = %d\n", log_level);
    log_set_level(3);
    printf("level = %d\n", log_get_level());
    return 0;
}
```

```text
level = 1
level = 3
```

The pattern is: `extern` in the header, plain definition in exactly one `.c` file. A plain definition in the header creates one copy per including `.c` file — a multiple-definition error (see below).

But prefer to avoid globals entirely: use a getter/setter pair instead of a writable `extern int`, pass a `const Config *` instead of an `extern Config`, and return error codes instead of incrementing an `extern int error_count`. Globals cannot be reasoned about locally — any function may have changed them — and one test's leftover value leaks into the next. If you must share state, hide it as `static` inside its `.c` file with accessor functions. Use `extern` only when an existing codebase forces you to.

---

## `static`: Private to This File

`static` on a file-scope function or variable means *internal linkage*: visible only inside this translation unit (this `.c` file plus everything it included).

```c
/* vec.c */
#include <math.h>

#include "vec.h"

/* private helper: nobody outside vec.c can call or even see this */
static double square(double x) {
    return x * x;
}

Vec2 vec_make(double x, double y) {
    Vec2 v = { x, y };
    return v;
}

Vec2 vec_add(Vec2 a, Vec2 b) {
    Vec2 r = { a.x + b.x, a.y + b.y };
    return r;
}

double vec_len(Vec2 v) {
    return sqrt(square(v.x) + square(v.y));
}
```

Benefits: no name collisions (two files can each own a `static square`), a smaller public surface in the header, and easier inlining (all callers are local).

Contrast with the default (external linkage): a non-`static` function is visible to the whole program. That is correct for `vec_add` (public API) and wrong for `square` (implementation detail). Habit: write every helper `static` first; remove it only when a second file genuinely needs the function, declaring it in the header at the same moment.

Proof that `static` hides the symbol (exercise 2 builds on this):

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -c vec.c -o vec.o
nm vec.o | grep square
```

```text
0000000000000000 t square
```

Lowercase `t` means a local (static) text symbol. A non-`static` function shows uppercase `T` — globally visible. If another file tries to call `square`, compilation succeeds (with its own prototype) but linking fails with `undefined reference`.

---

## The Compile + Link Cycle

When you run:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c vec.c -o app
```

`gcc` runs three stages behind one command:

```text
  main.c                  vec.c
    |                       |
    v                       v
 preprocessor (cpp)    preprocessor (cpp)
 expands #include,       expands #include,
 #define                 #define
    |                       |
    v                       v
 compiler (cc1)        compiler (cc1)
 checks types,           checks types,
 emits assembly         emits assembly
    |                       |
    v                       v
 assembler (as)        assembler (as)
  main.o                  vec.o        <-- object files (machine code,
    \                                     unresolved calls like vec_add)
     \                   /
      v                 v
      linker (ld)
      resolves vec_add in main.o against vec.o,
      adds libc (printf, sqrt needs -lm),
      writes executable ./app
```

You can stop after the compile stage with `-c` (compile only, no link):

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -c main.c -o main.o
gcc -std=c11 -Wall -Wextra -pedantic -g -c vec.c -o vec.o
gcc main.o vec.o -o app -lm
./app
```

Why care: `gcc -c` compiles one file without linking (build systems use it so unchanged files are not recompiled), `gcc main.o vec.o -o app` is the link step, `gcc main.c vec.c -o app` does both at once for small projects, and `gcc -E` shows preprocessor output for macro debugging.

Header search rules: `#include "vec.h"` searches your directory first (your headers, quotes), `#include <stdio.h>` searches system directories (library headers, angle brackets). `#include <vec.h>` will not find your local file unless you pass `-I.` — use quotes for your headers.

One more link detail: `sqrt` lives in the math library, separate from libc on Linux, so the final link needs `-lm`:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c vec.c -o app -lm
```

Forget it and you get the undefined-reference error from the next section, naming `sqrt`.

---

## Classic Error 1: Multiple Definition

Cause: a definition appears in more than one translation unit. The usual suspect is a function body or variable placed in a header, so every `.c` file that includes it emits its own copy.

Setup that triggers it:

```c
/* BAD vec.h */
#ifndef VEC_H
#define VEC_H

typedef struct { double x; double y; } Vec2;

int vec_calls = 0;   /* BAD: definition in header */

Vec2 vec_add(Vec2 a, Vec2 b) {   /* BAD: definition in header */
    Vec2 r = { a.x + b.x, a.y + b.y };
    return r;
}

#endif
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c vec.c -o app
```

```text
/usr/bin/x86_64-linux-gnu-ld.bfd: vec.o:(.bss+0x0): multiple definition of `vec_calls'; main.o:(.bss+0x0): first defined here
/usr/bin/x86_64-linux-gnu-ld.bfd: vec.o: in function `vec_add':
vec.c:(.text+0x0): multiple definition of `vec_add'; main.o:main.c:(.text+0x0): first defined here
collect2: error: ld returned 1 exit status
```

(The exact path to `ld` varies by system — `ld`, `x86_64-linux-gnu-ld`, Apple `ld` — but the phrases `multiple definition of` and `first defined here` are identical everywhere.)

Reading it:

1. `multiple definition of 'vec_add'` — the linker found two compiled copies.
2. `vec.o ... main.o ... first defined here` — the two object files that collide.
3. `collect2: error: ld returned 1 exit status` — `collect2` is gcc's wrapper around `ld`; the link step failed, so no executable was written.

Fix: move bodies to one `.c` file, leave only prototypes and `extern` declarations in the header. The correct `vec.h` from earlier in this part compiles cleanly with the same command.

A sneakier variant: the same `.c` file compiled twice on the command line (`gcc main.c vec.c vec.c -o app`) also produces multiple-definition errors. Same cause — two copies of every definition — same fix: list each source once.

---

## Classic Error 2: Undefined Reference

Cause: a declaration was seen (so compilation succeeded) but no definition was linked in. Either you forgot a `.c` file on the command line, or you misspelled the function name.

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o app
```

where `main.c` calls `vec_add` but `vec.c` was never passed:

```text
/usr/bin/x86_64-linux-gnu-ld.bfd: main.o: in function `main':
main.c:(.text+0x2b): undefined reference to `vec_add'
collect2: error: ld returned 1 exit status
```

Reading it: `main.o: in function 'main'` names the caller, `(.text+0x2b)` is the offset inside it, and `undefined reference to 'vec_add'` is the missing symbol — the compiler trusted your prototype, the linker found no body.

Fixes: forgot `vec.c` (add it to the gcc line), typo in the name (C is case-sensitive), missing `-lm` for `sqrt`, or calling a `static` helper from another file (expose a wrapper instead).

Note the asymmetry: the compiler reports *its* file (`main.c`), but the bug is the missing *other* file. When you see `undefined reference`, ask "which `.c` defines this symbol, and did I pass it to gcc?" before touching the calling code.

---

## Complete Example

Three files, one program:

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
double vec_len(Vec2 v);

#endif /* VEC_H */
```

```c
/* vec.c */
#include <math.h>

#include "vec.h"

static double square(double x) {
    return x * x;
}

Vec2 vec_make(double x, double y) {
    Vec2 v = { x, y };
    return v;
}

Vec2 vec_add(Vec2 a, Vec2 b) {
    Vec2 r = { a.x + b.x, a.y + b.y };
    return r;
}

double vec_len(Vec2 v) {
    return sqrt(square(v.x) + square(v.y));
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

    printf("a = (%.1f, %.1f)\n", a.x, a.y);
    printf("b = (%.1f, %.1f)\n", b.x, b.y);
    printf("sum = (%.1f, %.1f)\n", s.x, s.y);
    printf("len(sum) = %.2f\n", vec_len(s));

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c vec.c -o app -lm
./app
```

```text
a = (1.0, 2.0)
b = (3.0, 4.0)
sum = (4.0, 6.0)
len(sum) = 7.21
```

Check the math: `sum = (4, 6)`, length `sqrt(16 + 36) = sqrt(52) = 7.211...`, printed as `7.21`. If you get `7.21`, compile, link, and the math library all cooperated.

---

## Bugs You Will Meet

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| 1 | `multiple definition of 'f'` | function body in header, included by two `.c` files | keep prototype in `.h`, move body to one `.c` |
| 2 | `multiple definition of 'x'` | variable defined (no `extern`) in header | `extern` in `.h`, define once in one `.c` |
| 3 | `undefined reference to 'f'` | forgot the defining `.c` on the gcc line | add it: `gcc main.c vec.c -o app` |
| 4 | `undefined reference to 'sqrt'` | math library not linked on Linux | append `-lm` |
| 5 | `redefinition of 'struct ...'` | header included twice without guards | add `#ifndef` / `#define` / `#endif` |
| 6 | `fatal error: vec.h: No such file` | used `<vec.h>` or wrong directory | `#include "vec.h"`, check the file sits next to `main.c` |
| 7 | calls a `static` helper from another file | `static` means file-private by design | expose a public wrapper instead |

---

## Checklist

1. Does every header start with `#ifndef X_H` / `#define X_H` and end with `#endif`?
2. Does the header contain only prototypes, struct/enum/typedef definitions, `#define`s, and `extern` declarations?
3. Is every global (if any) declared `extern` in the header and defined exactly once in one `.c` file — or better, hidden as `static` with accessor functions?
4. Is every file-private helper marked `static`?
5. Does the build list every `.c` file exactly once (`gcc main.c vec.c -o app -lm`)?
6. Does it compile clean with `gcc -std=c11 -Wall -Wextra -pedantic -g` and run to the expected output?

---

#### [Previous](../04-file-io/exercises.md) | [Next](./02-part.md)
