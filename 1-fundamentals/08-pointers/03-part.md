# Pointers and Functions

C has no "pass by reference". Everything is passed by value — the compiler never makes an exception. What C *does* have is the ability to pass an address as a value, and that single fact covers every case people mean when they ask for by-reference parameters.

---

## By Value: the Copy That Dies at the End of the Call

```c
void add_one(int a) {
    a = a + 1;
}
```

```text
   caller's memory          function's memory

   +---------------+         +---------------+
   |  x = 41       |  -----> |  a = 41       |
   +---------------+  copy   +---------------+
                                     |
                                     | a = a + 1
                                     v
                              +---------------+
                              |  a = 42       |   changed, then forgotten
                              +---------------+
```

`a` is a brand-new variable. Changing it does nothing to `x`. This is called **pass by value**, and it is the only argument-passing mechanism C has.

---

## By Address: Copy the Address, Write Through It

```c
void add_one(int *a) {
    *a = *a + 1;
}
```

```text
   caller's memory          function's memory

   +---------------+         +---------------+
   |  x = 41       |         |  a = 0x..a8   |  copy of the ADDRESS
   +---------------+  addr   +---------------+
         ^                            |
         |         *a = *a + 1        |
         +----------------------------+
         x becomes 42
```

The parameter `a` is still just a copy — but it is a copy of an **address**, and writing through it lands on the original variable.

| | `f(x)` | `f(&x)` |
| --- | --- | --- |
| What is copied | the value of `x` | the address of `x` |
| Can the function change `x`? | no | yes |
| Can the function change the pointer? | n/a | only its own copy |

---

## The Canonical Example: `swap`

### The broken version

```c
void swap_by_value(int a, int b) {
    int tmp = a;
    a = b;
    b = tmp;
}
```

It swaps its own parameters perfectly, and the caller sees nothing. Side by side with the fix:

```c
void swap_by_value(int a, int b) {   /* WRONG: swaps local copies */
    int tmp = a;
    a = b;
    b = tmp;
}

void swap_by_address(int *a, int *b) {   /* RIGHT: writes to the caller's ints */
    int tmp = *a;
    *a = *b;
    *b = tmp;
}
```

Full program:

```c
#include <stdio.h>

void swap_by_value(int a, int b) {
    int tmp = a;
    a = b;
    b = tmp;
}

void swap_by_address(int *a, int *b) {
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

int main(void) {
    int x = 10;
    int y = 20;

    printf("before: x = %d, y = %d\n", x, y);

    swap_by_value(x, y);
    printf("after swap_by_value:   x = %d, y = %d\n", x, y);

    swap_by_address(&x, &y);
    printf("after swap_by_address: x = %d, y = %d\n", x, y);

    return 0;
}
```

```text
before: x = 10, y = 20
after swap_by_value:   x = 10, y = 20
after swap_by_address: x = 20, y = 10
```

Read the middle line again. The by-value call compiled cleanly, ran, and changed nothing. C did exactly what you asked. This is the trap: **if you want the caller to see a change, you must give it something it can reach.**

---

## `scanf`, the Original Case of This Problem

Now the `scanf("%d", &age)` from the first part looks obvious rather than strange:

```c
void read_int(const char *prompt, int *out) {
    printf("%s", prompt);
    if (scanf("%d", out) != 1) {
        printf("invalid input\n");
    }
}

void double_it(int *value) {
    *value = *value * 2;
}

int main(void) {
    int a;
    int b;

    read_int("First number:  ", &a);
    read_int("Second number: ", &b);

    printf("before: a = %d, b = %d\n", a, b);

    double_it(&a);
    printf("after double_it(&a): a = %d, b = %d (b is untouched)\n", a, b);

    double_it(&b);
    printf("after double_it(&b): a = %d, b = %d\n", a, b);

    return 0;
}
```

```text
First number:  21
Second number: 7
before: a = 21, b = 7
after double_it(&a): a = 42, b = 7 (b is untouched)
after double_it(&b): a = 42, b = 14
```

`double_it` received a pointer by value, exactly like `scanf` does, and it still managed to write into the caller's variable. Why? Because the copy of an address still points at the same memory.

---

## Passing a Pointer by Value Is Still by Value

This is the distinction students most often get wrong. Inside the function you have a **copy of the pointer**. Rebinding that copy does nothing to the caller; writing through it does.

```c
#include <stdio.h>

void set_to_null(int *p) {
    printf("  inside set_to_null, on entry p = %p\n", (void *)p);
    p = NULL;
    printf("  inside set_to_null, on exit  p = %p\n", (void *)p);
}

void zero_out(int *p) {
    *p = 0;
}

int main(void) {
    int x = 42;
    int *p = &x;

    printf("p == &x ? %s\n", (p == &x) ? "yes" : "no");

    set_to_null(p);
    printf("after set_to_null(p): p == &x ? %s, x = %d\n",
           (p == &x) ? "yes" : "no", x);

    zero_out(p);
    printf("after zero_out(p):    x = %d\n", x);

    return 0;
}
```

```text
p == &x ? yes
  inside set_to_null, on entry p = 0x7ffcc533812c
  inside set_to_null, on exit  p = (nil)
after set_to_null(p): p == &x ? yes, x = 42
after zero_out(p):    x = 0
```

Look closely: `p` really did become `NULL` inside the function — the printout proves it — and the caller's `p` still points at `x`. Meanwhile `zero_out(p)` changed `x` to `0`, permanently, because it wrote to the *target*, not to the pointer.

```text
   set_to_null(p)                     zero_out(p)
   rebinds a LOCAL copy               writes to the SHARED target
   -------------------                ---------------------------
   caller's p unchanged               caller's x changed

   +-------+   +-------+              +-------+   +-------+
   | p = A |   | p=NIL |  one copy    | p = A |   | p = A |
   +-------+   +-------+              +-------+   +-------+
        \          /                     |          |
         \--------/  no effect           +--------->| x = 0 |
        both point at x                            |
                                          +-------------------+
```

The four-line version of this rule:

- `p = something` → changes the function's copy. Caller sees nothing.
- `*p = something` → changes the target. Caller sees everything.
- `p` itself cannot be rebound by the caller unless the caller passes `int **`.

---

## A Non-Obvious Benefit: No More "Return Two Things"

Functions return one value. Before pointers you were forced into awkward workarounds.

```c
/* clunky: two output parameters, and the return value tells you nothing */
void find_min_max(int a, int b, int *min_out, int *max_out);

/* cleaner: the common answer comes back, the other goes through a pointer */
int  find_min_max(int a, int b, int *max_out);
```

```c
int find_min_max(int a, int b, int *max_out) {
    *max_out = (a > b) ? a : b;
    return (a < b) ? a : b;
}

int main(void) {
    int max;

    printf("min = %d\n", find_min_max(3, 9, &max));
    printf("max = %d\n", max);

    return 0;
}
```

```text
min = 3
max = 9
```

These parameters are called **out-parameters**. They show up constantly in real C: `fgets(buf, size, fp)`, `strtol(s, &end, 10)`, every `scanf` you write. The naming convention (`max_out`, `result_out`) exists to tell the reader at a glance that the argument will be written to.

### Rules of Thumb

| Situation | Signature |
| --- | --- |
| The function only reads the argument | take it by value, `int n` |
| The function reads a buffer/array | `const char *s` |
| The function must modify the caller's variable | take a pointer, `int *n` |
| The function must modify the pointer itself | `int **pp` (next part) |
| The function never modifies it | always add `const` |

And the one sentence to remember:

> If a function needs to modify the caller's variable, take a pointer.

### Const Pointers in Signatures

Since the pointer is a copy, you can freely promise not to use it to write:

```c
size_t strlen_like(const char *s);   /* cannot modify s through this pointer */
void   print_all(const int *arr, size_t n);
```

`const char *s` means "I promise not to modify the data at `s`". It is a promise the compiler **checks** — if the body tries to write through `s`, compilation fails. Free safety, and it documents intent.

---

#### [Previous](./02-part.md) | [Next](./04-part.md)