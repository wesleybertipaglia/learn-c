# Scope and Visibility

A C program is a set of **translation units** — one `.c` file becomes one translation unit. Functions and file-scope variables declared in a translation unit with external linkage are visible to every other file that declares them; everything else stays private. Getting this right is how a large C project stays comprehensible.

---

## Block Scope of Locals

A variable declared inside a `{ }` block exists only from its declaration to the closing brace:

```c
int x = 1;

if (x == 1) {
    int y = 2;        /* born here */
    printf("%d\n", y);
}
/* y is gone */
```

Locals inside a function — parameters, temporaries, loop variables declared in a `for` header — never leave that call. Two calls to the same function have separate copies of every local, which is what makes recursion possible (part 4).

The rule is short enough to memorize: **a name is visible from its declaration to the end of the enclosing block.**

---

## Shadowing

An inner block may declare a name that already exists in an outer scope. The inner declaration **shadows** the outer one for the whole inner block:

```c
#include <stdio.h>

int outer(int i) {
    int result = 0;

    {
        int i = 99;      /* shadows the parameter */
        result = result + i;
        printf("inner i = %d\n", i);
    }

    printf("after block, i = %d\n", i);

    return result + i;
}

int main(void) {
    printf("outer(5) = %d\n", outer(5));

    return 0;
}
```

```text
inner i = 99
after block, i = 5
outer(5) = 104
```

Inside the block, `i` is `99`. The instant the block ends, `i` is the parameter again, `5`. The arithmetic confirms it: `99 + 5 = 104`.

This is legal and sometimes useful, but it is a readability trap. Shadowing is not in `-Wall` or `-Wextra`, so you get no warning by default. Turn on the dedicated check:

```bash
gcc -std=c11 -Wall -Wextra -Wshadow -pedantic -g main.c -o main
```

```text
h1.c: In function ‘outer’:
h1.c:7:13: warning: declaration of ‘i’ shadows a parameter [-Wshadow]
    7 |         int i = 99;      /* shadows the parameter */
      |             ^
h1.c:3:15: note: shadowed declaration is here
    3 | int outer(int i) {
      |           ~~~~~
```

`-Wshadow` is worth adding to any project where more than one person touches the code. The bug it prevents is not a compile error; it is a `printf` printing 5 when you meant 99, or an accidental infinite loop because a loop counter is shadowed by an outer variable of the same name.

---

## File Scope

A function or variable declared **outside** any function is visible to the rest of the translation unit, from its declaration to the end of the file:

```c
int clamp(int value, int low, int high);   /* visible to everything below */

int main(void) { ... }

int clamp(int value, int low, int high) { ... }   /* definition */
```

Functions are declared at file scope. That is why C has no nested functions (part 1) and why a file-scope variable lives for the entire run of the program.

A file-scope variable that is non-`const` is a global, and part 2 covered why globals are a last resort.

---

## `static` Functions: Internal Linkage

`static` on a function means: **this function belongs to this one `.c` file and nowhere else**. The compiler will not even export the symbol, so no other translation unit can call it.

This is the standard way to hide helpers, and it is the single most useful visibility tool in C. A function named `is_hex_digit` in `parse.c` cannot collide with a function of the same name in `format.c`.

```c
#include <stdio.h>

static int is_hex_digit(int c);
int parse_hex(const char *text);

int main(void) {
    printf("parse_hex(\"1a\") = %d\n", parse_hex("1a"));
    printf("parse_hex(\"ff\") = %d\n", parse_hex("ff"));
    printf("parse_hex(\"z9\") = %d\n", parse_hex("z9"));

    return 0;
}

static int is_hex_digit(int c) {
    return (c >= '0' && c <= '9') || (c >= 'a' && c <= 'f');
}

int parse_hex(const char *text) {
    int value = 0;

    for (int i = 0; text[i] != '\0'; i++) {
        char c = text[i];

        if (!is_hex_digit(c)) {
            return -1;
        }

        if (c <= '9') {
            value = value * 16 + (c - '0');
        } else {
            value = value * 16 + (c - 'a' + 10);
        }
    }

    return value;
}
```

```text
parse_hex("1a") = 26
parse_hex("ff") = 255
parse_hex("z9") = -1
```

`is_hex_digit` is a detail of hex parsing. Nothing outside this file has any business calling it, and `static` makes that a guarantee rather than a naming convention.

### Two-File Example

Now split it, and try to reach into the helper from the other file.

`utils.h`:

```c
#ifndef UTILS_H
#define UTILS_H

int clamp_percent(int value);

#endif
```

`utils.c`:

```c
#include "utils.h"

static int clamp(int value, int low, int high) {
    if (value < low) {
        return low;
    }
    if (value > high) {
        return high;
    }
    return value;
}

int clamp_percent(int value) {
    return clamp(value, 0, 100);
}
```

`main.c`:

```c
#include <stdio.h>

#include "utils.h"

int main(void) {
    printf("clamp_percent(150) = %d\n", clamp_percent(150));
    printf("clamp_percent(-20) = %d\n", clamp_percent(-20));

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c utils.c -o app
./app
```

```text
clamp_percent(150) = 100
clamp_percent(-20) = 0
```

Now the other file tries to call `clamp` directly:

```c
#include "utils.h"

int clamp(int value, int low, int high);

int main(void) {
    return clamp(150, 0, 100) + clamp_percent(1);
}
```

It **compiles** — the prototype is perfectly valid on its own — and fails at link time:

```text
/usr/bin/x86_64-linux-gnu-ld.bfd: bad2.o: in function ‘main':
bad2.c:(.text+0x1d): undefined reference to `clamp'
collect2: error: ld returned 1 exit status
```

The exact wording of the linker message varies, but the content does not: **the compiler accepted the call and the linker could not find the function**, because `static` gave it internal linkage and it was never emitted as a linkable symbol.

Read that as confirmation the protection works. Note also that `undefined reference` is a **link** error, not a compile error — the same phase you meet whenever you forget to pass a source file to `gcc`:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o app   # forgot utils.c
```

### When to Use `static`

| Situation | Use `static`? |
| --- | --- |
| Helper only meaningful inside its own file | Yes |
| Function that is part of the file's public interface | No |
| Anything another `.c` file will call | No |
| Unsure | Yes, then remove `static` when a second caller appears |

Marking a function `static` early is cheap: adding it later means editing the definition and every prototype that mentions it.

### `static` on Variables Too

The same keyword, different meaning: a file-scope variable declared `static` is also private to its translation unit, and — unlike an ordinary global — it is initialized to `0` automatically and keeps its value across calls.

```c
static int call_count = 0;

void record_call(void) {
    call_count++;
}
```

That "keep its value between calls" property is what makes `static` locals useful for memoization and for lazily built lookup tables, and what makes it dangerous for anything else: the state is invisible to the caller and survives between unrelated operations. The Modules chapter uses both patterns.

---

## `extern`: Declaring Something Defined Elsewhere

`extern` says "this symbol exists, it is defined in another translation unit, and I am only declaring it here". It tells the compiler not to create storage for it.

`lib.c`:

```c
int counter = 0;

void bump(void) {
    counter++;
}
```

`app.c`:

```c
#include <stdio.h>

extern int counter;        /* defined in lib.c */
extern void bump(void);    /* defined in lib.c */

int main(void) {
    bump();
    bump();
    printf("counter = %d\n", counter);

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g app.c lib.c -o app
./app
```

```text
counter = 2
```

Two subtleties:

- `extern int counter;` and `int counter;` at file scope mean the same thing to the compiler. Both declare rather than define. Write `extern` when you want the reader to know the definition is elsewhere.
- If you **omit** `extern` in a file where the variable *is* defined, that file becomes the definition and allocates the storage. Getting this wrong is the classic duplicate-definition link error:

```text
multiple definition of `counter'; first defined in lib.o
```

Both `lib.c` and `app.c` claimed to define it. Exactly one file may define a variable; any number may declare it.

You rarely write `extern` for functions in C — a bare prototype `int clamp_percent(int value);` already means "declared here, defined elsewhere". `extern` is only mandatory when you need to be explicit, or when declaring a variable.

---

## Header Files: Where Prototypes Belong

A header file contains declarations only: prototypes for functions, and `extern` declarations for variables that other files must see. It contains **no definitions** — no function bodies, no `int counter = 0;`.

`utils.h`:

```c
#ifndef UTILS_H
#define UTILS_H

int clamp_percent(int value);

#endif
```

`utils.c`:

```c
#include "utils.h"

/* the definition, in the same file as the header so they cannot drift apart */

int clamp_percent(int value) { ... }
```

`main.c`:

```c
#include <stdio.h>

#include "utils.h"   /* for the prototype of clamp_percent */

int main(void) { ... }
```

### The Include Guard

The `#ifndef` / `#define` / `#endif` block at the top of a header is an **include guard**. It stops the preprocessor from pasting the header's contents a second time if several files include it:

```text
main.c includes utils.h  ->  contents pasted once
utils.c includes utils.h  ->  guard skips it, already pasted
```

This matters because `#include` is literal text substitution. Without the guard, `utils.h`'s prototypes would appear twice in one translation unit, and every prototype would be a redeclaration error. The guard's name must match the file name (`UTILS_H` for `utils.h`) and be unique across the project — this is why header guards are wrapped in `__` or `#pragma once` is used instead. `#pragma once` is simpler and supported by GCC, clang and MSVC, but it is not part of the C standard; guards are portable.

### Why Prototypes in Headers Matter

A header file makes the compiler check your calls: wrong argument count, wrong argument types, wrong return type. Without it, a call with three arguments to a two-parameter function compiles under the old implicit-declaration rules and produces garbage at runtime.

This is why the convention in part 1 applies at the project level: prototypes above every call site, in a header, included everywhere the function is used.

### Same Names, Different Files

Without `static`, two translation units that each define a helper called `clamp` collide at link time:

```text
multiple definition of `clamp'
```

`static` on each definition is the fix — each becomes private to its own file and the clash disappears. That is the practical reason to reach for `static` on helpers from the very beginning.

---

## The Visibility Summary

| Declaration | Visible where | Storage |
| --- | --- | --- |
| Inside a block | That block, from the declaration down | On the stack, gone after the block |
| Parameter | The function body | On the stack, one copy per call |
| Function at file scope | The whole translation unit, and every other file that declares it | Code |
| `static` function | This translation unit only | Code, symbol not exported |
| File-scope variable | The whole program (with `extern` declarations) | Static, lives until exit |
| `static` variable at file scope | This translation unit only | Static, zero-initialized |

---

## Checklist for a New File

1. Does every function intended for reuse have a prototype in a header?
2. Is every helper `static`?
3. Does each header have an include guard?
4. Do headers contain only declarations, never definitions?
5. Is a variable that is truly global marked `static`, or is it passed in as a parameter?

---

#### [Previous](./02-part.md) | [Next](./04-part.md)