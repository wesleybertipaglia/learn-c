# Arithmetic Operators

C has five arithmetic operators. Their behavior on integers is the source of a surprising number of bugs, because integer division throws away the fractional part.

## The Five Operators

| Operator | Name | Example | Result |
| --- | --- | --- | --- |
| `+` | addition | `7 + 3` | `10` |
| `-` | subtraction | `7 - 3` | `4` |
| `*` | multiplication | `7 * 3` | `21` |
| `/` | division | `7 / 3` | `2` (integers!) |
| `%` | remainder (modulo) | `7 % 3` | `1` |

There is no exponent operator. `2 ^ 3` is not eight — `^` is the bitwise XOR (see part 3). For powers you need a loop, `pow` from `<math.h>`, or repeated multiplication.

---

## Integer Division Truncates

When both operands are integers, `/` performs **integer division**: the fractional part is discarded. It does not round, it truncates toward zero.

```c
#include <stdio.h>

int main(void) {
    printf("7 / 2  = %d\n", 7 / 2);        /* 3 */
    printf("7.0 / 2 = %.1f\n", 7.0 / 2);   /* 3.5 */
    printf("7 / 2.0 = %.1f\n", 7 / 2.0);   /* 3.5 */

    printf("1 / 3  = %d\n", 1 / 3);        /* 0 */
    printf("100 / 10 = %d\n", 100 / 10);   /* 10 */
    printf("100 %% 10 = %d\n", 100 % 10);  /* 0 */

    return 0;
}
```

```text
7 / 2  = 3
7.0 / 2 = 3.5
7 / 2.0 = 3.5
1 / 3  = 0
100 / 10 = 10
100 % 10 = 0
```

### The Average Bug

This is the single most common C mistake in a first month of programming:

```c
#include <stdio.h>

int main(void) {
    int a = 10;
    int b = 20;

    int average = (a + b) / 2;

    printf("average = %d\n", average);   /* WRONG: should be 15 */

    return 0;
}
```

```text
average = 10
```

The division happened **before** the assignment, on two `int`s, so `30 / 2` became `15` — wait, that one works by accident. Use `a = 1` and `b = 2`:

```c
int average = (1 + 2) / 2;   /* 3 / 2 == 1, not 1.5 */
```

The fix is to make the division floating point:

```c
#include <stdio.h>

int main(void) {
    int a = 1;
    int b = 2;

    printf("integer average: %d\n", (a + b) / 2);         /* 1 */
    printf("correct average: %.2f\n", (a + b) / 2.0);    /* 1.50 */
    printf("cast approach:   %.2f\n", (double)(a + b) / 2);

    return 0;
}
```

The cheapest reliable habit is to write `2.0` instead of `2` when you mean division.

---

## Modulo and Negative Numbers

`%` gives the remainder of integer division. The rule that trips people up: **the result of `%` has the sign of the dividend** (the left operand). Truncation toward zero makes this inevitable.

```c
#include <stdio.h>

int main(void) {
    printf("7 %% 2  = %d\n",  7 %  2);   /*  1 */
    printf("-7 %% 2 = %d\n", -7 %  2);   /* -1 */
    printf("7 %% -2 = %d\n",  7 % -2);   /*  1 */
    printf("-7 %% -2 = %d\n", -7 % -2);  /* -1 */
    printf("-8 %% 3 = %d\n", -8 %  3);   /* -2 */

    printf("7 / 2  = %d\n",  7 / 2);    /*  3 */
    printf("-7 / 2 = %d\n", -7 / 2);    /* -3 */
    printf("7 / -2 = %d\n",  7 / -2);    /* -3 */
    printf("-7 / -2 = %d\n", -7 / -2);   /*  3 */

    return 0;
}
```

```text
7 % 2  = 1
-7 % 2 = -1
7 % -2 = 1
-7 % -2 = -1
-8 % 3 = -2
7 / 2  = 3
-7 / 2 = -3
7 / -2 = -3
-7 / -2 = 3
```

So `%` is not the mathematical modulo (which is always non-negative). If you need a non-negative remainder:

```c
int r = value % divisor;
if (r < 0) {
    r += divisor;   /* now r is in 0..divisor-1 */
}
```

### Common uses for `%`

```c
#include <stdio.h>

int main(void) {
    int seconds = 3725;

    int hours   = seconds / 3600;
    int minutes = (seconds % 3600) / 60;
    int secs    = seconds % 60;

    printf("%d:%02d:%02d\n", hours, minutes, secs);

    int n = 17;
    printf("even? %d\n", n % 2 == 0);
    printf("n mod 5 = %d\n", n % 5);

    return 0;
}
```

```text
1:02:05
even? 0
n mod 5 = 2
```

---

## Division by Zero Is Undefined Behavior

```c
#include <stdio.h>

int main(void) {
    int a = 10;
    int b = 0;

    printf("%d\n", a / b);

    return 0;
}
```

This program **crashes**. On Linux it dies with signal 8 (`SIGFPE`), and the shell reports `Floating point exception` (core dumped). There is no return value, no error code and no exception — the program is simply gone.

```bash
./div_zero
Floating point exception (core dumped)
```

Floating point is different: `1.0 / 0.0` produces `inf` (with `IEEE` floats, most platforms) and `0.0 / 0.0` produces `nan`. But integer division by zero is undefined behavior, always.

Never rely on the crash. Check the divisor before dividing:

```c
if (b != 0) {
    printf("%d\n", a / b);
} else {
    printf("cannot divide by zero\n");
}
```

---

## Compound Assignment

Instead of `x = x + 5`, C offers `x += 5`:

| Operator | Equivalent to |
| --- | --- |
| `+=` | `x = x + ...` |
| `-=` | `x = x - ...` |
| `*=` | `x = x * ...` |
| `/=` | `x = x / ...` |
| `%=` | `x = x % ...` |
| `<<=` `>>=` `&=` `\|=` `^=` | bitwise versions (part 3) |

```c
#include <stdio.h>

int main(void) {
    int x = 10;

    x += 5;   printf("x = %d\n", x);   /* 15 */
    x -= 3;   printf("x = %d\n", x);   /* 12 */
    x *= 2;   printf("x = %d\n", x);   /* 24 */
    x /= 5;   printf("x = %d\n", x);   /* 4 */
    x %= 3;   printf("x = %d\n", x);   /* 1 */

    int counter = 0;
    for (int i = 0; i < 5; i++) {
        counter++;
    }
    printf("counter = %d\n", counter);

    return 0;
}
```

Two notes:

- `x += 5` is **not** the same as `x = x + 5` in general. For a variable of a type with a narrower promoted result, the compound form assigns back using the original type's range, so `unsigned char c = 255; c += 1;` gives 0 (defined wraparound for the compound assignment) rather than an intermediate 256. Prefer compound assignment when you want assignment back into a narrow type.
- `x += 5` as an **expression** yields the new value of `x`, so `y = (x += 5) + 1;` is legal. Use it only for the side effect; chained versions like `a += b += c` are legal but unreadable.

---

## Precedence Within Arithmetic

From highest to lowest:

| Precedence | Operators | Associativity |
| --- | --- | --- |
| 1 (highest) | `+ -` (unary plus/minus) | right to left |
| 2 | `* / %` | left to right |
| 3 (lowest) | `+ -` (binary) | left to right |

```c
#include <stdio.h>

int main(void) {
    printf("%d\n", 2 + 3 * 4);        /* 14, not 20 */
    printf("%d\n", (2 + 3) * 4);      /* 20 */
    printf("%d\n", 100 / 10 / 2);     /* 5, left to right */
    printf("%d\n", 100 - 10 - 2);     /* 88, left to right */
    printf("%d\n", -3 % 2);           /* -1: unary minus binds first, so -(3 % 2) */

    return 0;
}
```

```text
14
20
5
88
-1
```

---

#### [Previous](../03-data-types/exercises.md) | [Next](./02-part.md)