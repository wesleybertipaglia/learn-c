# Limits, Overflow and Portability

C lets you write code that compiles everywhere and means something different on each machine. This part is about the tools that make that problem manageable, and about the single most important rule: **overflowing a signed integer is undefined behavior.**

---

## Limits with `<limits.h>`

This header defines the exact range of each integer type on the machine you compiled for.

| Macro | Meaning |
| --- | --- |
| `CHAR_BIT` | bits per byte (8 everywhere modern) |
| `SCHAR_MIN`, `SCHAR_MAX` | range of `signed char` |
| `UCHAR_MAX` | max of `unsigned char` (always 255) |
| `CHAR_MIN`, `CHAR_MAX` | range of plain `char` (depends on its signedness) |
| `SHRT_MIN`, `SHRT_MAX` | range of `short` |
| `INT_MIN`, `INT_MAX` | range of `int` |
| `LONG_MIN`, `LONG_MAX` | range of `long` |
| `LLONG_MIN`, `LLONG_MAX` | range of `long long` |

```c
#include <stdio.h>
#include <limits.h>

int main(void) {
    printf("CHAR_BIT   = %d\n", CHAR_BIT);
    printf("INT_MIN    = %d\n", INT_MIN);
    printf("INT_MAX    = %d\n", INT_MAX);
    printf("UCHAR_MAX  = %u\n", UCHAR_MAX);
    printf("LLONG_MAX  = %lld\n", LLONG_MAX);

    return 0;
}
```

Note that the limits are **macros**, not variables. They are resolved at compile time, so you can use them in `if` conditions and in preprocessor directives.

---

## Exact Sizes with `<stdint.h>`

`limits.h` tells you what *this* platform has. `<stdint.h>` gives you types that are **exactly** the size you ask for — or nothing at all, which is the point.

| Type | Exact width |
| --- | --- |
| `int8_t`, `uint8_t` | 8 bits (1 byte) |
| `int16_t`, `uint16_t` | 16 bits |
| `int32_t`, `uint32_t` | 32 bits |
| `int64_t`, `uint64_t` | 64 bits |

When an exact type does not exist on a platform, `<stdint.h>` simply does not define it. Code that uses `int32_t` therefore compiles only where a 32-bit type really exists — which is what makes binary file formats and network protocols reliable.

There are also "at least" and "fastest" variants:

| Family | Guarantees | Typical type chosen |
| --- | --- | --- |
| `int32_t` | exactly 32 bits | `int` |
| `int_least32_t` | at least 32 bits, smallest such type | `int` |
| `int_fast32_t` | at least 32 bits, fastest for arithmetic | `int` or `long` |
| `int32_t` vs `intptr_t` | wide enough to hold a pointer | `long` on LP64 |

```c
#include <stdio.h>
#include <stdint.h>
#include <inttypes.h>

int main(void) {
    int32_t  a = 2000000000;
    uint8_t  b = 255;

    printf("int32_t  max: %" PRId32 "\n", a);
    printf("uint8_t  val: %u\n", (unsigned)b);
    printf("sizes: %zu %zu %zu\n",
           sizeof(int32_t), sizeof(uint8_t), sizeof(int_fast32_t));

    return 0;
}
```

`int32_t` for a value you will print: `<inttypes.h>` provides `PRId32`, `PRIu32`, `PRId64` and friends, so the format string is always correct for the type, on every platform.

---

## `size_t` and `<stddef.h>`

`size_t` is the unsigned type returned by `sizeof`, and the type of the result of subtracting two pointers. It lives in `<stddef.h>` (also pulled in by `<stdio.h>` and `<string.h>`).

`ptrdiff_t` is the signed counterpart, used for pointer arithmetic that can be negative. `NULL` is a null pointer constant, defined as `((void *)0)` in practice.

```c
#include <stdio.h>
#include <stddef.h>

int main(void) {
    size_t    n = sizeof(int);
    ptrdiff_t d = 3 - 7;

    printf("n = %zu\n", n);
    printf("d = %td\n", d);
    printf("NULL prints as %p\n", (void *)NULL);

    return 0;
}
```

```text
n = 4
d = -4
NULL prints as (nil)
```

(`(nil)` is how glibc renders a null pointer; on other platforms you may see `0` or `000000000000`.)

Because `size_t` is unsigned, writing `int len = some_size_t - 1;` when the size is 0 wraps around to a huge positive number. Keep sizes in `size_t` all the way.

---

## `%zu` Exists Because `sizeof` Returns `size_t`

`printf` does not know the type of the argument you pass — it trusts the format string. If you tell it `%d` (an `int`) but pass a `size_t` (an `unsigned` value, possibly 8 bytes), you get garbage on x86-64 because the values do not line up in registers:

```c
#include <stdio.h>

int main(void) {
    printf("%d\n", sizeof(int));   /* WRONG: %d expects int */

    return 0;
}
```

The example above is deliberately wrong — it is here so you can see the warning your compiler would have given you. With the right flags, GCC catches it:

```bash
gcc -std=c11 -Wall -Wextra -Wformat=2 -pedantic -g prog.c -o prog
```

```text
warning: format '%d' expects argument of type 'int', but argument 2 has type 'size_t'
```

The correct specifiers:

| Type | `printf` | `printf` with `size_t` |
| --- | --- | --- |
| `size_t` | `%zu` | always `%zu` |
| `ptrdiff_t` | `%td` | `%td` |
| `int32_t` (via `<inttypes.h>`) | `%" PRId32 "` | |
| `int64_t` | `%" PRId64 "` | |

Length modifiers also exist for the rest: `%hhd` for `signed char`, `%hd` for `short`, `%ld` for `long`, `%lld` for `long long`.

---

## Signed Overflow Is Undefined Behavior

Here is a program that looks like it should print a negative number:

```c
#include <stdio.h>
#include <limits.h>

int main(void) {
    int big = INT_MAX;

    printf("INT_MAX = %d\n", big);
    printf("INT_MAX + 1 = %d\n", big + 1);

    return 0;
}
```

```text
INT_MAX = 2147483647
INT_MAX + 1 = -2147483648
```

On most machines you get that "wrapped" answer. But the C standard promises nothing: the overflow is **undefined behavior**, which means the compiler is allowed to do *anything at all*. It may print `-2147483648`, it may print `2147483647` again, it may optimize the whole `printf` away because the branch is impossible, and on some machines or with other optimization levels it may produce something else entirely.

Why so permissive? Because compilers use overflow to mean things. A smart compiler can see:

```c
int result = 0;
for (int i = 0; i < n; i++) {
    if (i == 0) {
        continue;   /* at this point, the compiler knows i != 0 */
    }
    result++;
}
```

and simplify the check away, because under the standard's rules `i` overflowing would make the loop meaningless. If overflow were defined to wrap, that assumption would be false and the "optimization" would break your program. C resolves this by leaving overflow undefined, so the compiler may assume it never happens.

The same rule applies to multiplication:

```c
#include <stdio.h>

int main(void) {
    int a = 100000;
    int b = 100000;

    printf("%d\n", a * b);   /* 10^10 does not fit in an int: UB */

    return 0;
}
```

With `-O0` you see a wrapped value. With `-O2` GCC will often assume the multiplication cannot overflow and optimize accordingly. Do not rely on either.

### How to prevent it

1. **Check before you operate.** Compare against the limits.
2. **Make the type wider.** `long long` has a much larger range.
3. **Use `unsigned`**, where overflow is defined and wraps.

```c
#include <stdio.h>
#include <limits.h>

int main(void) {
    int a = 100000;
    int b = 100000;

    if (b != 0 && a > INT_MAX / b) {
        printf("overflow\n");
    } else {
        printf("%lld\n", (long long)a * b);   /* safe */
    }

    long long product = (long long)a * b;
    printf("in long long: %lld\n", product);

    return 0;
}
```

The `a > INT_MAX / b` test is the standard idiom: instead of multiplying and hoping, you ask whether the multiplication could exceed the maximum, using a division that is guaranteed to fit.

---

## Unsigned Overflow Wraps (And That Is Fine)

For unsigned types, the standard defines the result exactly: arithmetic wraps modulo 2^N, where N is the number of bits.

```c
#include <stdio.h>

int main(void) {
    unsigned int max = 0u;

    printf("%u\n", max);          /* 0 */
    printf("%u\n", max - 1u);     /* 4294967295, defined */
    printf("%u\n", 4294967295u + 1u);  /* wraps back to 0 */

    unsigned char byte = 255;
    printf("%u\n", (unsigned)(byte + 1));   /* 0 */

    return 0;
}
```

This is a real feature, not a loophole: hash tables, counters and fixed-width arithmetic all rely on wraparound. Just never let it happen by accident — it is the unsigned counterpart of signed overflow's "undefined" rule, and equally capable of producing a nonsensical program.

---

## Why `long` Is 8 Bytes on Linux and 4 on Windows

Both platforms satisfy the C standard: `long` must be at least 32 bits, and on Windows it is exactly 32 bits.

The reason is historical. On 64-bit systems, Unix compilers adopted the **LP64** model, where pointers and `long` are both 64 bits:

| Type | Windows (LLP64) | Linux/macOS 64-bit (LP64) |
| --- | --- | --- |
| `char` | 1 | 1 |
| `short` | 2 | 2 |
| `int` | 4 | 4 |
| `long` | **4** | **8** |
| `long long` | 8 | 8 |
| pointer | 8 | 8 |

Windows kept `long` at 32 bits for backwards compatibility with decades of existing code, using the `LLP64` name to describe it. The Windows API was built around that choice and still is.

The practical consequence: **never assume `sizeof(long)` is the same on every machine.** If you need a fixed width, use `<stdint.h>`. If you need to print a `long`, use `%ld` — which is correct on both platforms, even though the value means a different size underneath.

---

#### [Previous](./03-part.md) | [Next](./exercises.md)