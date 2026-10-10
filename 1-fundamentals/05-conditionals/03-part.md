# The Ternary Operator and Choosing the Right Tool

You have three tools for branching: `if`/`else`, `switch`, and `?:`. This part is about picking the right one and avoiding the mistakes that make all three misbehave.

---

## The Ternary Operator in Depth

`?:` takes the general shape:

```c
condition ? value_if_true : value_if_false
```

Unlike `if`, it is an **expression**, so it produces a value.

```c
#include <stdio.h>

int main(void) {
    int a = 10;
    int b = 20;

    int max = (a > b) ? a : b;
    int min = (a < b) ? a : b;

    printf("max = %d, min = %d\n", max, min);
    printf("%s\n", (a % 2 == 0) ? "even" : "odd");

    return 0;
}
```

```text
max = 20, min = 10
even
```

### It Works Anywhere a Value Does

```c
int    x = 5;
int    y = 7;

printf("largest = %d\n", (x > y) ? x : y);

int arr[3] = { (x > y) ? x : y, 0, 0 };

int clamped = (x < 0) ? 0 : ((x > 100) ? 100 : x);   /* nested */

double price = (x > 5) ? 19.99 : 24.99;
```

Nested ternaries are legal but hard to read. Two levels is the practical limit; past that, use `if`.

### Only One Branch Runs

```c
int a = 10;
int b = 0;

int result = (b != 0) ? (a / b) : 0;   /* safe: a / b is not evaluated */
printf("%d\n", result);
```

```text
0
```

When the condition is false, the `a / b` branch is skipped entirely, so the division by zero never happens. Same short-circuit reasoning as `&&`.

### Type Rules

Both branches must have types that can be reconciled. If they differ, the usual arithmetic conversions apply and the result is the wider type:

```c
#include <stdio.h>

int main(void) {
    int    i = 3;
    double d = 3.5;

    double mixed = (1) ? i : d;
    printf("size = %zu\n", sizeof(mixed));   /* 8: promoted to double */

    int flags = (1) ? 'a' : 'b';
    printf("%c\n", flags);

    return 0;
}
```

### Common Uses

```c
printf("%s\n", (score >= 60) ? "pass" : "fail");
n = (n < 0) ? 0 : n;
value = (value > limit) ? limit : value;
printf("%c\n", (n % 2 == 0) ? '.' : '*');
```

The last one is the classic divisor pattern: you would otherwise need a whole `if` block just to pick a character.

---

## When `if` Is Better Than `?:`

Use `?:` when you are **choosing a value**. Use `if` when you are **doing something**.

```c
/* ?: is fine: pick one of two values */
int max = (a > b) ? a : b;
printf("%s\n", ready ? "yes" : "no");

/* ?: is bad: two unrelated actions in one expression */
(a > b) ? printf("a is bigger\n") : printf("b is bigger\n");

/* if is clearer: several statements, or none */
if (a > b) {
    printf("a is bigger\n");
    printf("by %d\n", a - b);
} else {
    printf("b is bigger\n");
}
```

Rule of thumb: if you find yourself writing `?:` with a `printf` or a function call on one branch and something else on the other, or if the line needs a scroll, use `if`.

For an assignment that is already in an `if`, `?:` is fine:

```c
if (grade >= 60) {
    result = 1;
} else {
    result = 0;
}

/* better when the body really is just this */
result = (grade >= 60) ? 1 : 0;
```

---

## `switch` vs Chains of `if`

You saw this in the previous part, summarized:

| Use `switch` when | Use `if`/`else` when |
| --- | --- |
| one value against many integer constants | conditions involve ranges |
| each branch is a distinct exact case | conditions are complex expressions |
| you want the compiler to build a jump table | you are comparing two variables |
| the value is `int`, `char` or an enum | you are comparing strings or floats |

When in doubt, `if` is always correct. `switch` is an optimization for readability and speed, never a requirement.

---

## Common Mistakes

### Mistake 1: Assignment Instead of Comparison

```c
int x = 5;

if (x = 5) {          /* assigns 5, then tests 5, which is true */
    printf("always\n");
}

if (x == 5) {         /* compares */
    printf("true\n");
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g prog.c -o prog
```

```text
warning: suggest parentheses around assignment used as truth value [-Wparentheses]
```

This is why `-Wall` is not optional. When the warning appears, look at the `=`. There are rare legitimate cases (in a `while` loop: `while ((c = getchar()) != EOF)`), and there they need a **pair** of parens to show the intent. One pair of parens is a bug.

### Mistake 2: Comparing Strings with `==`

```c
char a[] = "ana";
char b[] = "ana";

if (a == b) {          /* WRONG */
    printf("equal\n");
}
```

`a` and `b` are arrays. In a comparison each one decays to a pointer to its first character, and you end up comparing **addresses**, not contents. Since these are two different arrays, the addresses differ and the comparison is false.

The three comparisons below are deliberately wrong, and GCC warns about each one. Run the program to see the real output, then fix it with `strcmp`:

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char a[] = "ana";
    char b[] = "ana";
    const char *c = "ana";

    printf("a == b        ? %d\n", a == b);        /* 0: different addresses */
    printf("a == c        ? %d\n", a == c);        /* 0: a is a copy */
    printf("c == \"ana\"   ? %d\n", c == "ana");   /* may be 1: same literal */

    printf("strcmp(a, b)  ? %d\n", strcmp(a, b));  /* 0: equal contents */

    if (strcmp(a, b) == 0) {
        printf("equal\n");
    }

    return 0;
}
```

```text
a == b        ? 0
a == c        ? 0
c == "ana"   ? 1
strcmp(a, b)  ? 0
equal
```

The third line is worth understanding: the standard allows identical string literals to share storage, so `c == "ana"` may be true. Relying on it is non-portable — never compare pointers to decide whether text matches. The Strings chapter covers `strcmp` properly.

GCC flags the first and third comparisons:

```text
warning: comparison between two arrays [-Warray-compare]
note: use '&a[0] == &b[0]' to compare the addresses

warning: comparison with string literal results in unspecified behavior [-Waddress]
```

The `a == c` line (pointer versus `const char *`) warns about nothing — it is legal and simply wrong at runtime.

### Mistake 3: Comparing Floats with `==`

```c
double a = 0.1;
double b = 0.2;

if (a + b == 0.3) {      /* FALSE */
    printf("equal\n");
}
```

As covered in the Data Types chapter, `double` stores a binary approximation, so `0.1 + 0.2` is `0.30000000000000004`. Compare with a tolerance instead:

```c
#include <stdio.h>
#include <math.h>

int main(void) {
    double a = 0.1;
    double b = 0.2;

    if (fabs(a + b - 0.3) < 1e-9) {
        printf("equal within tolerance\n");
    }

    /* checking for exactly zero is fine */
    double zero = 0.0;
    if (zero == 0.0) {
        printf("zero\n");
    }

    return 0;
}
```

```text
equal within tolerance
zero
```

An epsilon comparison is the right tool when you are asking "is this the same physical quantity, computed two ways" — geometry, physics, sensor readings, comparing against a threshold from a specification.

It is **not** the right tool when you are comparing integers, when you need an exact match, or when the numbers come from a source with accumulated error far larger than the epsilon you chose. In those cases the design is wrong, and an epsilon just hides it.

Comparing against **zero** deserves a special note: exact zero is safe for values that came from an assignment, but not for accumulated arithmetic. `x == 0.0` after a long sum may fail while `fabs(x) < 1e-9` succeeds.

---

## Decision Flow Recap

| I want to... | Use | Example |
| --- | --- | --- |
| pick one of two values | `?:` | `int m = (a > b) ? a : b;` |
| pick one of several values | `switch` | `switch (menu) { case 1: ... }` |
| run different code per case | `if`/`else if` | `if (x > 0) ... else ...` |
| test a range | `if` | `if (score >= 90.0)` |
| test two arbitrary variables | `if` | `if (a == b)` |
| test whether a pointer is valid | `if` | `if (p != NULL)` |
| test whether a value is nonzero | `if` | `if (count)` |
| test strings for equality | `if` + `strcmp` | `if (strcmp(a, b) == 0)` |
| test floats for equality | `if` + epsilon | `if (fabs(x - y) < 1e-9)` |
| clamp a value to a range | `if` | `if (v < 0) v = 0;` |
| choose one of several expressions | nested `?:`, or `switch` on an enum | |

One last habit: when a condition gets complicated, give it a name.

```c
int in_range = (value >= 0 && value <= 100);
int needs_id = (age < 18 && !has_parent);

if (in_range) {
}
```

Named conditions are self-documenting and easy to unit test. A 120-character condition is neither.

---

#### [Previous](./02-part.md) | [Next](./exercises.md)