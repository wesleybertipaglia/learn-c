# Syntax

Every C program is built from the same small set of building blocks. This chapter introduces all of them: how a program is structured, how statements end, and how the compiler transforms your source code.

## The Skeleton of a C Program

```c
#include <stdio.h>

int main(void) {
    printf("Hello, World!\n");
    return 0;
}
```

Line by line:

| Line | Meaning |
| --- | --- |
| `#include <stdio.h>` | import the declarations of the standard I/O functions |
| `int main(void)` | the entry point: `main` receives no arguments, returns an `int` |
| `{ ... }` | a block; everything inside it is the body of `main` |
| `printf(...)` | a function call, followed by `;` to end the statement |
| `return 0;` | tell the operating system "finished successfully" |

`main` is the only function the operating system calls directly. Everything else in your program must be called by `main` (directly or indirectly).

---

## Statements and the Semicolon

Almost every statement in C ends with a semicolon:

```c
int x = 10;        /* declaration with initialization */
x = x + 5;         /* assignment */
printf("%d\n", x); /* function call */
return 0;          /* return statement */
```

Statements that *contain* other statements do **not** end with `;`:

```c
if (x > 0) {
    printf("positive\n");
}                 /* no semicolon here — the if is a statement, and a block ends with } */

while (x > 0) {
    x--;
}
```

The single most common beginner error is a missing semicolon. The compiler usually points at the line *after* the problem, because it only notices the error at the end of the statement.

---

## Comments

```c
/* A comment. Everything between the markers is ignored. */

// Another style (C99 and later).

/*
   Comments can
   span multiple lines.
*/
```

Comments are stripped by the compiler and do not generate machine code.

---

## Identifiers

Names of variables, functions, types and constants follow these rules:

- Letters, digits and `_` only.
- Cannot start with a digit.
- Case-sensitive: `total`, `Total` and `TOTAL` are three different names.
- Cannot be a keyword: `int`, `if`, `while`, `return`, `struct`, `const`, ...

```c
int age = 26;          /* ok */
int _total = 100;      /* ok, though a leading underscore is reserved */
int my var = 10;       /* error: whitespace is not allowed */
int 2fast = 1;         /* error: cannot start with a digit */
int float = 3;         /* error: 'float' is a keyword */
```

---

## Blocks and Scope

A `{ }` block creates a **scope**: variables declared inside it only exist inside it.

```c
int x = 1;

if (x == 1) {
    int y = 2;   /* y only exists inside this block */
    printf("%d\n", y);
}

/* printf("%d\n", y);  ← compile error: 'y' undeclared */
```

This is one of C's best features: it lets you reuse the same variable name in different parts of a function without conflicts, and it makes it obvious when an object should be destroyed.

---

## Preprocessor Directives

Lines that start with `#` are instructions for the preprocessor, not for the compiler. They must be the **first non-whitespace** on the line.

```c
#include <stdio.h>   /* system header, searched in standard paths */
#include "my.h"     /* your own header, searched in the current directory first */

#define PI 3.14159             /* object-like macro */
#define SQUARE(x) ((x) * (x))  /* function-like macro */

int area = (int)(PI * 2 * 2);  /* macros are text substitution */
```

> The extra parentheses in `SQUARE` are not decoration. Without them, `SQUARE(1 + 2)` expands to `1 + 2 * 1 + 2` = 5, not 9. Always parenthesize macro parameters.

---

## Standard Headers You Will Use in This Course

```c
#include <stdio.h>   /* printf, scanf, fopen, FILE */
#include <stdlib.h>  /* malloc, free, exit, atoi, qsort */
#include <string.h>  /* strlen, strcpy, memcpy, strcmp */
#include <stdint.h>  /* int32_t, uint8_t, SIZE_MAX */
#include <stdbool.h> /* bool, true, false */
#include <limits.h>  /* INT_MAX, CHAR_BIT */
#include <assert.h>  /* assert */
#include <math.h>    /* sqrt, pow, fabs */
```

If you see `undefined reference to 'sqrt'` when linking a math function, add `-lm` at the end of the `gcc` command.

---

## Compilation Process in Practice

Comments and whitespace are for humans; the compiler sees a stream of tokens. This code:

```c
int x = 1 + 2; // equals 3
```

is tokenized as `int`, `x`, `=`, `1`, `+`, `2`, `;`. The compiler does not care how you spaced or formatted them — only where tokens are unambiguous. This is also why `int count = 10;` and `intcount=10;` differ, while `int count=10;` and `int count = 10;` do not.

---

#### [Previous](../01-getting-started/exercises.md) | [Next](./02-part.md)