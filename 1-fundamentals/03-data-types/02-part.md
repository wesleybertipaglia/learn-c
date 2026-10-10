# Floating Point and Boolean Types

Integers are not enough for real-world numbers. C gives you three floating point types, and one small type that is either true or false.

## Float, Double, Long Double

```c
float       f = 1.5f;
double      d = 1.5;
long double ld = 1.5L;
```

| Type | Typical size | Precision |
| --- | --- | --- |
| `float` | 4 bytes | ~6 to 9 significant decimal digits |
| `double` | 8 bytes | ~15 to 17 significant decimal digits |
| `long double` | 8 or 16 bytes | at least as good as `double` |

The standard only requires `float` to be at least as wide as `double` in range, but every real platform implements `double` as the default choice.

> Use `double` by default. Reach for `float` only to save memory in a large array, and for `long double` only when you genuinely need more precision than `double` provides.

---

## Literals and Suffixes

A number with a dot or an exponent is a floating point literal:

| Literal | Type |
| --- | --- |
| `1.5` | `double` |
| `1.5f` / `1.5F` | `float` |
| `1.5L` / `1.5L` | `long double` |
| `.5` | `double` (the digit before the dot is optional) |
| `5.` | `double` (same thing) |
| `1e3` | `double` (1000.0) |
| `1e-2` | `double` (0.01) |

Without the suffix, a floating point literal is a `double` — the same way `100` is an `int`, not a `long`.

---

## Floats Cannot Represent 0.1

A `double` stores a **binary** number: a sign, an exponent, and a fixed number of binary digits. Base 10 has no finite representation in base 2, exactly like 1/3 has none in base 10 — the digit 3 repeats forever.

So `0.1` is stored as the closest binary value the format can hold, which is very slightly *above* `0.1`:

```c
#include <stdio.h>

int main(void) {
    double a = 0.1;
    double b = 0.2;

    printf("0.1 + 0.2 = %.20f\n", a + b);
    printf("0.1 + 0.2 == 0.3 ? %d\n", a + b == 0.3);
    printf("0.1 == 0.1        ? %d\n", a == a);

    return 0;
}
```

```text
0.1 + 0.2 = 0.30000000000000004441
0.1 + 0.2 == 0.3 ? 0
0.1 == 0.1        ? 1
```

This is not a bug in your code and it is not a compiler bug. It is arithmetic in base 2. The practical rules:

1. **Never compare floats with `==`.** Accumulated rounding will bite you.
2. **Compare with a tolerance** when you really need equality:

```c
#include <stdio.h>
#include <math.h>

int main(void) {
    double a = 0.1;
    double b = 0.2;

    if (fabs(a + b - 0.3) < 1e-9) {
        printf("equal within tolerance\n");
    }

    return 0;
}
```

Compile math functions with `-lm` at the end: `gcc -std=c11 -Wall -Wextra -pedantic -g prog.c -o prog -lm`.

3. **Round for display, not for storage.** `printf("%.2f", x)` gives the user a clean `0.30` while the variable keeps its full precision.

---

## `printf` and `scanf` with Floats

This asymmetry catches everyone once:

| Context | `float` | `double` |
| --- | --- | --- |
| `printf` | `%f` | `%f` |
| `scanf` | `%f` | `%lf` |

The reason is default argument promotion: when you call `printf("...", f)` with a `float`, the compiler promotes it to `double`, so `%f` reads a `double` in both cases. `scanf` receives *pointers*, so nothing is promoted and the format must describe the real type exactly.

```c
#include <stdio.h>

int main(void) {
    float  f = 1.5f;
    double d = 2.5;

    printf("float as double: %f\n", f);    /* correct: %f reads a double */
    printf("double:         %f\n", d);

    printf("with precision: %.3f\n", d);
    printf("exponent form:  %e\n", d);
    printf("long double:    %Lf\n", 1.5L);

    float  in_f;
    double in_d;

    printf("float: ");
    if (scanf("%f", &in_f) != 1) {
        return 1;
    }
    printf("double: ");
    if (scanf("%lf", &in_d) != 1) {
        return 1;
    }

    printf("read back: %f %f\n", in_f, in_d);

    return 0;
}
```

Using `%f` with `scanf("%f", &d)` where `d` is a `double` writes 4 bytes into an 8-byte object: undefined behavior, and usually a wrong value or a crash. GCC warns about it (`-Wformat`), which is one more reason to always build with `-Wall -Wextra`.

---

## `bool`, `true`, `false`

C99 added a boolean type. It lives in `<stdbool.h>`, which does not define anything new — it defines **three macros** on top of plain integers:

| Name | Expands to |
| --- | --- |
| `bool` | `_Bool` |
| `true` | `1` |
| `false` | `0` |

```c
#include <stdio.h>
#include <stdbool.h>

int main(void) {
    bool is_valid = true;
    bool is_empty  = false;

    printf("%d %d\n", is_valid, is_empty);
    printf("%zu\n", sizeof(bool));   /* usually 1 */

    return 0;
}
```

```text
1 0
1
```

So `bool` **behaves exactly like `int`**: it holds 0 or 1, it is truthy/falsy under the usual rules, and any nonzero value converts to `true`. Its value as a type is in *intent*: a function returning `bool` says "this is a yes or no question", and no reader wonders whether `0` means failure or `42` means success.

### `_Bool`

`_Bool` is the built-in type; `<stdbool.h>` only gives it the friendlier name. The conversions are a nice demonstration of how C converts between types:

```c
#include <stdio.h>

int main(void) {
    _Bool a = 0.5;    /* any nonzero value becomes 1 */
    _Bool b = 256;    /* any nonzero value becomes 1 */
    _Bool c = 0;      /* stays 0 */

    printf("%d %d %d\n", a, b, c);

    return 0;
}
```

```text
1 1 0
```

You can always test a boolean with `if`, `while`, `?:` or `!`. The idiomatic way to negate is `!flag`:

```c
bool failed = !success;
```

---

## Summary

| Type | Use it for | Watch out for |
| --- | --- | --- |
| `int`, `long`, `long long` | whole numbers, counts, IDs | overflow is undefined behavior |
| `unsigned` variants | sizes, raw bytes, non-negative values | subtraction wraps silently |
| `float` | memory-saving arrays | low precision |
| `double` | everyday arithmetic | cannot represent 0.1 exactly |
| `long double` | extra precision | larger, slower, not always 16 bytes |
| `bool` / `_Bool` | yes/no flags | never store anything but 0 or 1 in it |

---

#### [Previous](./01-part.md) | [Next](./03-part.md)