# Exercise List 2 - Bit Manipulation

All exercises use unsigned types (`uint8_t`, `uint32_t` from `<stdint.h>`), compile with `gcc -std=c11 -Wall -Wextra -pedantic -g`, and keep every shift count in `0 <= n < WIDTH`. Helpers are `static`, shared masks are `static const` or `enum`.

---

## Exercise 1: Print Bits and Count Ones

Write a function that prints a `uint8_t` in binary and a function that counts how many bits are 1 (the population count):

```c
#include <stdio.h>
#include <stdint.h>

static void print_bits_u8(uint8_t v);
static unsigned popcount_u8(uint8_t v);

int main(void) {
    static const uint8_t inputs[] = { 0x00, 0x05, 0xAC, 0xFF };
    for (size_t i = 0; i < sizeof(inputs) / sizeof(inputs[0]); i++) {
        printf("0x%02x = ", inputs[i]);
        print_bits_u8(inputs[i]);
        printf(" ones=%u\n", popcount_u8(inputs[i]));
    }
    return 0;
}
```

```text
0x00 = 0000 0000 ones=0
0x05 = 0000 0101 ones=2
0xac = 1010 1100 ones=4
0xff = 1111 1111 ones=8
```

Requirements:

- `print_bits_u8` prints exactly 8 characters `0`/`1` with a space between the nibbles (`0000 0101`, not `00000101`). Promote to `unsigned int` once, then shift the unsigned copy.
- `popcount_u8` uses only shifts and `& 1u` in a loop over the 8 positions (`(v >> i) & 1u`). No lookup table.
- Verify: `0x00 -> 0`, `0xFF -> 8`, `0xAC -> 4`. Count by hand from the printed bits.
- Extra credit: write `print_bits_u32` (groups of 8, `%08x` label) and confirm `0xDEADBEEF` has 24 ones.

---

## Exercise 2: Set, Clear, Toggle, Test

Implement the four single-bit idioms as functions and demonstrate each on `0xAC`:

```c
#include <stdio.h>
#include <stdint.h>

#define BIT(n) (1u << (n))

static uint8_t set_bit(uint8_t v, unsigned n);
static uint8_t clear_bit(uint8_t v, unsigned n);
static uint8_t toggle_bit(uint8_t v, unsigned n);
static int test_bit(uint8_t v, unsigned n);

int main(void) {
    uint8_t v = 0xAC;
    printf("start:        0x%02x\n", v);
    printf("set bit 0:    0x%02x\n", set_bit(v, 0));
    printf("clear bit 7:  0x%02x\n", clear_bit(v, 7));
    printf("toggle bit 2: 0x%02x\n", toggle_bit(v, 2));
    printf("test bit 5:   %d\n", test_bit(v, 5));
    printf("test bit 0:   %d\n", test_bit(v, 0));
    return 0;
}
```

```text
start:        0xac
set bit 0:    0xad
clear bit 7:  0x2c
toggle bit 2: 0xa8
test bit 5:   1
test bit 0:   0
```

Requirements:

- Set is `v | BIT(n)`, clear is `v & ~BIT(n)`, toggle is `v ^ BIT(n)`, test is `((unsigned)v & BIT(n)) != 0u`. Cast results back to `uint8_t`.
- Guard the position: if `n >= 8u`, return `v` unchanged (set/clear/toggle) or `0` (test). Shifting by `>=` width is undefined behavior.
- Verify each line against the starting pattern `1010 1100` by hand: setting bit 0 gives `1010 1101`, clearing bit 7 gives `0010 1100`.
- Extra credit: add `set_bit32` / `clear_bit32` for `uint32_t` with a `n >= 32u` guard and test bit 31 with `1u << 31u`.

---

## Exercise 3: Permission Flags (Grant / Revoke / Check)

Model Unix-style `rwx` permissions as three flags and implement grant, revoke, and check:

```c
#include <stdio.h>
#include <stdint.h>

enum Perm {
    PERM_READ = 1u << 0,
    PERM_WRITE = 1u << 1,
    PERM_EXEC = 1u << 2
};

static uint32_t perm_grant(uint32_t set, uint32_t flag);
static uint32_t perm_revoke(uint32_t set, uint32_t flag);
static int perm_has(uint32_t set, uint32_t flag);
static void perm_print(uint32_t set); /* prints e.g. "r-x" */

int main(void) {
    uint32_t p = 0u;
    p = perm_grant(p, (uint32_t)(PERM_READ | PERM_WRITE));
    printf("granted rw:  ");
    perm_print(p);
    printf(" (0x%02x) read=%d exec=%d\n", p,
           perm_has(p, (uint32_t)PERM_READ),
           perm_has(p, (uint32_t)PERM_EXEC));
    p = perm_grant(p, (uint32_t)PERM_EXEC);
    printf("granted x:   ");
    perm_print(p);
    printf(" (0x%02x)\n", p);
    p = perm_revoke(p, (uint32_t)PERM_WRITE);
    printf("revoked w:   ");
    perm_print(p);
    printf(" (0x%02x) write=%d\n", p,
           perm_has(p, (uint32_t)PERM_WRITE));
    return 0;
}
```

```text
granted rw:  rw- (0x03) read=1 exec=0
granted x:   rwx (0x07)
revoked w:   r-x (0x05) write=0
```

Requirements:

- Grant combines with `|`, revoke clears with `& ~flag`, check tests with `(set & flag) != 0u`. Revoking twice must stay revoked (prove `^=` would be wrong in a comment).
- `perm_print` prints `r`/`w`/`x` or `-` per position, so `0x05` prints `r-x`.
- Demonstrate the all-vs-any distinction: with `p = r-x`, `(p & (R|X)) == (R|X)` is true (has all) but `(p & (R|W|X)) == (R|W|X)` is false while `(p & (R|W|X)) != 0` is true.
- Extra credit: add `PERM_ADMIN = 1u << 3` and a `perm_has_all(set, need)` helper using `(set & need) == need`.

---

## Exercise 4: Pack and Unpack a Date or an RGB Color

Pick one (RGB is recommended) and write exact round-trip pack/unpack functions. The skeleton below packs an RGB color into a `uint32_t` as `0xRRGGBB`:

```c
#include <stdio.h>
#include <stdint.h>
#include <assert.h>

typedef struct {
    uint8_t r;
    uint8_t g;
    uint8_t b;
} Rgb;

static uint32_t rgb_pack(Rgb c);
static Rgb rgb_unpack(uint32_t packed);

int main(void) {
    Rgb orange = { 255u, 165u, 0u };
    uint32_t packed = rgb_pack(orange);
    Rgb back = rgb_unpack(packed);
    printf("rgb(%u,%u,%u) -> 0x%06x -> rgb(%u,%u,%u)\n",
           orange.r, orange.g, orange.b, packed,
           back.r, back.g, back.b);
    assert(back.r == orange.r && back.g == orange.g && back.b == orange.b);
    assert(rgb_pack(back) == packed);
    printf("round-trip OK\n");
    return 0;
}
```

```text
rgb(255,165,0) -> 0xffa500 -> rgb(255,165,0)
round-trip OK
```

Alternative (date): pack day (1..31, 5 bits), month (1..12, 4 bits), year offset from 2000 (0..127, 7 bits) into a `uint16_t`:

```text
packed = day | (month << 5) | (year_off << 9)
# input
day=25 month=12 year_off=26
# output
packed=0x3599 date=25/12/2026
round-trip OK
```

Requirements:

- Widen before shifting: `(uint32_t)c.r << 16u`, never shift the `uint8_t` directly (it promotes to signed `int`).
- Unpack with shift-then-mask: `(packed >> 16u) & 0xFFu`, cast back to `uint8_t`.
- Assert the round-trip both ways: unpacked fields equal the input AND `pack(unpack(w)) == w` for at least three values including `0x000000` and `0xFFFFFF`.
- Date variant: `assert` each range on pack (`day 1..31`, `month 1..12`, `year_off < 128`) and mask each field with its width on unpack.
- Extra credit: reserve the top byte of the `uint32_t` for an alpha channel and prove `0x80FFA500` round-trips.

---

## Exercise 5: Power of Two and Next Multiple With Shifts

Use shifts to test for powers of two and to round addresses/sizes up to an alignment — the two shift tricks behind every allocator:

```c
#include <stdio.h>
#include <stdint.h>

static int is_power_of_two(uint32_t v);
static uint32_t next_pow2(uint32_t v);
static uint32_t align_up(uint32_t v, uint32_t align);

int main(void) {
    static const uint32_t probes[] = { 0u, 1u, 5u, 8u, 12u, 16u };
    for (size_t i = 0; i < sizeof(probes) / sizeof(probes[0]); i++) {
        printf("%3u pow2=%d next_pow2=%u align16=%u\n", probes[i],
               is_power_of_two(probes[i]), next_pow2(probes[i]),
               align_up(probes[i], 16u));
    }
    return 0;
}
```

```text
  0 pow2=0 next_pow2=1 align16=0
  1 pow2=1 next_pow2=1 align16=16
  5 pow2=0 next_pow2=8 align16=16
  8 pow2=1 next_pow2=8 align16=16
 12 pow2=0 next_pow2=16 align16=16
 16 pow2=1 next_pow2=16 align16=16
```

Requirements:

- `is_power_of_two`: `v != 0 && (v & (v - 1u)) == 0u`. Explain in a comment why `v - 1` clears the lowest set bit (borrow chain through the trailing zeros).
- `next_pow2`: handle `0 -> 1` explicitly, return `v` unchanged if already a power of two, and build the answer by spreading the top bit down (`v |= v >> 1; v |= v >> 2; ...`) on `uint32_t` only. Never loop a shift by `>= 32`.
- `align_up`: require `align` to be a nonzero power of two (`assert`), then `(v + (align - 1u)) & ~(align - 1u)`. Test `align_up(13, 8) == 16` and `align_up(16, 8) == 16`.
- All arithmetic unsigned. Signed `-` or `>>` anywhere in these three functions is a bug.
- Extra credit: prove `align_up` never overflows for `v <= 0xFFFFFF00u` with alignment 16, and document what happens above that.

---

## Exercise 6: XOR Swap and Hamming Distance

The XOR swap exchanges two integers with no temporary. Implement it, show it works, then explain why you should never use it — and compute the Hamming distance (differing bits) with XOR plus your popcount from exercise 1:

```c
#include <stdio.h>
#include <stdint.h>

static void xor_swap(uint32_t *a, uint32_t *b);
static unsigned popcount_u32(uint32_t v);
static unsigned hamming_u8(uint8_t a, uint8_t b);

int main(void) {
    uint32_t a = 0x12345678u, b = 0xDEADBEEFu;
    printf("before: a=0x%08x b=0x%08x\n", a, b);
    xor_swap(&a, &b);
    printf("after:  a=0x%08x b=0x%08x\n", a, b);
    printf("dist(0xAC, 0x3F) = %u\n", hamming_u8(0xACu, 0x3Fu));
    printf("dist(0xFF, 0xFF) = %u\n", hamming_u8(0xFFu, 0xFFu));
    return 0;
}
```

```text
before: a=0x12345678 b=0xdeadbeef
after:  a=0xdeadbeef b=0x12345678
dist(0xAC, 0x3F) = 4
dist(0xFF, 0xFF) = 0
```

Requirements:

- `xor_swap` is the three-XOR sequence (`*a ^= *b; *b ^= *a; *a ^= *b;`). Guard the aliasing case: `if (a == b) return;`, because swapping a variable with itself through XOR zeroes it.
- `hamming_u8(a, b)` returns `popcount(a ^ b)`: XOR marks exactly the differing positions, popcount counts them. Verify by hand: `0xAC ^ 0x3F = 0x93 = 1001 0011`, which has 4 ones (bits 7, 4, 1, 0).
- Write a comment block giving three reasons to avoid XOR swap in real code: (1) unreadable vs a temporary, (2) no faster — modern compilers and register renaming beat it, (3) the aliasing bug above plus it cannot swap pointed-to aggregates or overlapping memory, unlike a temporary swap.
- Extra credit: implement `hamming_u32` with the `v & (v - 1u)` clear-lowest-bit loop (each iteration removes exactly one `1`) and compare its step count against the shift-and-mask version on `0x80000000`.

---

#### [Previous](./02-part.md) | [Next](../03-function-pointers/01-part.md)
