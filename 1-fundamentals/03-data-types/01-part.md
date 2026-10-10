# Integer Types

Every variable you declare has a type, and the type decides how many bits are reserved, how those bits are interpreted, and which operations are allowed. In C there are five signed integer types and their unsigned counterparts.

## The Integer Family

```c
char           letter = 'A';
signed char    sc    = -1;
unsigned char  uc    = 255;
short          s     = -30000;
int            i     = 100000;
long           l     = 100000L;
long long      ll    = 10000000000LL;

unsigned int           ui  = 4000000000u;
unsigned long          ul  = 4000000000UL;
unsigned long long     ull = 18000000000000000000ULL;
```

Note the suffix letters (`L`, `LL`, `u`, `UL`): they tell the compiler which type the literal should become, because a bare number like `100000` is an `int` unless it does not fit.

## Measuring the Sizes

`sizeof` is a **compile-time operator** that returns the size of a type in **bytes**. On a typical 64-bit Linux machine:

```c
#include <stdio.h>

int main(void) {
    printf("%zu\n", sizeof(char));
    printf("%zu\n", sizeof(short));
    printf("%zu\n", sizeof(int));
    printf("%zu\n", sizeof(long));
    printf("%zu\n", sizeof(long long));

    printf("%zu\n", sizeof(float));
    printf("%zu\n", sizeof(double));
    printf("%zu\n", sizeof(void *));

    return 0;
}
```

```text
1
2
4
8
8
4
8
8
```

Use `%zu` when printing a `sizeof`, because `sizeof` yields a `size_t` (an unsigned type whose size is platform-dependent). See the next parts for why `%d` would be wrong here.

### The Rules the Standard Actually Guarantees

`sizeof(char)` is **always 1**, by definition — one `char` is one byte, and a byte is `CHAR_BIT` bits (8 on essentially every modern machine).

Everything else is **up to the implementation**. C only promises *minimums*:

| Type | Minimum size | Meaning |
| --- | --- | --- |
| `char` | exactly 1 byte | at least 8 bits |
| `short` | at least 2 bytes | `SHRT_MAX >= 32767` |
| `int` | at least 2 bytes | `INT_MAX >= 32767` |
| `long` | at least 4 bytes | `LONG_MAX >= 2147483647` |
| `long long` | at least 8 bytes | `LLONG_MAX >= 9223372036854775807` |
| `float` | at least 4 bytes | |
| `double` | at least 8 bytes | |

This is why you should **never assume** `sizeof(int) == 4` when writing portable code. Ask the compiler:

```c
#include <stdio.h>

int main(void) {
    if (sizeof(int) == 2) {
        printf("16-bit int\n");
    } else if (sizeof(int) == 4) {
        printf("32-bit int\n");
    } else {
        printf("something else: %zu bytes\n", sizeof(int));
    }

    return 0;
}
```

On Linux/macOS, `long` is 8 bytes. On 64-bit Windows it is 4 bytes, because Windows did not follow the convention of making `long` match pointer width. Portable code that needs "at least 8 bytes" uses `long long` or the exact-width types from `<stdint.h>` (covered in part 4).

---

## Signed and Unsigned

Signed types can hold negative, zero and positive values. Unsigned types hold only zero and positive values, but they get roughly **twice the range**.

| Type | Minimum range |
| --- | --- |
| `signed int` | -2147483647 to 2147483647 |
| `unsigned int` | 0 to 4294967295 |
| `signed char` | -127 to 127 |
| `unsigned char` | 0 to 255 |

Notice the asymmetry: signed types have one more negative value than positive values. The reason is in the next section.

A common misconception is that you can change the sign of a variable at will:

```c
int          i = -1;
unsigned int u = 1;

u = i;   /* the -1 becomes 4294967295 */
i = u;   /* 4294967295 does not fit in an int: this is undefined behavior */
```

Converting a negative value into an unsigned type is **well defined** and wraps. Converting a large unsigned into a signed type that cannot represent it is **undefined behavior** — see part 4.

---

## How Values Are Stored

Every integer is a fixed-size block of bits. To represent negative numbers, C uses **two's complement**, which means: *the value of a signed integer is the bit pattern read as unsigned, minus the largest unsigned value plus one.*

A 4-byte signed `int` uses 32 bits. The bit at the far left (the **sign bit**) has a special role.

| Value | Bit pattern |
| --- | --- |
| `0` | `0000 0000 0000 0000 0000 0000 0000 0000` |
| `1` | `0000 0000 0000 0000 0000 0000 0000 0001` |
| `2147483647` | `0111 1111 1111 1111 1111 1111 1111 1111` |
| `-1` | `1111 1111 1111 1111 1111 1111 1111 1111` |
| `-2` | `1111 1111 1111 1111 1111 1111 1111 1110` |
| `-2147483648` | `1000 0000 0000 0000 0000 0000 0000 0000` |

So `-1` is simply *every bit set*. Two useful consequences:

1. `-x` can be computed by flipping every bit and adding one: `~x + 1`.
2. `0` has exactly one representation (`-0` does not exist), which means comparison is simple bit comparison.

Two's complement also explains the missing negative value: a 32-bit pattern has 2^32 possible values, and C splits them into 2^31 positive/zero and 2^31 negative, giving -2147483648 to 2147483647.

We can see the bits directly with an **unsigned** copy, because converting a negative `int` to `unsigned int` is defined and wraps:

```c
#include <stdio.h>

int main(void) {
    int          n = -1;
    unsigned int bits = (unsigned int)n;   /* defined: wraps to 4294967295 */

    printf("value:  %d\n", n);
    printf("as unsigned: %u\n", bits);
    printf("in hex: %08X\n", bits);

    return 0;
}
```

```text
value:  -1
as unsigned: 4294967295
in hex: FFFFFFFF
```

Notice that `bits` still prints as `4294967295` — the *bits* did not change, only our interpretation of them did. Memory holds bits, not "negative numbers".

---

## Choosing a Type

| Use | Type | Why |
| --- | --- | --- |
| Counting items in a small array | `int` | the default; readable, fast |
| A memory address or array size | `size_t` | cannot be negative; see part 4 |
| A byte of raw data | `unsigned char` | covers the full 0-255 range |
| A value that must be exactly 32 bits | `int32_t` | see part 4 |
| A quantity that must never be negative | `unsigned int` | doubles the range, catches mistakes |

The habit worth building: **prefer `int`** for ordinary arithmetic, reach for `size_t` when dealing with sizes or indices, and use the exact-width types only when a file format or protocol demands it.

### Pitfall: `unsigned` subtraction

```c
#include <stdio.h>

int main(void) {
    unsigned int a = 1;
    unsigned int b = 3;

    printf("%u\n", a - b);   /* wraps around to a huge number */

    return 0;
}
```

```text
4294967295
```

Because both operands are unsigned, `-2` is not representable, so the result wraps modulo 2^32. Nothing crashed and no warning appeared — the result is simply nonsense. If you need to detect this, compare like this instead:

```c
if (a < b) {
    /* a is smaller; do not subtract */
} else {
    printf("%u\n", a - b);
}
```

Signed integers do **not** wrap: overflowing them is undefined behavior. See part 4.

---

#### [Previous](../02-syntax/exercises.md) | [Next](./02-part.md)