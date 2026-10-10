# What is memory

Every variable you declare is a piece of memory with a numeric address. Pointers are simply variables that store such addresses. Before you can write `int *p`, you need to see what is really going on when you write `int x = 42;`.

---

## RAM as a Huge Array of Bytes

Conceptually, the memory of your program is one enormous array of **bytes**, and every byte has a position in it. That position is its **address**, and an address is just a number.

```text
address:  0x7ffd977ac9a0  0x7ffd977ac9a1  0x7ffd977ac9a2  0x7ffd977ac9a3 ...
           +-------------+ +-------------+ +-------------+ +-------------+
           |             | |             | |             | |             |
           +-------------+ +-------------+ +-------------+ +-------------+
             ... a 4-byte int could live here ...
```

Two facts follow immediately:

- A variable of type `int` does not occupy one byte, it occupies `sizeof(int)` bytes, side by side.
- Each byte has its own address, so a variable has a range of addresses. When people say "the address of `x`", they mean the address of its first byte.

> Addresses are usually shown in hexadecimal (`0x7ffd977ac9a8`) because that notation is compact. They are still just numbers: `0x10` is `16`.

---

## The Address Operator `&`

The `&` operator gives you the address of a variable. It takes the variable itself and produces its address:

```c
int x = 42;
int address = (int)&x;   /* the address, forced into an int so you can see it */
```

In a picture, `&x` is a note that says "the box is over there":

```text
+--------+          +-------------+
|  x = 42|   &x --> | 0x7ffd97a8  |
+--------+          +-------------+
```

`&` binds tightly to its operand and applies to a single variable. It does not "give you a pointer variable" — it gives you a *value*, which happens to be an address. Storing that value in a variable is what turns it into a pointer.

---

## Printing Addresses with `%p`

`printf` prints pointers with `%p`. The standard requires you to cast the pointer to `void *` (a generic pointer, covered in the next part) so the format matches:

```c
#include <stdio.h>

int main(void) {
    int  x      = 42;
    int  y      = 7;
    char letter = 'A';
    double pi   = 3.14;

    printf("x      = %d, address = %p\n", x, (void *)&x);
    printf("y      = %d, address = %p\n", y, (void *)&y);
    printf("letter = %c, address = %p\n", letter, (void *)&letter);
    printf("pi     = %.2f, address = %p\n", pi, (void *)&pi);

    printf("size of int    = %zu bytes\n", sizeof(int));
    printf("size of char   = %zu bytes\n", sizeof(char));
    printf("size of double = %zu bytes\n", sizeof(double));

    printf("distance between x and y = %ld bytes\n", (long)((char *)&y - (char *)&x));

    return 0;
}
```

A typical run:

```text
x      = 42, address = 0x7ffd977ac9a8
y      = 7, address = 0x7ffd977ac9ac
letter = A, address = 0x7ffd977ac9a7
pi     = 3.14, address = 0x7ffd977ac9b0
size of int    = 4 bytes
size of char   = 1 bytes
size of double = 8 bytes
distance between x and y = 4 bytes
```

Look at `x` and `y`: they are 4 bytes apart, exactly `sizeof(int)`. The compiler is free to place your variables in any order and to leave gaps for alignment, so do not rely on a specific layout.

---

## Addresses Change Between Runs

Run that program again and the addresses will almost certainly be different:

```bash
./main          # x at 0x7ffd977ac9a8
./main          # x at 0x7ffe0c4ff1ac  (different every time)
```

Why? Local variables live on the **stack**, which the operating system picks for you when the program starts. The stack grows *downward* from near the top of the address space, and the environment (arguments, environment variables, previous runs) shifts where it lands.

This is why you must never hardcode an address, never store an address in a file expecting it to work next week, and never compare addresses you got from different runs.

### What the Compiler Does Not Tell You

`&x` produces a value whose *runtime size depends on the architecture*: 4 bytes on a 32-bit system, 8 on a 64-bit one. C does not expose a portable "address type" that you can declare, print or store safely. That is why `%p` prints addresses in implementation-defined format, and why casting an address to `int` (as in `int address = (int)&x;`) is only legal on machines where pointers fit in an `int`.

The honest rule: treat addresses as opaque. Use `%p` to print them, `void *` to store them generically, and a typed pointer (`int *`) to work with them.

---

## Why `scanf` Needs `&`

You already met this in the I/O chapter:

```c
int age;

printf("Age: ");
if (scanf("%d", &age) != 1) {
    fprintf(stderr, "expected a number\n");
    return 1;
}
```

Full program:

```c
#include <stdio.h>

int main(void) {
    int age;
    int height;

    printf("Age: ");
    if (scanf("%d", &age) != 1) {
        fprintf(stderr, "expected a number\n");
        return 1;
    }

    printf("Height (cm): ");
    if (scanf("%d", &height) != 1) {
        fprintf(stderr, "expected a number\n");
        return 1;
    }

    printf("age = %d, height = %d, sum = %d\n", age, height, age + height);

    return 0;
}
```

```text
Age: 26
Height (cm): 180
age = 26, height = 180, sum = 206
```

Now the mystery. When you call any function, C copies each argument into the callee's parameter — **passing by value**. When `scanf` receives `age`, it receives a *copy* of the number inside it:

```text
  main's memory            scanf's memory

  +-----------+            +-----------+
  |  age = 26 | ---------> |    a = 26 |    a is a copy
  +-----------+            +-----------+

  writing to a here does NOT touch age
```

So how can `scanf` possibly fill in `age`? Because C also lets you pass **the address itself**:

```text
  main's memory            scanf's memory

  +-----------+            +-------------+
  |  age = 26 | ----------->|  p = 0x...a8|   p holds a copy of the ADDRESS
  +-----------+            +-------------+
        ^                          |
        |                          | scanf does  *p = 42
        |                          v
        +--------------------------+
        writing through p modifies the real age
```

`&age` is what tells `scanf` **where** to write. Without it, `scanf` would be handed a number where it expects a place to put a number.

---

## The Bug and the Fix, Side by Side

This is wrong. Do not compile and run it:

```c
#include <stdio.h>

int main(void) {
    int age;

    printf("Age: ");
    scanf("%d", age);      /* BUG: passing the value, not the address */

    printf("age = %d\n", age);
    return 0;
}
```

gcc tells you exactly why:

```text
p1_noscansf.c:5:13: warning: format '%d' expects argument of type 'int *',
                              but argument 2 has type 'int' [-Wformat=]
    5 |     scanf("%d", age);
      |            ~^   ~~~
      |             |   |
      |             int  int *
p1_noscansf.c:5:5: warning: 'age' is used uninitialized [-Wuninitialized]
```

And the fix:

```c
scanf("%d", &age);     /* pass the address so scanf knows where to write */
```

| What you pass | What `scanf` receives | Result |
| --- | --- | --- |
| `age` | the number stored in `age` | writes to random memory; `age` untouched |
| `&age` | the address of `age` | writes into `age` — correct |

Note that this bug compiles. It is a **warning**, not an error, and that is the entire reason `-Wall -Wextra` is mandatory in this course: it is often the only thing standing between you and a mystery crash.

---

## `&` Requires an lvalue

You cannot take the address of something that has no address.

**Wrong — both lines are compile errors:**

```c
int a = 1;
int b = 2;

int *p = &5;        /* error: lvalue required as unary '&' operand */
int *q = &(a + b);  /* error: lvalue required as unary '&' operand */
```

```text
p1_lvalue.c:6:14: error: lvalue required as unary '&' operand
    6 |     int *p = &5;
      |              ^
p1_lvalue.c:7:14: error: lvalue required as unary '&' operand
    7 |     int *q = &(a + b);
      |              ^
```

An **lvalue** is an expression that names a place in memory — a variable, an array element, a dereferenced pointer. `5` is a literal that lives in the instruction stream, and `a + b` produces a temporary that the compiler is free to discard, so neither has an address worth taking.

Valid targets:

```c
int  x   = 1;
int  arr[3] = {10, 20, 30};
int *p;

p = &x;        /* a variable       */
p = &arr[1];   /* an array element */
p = &x;        /* reassignment, no trickery needed */
```

Note what is *not* in that list: `&arr` (an array name is already an address, so `arr` alone is what you want) and any arithmetic on `&x`. `&x + 1` parses as `(&x) + 1` — the address of `x` moved forward by 4 bytes — which is undefined behaviour unless that address happens to point into a bigger array. You do not need it: take the address, store it, then do arithmetic on the pointer. That is covered in [Pointer Arithmetic](./04-part.md).

---

## Mental Model So Far

```text
   SOURCE CODE              MEMORY
   ---------                ------
   int x = 42;    -->       +-----------+
                              |  x = 42  |
   &x             -->       +-----------+
                               0x7ffd...   a number, not a pointer variable
```

- A variable is a name for a region of memory.
- The region's position is its **address**, a number.
- `&x` reads that number out.
- Nothing in that number is tied to `x` — if `x` goes out of scope, the address becomes meaningless.

The next part stores that address in a variable. That variable is the pointer.

---

#### [Previous](../07-functions/exercises.md) | [Next](./02-part.md)