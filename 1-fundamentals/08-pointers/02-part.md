# Declaring and Dereferencing Pointers

A pointer is a variable whose value is an address. That is the whole definition. Everything else — `*`, `NULL`, `void *` — is syntax and discipline built on top of it.

---

## Declaring a Pointer

```c
int *p;
```

Read it as "p is a pointer to an int". The `*` is part of the declarator, so it sticks to the **name**:

```text
int * p;
    ^ the star belongs to p, not to int
```

`int` is the *pointee type* — it tells the compiler how many bytes to read or write when you use `*p`. A pointer to an `int` knows that its target occupies 4 bytes on a typical machine.

### The Trap of the Comma

```c
int *a, b;
```

This declares **two** variables, and only one of them is a pointer:

```c
int *a, b;

b = 7;
a = &b;
printf("*a = %d\n", *a);
```

```text
*a = 7
a is a pointer (8 bytes), b is a plain int (4 bytes)
```

`a` is a pointer; `b` is a plain `int`. The `*` is consumed by the first declarator and does not apply to the rest. If you want two pointers, repeat the star:

```c
int *a, *b;    /* both are pointers */
int  a, *b;    /* a is an int, b is a pointer */
```

> A lint-heavy reader may prefer writing `int *a; int *b;` on separate lines, which removes the question entirely. One declaration per line is a good habit in code that will be read by tired people at 3am.

---

## Two Operators, Two Jobs

| Operator | Name | Takes | Gives |
| --- | --- | --- | --- |
| `&x` | address-of | a variable | the address where `x` lives |
| `*p` | dereference | a pointer | the value stored at the address `p` holds |

They are opposites: `&` goes from the thing to its address, `*` goes from the address to the thing.

```text
   &x  reads the address        x        reads the value
  +-----------+  ---------->  +---------+
  | 0x7ffd..a8|               |  x = 42 |
  +-----------+  <----------  +---------+

   p holds 0x7ffd..a8         *p reads/writes the memory there
  +-----------+  ---------->  +---------+
  |  p = ...a8|               |  x = 42 |
  +-----------+               +---------+
```

---

## The Three-Step Mental Model

Keep exactly this in your head and pointers stop being mysterious:

```text
1.  p  HOLDS an address
2.  *p READS or WRITES the memory at that address
3.  &x GIVES you the address of x, so you can put it into p
```

That is the entire model. A full program:

```c
#include <stdio.h>

int main(void) {
    int x = 42;
    int *p;

    p = &x;

    printf("x        = %d\n", x);
    printf("p points to %d\n", *p);

    *p = 10;

    printf("after *p = 10, x = %d\n", x);

    return 0;
}
```

```text
x        = 42
p points to 42
after *p = 10, x = 10
```

Notice that `x` changed. We never wrote to `x` directly; we wrote *through* `p`, and the compiler resolved that to a write at the address `p` holds — which is `x`'s address.

Before and after, the picture is identical:

```text
        p                                  p
  +-------------+                     +-------------+
  | 0x7ffd..a8  | ------------------->| 0x7ffd..a8  |
  +-------------+                     +-------------+
        |                                    |
        v                                    v
  +-------------+   *p = 10            +-------------+
  |   x = 42    | ------ becomes ---> |   x = 10    |
  +-------------+                     +-------------+
```

Two pointers can point at the same variable and see each other's writes. Aliasing is a feature when you want it and a bug when you do not.

---

## NULL: The Pointer That Points Nowhere

There is one address guaranteed not to be a valid object: the **null pointer constant**, written `NULL`. It comes from `<stddef.h>`, `<stdio.h>` and `<stdlib.h>` (usually defined as `((void *)0)`).

```c
#include <stdio.h>
#include <stddef.h>

int main(void) {
    int  x   = 42;
    int *a   = NULL;
    int *b   = &x;
    int  c   = 7;
    void *v  = &x;

    printf("x = %d, x is at %p\n", x, (void *)&x);
    printf("*b = %d, b holds the address %p\n", *b, (void *)b);

    if (a == NULL) {
        printf("a is NULL\n");
    } else {
        printf("a is not NULL\n");
    }

    printf("b == NULL ? %s\n", (b == NULL) ? "yes" : "no");
    printf("v points at x ? %s\n", (*(int *)v == x) ? "yes" : "no");

    show_size("an int", sizeof(int));
    show_size("an int pointer", sizeof(int *));
    show_size("a void pointer", sizeof(void *));

    printf("c is still %d, untouched by the pointer work\n", c);

    return 0;
}
```

with the helper it calls:

```c
void show_size(const char *label, size_t size) {
    printf("%s occupies %zu bytes\n", label, size);
}
```

```text
x = 42, x is at 0x7ffdb848d3f8
*b = 42, b holds the address 0x7ffdb848d3f8
a is NULL
b == NULL ? no
v points at x ? yes
an int occupies 4 bytes
an int pointer occupies 8 bytes
a void pointer occupies 8 bytes
c is still 7, untouched by the pointer work
```

Two things to take from this:

- A pointer's size is determined by the **architecture** (8 bytes on a 64-bit machine), not by what it points to. `int *` and `double *` have the same size.
- `NULL` is not a memory location. It is a sentinel meaning "no address". Comparing against it is safe and cheap; dereferencing it is not.

### Dereferencing NULL Crashes

This is undefined behaviour. Do not run it:

```c
#include <stdio.h>
#include <stddef.h>

int main(void) {
    int *p = NULL;
    printf("%d\n", *p);   /* crash: reading address 0 */
    return 0;
}
```

What you get:

```text
Segmentation fault (core dumped)
```

The operating system maps address `0` to nothing, so the CPU raises a fault and the kernel kills the process. The rule is simple: **check for `NULL` before you follow a pointer that might not be valid.** The `malloc` chapter exists almost entirely because of this.

---

## Uninitialized Pointers Are Garbage

```c
int *p;      /* holds whatever was on the stack: nonsense */
*p = 5;      /* writes to a random address */
```

Declaring a pointer does **not** initialize it. The memory it occupies used to belong to a function that already returned, so `p` starts out holding whatever those bytes happened to be. Sometimes that happens to be `0`, sometimes it is a valid address of something unrelated, sometimes it is a mapped page and the write "works". The last case is the dangerous one, because it fails intermittently and far from the cause.

Two habits prevent the whole class of bug:

1. **Initialize every pointer at declaration** — `int *p = &x;` or `int *p = NULL;`.
2. **Initialize to `NULL` by default** when you do not yet have a target, so the mistake becomes an obvious crash instead of silent corruption.

```c
char *input  = NULL;   /* honest: not ready yet */
char *result = buffer; /* honest: already valid */
```

---

## `void *`: The Generic Pointer

`void` means "no type information". A `void *` can hold the address of *any* object, which makes it the natural currency for generic code: `malloc` returns one, and you cast it to the type you actually need.

```c
int *v = (int *)malloc(10 * sizeof(int));   /* fix the type immediately */
```

Reading through a `void *` is not allowed — the compiler does not know how many bytes or how to interpret them:

```c
void *v = &x;
printf("%d\n", *v);   /* error: invalid use of void expression */
```

You must cast it back first:

```c
printf("%d\n", *(int *)v);   /* legal: you told the compiler the type */
```

Casting a pointer is not converting data; it is a promise about how to read the bytes already there. Break the promise and you get garbage, or worse.

---

## The Three Classic Bugs

### Bug 1: `p = x` — missing `&`

Wrong, and gcc catches it:

```c
int x = 42;
int *p;
p = x;                /* error: assignment to 'int *' from 'int' makes
                         pointer from integer without a cast */
```

```text
p2_bad1.c:6:7: error: assignment to 'int *' from 'int' makes pointer from
                   integer without a cast [-Wint-conversion]
    6 |     p = x;
      |       ^
```

Fix: `p = &x;`

The general rule: a variable *is* its value; `&variable` *is* its address. Copying `x` into a pointer gives a pointer to whatever address `42` happens to be. On a 64-bit system, `42` as a pointer is address `0x2a`, which is unmapped, so the program segfaults on the next line.

### Bug 2: `*p = x` — missing the address

This one compiles silently, which is why it is more dangerous:

```c
int x = 42;
int y = 0;
int *p = &y;

*p = x;      /* no error, no warning, and y is now 42 */
```

```text
y = 42
```

You meant to store `x` into `x`. Instead you stored `x`'s value into `y`. Nothing crashes, the answer is just wrong. **Bug and fix side by side:**

```c
/* WRONG: writes x's value into whatever p points to (here: y) */
*p = x;

/* RIGHT: p is a pointer; assign the address so it points at x */
p = &x;

/* RIGHT: if you really want to store x's value into y, say so explicitly */
y = x;
```

The lesson: before writing `*` on the left of `=`, say out loud what variable the pointer points at. If the answer is "I have no idea", you have a bug.

### Bug 3: confusing `&` and `*`

```text
p = &x;    /* correct: fetch the address of x into p */
p = x;     /* wrong:   copy the value of x into p */
*p = x;    /* wrong:   write x's value into whatever p points at */
*p = &x;   /* wrong:   write an address into an int */
```

A memory aid that actually works:

| Where you see it | What it means |
| --- | --- |
| `&x` | "give me the *place* of x" |
| `*p` | "give me the *thing* at p's place" |
| `int *p` | "p is a variable that holds a place" |

Mnemonic: **ampersand gives the address, star gives the content.**

---

## Checklist Before You Use a Pointer

```text
[ ] Does it point at something that is still alive (not a returned local)?
[ ] Was it initialized at declaration?
[ ] Am I using & where the compiler expects a pointer?
[ ] Am I using * only on things that really are pointers?
[ ] Could it be NULL here?
```

---

#### [Previous](./01-part.md) | [Next](./03-part.md)