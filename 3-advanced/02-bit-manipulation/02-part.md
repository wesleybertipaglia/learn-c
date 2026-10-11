# Flags, Masks and Bit Fields

Part 1 gave you the operators. This part gives you the patterns built from them: flag sets where each bit is a yes/no answer, integer packing where a struct is squeezed into one word, and C bit fields where the compiler does the shifting for you.

The running theme: **one integer, many meanings.** A permission word, a file attribute set, a network header — all of them are a single unsigned value read through a set of named masks.

```c
#include <stdio.h>
#include <stdint.h>
#include <assert.h>

enum Perm { PERM_READ = 1u << 0, PERM_WRITE = 1u << 1, PERM_EXEC = 1u << 2 };

int main(void) {
    uint32_t p = (uint32_t)(PERM_READ | PERM_WRITE);
    printf("p = 0x%02x can_read=%d can_exec=%d\n",
           p, (p & (uint32_t)PERM_READ) != 0u, (p & (uint32_t)PERM_EXEC) != 0u);
    assert(((p & (uint32_t)PERM_READ) != 0u));
    return 0;
}
```

```text
p = 0x03 can_read=1 can_exec=0
```

Compile with the usual flags:

```text
gcc -std=c11 -Wall -Wextra -pedantic -g flags.c -o flags
```

Everything here stays unsigned (`uint8_t`, `uint16_t`, `uint32_t` from `<stdint.h>`). Masks are `static const` or `enum` constants, helpers are `static`, and shift counts are always checked to be below the width.

---

## Flags: One Bit per Yes/No Answer

A flag is a named single bit. A flag set is several flags OR-ed into one integer. The three operations are grant (OR), revoke (AND-NOT), and check (AND):

```text
  bit:    7   6   5   4   3   2   1   0
        +---+---+---+---+---+---+---+---+
        | 0 | 0 | 0 | 0 | 0 | X | W | R |
        +---+---+---+---+---+---+---+---+
                              ^   ^   ^
                              |   |   +-- bit 0: READ
                              |   +------ bit 1: WRITE
                              +---------- bit 2: EXEC

  READ | WRITE = 0000 0011 = 0x03
```

Define each flag as one shifted bit. An `enum` is the idiomatic home for them: the constants are compile-time, visible in the debugger, and grouped under one name.

```c
#include <stdio.h>
#include <stdint.h>

enum Perm {
    PERM_READ  = 1u << 0, /* 0x01 */
    PERM_WRITE = 1u << 1, /* 0x02 */
    PERM_EXEC  = 1u << 2  /* 0x04 */
};

static uint32_t perm_grant(uint32_t set, uint32_t flag) {
    return set | flag;
}

static uint32_t perm_revoke(uint32_t set, uint32_t flag) {
    return set & (uint32_t)(~flag);
}

static int perm_has(uint32_t set, uint32_t flag) {
    return (set & flag) != 0u;
}

static void perm_print(uint32_t set) {
    putchar((set & (uint32_t)PERM_READ) != 0u ? 'r' : '-');
    putchar((set & (uint32_t)PERM_WRITE) != 0u ? 'w' : '-');
    putchar((set & (uint32_t)PERM_EXEC) != 0u ? 'x' : '-');
}

int main(void) {
    uint32_t p = 0u;

    p = perm_grant(p, (uint32_t)PERM_READ);
    p = perm_grant(p, (uint32_t)PERM_WRITE);
    printf("after grant r,w: 0x%02x ", p);
    perm_print(p);
    printf("  read=%d exec=%d\n", perm_has(p, (uint32_t)PERM_READ),
           perm_has(p, (uint32_t)PERM_EXEC));

    p = perm_grant(p, (uint32_t)(PERM_READ | PERM_WRITE | PERM_EXEC));
    printf("after grant all: 0x%02x ", p);
    perm_print(p);
    putchar('\n');

    p = perm_revoke(p, (uint32_t)PERM_WRITE);
    printf("after revoke w:  0x%02x ", p);
    perm_print(p);
    printf("  write=%d\n", perm_has(p, (uint32_t)PERM_WRITE));

    /* testing several flags at once: all must be present */
    uint32_t need = (uint32_t)(PERM_READ | PERM_EXEC);
    printf("has r+x? %d\n", (p & need) == need);
    return 0;
}
```

```text
after grant r,w: 0x03 rw-  read=1 exec=0
after grant all: 0x07 rwx
after revoke w:  0x05 r-x  write=0
has r+x? 1
```

Three rules that prevent every classic flag bug:

1. **Combine with `|`, test with `&`.** `PERM_READ | PERM_WRITE` builds the set. `(p & FLAG) != 0u` tests one flag. `(p & need) == need` tests that *all* of several flags are present; `(p & need) != 0u` tests that *any* is present. Pick the wrong comparison and `r--` passes an `rwx` check.
2. **Revoke with `& ~flag`.** `p &= ~FLAG` clears exactly the flagged bits. `p ^= FLAG` toggles — revoking twice re-grants, which is never what "revoke" means.
3. **Keep flags unsigned and single-bit.** Every enumerator must be exactly one `1u << n` with distinct `n`. `PERM_ALL = PERM_READ | PERM_WRITE | PERM_EXEC` is a combination, not a flag — never pass it to a function that expects one bit if that function counts or shifts it.

| Operation | Code | Reads as |
| --------- | ---- | -------- |
| Grant | `p \|= FLAG` | turn these bits on |
| Revoke | `p &= ~FLAG` | turn these bits off |
| Toggle | `p ^= FLAG` | flip these bits |
| Has one | `(p & FLAG) != 0` | is this bit on? |
| Has all | `(p & NEED) == NEED` | are all these bits on? |
| Has any | `(p & NEED) != 0` | is at least one on? |
| Clear all | `p = 0` | no flags |

An all-or-nothing check is worth a picture. With `p = 0x05` (`r-x`) and `need = 0x05`:

```text
  p    = 0000 0101
  need = 0000 0101
  p & need = 0000 0101 == need  -> has all: YES

  need = 0000 0111 (rwx)
  p & need = 0000 0101 != need  -> has all: NO (w missing)
  p & need = 0000 0101 != 0     -> has any: YES
```

---

## Masks Wider Than One Bit

Real fields are wider than one bit: a 3-bit priority, a 4-bit type, an 8-bit length. Each field needs a WIDTH, a SHIFT (where it starts), and a MASK (the width-worth of 1s moved into place).

```text
  8-bit status word example:

   bit:    7   6   5   4   3   2   1   0
         +---+---+---+---+---+---+---+---+
         |  E  |   TYPE    |    CODE     |
         +---+---+---+---+---+---+---+---+
           ^    ^^^^^^^     ^^^^^^^^^
           |    bits 5..3   bits 2..0
           bit 7 (1-bit flag)
```

Build masks from width and shift, never by hand-counting bits:

```c
#include <stdio.h>
#include <stdint.h>

static const unsigned CODE_SHIFT = 0u;
static const unsigned CODE_WIDTH = 3u;
static const unsigned TYPE_SHIFT = 3u;
static const unsigned TYPE_WIDTH = 3u;
static const unsigned ERR_SHIFT = 7u;

static uint32_t field_mask(unsigned width, unsigned shift) {
    /* width in 1..31, shift + width <= 32: caller guarantees it */
    return ((1u << width) - 1u) << shift;
}

static uint32_t field_get(uint32_t word, unsigned width, unsigned shift) {
    return (word >> shift) & ((1u << width) - 1u);
}

static uint32_t field_put(uint32_t word, uint32_t value,
                          unsigned width, unsigned shift) {
    uint32_t max = (1u << width) - 1u;
    uint32_t clean = word & (uint32_t)(~(max << shift));
    return clean | ((value & max) << shift);
}

int main(void) {
    printf("CODE mask = 0x%02x\n", field_mask(CODE_WIDTH, CODE_SHIFT));
    printf("TYPE mask = 0x%02x\n", field_mask(TYPE_WIDTH, TYPE_SHIFT));
    printf("ERR  mask = 0x%02x\n", field_mask(1u, ERR_SHIFT));

    uint32_t w = 0u;
    w = field_put(w, 5u, CODE_WIDTH, CODE_SHIFT);
    w = field_put(w, 3u, TYPE_WIDTH, TYPE_SHIFT);
    w = field_put(w, 1u, 1u, ERR_SHIFT);
    printf("packed = 0x%02x\n", w);
    printf("code=%u type=%u err=%u\n",
           field_get(w, CODE_WIDTH, CODE_SHIFT),
           field_get(w, TYPE_WIDTH, TYPE_SHIFT),
           field_get(w, 1u, ERR_SHIFT));
    return 0;
}
```

```text
CODE mask = 0x07
TYPE mask = 0x38
ERR  mask = 0x80
packed = 0x9d
code=5 type=3 err=1
```

`field_put` does three things in order: mask the incoming value down (`value & max`) so a too-large input cannot bleed into neighbors, clear the target field (`& ~mask`), then OR the shifted value in. Skipping the first step is the bug: `field_put(w, 9, 3-bit-field)` would silently set a bit outside the field.

`(1u << width) - 1u` is the standard "width ones" idiom: width 3 gives `0b1000 - 1 = 0b0111`. It requires `width >= 1` and `width < 32` — width 32 would shift by the full width, which is undefined. For a full-width field there is no mask to build; the mask is `0xFFFFFFFFu`.

---

## Packing a Struct Into One Integer

A struct with small ranges wastes whole `int`s. Packing stores each member as a bit field inside one unsigned word. The struct stays readable; the integer is what gets stored or sent.

Example: an RGB color. Each channel is 0..255 (8 bits). Three channels fit exactly in 24 bits of a `uint32_t`:

```text
  uint32_t packed RGB (0xRRGGBB):

  +--------+--------+--------+
  |   R    |   G    |   B    |   8 bits each
  +--------+--------+--------+
  bits      bits      bits
  23..16    15..8     7..0
```

```c
#include <stdio.h>
#include <stdint.h>
#include <assert.h>

typedef struct {
    uint8_t r;
    uint8_t g;
    uint8_t b;
} Rgb;

static const unsigned R_SHIFT = 16u;
static const unsigned G_SHIFT = 8u;
static const unsigned B_SHIFT = 0u;

static uint32_t rgb_pack(Rgb c) {
    return ((uint32_t)c.r << R_SHIFT) |
           ((uint32_t)c.g << G_SHIFT) |
           ((uint32_t)c.b << B_SHIFT);
}

static Rgb rgb_unpack(uint32_t packed) {
    Rgb c;
    c.r = (uint8_t)((packed >> R_SHIFT) & 0xFFu);
    c.g = (uint8_t)((packed >> G_SHIFT) & 0xFFu);
    c.b = (uint8_t)((packed >> B_SHIFT) & 0xFFu);
    return c;
}

int main(void) {
    Rgb orange = { 255u, 165u, 0u };
    uint32_t packed = rgb_pack(orange);
    Rgb back = rgb_unpack(packed);

    printf("rgb(%u,%u,%u) -> 0x%06x\n", orange.r, orange.g, orange.b, packed);
    printf("unpacked rgb(%u,%u,%u)\n", back.r, back.g, back.b);

    assert(back.r == orange.r && back.g == orange.g && back.b == orange.b);
    assert(rgb_pack(back) == packed); /* round-trip is exact */
    printf("round-trip OK\n");
    return 0;
}
```

```text
rgb(255,165,0) -> 0xffa500
unpacked rgb(255,165,0)
round-trip OK
```

Every shift here is safe by construction: each channel promotes to `uint32_t` before shifting (the `(uint32_t)c.r` cast), and the counts 16, 8, 0 are constants below 32. Unpacking masks with `& 0xFFu` after shifting so higher bits can never leak through, even if the packed word had flags stored in its top byte.

Packing rules:

- Pack with `|`: each `(uint32_t)value << SHIFT` occupies disjoint bits, so OR assembles them.
- Unpack with `>>` then `&`: shift the field down to bit 0, then mask to its width.
- Widen before shifting: cast the narrow member to the wide word type *first*. Shifting a `uint8_t` 16 places without widening shifts a promoted signed `int` and risks signed-shift pitfalls on some widths.
- `static_assert`-style range checks (or `assert`) on pack prove the round-trip. The exercises require them.

---

## Bit Fields (`:n`): Letting the Compiler Shift

C lets you declare struct members with a width in bits. The compiler generates the masks and shifts:

```c
#include <stdio.h>
#include <stdint.h>

typedef struct {
    unsigned int code : 3; /* bits 0..2  */
    unsigned int type : 3; /* bits 3..5  */
    unsigned int reserved : 1;
    unsigned int err : 1;  /* bit 7      */
} StatusBits;

int main(void) {
    StatusBits s = { 0 };
    s.code = 5u;
    s.type = 3u;
    s.err = 1u;
    printf("code=%u type=%u err=%u sizeof=%zu\n",
           s.code, s.type, s.err, sizeof(s));
    return 0;
}
```

```text
code=5 type=3 err=1 sizeof=4
```

That looks simpler than `field_put`/`field_get`, and for in-memory flags it is. But bit fields come with a portability warning you must take seriously:

> **Bit fields are not portable across compilers.** The standard does not fix which bit is first, how padding is inserted, whether a plain `int` field is signed or unsigned, or how fields cross storage-unit boundaries. Two compilers (or two optimization flags) may lay the same struct out differently.

Consequences:

| Use | Verdict |
| --- | ------- |
| In-memory flag struct inside one program, one compiler | fine |
| Value sent over a socket, written to a file, mapped onto hardware registers | **do not use bit fields** — use explicit shifts and masks |

The rest of this part uses explicit shifts and masks for everything that leaves the process. Bit fields appear only as a local convenience, never as a wire format. (Unsigned 1-bit fields hold only 0 and 1; assigning anything wider truncates, and signed fields risk UB.)

If you need the layout to be exact — and for protocols and files you always do — write the pack/unpack functions by hand. They are five lines each and behave identically everywhere.

---

## Real Example: Packet Header

A tiny 16-bit packet header packs four fields. This is the shape of real protocol work (IP, TCP, and DNS headers all look like this, only larger):

```text
  16-bit header, MSB on the left:

   bit: 15 14 13 12 11 10  9  8   7  6  5  4  3  2  1  0
        +--+--+--+--+--+--+--+--++--+--+--+--+--+--+--+--+
        | version  |   type    | flags |     length      |
        |  4 bits  |  4 bits   | 4 bits|     4 bits      |
        +--+--+--+--+--+--+--+--++--+--+--+--+--+--+--+--+
        bits 15..12  bits 11..8  7..4      3..0
```

```c
#include <stdio.h>
#include <stdint.h>
#include <assert.h>

static const unsigned HDR_VERSION_SHIFT = 12u;
static const unsigned HDR_VERSION_WIDTH = 4u;
static const unsigned HDR_TYPE_SHIFT = 8u;
static const unsigned HDR_TYPE_WIDTH = 4u;
static const unsigned HDR_FLAGS_SHIFT = 4u;
static const unsigned HDR_FLAGS_WIDTH = 4u;
static const unsigned HDR_LEN_SHIFT = 0u;
static const unsigned HDR_LEN_WIDTH = 4u;

typedef struct {
    uint8_t version; /* 0..15  */
    uint8_t type;    /* 0..15  */
    uint8_t flags;   /* 4 bits */
    uint8_t length;  /* 0..15  */
} Header;

static uint16_t header_pack(Header h) {
    assert(h.version <= 15u && h.type <= 15u);
    assert(h.flags <= 15u && h.length <= 15u);
    return (uint16_t)(((uint16_t)(h.version & 0xFu) << HDR_VERSION_SHIFT) |
                      ((uint16_t)(h.type & 0xFu) << HDR_TYPE_SHIFT) |
                      ((uint16_t)(h.flags & 0xFu) << HDR_FLAGS_SHIFT) |
                      ((uint16_t)(h.length & 0xFu) << HDR_LEN_SHIFT));
}

static Header header_unpack(uint16_t word) {
    Header h;
    h.version = (uint8_t)(((unsigned)word >> HDR_VERSION_SHIFT) & 0xFu);
    h.type = (uint8_t)(((unsigned)word >> HDR_TYPE_SHIFT) & 0xFu);
    h.flags = (uint8_t)(((unsigned)word >> HDR_FLAGS_SHIFT) & 0xFu);
    h.length = (uint8_t)(((unsigned)word >> HDR_LEN_SHIFT) & 0xFu);
    return h;
}

int main(void) {
    Header out = { 2u, 7u, 0xAu, 12u };
    uint16_t wire = header_pack(out);
    Header in = header_unpack(wire);

    printf("header v=%u type=%u flags=0x%x len=%u\n",
           out.version, out.type, out.flags, out.length);
    printf("wire: 0x%04x\n", wire);
    printf("back: v=%u type=%u flags=0x%x len=%u\n",
           in.version, in.type, in.flags, in.length);

    assert(in.version == out.version);
    assert(in.type == out.type);
    assert(in.flags == out.flags);
    assert(in.length == out.length);
    assert(header_pack(in) == wire);
    printf("round-trip OK\n");
    return 0;
}
```

```text
header v=2 type=7 flags=0xa len=12
wire: 0x27ac
back: v=2 type=7 flags=0xa len=12
round-trip OK
```

Check the wire value by hand: version 2 is `0010`, type 7 is `0111`, flags `0xA` is `1010`, length 12 is `1100`. Concatenated: `0010 0111 1010 1100` = `0x27AC`. The `& 0xFu` on every field before shifting is what keeps a bad input like `version = 17` from corrupting the type field — combined with the `assert`, it fails loudly in debug and degrades safely in release.

The Unix `rwx` permission word from the flags section is the same idea at its smallest: three 1-bit fields tested with `&`, combined with `|`. The packet header is the same idea scaled to multi-bit fields with shifts.

---

## Common Bugs

| Bug | Symptom | Fix |
| --- | ------- | --- |
| Testing all-flags with `!= 0` instead of `== need` | `r--` passes an `rwx` check | any-of is `!= 0`, all-of is `== need` |
| Revoking with `^=` | revoke twice re-grants | revoke is `&= ~flag` |
| `field_put` without `value & max` | large input bleeds into the neighbor field | mask the value first, `assert` the range |
| Hand-written mask off by one (`0x07` vs `0x0F`) | field reads half its bits | build masks as `((1u << w) - 1u) << s` |
| `1u << 32` for a full-width mask | undefined behavior | full-width mask is `0xFFFFFFFFu`, no shift |
| Shifting `uint8_t` without widening | promotion to signed `int`, sign surprises | `(uint32_t)member << SHIFT` |
| Serializing a bit-field struct | other end reads garbage | bit fields are local-only; pack by hand for I/O |
| Overlapping field definitions | pack then unpack returns different values | assert `pack(unpack(w)) == w` for boundary inputs |

---

## Checklist

- [ ] I define single-bit flags as `1u << n` enumerators and combine them with `|`.
- [ ] I grant with `|=`, revoke with `&= ~`, test one with `& ... != 0`, test all with `& ... == need`.
- [ ] I build multi-bit masks from `(width, shift)`, never by counting bits by hand.
- [ ] `field_put` masks the value, clears the field, then ORs — in that order.
- [ ] I cast narrow members to `uint32_t`/`uint16_t` before shifting.
- [ ] All shift counts are constants or guarded to be `<` the width.
- [ ] Bit fields (`:n`) appear only in memory-local structs, never in file/network formats.
- [ ] Every pack has an unpack, and a round-trip `assert` proves `unpack(pack(x)) == x`.
- [ ] Clean build with `gcc -std=c11 -Wall -Wextra -pedantic -g`.

---

## Summary

- Flags are named single bits: OR to grant, AND-NOT to revoke, AND to test. `== need` means all, `!= 0` means any.
- Multi-bit fields need `(width, shift, mask)` triples; put = mask value, clear field, OR in.
- Packing turns a struct into one integer for storage or transmission; unpacking reverses it with `>>` and `&`.
- Bit fields ask the compiler to do the shifting, but their layout is implementation-defined: local use only.
- The packet header shows the full pattern — named shifts, masked inputs, asserts, and a round-trip proof.

Next: exercises that make you write every idiom in this part from scratch.

---

#### [Previous](./01-part.md) | [Next](./exercises.md)
