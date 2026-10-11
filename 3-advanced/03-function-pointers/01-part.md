# Function Pointers

A function pointer is a variable that holds the address of a function. Instead of calling a function by its name, you call it through the pointer — which means the *same* line of code can run *different* functions depending on what the pointer holds.

That single idea unlocks comparators, dispatch tables, menus without giant `switch` statements, and callbacks. This part covers the syntax, `typedef` for readability, passing comparators into sort and search, arrays of function pointers, and the mandatory `NULL` check before every indirect call.

Compile everything in this chapter with:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

Zero warnings is the bar. A warning on a function-pointer line is almost always a wrong signature, and a wrong signature through a pointer is undefined behaviour.

---

## 1. Functions Have Addresses Too

Every function in your program lives somewhere in memory, in the code (text) segment. The name of a function, used without parentheses, decays to its address — just like an array name decays to the address of its first element.

```c
#include <stdio.h>

int add(int a, int b) {
    return a + b;
}

int main(void) {
    printf("add(2, 3)      = %d\n", add(2, 3));
    printf("address of add = %p\n", (void *)add);
    return 0;
}
```

```text
add(2, 3)      = 5
address of add = 0x55a1c2d111a9
```

The mental picture:

```text
  SOURCE CODE              MEMORY (text segment)
  -----------              --------------------
  int add(int a, int b)    +---------------+
  { ... }            --->  | machine code  |  at 0x55a1...1a9
                           | for add       |
                           +---------------+

  add  (no parens)  -->  0x55a1...1a9   (a number, the entry address)
  add(2, 3)         -->  RUN the code at that address, get 5
```

A function pointer stores that number so you can run the code later, possibly choosing *which* code at runtime.

---

## 2. The Syntax: `int (*cmp)(int, int)`

Here is the declaration the whole chapter is built on:

```c
int (*cmp)(int, int);
```

Read it inside-out: "`cmp` is a pointer (`*cmp`) to a function taking `(int, int)` and returning `int`."

The parentheses around `*cmp` are mandatory. Without them the meaning changes completely:

| Declaration | Meaning |
| --- | --- |
| `int (*cmp)(int, int);` | `cmp` is a pointer to a function `(int, int) -> int` — what we want |
| `int *cmp(int, int);` | `cmp` is a *function* `(int, int)` returning `int *` — not a pointer at all |
| `int (*cmp)(void);` | pointer to a function `(void) -> int` — different signature, not compatible |
| `void (*handler)(int);` | pointer to a function `(int) -> void` — different signature |

A complete first program:

```c
#include <stdio.h>

int add(int a, int b) {
    return a + b;
}

int sub(int a, int b) {
    return a - b;
}

int main(void) {
    /* declare a pointer to a function (int, int) -> int */
    int (*op)(int, int);

    op = add;
    printf("op = add: %d\n", op(2, 3));
    printf("op = add: %d\n", (*op)(2, 3));  /* explicit dereference, same thing */

    op = sub;
    printf("op = sub: %d\n", op(2, 3));

    return 0;
}
```

```text
op = add: 5
op = add: 5
op = sub: -1
```

Both call styles — `op(2, 3)` and `(*op)(2, 3)` — are identical in C11. The course uses `op(2, 3)` because it is shorter and matches how `qsort` and callbacks call you back.

Assignment rules:

```c
int (*op)(int, int);

op = add;    /* preferred: name decays to address */
```

---

## 3. `typedef` Makes It Readable

Raw function-pointer syntax gets unreadable fast, especially as parameters. The course rule: **always `typedef` the function type** once, then use the name everywhere.

```c
#include <stdio.h>

/* BinaryOp is "pointer to function (int, int) -> int" */
typedef int (*BinaryOp)(int, int);

int add(int a, int b) {
    return a + b;
}

int sub(int a, int b) {
    return a - b;
}

int mul(int a, int b) {
    return a * b;
}

/* takes any BinaryOp -- no syntax noise at the call site */
int apply(BinaryOp op, int x, int y) {
    return op(x, y);
}

int main(void) {
    BinaryOp op = add;
    printf("%d\n", apply(op, 10, 5));   /* 15 */
    printf("%d\n", apply(sub, 10, 5));  /* 5  */
    printf("%d\n", apply(mul, 10, 5));  /* 50 */
    return 0;
}
```

```text
15
5
50
```

Why the `typedef` matters:

| Without `typedef` | With `typedef` |
| --- | --- |
| `int (*ops[4])(int, int);` | `BinaryOp ops[4];` |
| `int apply(int (*op)(int, int), int x, int y);` | `int apply(BinaryOp op, int x, int y);` |
| `int (*choose(char c))(int, int);` | `BinaryOp choose(char c);` |

The pattern for naming:

```c
typedef return_type (*Name)(param_types...);
```

---

## 4. Passing a Comparator to Sort and Search

The payoff: write the sorting loop *once*, and inject the ordering through a pointer.

```c
#include <stdio.h>

typedef int (*Comparator)(int a, int b);

/* strict, overflow-safe comparators: return -1 / 0 / 1, never (a - b) */
int ascending(int a, int b) {
    if (a < b) {
        return -1;
    }
    if (a > b) {
        return 1;
    }
    return 0;
}

int descending(int a, int b) {
    if (a > b) {
        return -1;
    }
    if (a < b) {
        return 1;
    }
    return 0;
}

/* bubble sort driven by *any* Comparator */
void sort_ints(int *arr, int n, Comparator cmp) {
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - 1 - i; j++) {
            if (cmp(arr[j], arr[j + 1]) > 0) {
                int tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
}

void print_all(const int *arr, int n) {
    for (int i = 0; i < n; i++) printf("%d ", arr[i]);
    printf("\n");
}

int main(void) {
    int data1[5] = {5, 2, 8, 1, 9};
    int data2[5] = {5, 2, 8, 1, 9};

    sort_ints(data1, 5, ascending);
    printf("ascending:  ");
    print_all(data1, 5);

    sort_ints(data2, 5, descending);
    printf("descending: ");
    print_all(data2, 5);

    return 0;
}
```

```text
ascending:  1 2 5 8 9
descending: 9 8 5 2 1
```

The same trick works for search — pass the predicate:

```c
#include <stdio.h>
#include <stdbool.h>

typedef bool (*Predicate)(int value);

bool is_even(int v) {
    return (v % 2) == 0;
}

int find_first(const int *arr, int n, Predicate pred) {
    for (int i = 0; i < n; i++) {
        if (pred(arr[i])) {
            return i;
        }
    }
    return -1;
}

int main(void) {
    int data[6] = {-3, -7, 5, 8, 11, 4};
    printf("first even at %d\n", find_first(data, 6, is_even));
    return 0;
}
```

```text
first even at 3
```

One loop, many behaviours. The function owns the *how* (iterate, swap, count); the pointer supplies the *which* (order, match).

---

## 5. Array of Function Pointers: Dispatch Table and Menu

An array of function pointers replaces a long `switch` or `if/else` chain with a table lookup. This is the dispatch pattern.

```c
#include <stdio.h>

typedef int (*BinaryOp)(int, int);

int add(int a, int b) {
    return a + b;
}

int sub(int a, int b) {
    return a - b;
}

int mul(int a, int b) {
    return a * b;
}

int divide(int a, int b) {
    if (b == 0) {
        fprintf(stderr, "division by zero\n");
        return 0;
    }
    return a / b;
}

int main(void) {
    /* dispatch table: index 0..3 selects the operation */
    BinaryOp ops[4] = {add, sub, mul, divide};
    const char *names[4] = {"add", "sub", "mul", "div"};
    int x = 20;
    int y = 4;
    for (int i = 0; i < 4; i++) {
        printf("%s(%d, %d) = %d\n", names[i], x, y, ops[i](x, y));
    }
    return 0;
}
```

```text
add(20, 4) = 24
sub(20, 4) = 16
mul(20, 4) = 80
div(20, 4) = 5
```

Picture of the table:

```text
  ops (array of 4 pointers)        CODE
  -------------------------       ---------------
  ops[0]  -----------------> | add       |
  ops[1]  -----------------> | sub       |
  ops[2]  -----------------> | mul       |
  ops[3]  -----------------> | divide    |
```

Rules for the table:

1. Every entry must have exactly the `typedef` signature.
2. Bounds-check the index before calling. `ops[99](x, y)` jumps to garbage.
3. Keep a parallel `names[]` array for printing — function pointers cannot tell you their own names.
---

## 6. Never Call a `NULL` Pointer

A function pointer is still a pointer. It can be `NULL`, and calling `NULL` is a segfault — there is no code at address zero to run.

```c
#include <stdio.h>

typedef int (*BinaryOp)(int, int);

int add(int a, int b) {
    return a + b;
}

/* SAFE: check before every indirect call that might be NULL */
int safe_apply(BinaryOp op, int x, int y, int fallback) {
    if (op == NULL) {
        fprintf(stderr, "no operation selected\n");
        return fallback;
    }
    return op(x, y);
}

int main(void) {
    BinaryOp op = NULL;

    /* ... later, maybe assigned ... */
    printf("result = %d\n", safe_apply(op, 10, 5, 0));

    op = add;
    printf("result = %d\n", safe_apply(op, 10, 5, 0));

    return 0;
}
```

```text
no operation selected
result = 0
result = 15
```

The discipline, every time:

| Rule | Code |
| --- | --- |
| Initialize at declaration | `BinaryOp op = NULL;` — never a bare `BinaryOp op;` |
| Check before call | `if (op == NULL) { ... } else { op(x, y); }` |
| Validate table index | `if (i >= 0 && i < 4 && ops[i] != NULL)` |
| Validate callback parameter | `void run(BinaryOp cb) { if (cb != NULL) cb(1, 2); }` |

What the crash looks like if you forget (do not run this):

```c
BinaryOp op = NULL;
int r = op(1, 2);   /* BOOM: segfault, address 0 is not callable */
```

```text
Segmentation fault (core dumped)
```

There is no compiler warning that saves you here — `op(1, 2)` is perfectly typed. Only the explicit `== NULL` check saves you. Treat every incoming function pointer like a possibly-`NULL` data pointer.

---

## 7. Pointer to Code: The Full Diagram

Data pointers point at data. Function pointers point at code. Same pointer mechanics, different segment:

```text
  DATA POINTER                        FUNCTION POINTER
  ------------                        ----------------

  int x = 42;                         int add(int a, int b) { ... }
  int *p = &x;                        BinaryOp op = add;

  +-------+  +-----------+            +-------+  +---------------+
  | p     |->| x = 42    |            | op    |->| add's machine |
  +-------+  +-----------+            +-------+  | code          |
   (stack)    (stack)                  (stack)   +---------------+
                                                  (text segment)

  *p == 42  (read data)               op(2,3) == 5 (run code)
  *p = 7    (write data)              op = sub;   (point elsewhere)
```

Consequences worth memorizing:

- `sizeof(op)` is the size of a pointer (8 bytes on 64-bit), not the size of the function.
- You cannot do arithmetic on function pointers (`op + 1` is an error). Code is not an array you walk.
- Copying the pointer is cheap (one address). Copying the function is impossible.

---

## Common Bugs

**Bug 1 — Missing parentheses: declaring a function instead of a pointer.**

```c
int *cmp(int, int);      /* WRONG: function returning int * */
int (*cmp)(int, int);    /* RIGHT: pointer to function returning int */
```

```text
error: passing argument 3 of 'sort_ints' from incompatible pointer type
```

The compiler message says "incompatible pointer type" at the call site, far from the declaration. Fix the declaration.

**Bug 2 — Calling instead of assigning.**

```c
op = add(2, 3);   /* WRONG: stores int 5 into a pointer */
op = add;         /* RIGHT: stores the address */
```

```text
warning: assignment to 'BinaryOp' from 'int' makes pointer from integer without a cast [-Wint-conversion]
```

**Bug 3 — Signature mismatch silently accepted without a `typedef`.**

```c
void greet(int x);                /* returns void, takes one int */
BinaryOp op = (BinaryOp)greet;    /* WRONG: forced cast hides the mismatch */
```

Never cast to silence an incompatible-pointer-type warning. Fix the signature instead.

**Bug 4 — Calling `NULL` or indexing the table out of bounds.**

```c
BinaryOp ops[4] = {add, sub, mul, divide};
int r = ops[choice](10, 5);   /* WRONG if choice is unchecked or entry is NULL */
```

---

## Checklist Before You Run

| Check | Why |
| --- | --- |
| `typedef` names every function-pointer type (`BinaryOp`, `Comparator`) | Raw syntax hides arity/return mistakes |
| Every pointer initialized (`= func` or `= NULL`) | Uninitialized pointer jumps to garbage |
| `NULL` checked before every indirect call | Calling `NULL` segfaults with no warning |
| Dispatch index bounds-checked *and* entry `NULL`-checked | Table lookup with a bad index is a wild jump |
| Comparator returns `-1` / `0` / `1`, never `a - b` | Subtraction overflows on `INT_MIN` / `INT_MAX` |
| Signatures match exactly — no silencing casts | Mismatched call through a pointer is undefined behaviour |
| Compiles clean with `-std=c11 -Wall -Wextra -pedantic -g` | Every warning here is a crash or UB |

---

#### [Previous](../02-bit-manipulation/exercises.md) | [Next](./02-part.md)
