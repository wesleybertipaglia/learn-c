# Bitwise Operators

Bitwise operators work on the individual bits of an integer. They are the fastest tools for packing flags into small values, and they show up in every real code base — parsers, compression, crypto, graphics, embedded firmware.

## The Operators

| Operator | Name | Meaning |
| --- | --- | --- |
| `&` | AND | bit is 1 only if both operands have it |
| `\|` | OR | bit is 1 if at least one operand has it |
| `^` | XOR | bit is 1 if exactly one operand has it |
| `~` | NOT | flips every bit |
| `<<` | left shift | moves bits left, filling with zeros |
| `>>` | right shift | moves bits right |

## Reading the Examples

We will use these two values:

```text
a = 12 = 1100
b = 10 = 1010
```

### AND `&`

```text
   1100     12
 & 1010     10
   -----
   1000      8
```

Bit 3: both are 1. Bits 2 and 0: not both. Result: 8.

### OR `|`

```text
   1100     12
 | 1010     10
   -----
   1110     14
```

Every bit set in either operand stays set.

### XOR `^`

```text
   1100     12
 ^ 1010     10
   -----
   0110      6
```

Bits that differ are set. Note that `a ^ b == 6`, and `12 + 10 - 2 * (12 & 10) == 12 + 10 - 16 == 6` — XOR is addition without carrying.

### NOT `~`

```text
   1100     12
 ~ ----
   0011      3
```

`~n` is equivalent to `-n - 1`, so `~12 == -13` for a signed `int`.

### Shifts

```text
12 << 1  = 11000  = 24
12 >> 2  = 11     = 3
```

Left shift multiplies by 2 per bit. Right shift of an unsigned value divides by 2 per bit (rounding down).

---

## All Together in One Program

```c
#include <stdio.h>

int main(void) {
    unsigned int a = 12;   /* 1100 */
    unsigned int b = 10;   /* 1010 */

    printf("a     = %u  (%u << 3 | %u)\n", a, a >> 3, a & 7u);
    printf("b     = %u\n", b);
    printf("a & b = %u\n", a & b);   /* 8 */
    printf("a | b = %u\n", a | b);   /* 14 */
    printf("a ^ b = %u\n", a ^ b);   /* 6 */
    printf("~a    = %u\n", ~a);      /* 4294967283 */
    printf("a << 1 = %u\n", a << 1); /* 24 */
    printf("a >> 2 = %u\n", a >> 2); /* 3 */

    return 0;
}
```

```text
a     = 12  (1 << 3 | 4)
b     = 10
a & b = 8
a | b = 14
a ^ b = 6
~a    = 4294967283
a << 1 = 24
a >> 2 = 3
```

Use `%u` with `unsigned int` values. `~a` on a signed `int` would be negative, and `%u` reinterprets the bits — which is exactly the two's complement behavior from part 1 of this chapter.

---

## Bitmasks and Flags

A **mask** is a value with selected bits set. A single `int` can hold many independent yes/no flags, which is cheaper and faster than a struct or an array of bools.

The classic pattern is power-of-two constants:

```c
#define FLAG_A 1   /* 0001 */
#define FLAG_B 2   /* 0010 */
#define FLAG_C 4   /* 0100 */
```

Because each flag occupies exactly one bit, combining flags is just OR-ing them together.

```c
#include <stdio.h>

#define FLAG_A 1u   /* 0001 */
#define FLAG_B 2u   /* 0010 */
#define FLAG_C 4u   /* 0100 */

void print_flags(unsigned int flags) {
    printf("A:%d B:%d C:%d (raw %u)\n",
           (flags & FLAG_A) != 0,
           (flags & FLAG_B) != 0,
           (flags & FLAG_C) != 0,
           flags);
}

int main(void) {
    unsigned int flags = 0;

    print_flags(flags);           /* nothing set */

    /* setting: OR in the bit */
    flags |= FLAG_A;
    print_flags(flags);           /* A on */

    flags |= FLAG_C;              /* A and C on */
    print_flags(flags);

    /* testing: AND with the bit, then compare to 0 */
    if (flags & FLAG_C) {
        printf("FLAG_C is set\n");
    }
    if ((flags & FLAG_B) == 0) {
        printf("FLAG_B is NOT set\n");
    }

    /* clearing: AND with the inverted mask */
    flags &= ~FLAG_A;
    print_flags(flags);           /* only C left */

    /* toggling: XOR flips the bit */
    flags ^= FLAG_A;
    print_flags(flags);           /* A and C */
    flags ^= FLAG_A;
    print_flags(flags);           /* only C again */

    return 0;
}
```

```text
A:0 B:0 C:0 (raw 0)
A:1 B:0 C:0 (raw 1)
A:1 B:0 C:1 (raw 5)
FLAG_C is set
FLAG_B is NOT set
A:0 B:0 C:1 (raw 4)
A:1 B:0 C:1 (raw 5)
A:0 B:0 C:1 (raw 4)
```

### The Four Operations, Summarized

| Goal | Code | Why |
| --- | --- | --- |
| test | `if (flags & FLAG_A)` | AND leaves only that bit; nonzero means set |
| test (exact) | `if ((flags & FLAG_A) != 0)` | explicit, clearer |
| set | `flags \|= FLAG_A` | OR forces the bit to 1 |
| clear | `flags &= ~FLAG_A` | AND with 0 in that position forces 0 |
| toggle | `flags ^= FLAG_A` | XOR flips whatever is there |

The `!= 0` comparison matters in `if ((flags & FLAG_A) != 0)` because `if (flags & FLAG_A)` treats the result as a truthiness value — correct, but easy to misread as a boolean. Write it both ways until it becomes automatic, then prefer the explicit form.

This pattern is exactly how the C standard library stores `errno`-style status, how file permission bits work (`chmod`), and how window/control flags work in every GUI toolkit.

---

## Packing Values Into Bits

The same bitmask idea scales: you can store several small values in the bits of one integer.

```c
#include <stdio.h>

#define MODE_MASK  0x3u   /* lowest 2 bits */
#define VALUE_MASK 0xFFu  /* next 8 bits */

int main(void) {
    unsigned int packed = 0;

    unsigned int mode = 2;
    unsigned int value = 200;

    packed = (mode & MODE_MASK) | ((value & VALUE_MASK) << 2);

    printf("packed   = 0x%X\n", packed);
    printf("mode     = %u\n", packed & MODE_MASK);
    printf("value    = %u\n", (packed >> 2) & VALUE_MASK);

    return 0;
}
```

```text
packed   = 0x322
mode     = 2
value    = 200
```

This is how network protocols and image formats pack fields. It is also how you convert a value to a fixed byte layout for a file you must read on another machine.

---

## Shifts on Signed Values: Implementation-Defined

For **unsigned** operands, shifting is fully defined: bits that fall off are discarded, and zeros are shifted in.

For **signed** operands it is messier:

- `E1 << E2` with a **negative** `E1`, or a shift count that is negative or not less than the width, is **undefined behavior**.
- `E1 >> E2` with a **negative** `E1` is **implementation-defined**: the compiler decides, and it must document what it does.

```c
#include <stdio.h>

int main(void) {
    int n = -8;

    printf("unsigned: %u\n", (unsigned)n >> 1);   /* defined: 2147483644 */
    printf("signed  : %d\n", n >> 1);             /* implementation-defined: -4 or -3 */

    /* This is undefined behavior — do not run it:
       int bad = 1 << 31 << 1;
       int worse = 1 << 32;   on a 32-bit int, the shift count is too large
    */

    printf("1 << 31 = %d\n", 1 << 31);   /* INT_MIN on 32-bit int */

    return 0;
}
```

```text
unsigned: 2147483644
signed  : -4
1 << 31 = -2147483648
```

The rules of thumb that avoid all of this:

1. Shift **unsigned** values when you mean to move bits around.
2. Never shift by a negative count or by more than the bit width (`sizeof(x) * 8 - 1`).
3. Avoid left-shifting a signed value into or past the sign bit — cast to `unsigned` first.
4. For dividing by a power of two, use `/` on unsigned values; for multiplying, use `*`. The compiler already turns those into shifts when it is safe to do so.

```c
unsigned int value = 40;

printf("%u\n", value >> 2);   /* 10: same as value / 4 */
printf("%u\n", value << 1);   /* 80: same as value * 2 */
```

---

#### [Previous](./02-part.md) | [Next](./04-part.md)