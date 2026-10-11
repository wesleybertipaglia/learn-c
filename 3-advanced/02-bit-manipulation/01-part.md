# Bits and Bitwise Operators

Everything you have stored so far — an `int`, a `char`, a struct — is, at the bottom, a row of bits. Arithmetic operators (`+`, `-`, `*`) treat those bits as a number. Bitwise operators treat them as what they really are: individual switches you can read and flip one by one.

This is the skill behind flags, permissions, protocol headers, graphics pixels, and embedded registers. A single `uint32_t` can hold 32 yes/no answers, and one machine instruction can test all of them at once.

```c
#include <stdio.h>
#include <stdint.h>

int main(void) {
    uint8_t v = 0xAC; /* 172 decimal, 10101100 binary */
    printf("v = %u (0x%02x)\n", v, v);
    printf("v & 0x0F = %u (0x%02x)\n", (unsigned)(v & 0x0Fu), (unsigned)(v & 0x0Fu));
    printf("v | 0x0F = %u (0x%02x)\n", (unsigned)(v | 0x0Fu), (unsigned)(v | 0x0Fu));
    return 0;
}
```

```text
v = 172 (0xac)
v & 0x0F = 12 (0x0c)
v | 0x0F = 175 (0xaf)
```

Compile every example in this chapter with:

```text
gcc -std=c11 -Wall -Wextra -pedantic -g bits.c -o bits
```

All shift code here uses unsigned types (`uint8_t`, `uint32_t`) from `<stdint.h>`. Never shift a negative signed value, and never shift by an amount greater than or equal to the width of the type. Both are undefined behavior.

---

## Binary and Hex: the Two Languages of Bits

Decimal is for humans. Bits are read in binary (base 2) and written in hex (base 16), because one hex digit is exactly four bits — one nibble.

| Decimal | Binary     | Hex  | Meaning            |
| ------- | ---------- | ---- | ------------------ |
| 0       | `00000000` | `0x00` | nothing set      |
| 1       | `00000001` | `0x01` | bit 0 set        |
| 2       | `00000010` | `0x02` | bit 1 set        |
| 4       | `00000100` | `0x04` | bit 2 set        |
| 8       | `00001000` | `0x08` | bit 3 set        |
| 15      | `00001111` | `0x0F` | low nibble set  |
| 16      | `00010000` | `0x10` | bit 4 set        |
| 128     | `10000000` | `0x80` | high bit of a byte |
| 172     | `10101100` | `0xAC` | running example |
| 255     | `11111111` | `0xFF` | all 8 bits set  |

Powers of two are the whole game. Bit `n` (counting from 0 at the right) has value `2^n`:

| Bit index | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
| --------- | - | - | - | - | - | - | - | - |
| Value     | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |
| `1u << n` | `0x80` | `0x40` | `0x20` | `0x10` | `0x08` | `0x04` | `0x02` | `0x01` |

In C, write bit patterns in hex, never in decimal. `0xAC` shows you the nibbles `A = 1010` and `C = 1100`. The decimal `172` shows you nothing. There is no binary literal in standard C11 (the `0b` prefix is a compiler extension and forbidden with `-pedantic`), so hex is the portable way to say "this bit pattern".

Rule of thumb: decimal for quantities ("172 pixels"), hex for patterns ("mask `0x0F`"). `0xAC` is the nibbles `A = 1010` and `C = 1100`; the decimal `172` shows you nothing.

---

## Anatomy of a Byte

A `uint8_t` is eight bits. Number them 7 down to 0, left to right. Bit 0 is the least significant bit (LSB, worth 1). Bit 7 is the most significant bit (MSB, worth 128).

Here is `0xAC` (`10101100`, decimal 172):

```text
  value 0xAC = 172 = 10101100 binary

   bit:    7   6   5   4   3   2   1   0
         +---+---+---+---+---+---+---+---+
         | 1 | 0 | 1 | 0 | 1 | 1 | 0 | 0 |
         +---+---+---+---+---+---+---+---+
  worth:  128  64  32  16   8   4   2   1
           ^               ^   ^
           |               |   |
         128 set        16 set  8 set  -> 128 + 32 + 8 + 4 = 172
                           (bit 5 = 32 and bit 2 = 4 also set)
```

Read it nibble by nibble: the high nibble `1010` is `0xA`, the low nibble `1100` is `0xC`. That split is why hex dumps group digits in pairs. Bit 0 is still on the far right for wider types; C does not fix the byte order in memory (endianness), only this logical numbering.

---

## Printing Bits: `%u`, `%x`, `%02x`

`printf` has no binary format. You print bit patterns with `%u` (decimal), `%x` (hex), and `%02x` (exactly two hex digits, zero-padded). There is no `%b` in standard C.

| Specifier | Prints | Example for `0xAC` |
| --------- | ------ | ------------------ |
| `%u`      | unsigned decimal | `172` |
| `%x`      | hex, no padding | `ac` |
| `%02x`    | hex, 2 digits, zero-padded | `ac` |
| `%02x`    | hex, 2 digits, zero-padded | `05` for value 5 |

The `02` matters: `%x` prints `5` for the value 5, hiding the leading zero nibble. `%02x` prints `05`, so every byte lines up in a dump.

```c
#include <stdio.h>
#include <stdint.h>

static void print_bits_u8(uint8_t v) {
    unsigned int x = v; /* promote once, shift the unsigned copy */
    for (int i = 7; i >= 0; i--) {
        putchar(((x >> (unsigned)i) & 1u) ? '1' : '0');
        if (i == 4) {
            putchar(' '); /* gap between nibbles */
        }
    }
}

int main(void) {
    uint8_t values[] = { 0x00, 0x05, 0xAC, 0xFF };
    for (size_t i = 0; i < sizeof(values) / sizeof(values[0]); i++) {
        printf("dec %3u  hex 0x%02x  bin ", values[i], values[i]);
        print_bits_u8(values[i]);
        putchar('\n');
    }
    return 0;
}
```

```text
dec   0  hex 0x00  bin 0000 0000
dec   5  hex 0x05  bin 0000 0101
dec 172  hex 0xac  bin 1010 1100
dec 255  hex 0xff  bin 1111 1111
```

Three details in that helper are load-bearing:

1. `unsigned int x = v;` — a `uint8_t` promotes to `int` (signed) in most expressions. Copying it into an `unsigned int` first means every later `>>` is an unsigned shift, which is defined.
2. `(unsigned)i` — the shift count itself is unsigned, and the loop guarantees `0 <= i <= 7`, always below the width.
3. `& 1u` — masks everything except the bit that was shifted down to position 0.

For 32 bits the same helper loops `i` from 31 down to 0 and the label uses `%08x` (eight hex digits).

---

## The Six Operators

| Operator | Name | What it does to each bit |
| -------- | ---- | ------------------------ |
| `&`      | AND  | 1 only if **both** inputs are 1 |
| `\|`     | OR   | 1 if **either** input is 1 |
| `^`      | XOR  | 1 if the inputs **differ** |
| `~`      | NOT  | flips every bit (unary) |
| `<<`     | shift left | moves bits toward the MSB |
| `>>`     | shift right | moves bits toward the LSB |

The first three are binary (two operands) and work bit by bit, column by column. `~` is unary. The shifts move whole patterns sideways.

### AND (`&`): masking bits out

`1 & 1 = 1`. Every other combination is 0. AND is how you **keep** selected bits and force the rest to zero:

```text
  0xAC = 1010 1100
& 0x0F = 0000 1111
-------------------
  0x0C = 0000 1100    (low nibble kept, high nibble cleared)
```

### OR (`|`): forcing bits on

`0 | 0 = 0`. Every other combination is 1. OR is how you **set** bits without touching the rest:

```text
  0xAC = 1010 1100
| 0x0F = 0000 1111
-------------------
  0xAF = 1010 1111    (low nibble forced to 1111)
```

### XOR (`^`): toggling and comparing

`1 ^ 1 = 0`, `1 ^ 0 = 1`. XOR is 1 where the inputs differ. It toggles bits, and `a ^ a` is always zero:

```text
  0xAC = 1010 1100
^ 0xFF = 1111 1111
-------------------
  0x53 = 0101 0011    (every bit flipped: same as ~ within 8 bits)
```

### NOT (`~`): flipping everything

`~` flips all bits of its operand, **including the ones you were not thinking about**. On a 32-bit `unsigned int`, `~0xAC` is `0xFFFFFF53`, not `0x53`. Always mask it back down:

`~` flips all bits of its operand, **including the ones you were not thinking about**. On a 32-bit `unsigned int`, `~0xAC` is `0xFFFFFF53`, not `0x53`. Always mask it back down with `& 0xFFu` when you want one byte.

Full truth tables, one column per bit position:

| A | B | A `&` B | A `\|` B | A `^` B |
| - | - | ------- | -------- | ------- |
| 0 | 0 | 0 | 0 | 0 |
| 0 | 1 | 0 | 1 | 1 |
| 1 | 0 | 0 | 1 | 1 |
| 1 | 1 | 1 | 1 | 0 |

A runnable demo of all four (`a = 0xAC`, `b = 0x3F`): `a & b = 0x2C`, `a | b = 0xBF`, `a ^ b = 0x93`, `~a & 0xFF = 0x53`. Check the XOR column by hand: `1010 1100 ^ 0011 1111 = 1001 0011`.

Do not confuse `&` with `&&`, or `|` with `||`. The single-character forms work on every bit and return a bit pattern. The double-character forms work on truth values and return 0 or 1. `0xAC & 0xF0` is `0xA0`; `0xAC && 0xF0` is `1`.

---

## Shifts: `<<` and `>>`

`v << n` moves every bit `n` positions toward the MSB. Zeros enter on the right. `v >> n` moves every bit toward the LSB. For unsigned types, zeros enter on the left (a logical shift).

```text
  0xAC << 1:

  before: 1 0 1 0 1 1 0 0
  after:  0 1 0 1 1 0 0 0   = 0x58 (low 8 bits of 0x158)
           ^                 ^
           |                 +-- 0 enters on the right
           +-- 1 falls off the left edge (lost)

  0xAC >> 2:

  before: 1 0 1 0 1 1 0 0
  after:  0 0 1 0 1 0 1 1   = 0x2B
           ^ ^
           ++++ 0s enter on the left (unsigned shift)
```

Shifting by `n` multiplies (left) or divides (right) by `2^n`, as long as nothing falls off the edge:

```c
#include <stdio.h>
#include <stdint.h>

int main(void) {
    uint32_t v = 13u;

    printf("%u << 1 = %u\n", v, v << 1u); /* 26  = 13 * 2 */
    printf("%u << 3 = %u\n", v, v << 3u); /* 104 = 13 * 8 */
    printf("%u >> 1 = %u\n", v, v >> 1u); /* 6   = 13 / 2, truncated */
    printf("%u >> 2 = %u\n", v, v >> 2u); /* 3   = 13 / 4, truncated */
    return 0;
}
```

```text
13 << 1 = 26
13 << 3 = 104
13 >> 1 = 6
13 >> 2 = 3
```

Right shift truncates toward zero for unsigned values: `13 >> 1` is 6, not 6.5. Compilers use shifts for `* 2` and `/ 2` on unsigned values automatically, so write the arithmetic and let the optimizer do the shifting — use explicit shifts only when you mean "move these bits", not "multiply this number". The most common shift builds a single-bit mask: bit `n` is `1u << n` (`1u << 0 = 0x01` through `1u << 7 = 0x80`). For 32-bit masks use `1u << n` with `0 <= n < 32`.

---

## Masks: Set, Clear, Toggle, Test

A **mask** is a bit pattern whose 1-bits select the positions you care about. Four one-line idioms cover almost all bit work. Here `BIT(n)` is `(1u << (n))` computed on unsigned values:

| Goal | Idiom | Example (`n = 3`) |
| ---- | ----- | ----------------- |
| Set bit `n` | `v \|= BIT(n)` | `1010 1100` -> `1010 1100` (already set) |
| Clear bit `n` | `v &= ~BIT(n)` | `1010 1100` -> `1010 0100` |
| Toggle bit `n` | `v ^= BIT(n)` | `1010 1100` -> `1010 0100` |
| Test bit `n` | `(v & BIT(n)) != 0` | bit 3 of `0xAC` is `1` |

```c
#include <stdio.h>
#include <stdint.h>

#define BIT(n) (1u << (n))

static uint8_t set_bit(uint8_t v, unsigned n) {
    return (uint8_t)(v | BIT(n));
}

static uint8_t clear_bit(uint8_t v, unsigned n) {
    return (uint8_t)(v & (uint8_t)(~BIT(n)));
}

static uint8_t toggle_bit(uint8_t v, unsigned n) {
    return (uint8_t)(v ^ BIT(n));
}

static int test_bit(uint8_t v, unsigned n) {
    return ((unsigned)v & BIT(n)) != 0u;
}

int main(void) {
    uint8_t v = 0xAC; /* 10101100 */

    printf("start:       0x%02x\n", v);
    printf("set bit 0:   0x%02x\n", set_bit(v, 0));
    printf("clear bit 3: 0x%02x\n", clear_bit(v, 3));
    printf("toggle bit 7: 0x%02x\n", toggle_bit(v, 7));
    printf("test bit 2:  %d\n", test_bit(v, 2));
    printf("test bit 0:  %d\n", test_bit(v, 0));

    /* extracting a field: keep bits 4..7 */
    printf("high nibble: 0x%02x\n", (unsigned)(uint8_t)(((unsigned)v >> 4u) & 0x0Fu));
    /* inserting a field: replace low nibble with 0x5 */
    printf("low=0x5:     0x%02x\n",
           (unsigned)(uint8_t)(((unsigned)(uint8_t)((unsigned)v & 0xF0u)) | 0x05u));
    return 0;
}
```

```text
start:       0xac
set bit 0:   0xad
clear bit 3: 0xa4
toggle bit 7: 0x2c
test bit 2:  1
test bit 0:  0
high nibble: 0x0a
low=0x5:     0xa5
```

Note the casts. `~BIT(n)` is computed as `unsigned int` (at least 32 bits), so casting back to `uint8_t` documents "I want the low 8 bits". The test returns an `int` 0 or 1, which is exactly what an `if` wants.

Multi-bit masks follow the same four shapes with a mask instead of a single bit: `(v & 0x0Fu)` keeps the low nibble, `(v & 0xF0u)` keeps the high nibble, `(v | 0x0Fu)` forces the low nibble on.

---

## The Two Shift Traps

### Trap 1: shifting by `>=` the width is undefined

`v << n` and `v >> n` require `0 <= n < WIDTH`, where WIDTH is the number of bits in the (promoted) left operand. For a 32-bit `unsigned int`, shifting by 32, 40, or 100 is undefined behavior — not "gives zero", undefined. On x86 the hardware masks the count (`32` behaves like `0`), on ARM it may give zero. The compiler may assume it never happens and delete your error handling.

```c
#include <stdio.h>
#include <stdint.h>

static uint32_t shift_left_safe(uint32_t v, unsigned n) {
    if (n >= 32u) {
        return 0u; /* caller asked for something impossible: define it */
    }
    return v << n;
}

int main(void) {
    printf("1u << 31 = 0x%08x\n", 1u << 31u);
    printf("safe(1, 32) = 0x%08x\n", shift_left_safe(1u, 32u));
    printf("safe(1, 1)  = 0x%08x\n", shift_left_safe(1u, 1u));
    return 0;
}
```

```text
1u << 31 = 0x80000000
safe(1, 32) = 0x00000000
safe(1, 1)  = 0x00000002
```

Guard every shift whose count comes from input, a file, or the network. A shift count of exactly the width is the classic fuzzer find.

### Trap 2: signed and negative shifts are undefined

Three related ways to go wrong, all undefined behavior:

| Code | Why it is UB |
| ---- | ------------ |
| `int x = -8; x >> 1;` | right shift of a negative value is implementation-defined (arithmetic vs logical); do not rely on it |
| `x << 1` where the result overflows `int` | signed overflow is UB, even via shift |
| `v << -1` or `v >> -2` | negative shift count is always UB, for any type |

The fix is one rule: **shift only unsigned, non-negative, in-range values.**

```c
#include <stdio.h>
#include <stdint.h>

int main(void) {
    /* WRONG (do not write this): */
    /* int s = -8; printf("%d\n", s >> 1); */

    /* RIGHT: do the bit work unsigned, convert at the boundary. */
    uint32_t magnitude = 8u;
    uint32_t halved = magnitude >> 1u;
    int result = -(int)halved;

    printf("halved magnitude: %u\n", halved);
    printf("as signed: %d\n", result);

    /* RIGHT: guard the count, keep the value unsigned. */
    uint32_t v = 0xACu;
    unsigned n = 2u;
    uint32_t out = (n < 32u) ? (v >> n) : 0u;
    printf("0x%02x >> %u = 0x%02x\n", v, n, out);
    return 0;
}
```

```text
halved magnitude: 4
as signed: -4
0xac >> 2 = 0x2b
```

`-Wall -Wextra` catches constant shift counts that are too large, but not counts known only at runtime. The guard `n < 32u` is your responsibility. One last promotion warning: in `uint8_t x = 0xFF; x << 8;`, `x` promotes to signed `int` before the shift — for general helpers, cast to `unsigned int` first and narrow back with `(uint8_t)((wide << 1u) & 0xFFu)`.

---

## Complete Example: Nibble Swap

Swap the two nibbles of a byte: `0xAC` becomes `0xCA`. It uses both shifts, both masks, and prints every format from this part.

```c
#include <stdio.h>
#include <stdint.h>
#include <assert.h>

static void print_bits_u8(uint8_t v) {
    unsigned int x = v;
    for (int i = 7; i >= 0; i--) {
        putchar(((x >> (unsigned)i) & 1u) ? '1' : '0');
        if (i == 4) {
            putchar(' ');
        }
    }
}

static uint8_t swap_nibbles(uint8_t v) {
    unsigned int x = v;
    unsigned int hi = (x >> 4u) & 0x0Fu;
    unsigned int lo = (x & 0x0Fu) << 4u;
    return (uint8_t)(hi | lo);
}

int main(void) {
    static const uint8_t inputs[] = { 0x00, 0x12, 0xAC, 0xFF };
    for (size_t i = 0; i < sizeof(inputs) / sizeof(inputs[0]); i++) {
        uint8_t v = inputs[i];
        uint8_t s = swap_nibbles(v);
        assert(swap_nibbles(s) == v); /* swapping twice restores v */

        printf("0x%02x (%3u) ", v, v);
        print_bits_u8(v);
        printf(" -> 0x%02x (%3u) ", s, s);
        print_bits_u8(s);
        putchar('\n');
    }
    return 0;
}
```

```text
0x00 (  0) 0000 0000 -> 0x00 (  0) 0000 0000
0x12 ( 18) 0001 0010 -> 0x21 ( 33) 0010 0001
0xac (172) 1010 1100 -> 0xca (202) 1100 1010
0xff (255) 1111 1111 -> 0xff (255) 1111 1111
```

---

## Common Bugs

| Bug | Symptom | Fix |
| --- | ------- | --- |
| `&` instead of `&&` (or vice versa) | mask test always true, or logic reduced to 0/1 | single char for bits, double char for truth |
| `~mask` without re-masking | high bits you never wanted get set | `(uint8_t)(~m & 0xFFu)` |
| `1 << 31` with signed `1` | undefined behavior, warning with `-Wall` | `1u << 31u` |
| shift count `>=` width | works on x86, zero on ARM, deleted by optimizer | `if (n >= 32u) return 0;` |
| shifting a negative `int` | sign bit replicated or not, per compiler | shift `uint32_t`, convert after |
| `%d` for a mask | `0xFFFFFFFF` prints as `-1` | `%u` / `%x` / `%02x` for bit patterns |
| forgetting `u` suffix in `BIT(n)` macro | signed overflow at bit 15/31 | `#define BIT(n) (1u << (n))` |
| `x >> 8` on `uint8_t x` without cast | promotion to signed `int` before shift | `((unsigned int)x >> 8)` |

---

## Checklist

- [ ] I write masks in hex (`0x0F`), not decimal (`15`).
- [ ] I print bytes with `%02x` and words with `%08x` so leading zeros are visible.
- [ ] I can hand-evaluate `&`, `|`, `^`, `~` on two nibbles.
- [ ] I can set, clear, toggle, and test bit `n` from memory.
- [ ] Every shift in my code is on an unsigned value with `0 <= n < WIDTH`.
- [ ] I guard runtime shift counts with `if (n >= 32u)` before shifting.
- [ ] I never right-shift a negative signed value.
- [ ] The program compiles clean with `gcc -std=c11 -Wall -Wextra -pedantic -g`.

---

## Summary

- Binary is read column by column; hex is how you write those columns. One hex digit is four bits.
- `%u` prints the quantity, `%x` / `%02x` print the pattern. There is no binary `printf` specifier.
- `&` keeps bits, `|` forces bits on, `^` toggles, `~` flips everything (mask it back down).
- `<< n` multiplies by `2^n`, `>> n` divides by `2^n`, for unsigned values with no bits lost.
- Set/clear/toggle/test are `|=`, `&= ~`, `^=`, and `& ... != 0`.
- Shift UB in two flavors: count `>=` width, and any shift of a negative signed value. Unsigned plus a range check removes both.

Next: combining these operators into flags, masks, and packed fields.

---

#### [Previous](../01-recursion-and-algorithms/exercises.md) | [Next](./02-part.md)
