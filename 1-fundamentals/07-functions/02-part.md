# Passing Arguments

Everything in C is passed **by value**. This is not a default you can flip — it is the model, and every argument you ever pass is a copy. Understanding this one rule explains why `scanf` needs `&`, why functions that "should" modify their arguments do not, and why the standard library separates data from pointers.

---

## By Value: Arguments Are Copies

When you call a function, C copies each argument into the callee's parameter. The callee works on its own copy:

```text
   caller's memory              callee's memory

  +-----------+                +-----------+
  | value = 10 | ---copy-------> |   n = 10  |
  +-----------+                +-----------+
                                    |
                     n = n * 2 modifies only this box
```

```c
#include <stdio.h>

void try_to_double(int n);

int main(void) {
    int value = 10;

    printf("before: %d\n", value);
    try_to_double(value);
    printf("after:  %d\n", value);

    return 0;
}

void try_to_double(int n) {
    n = n * 2;
    printf("inside: %d\n", n);
}
```

```text
before: 10
inside: 20
after:  10
```

`try_to_double` doubles its copy, prints it, and throws the copy away. `value` still holds 10. The function did exactly what you asked; you just asked for the wrong thing.

This is not a defect in C. It is a feature: a function cannot surprise its caller by quietly rewriting variables it was handed.

---

## To Modify the Caller's Variable, Pass Its Address

You already met this with `scanf`. The fix is to pass the **address** of the variable instead of its value, which means the parameter becomes a pointer:

```c
#include <stdio.h>

void double_it(int *n);

int main(void) {
    int value = 10;

    printf("before: %d\n", value);
    double_it(&value);
    printf("after:  %d\n", value);

    return 0;
}

void double_it(int *n) {
    *n = *n * 2;
    printf("inside: %d\n", *n);
}
```

```text
before: 10
inside: 20
after:  20
```

Now the call works, and the diagram is the one from the Pointers chapter:

```text
   caller's memory              callee's memory

  +-----------+                +-------------+
  | value = 10 | ---address---> | n = 0x...a8|
  +-----------+                +-------------+
        ^                            |
        |                            | *n = *n * 2  writes through the address
        +----------------------------+
```

The parameter `n` is still a copy — of the *address*. But the address points at the caller's `value`, so writing through it modifies the original.

### The Three-Step Pattern

```c
void double_it(int *n) {
    *n = *n * 2;
}
```

| Step | Code | What happens |
| --- | --- | --- |
| Copy the address into the parameter | `int *n` | the parameter receives `&value` |
| Read through the address | `*n` | gives the current value of `value` |
| Write through the address | `*n = ...` | changes `value` itself |

### Two Rules That Follow Immediately

1. **The parameter must be `T *` when the argument is a `T`.** Forgetting this is a type error the compiler catches, and it is the first thing to check when you get "incompatible pointer type".
2. **The caller must pass `&`.** Passing `value` instead of `&value` is a *warning* about type mismatch, and it is the second thing to check:

```c
double_it(value);      /* WRONG: warning about incompatible pointer types */
double_it(&value);     /* right */
```

### Arrays Are the Exception

Array parameters are **not** copied. When a parameter is declared as an array (`int out[]`), C adjusts it to a pointer (`int *out`) before passing. So a function that fills an array really does modify the caller's array:

```c
void fill_range(int *out, int count);
```

```c
#include <stdio.h>

void swap(int *a, int *b);
void fill_range(int *out, int count);

int main(void) {
    int x = 1;
    int y = 2;

    swap(&x, &y);
    printf("after swap: x=%d y=%d\n", x, y);

    int data[4];
    fill_range(data, 4);
    for (int i = 0; i < 4; i++) {
        printf("data[%d] = %d\n", i, data[i]);
    }

    return 0;
}

void swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

void fill_range(int *out, int count) {
    for (int i = 0; i < count; i++) {
        out[i] = i * i;
    }
}
```

```text
after swap: x=2 y=1
data[0] = 0
data[1] = 1
data[2] = 4
data[3] = 9
```

Notice `fill_range(data, 4)` — no `&` needed. `data` *is already* an address (arrays decay to pointers), which is exactly why `&data` would be wrong here. This is also why a function must be told how many elements the array holds: the pointer carries no length. Getting that count wrong means writing past the end of the buffer, which is the memory bug from the start of this course.

---

## Constraining a Pointer Parameter

If a function only reads through a pointer, say so with `const`:

```c
int total(const int *values, int count);
```

Now the function may read `values[i]` but a line like `values[i] = 0;` will not compile. `const` on the pointer target is checked at compile time and costs nothing at runtime — it is documentation the compiler enforces.

There is no way to say "this `char *` is read-only" in C. The `char *` API of the standard library is the source of a whole family of buffer bugs, which is why `const char *` exists in C++ but not in C.

---

## Returning More Than One Value

Here is the problem. A C function returns exactly one value:

```c
int divide(int a, int b);   /* a / b, or... a % b? only one fits */
```

Computing both the quotient and the remainder of an integer division is a completely normal thing to want, and there is no way to return two `int`s.

There are three solutions, in order of how much you should reach for them.

---

### Solution 1: Out-Parameters With Pointers

The caller declares the variables and passes their addresses; the function writes through them:

```c
void divide(int numerator, int denominator, int *quotient, int *remainder);
```

```c
#include <stdio.h>

void divide(int numerator, int denominator, int *quotient, int *remainder);

int main(void) {
    int numerator = 47;
    int denominator = 5;
    int quotient = 0;
    int remainder = 0;

    divide(numerator, denominator, &quotient, &remainder);

    printf("%d / %d = %d remainder %d\n",
           numerator, denominator, quotient, remainder);
    printf("check: %d * %d + %d = %d\n",
           quotient, denominator, remainder,
           quotient * denominator + remainder);

    return 0;
}

void divide(int numerator, int denominator, int *quotient, int *remainder) {
    *quotient = numerator / denominator;
    *remainder = numerator % denominator;
}
```

```text
47 / 5 = 9 remainder 2
check: 9 * 5 + 2 = 47
```

Two patterns make this safe and readable:

- **The caller must initialize the variables.** An out-parameter is only assigned, never read, so an uninitialized `int` passed as `&quotient` is fine — but initialize anyway, it costs nothing and makes the code obvious.
- **Document the convention in a comment.** Does `remainder` have the sign of the numerator? What happens when `denominator` is 0? For positive divisors, C already gives you `quotient * denominator + remainder == numerator`, and printing that check (as above) makes the property self-evident at runtime.

`const` applies here too: if a function only reads an out-parameter, declare it `const int *`.

---

### Solution 2: Return a `struct`

Group the values into a struct and return that. This is the cleanest answer when the results belong together:

```c
#include <stdio.h>

struct division {
    int quotient;
    int remainder;
};

struct division divide(int numerator, int denominator);

int main(void) {
    int numerator = 47;
    int denominator = 5;

    struct division result = divide(numerator, denominator);

    printf("%d / %d = %d remainder %d\n",
           numerator, denominator, result.quotient, result.remainder);

    return 0;
}

struct division divide(int numerator, int denominator) {
    struct division result;

    result.quotient = numerator / denominator;
    result.remainder = numerator % denominator;

    return result;
}
```

```text
47 / 5 = 9 remainder 2
```

The caller gets one value it can store in a single variable, and the result has a name. The names `quotient` and `remainder` travel with the data, which out-parameters do not give you. Returning a small struct by value is a struct copy, which is cheap and perfectly safe; the Arrays chapter covers the size threshold where that stops being true.

Structs as return values come back in full detail in the Structs chapter.

---

### Solution 3: A Global Variable

And then there is this:

```c
#include <stdio.h>

int accumulated;

void add(int value);

int main(void) {
    add(10);
    add(20);
    add(12);

    printf("accumulated = %d\n", accumulated);

    return 0;
}

void add(int value) {
    accumulated = accumulated + value;
}
```

```text
accumulated = 42
```

It works, and it is the wrong design. Why:

| Problem | Consequence |
| --- | --- |
| The dependency is invisible | `add` looks like it takes one argument and returns `void`, but it also reads and writes program state you cannot see |
| Not reentrant | Two calls in progress at once — a thread, a signal handler, a callback — corrupt each other |
| Not testable | To test `add`, you must reset the global by hand and check it by hand |
| Every file can see it | File-scope variables are visible to all translation units in the program, so anything can overwrite `accumulated` |

A global is acceptable in exactly two places: `main`'s local state that no function needs, and a constant lookup table declared `static const`. Everywhere else, pass the value in and return it. If you catch yourself wanting a global, that is usually a sign a function is missing a parameter.

---

## Which Solution to Use

| Situation | Use |
| --- | --- |
| Two or more values, tightly related, always together | Return a `struct` |
| A couple of simple values, C-like style, no header yet | Out-parameters |
| Read-only inputs alongside the outputs | `const` pointers |
| Nothing | Do not do it |

---

## Common Mistakes

```c
void f(int *n) {
    *n = 5;         /* correct: writes the caller's variable */
}

void g(int n) {
    n = 5;          /* only writes the copy — caller's variable untouched */
}

void h(int n) {
    *n = 5;         /* error: 'n' is an int, not a pointer */
}

int main(void) {
    int value = 1;

    f(&value);
    g(value);

    return 0;
}
```

Two of those are caught at compile time, and the third one is caught by the compiler too — with a warning that is more useful than the symptom you would have felt:

```text
g7.c: In function ‘g’:
g7.c:5:12: warning: parameter ‘n’ set but not used [-Wunused-but-set-parameter]
    5 |     n = 5;
      |        ^
g7.c: In function ‘h’:
g7.c:10:5: error: invalid type argument of unary ‘*’ (have ‘int’)
   10 |     *n = 5;
      |     ^~
```

`set but not used` is the compiler telling you that `g` did all its work on a copy that went nowhere. That warning is the by-value rule announcing itself.

| Symptom | Cause |
| --- | --- |
| Warning about incompatible pointer type | Passed the variable, needed `&` |
| The caller's value did not change | Parameter is `int`, should be `int *` |
| Error: invalid type argument of unary `*` | Parameter is `int`, needs `int *` |
| Error: too few arguments to function | Forgot an out-parameter, or forgot to pass `&` for it |
| Works only sometimes | Passing an address of a variable that has gone out of scope |

---

#### [Previous](./01-part.md) | [Next](./03-part.md)