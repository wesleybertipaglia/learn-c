# Precedence, the Ternary Operator and Type Conversion

Three topics that all cause the same class of bug: the compiler does something legal and surprising, and you do not notice.

---

## Operator Precedence

Precedence decides the order in which sub-expressions are grouped. From highest to lowest:

| Level | Operators | Associativity |
| --- | --- | --- |
| 1 | postfix `()` `[]` `.` `->` | left to right |
| 2 | prefix `++` `--` `!` `~` `+` `-` `*` `&` `sizeof` `(cast)` | right to left |
| 3 | `*` `/` `%` | left to right |
| 4 | `+` `-` | left to right |
| 5 | `<<` `>>` | left to right |
| 6 | `<` `<=` `>` `>=` | left to right |
| 7 | `==` `!=` | left to right |
| 8 | `&` | left to right |
| 9 | `^` | left to right |
| 10 | `\|` | left to right |
| 11 | `&&` | left to right |
| 12 | `\|\|` | left to right |
| 13 | `?:` | right to left |
| 14 (lowest) | `=` `+=` `-=` `*=` `/=` `%=` `&=` `\|=` `^=` `<<=` `>>=` | right to left |

The important consequence of levels 6 and 7 being separate: **`==` binds tighter than `&`**, and comparison binds tighter than `&&` which binds tighter than `||`.

The parentheses below are **not** in the original expressions — they are added to show what the compiler does with them. Written without them, GCC warns under `-Wparentheses`, which is the point: the language lets you write confusing expressions, and the warning is your only signal.

```c
#include <stdio.h>

int main(void) {
    printf("%d\n", 2 + 3 * 4);        /* 14 */
    printf("%d\n", 1 << (2 + 3));     /* 32: + is higher than << */
    printf("%d\n", (1 + 2) > 2);      /* 1 */
    printf("%d\n", (1 & 2) == 0);     /* 1: == is higher than & */
    printf("%d\n", 1 | (2 ^ (3 & 4)));/* 3: & highest, then ^, then | */

    return 0;
}
```

```text
14
32
1
1
3
```

### The Assignment Trap

`=` has the **lowest** precedence of the common operators, so it grabs everything on its right:

```c
#include <stdio.h>

int main(void) {
    int a = 1;
    int b = 2;

    printf("%d\n", a == b);        /* 0: comparison */
    printf("%d\n", (a = b) == 2);  /* 1: assigns b to a, then compares */
    printf("%d\n", (a = b));       /* assigns, and prints the assigned value 2 */

    return 0;
}
```

```text
0
1
2
```

The last two lines **change `a`**. Assignment is an expression, so it can be nested inside a comparison or passed to a function. Each use here is wrapped in a pair of parentheses, which is how you tell a reader (and `-Wparentheses`) that the assignment is deliberate:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g prog.c -o prog
```

```text
warning: suggest parentheses around assignment used as truth value [-Wparentheses]
```

One parenthesized pair means "I meant this". No parentheses means the warning is the truth.

### Associativity

Left-to-right for arithmetic:

```c
printf("%d\n", 100 / 5 / 2);   /* 10, not 40 */
```

Right-to-left for assignment:

```c
int a, b, c;
a = b = c = 5;   /* c gets 5, then b, then a */
printf("%d %d %d\n", a, b, c);  /* 5 5 5 */
```

### Parenthesize Anyway

Precedence tables are something you look up, not something you trust yourself with. Whenever the intent is not obvious to a human reading the line, add parentheses:

```c
/* legal but hard to review */
if (x > 0 && y > 0 || z == 0) { }

/* the same logic, obvious */
if ((x > 0 && y > 0) || (z == 0)) { }
```

The cost of parentheses is zero. The cost of a misread expression is a debugging session.

---

## The Ternary Conditional Operator

`?:` is an expression that chooses between two values:

```c
condition ? value_if_true : value_if_false
```

```c
#include <stdio.h>

int main(void) {
    int a = 10;
    int b = 20;

    int max = (a > b) ? a : b;
    printf("max = %d\n", max);

    printf("%s\n", (a % 2 == 0) ? "even" : "odd");

    double price = (a > 15) ? 19.99 : 24.99;
    printf("price = %.2f\n", price);

    return 0;
}
```

Because it is an **expression**, `?:` produces a value and can appear anywhere a value can:

```c
printf("%d\n", (a < b) ? a : b);          /* inside a call */
int arr[3] = { (a > b) ? a : b, 0, 0 };    /* inside an initializer */

int clamped = (value < 0) ? 0 : ((value > 100) ? 100 : value);
```

Only **one** branch is evaluated at runtime, so this is safe:

```c
int result = (b != 0) ? (a / b) : 0;   /* a / b is not evaluated when b == 0 */
```

When the branches have different types, the usual arithmetic conversions apply (see below): `?:` with an `int` and a `double` yields a `double`.

---

## Type Conversion

### Implicit Conversion

C converts types silently whenever the types do not match. It is convenient and dangerous in equal measure.

```c
#include <stdio.h>

int main(void) {
    int    i = 65;
    double d = 3.99;
    char   c = 'A';

    double from_int = i;      /* int converted to double: 65.0 */
    int    from_double = d;   /* double converted to int: 3 */
    int    from_char = c;     /* char widened to int: 65 */
    char   from_int_char = (char)97;   /* int narrowed to char: 'a' */

    printf("int -> double: %.2f\n", from_int);
    printf("double -> int: %d\n", from_double);
    printf("char -> int:   %d\n", from_char);
    printf("int -> char:   %c\n", from_int_char);

    return 0;
}
```

```text
int -> double: 65.00
double -> int: 3
char -> int:   65
int -> char:   a
```

> Note on `printf`: `printf` does **not** convert between types for you. `printf("%f\n", i)` passes an `int` where a `double` is expected, which is undefined behavior and a `-Wformat` warning. Declare a variable of the right type first, as above.

#### Integer Promotion

In any expression involving arithmetic or comparison, `char` and `short` are **promoted to `int`** first. This is why `char` and `int` have the same range in expressions, and why you must cast before printing with `%c`:

```c
#include <stdio.h>

int main(void) {
    char c = 'A';

    printf("%zu\n", sizeof(c));         /* 1 */
    printf("%zu\n", sizeof(c + 0));     /* 4: promoted to int */
    printf("%zu\n", sizeof(c + c));     /* 4 */
    printf("%zu\n", sizeof('A'));       /* 4: a character literal is an int */

    return 0;
}
```

```text
1
4
4
4
```

#### The Usual Arithmetic Conversions

When two different arithmetic types meet in a binary operator, C promotes the smaller one to the larger. The rough order is:

```text
_Bool  <  char  <  short  <  int  <  long  <  long long  <  float  <  double  <  long double
```

So in `int + double`, the `int` becomes a `double` and the whole expression is `double`. In `int + long`, the `int` becomes `long`.

#### The Dangerous double to int Truncation

Assigning a `double` to an `int` **discards the fractional part**. It does not round, and it does not warn:

```c
#include <stdio.h>

int main(void) {
    double price = 19.99;
    int    dollars;

    dollars = price;      /* 19 — silent data loss */
    printf("dollars = %d\n", dollars);

    price = -19.99;
    dollars = price;      /* -19, not -20 */
    printf("dollars = %d\n", dollars);

    double huge = 1e20;
    int overflow = huge;  /* UNDEFINED BEHAVIOR: does not fit in an int */
    printf("overflow = %d\n", overflow);

    return 0;
}
```

The last case is undefined behavior, not just wrong: a `double` outside the range of `int` cannot be represented, and the standard allows the compiler to do anything. It often prints a garbage value like `-1094795586` instead of crashing, which is worse.

### Explicit Casts

A **cast** tells the compiler to convert a value to a type you name:

```c
(target_type) expression
```

```c
#include <stdio.h>

int main(void) {
    double pi = 3.14159265;

    printf("%d\n", (int)pi);              /* 3, truncates */
    printf("%.2f\n", (double)7 / 2);      /* 3.50 */
    printf("%d\n", 7 / 2);                /* 3, still integer division */
    printf("%.2f\n", 7 / (double)2);      /* 3.50: the int becomes a double */

    unsigned int big = 4000000000u;
    printf("%d\n", (int)big);             /* some negative number: does not fit */

    char c = (char)300;                   /* 300 mod 256 = 44, printed as ',' */
    printf("%d\n", (int)c);               /* 44 */

    return 0;
}
```

```text
3
3.50
3
3.50
-294967296
44
```

The `(int)big` line is undefined behavior, so the exact number printed varies between machines and optimization levels. The point stands: do not cast a value into a type that cannot represent it.

### A Cast Does Not Round

This is worth stating twice: `(int)3.99` is `3`, not `4`. For rounding you must do it explicitly with `round` from `<math.h>`, or with `(int)(x + 0.5)` for non-negative values.

```c
#include <stdio.h>
#include <math.h>

int main(void) {
    double values[] = { 2.4, 2.5, 2.6, -2.6 };

    for (size_t i = 0; i < sizeof(values) / sizeof(values[0]); i++) {
        printf("%.1f -> cast %d, round %.0f, floor %.0f\n",
               values[i],
               (int)values[i],
               round(values[i]),
               floor(values[i]));
    }

    return 0;
}
```

```text
2.4 -> cast 2, round 2, floor 2
2.5 -> cast 2, round 3, floor 2
2.6 -> cast 2, round 3, floor 2
-2.6 -> cast -2, round -3, floor -3
```

Compile with `-lm` for `math.h`.

### Casting Between Signed and Unsigned

Casting an out-of-range value to a signed type that cannot hold it is undefined behavior, but in practice it reinterprets the bits (two's complement). The reliable pattern is to mask with `& 0xFF`:

```c
#include <stdio.h>

int main(void) {
    int value = 300;

    printf("value & 0xFF      = %d\n", value & 0xFF);   /* 44, defined */
    printf("(unsigned char)value = %d\n", (int)(unsigned char)value);  /* 44 */
    printf("(signed char)value   = %d\n", (int)(signed char)value);    /* 44, defined for unsigned source */

    return 0;
}
```

Going the other way, converting a negative `int` to an `unsigned` type is **defined** and wraps; converting a large `unsigned` back to `int` is **undefined**. Mask or range-check first.

### When to Cast vs. When to Let It Happen

| Situation | What to do |
| --- | --- |
| Integer average needs decimals | cast one operand, or write `2.0` |
| Printing a `size_t` | do not cast to `int`; use `%zu` |
| Truncating on purpose | `(int)value`, and write a comment |
| Reading a byte of raw data | mask with `& 0xFF` |
| Silently relying on a conversion | do not — it is a bug waiting for a bigger value |

Prefer making the types match explicitly rather than relying on implicit conversions in code where precision or overflow matters. Implicit conversions are fine for `int` to `double` in a `printf`.

---

#### [Previous](./03-part.md) | [Next](./exercises.md)