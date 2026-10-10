# Unions, Bitfields and Layout

A struct answers "how do I put different types together?". Three other features answer neighbouring questions:

- **unions** — how do I store one of several types in the same bytes?
- **bitfields** — how do I pack small values into fewer bytes?
- **padding and alignment** — why is my struct bigger than the sum of its parts?

---

## Unions: Several Types, One Block of Memory

A union is declared like a struct but behaves completely differently. Every member starts at the **same address**, and the union is exactly as large as its **largest** member:

```c
union Value {
    int   i;
    float f;
};
```

```text
  struct with 2 members (8 bytes)     union with 2 members (4 bytes)

  +---------------+---------------+   +-----------------------+
  | i = 65        | f = ...       |   | i and f BOTH start    |
  +---------------+---------------+   | here                  |
    offset 0        offset 4         +-----------------------+
                                       4 bytes
                                    sizeof = max(4, 4)
```

Writing one member and reading another **reinterprets** the same bits. Nothing is copied, nothing is converted — the bytes are simply read under a different type.

```c
#include <stdio.h>

union Value {
    int   i;
    float f;
};

int main(void) {
    union Value v;

    printf("sizeof(int)   = %zu\n", sizeof(int));
    printf("sizeof(float) = %zu\n", sizeof(float));
    printf("sizeof(union Value) = %zu\n", sizeof(union Value));

    v.i = 1080033280;          /* 0x40600000 */
    printf("\nv.i = %d\n", v.i);

    v.f = 3.5f;                /* the same 32 bits, read as a float */
    printf("v.f = %.1f\n", v.f);
    printf("v.i = %d   <- the same 4 bytes, read as an int\n", v.i);

    return 0;
}
```

```text
sizeof(int)   = 4
sizeof(float) = 4
sizeof(union Value) = 4

v.i = 1080033280
v.f = 3.5
v.i = 1080033280   <- the same 4 bytes, read as an int
```

Three rules follow:

1. Only **one member is valid at a time**. Reading a member you did not write last is meaningless.
2. `sizeof` is the size of the **largest** member, plus any padding needed to align it.
3. A union with a large member makes every variable of that type large. A union of `int` and `double[1000]` is 8000 bytes the moment you declare a variable of it.

### The Type Tag Pattern

A union by itself is dangerous, because nothing records which member is currently live. The fix is to store a tag next to it:

```text
  struct Value, 8 bytes

  +-----------------------+-----------------------+
  | type (enum)           | value (union)         |
  +-----------------------+-----------------------+
  | 0 = TYPE_INT          | .i  or .f             |
  | 1 = TYPE_FLOAT        | (same 4 bytes)        |
  +-----------------------+-----------------------+

  the tag says which member of the union is allowed to be read
```

```c
#include <stdio.h>

typedef enum {
    TYPE_INT,
    TYPE_FLOAT
} ValueType;

typedef struct {
    ValueType type;       /* the tag: which member is valid right now */
    union {
        int   i;
        float f;
    } value;              /* named union: always write value.i */
} Value;

static void print_value(const Value *v) {
    switch (v->type) {
        case TYPE_INT:
            printf("int   : %d\n", v->value.i);
            break;
        case TYPE_FLOAT:
            printf("float : %.2f\n", v->value.f);
            break;
        default:
            printf("unknown type\n");
            break;
    }
}

int main(void) {
    Value a = {TYPE_INT,   {.i = 42}};
    Value b = {TYPE_FLOAT, {.f = 3.5f}};

    print_value(&a);
    print_value(&b);

    a.type = TYPE_FLOAT;    /* retag: now the same 4 bytes mean something else */
    a.value.f = 9.25f;
    print_value(&a);

    printf("\nsizeof(Value) = %zu bytes\n", sizeof(Value));

    return 0;
}
```

```text
int   : 42
float : 3.50
float : 9.25

sizeof(Value) = 8 bytes
```

This is the shape of every variant type in every other language: a type tag, a payload, and a `switch` that refuses to read the wrong member. The `default` branch is not decoration — a tag value the switch does not know about must not fall through into a `printf` of garbage.

### Unions in Real Code

| Where you meet it | Why |
| --- | --- |
| Variant / tagged value types | as above |
| Saving memory for a "one of these" field | a `Variant` of 6 possible types costs only the largest one plus the tag |
| Peeking at the representation of a value | `memcpy` into a union to inspect bits — the standard-sanctioned alternative to type punning through a pointer cast |
| Interpreter value stacks | an `int` or a `double` or a pointer, one per stack slot |

They are a real tool, but they are never the first tool. Reach for a struct first.

---

## Bitfields: Packing Small Values

A bitfield declares a member with an explicit width in bits:

```c
struct Flags {
    unsigned int is_active : 1;   /* 0 or 1            */
    unsigned int level     : 3;   /* 0..7              */
    unsigned int reserved  : 4;   /* unused, kept at 0 */
};
```

The point is size. Three `unsigned int` members take 12 bytes; the bitfield version takes 4.

```c
#include <stdio.h>

struct Flags {
    unsigned int is_active : 1;   /* 0 or 1                */
    unsigned int level     : 3;   /* 0..7                  */
    unsigned int reserved  : 4;   /* unused, kept at zero  */
};

struct Plain {
    unsigned int is_active;
    unsigned int level;
    unsigned int reserved;
};

int main(void) {
    struct Flags f = {0};

    printf("sizeof(struct Flags) = %zu bytes\n", sizeof(struct Flags));
    printf("sizeof(struct Plain) = %zu bytes\n", sizeof(struct Plain));

    f.is_active = 1;
    f.level     = 5;
    printf("\nis_active = %u, level = %u\n", f.is_active, f.level);

    /* values come from somewhere else at runtime, so the compiler cannot
       complain: the extra bits are simply dropped */
    int from_user = 9;
    f.level = (unsigned int)from_user;   /* 9 & 0b111 == 1 */
    printf("after level = 9 (a 3-bit field): level = %u\n", f.level);

    from_user = 2;
    f.is_active = (unsigned int)from_user;   /* 2 & 0b1 == 0 */
    printf("after is_active = 2 (a 1-bit field): is_active = %u\n", f.is_active);

    return 0;
}
```

```text
sizeof(struct Flags) = 4 bytes
sizeof(struct Plain) = 12 bytes

is_active = 1, level = 5
after level = 9 (a 3-bit field): level = 1
after is_active = 2 (a 1-bit field): is_active = 0
```

Assigning a value too large for the field silently truncates it to the low bits. A literal like `f.level = 9;` gets a `-Woverflow` warning at compile time; a value coming from a variable at runtime does not, so **validate the range yourself** if the input is not already constrained.

### The Warning That Matters

The C standard does not define how a compiler must lay bitfields out. It may reorder them, pack them in any order, and put them in any type it likes. So:

> **Never read or write a struct containing bitfields with `fread`, `fwrite`, `write`, or by casting to bytes.** Write your own packing, or store each field as a separate `unsigned int` and pack by hand.

```c
struct Flags {
    unsigned int a : 1;
    unsigned int b : 3;
    unsigned int c : 4;
    unsigned int d : 24;   /* 1 + 3 + 4 + 24 = 32 bits exactly */
};

struct Flags2 {
    unsigned int a : 1;
    unsigned int b : 3;
    unsigned int c : 4;
    unsigned int e : 3;    /* 11 bits total: the compiler pads to 32 */
};
```

```c
#include <stdio.h>

struct Flags {
    unsigned int a : 1;
    unsigned int b : 3;
    unsigned int c : 4;
    unsigned int d : 24;   /* 1 + 3 + 4 + 24 = 32 bits exactly */
};

struct Flags2 {
    unsigned int a : 1;
    unsigned int b : 3;
    unsigned int c : 4;
    unsigned int e : 3;    /* 11 bits total: the compiler pads to 32 */
};

int main(void) {
    printf("sizeof(struct Flags)  = %zu bytes\n", sizeof(struct Flags));
    printf("sizeof(struct Flags2) = %zu bytes\n", sizeof(struct Flags2));

    printf("\nthere is no offsetof for a bitfield:\n");
    printf("  bitfields have no address, so they cannot be pointed at\n");

    return 0;
}
```

```text
sizeof(struct Flags)  = 4 bytes
sizeof(struct Flags2) = 4 bytes

there is no offsetof for a bitfield:
  bitfields have no address, so they cannot be pointed at
```

11 bits of fields occupy 32 bits of memory. That is alignment, not a compiler bug.

A bitfield also has **no address**, which the compiler enforces:

```text
error: attempt to take address of bit-field structure member 'a'
    4 | int main(void) { printf("%zu\n", offsetof(struct Flags, a)); return 0; }
      |                                                  ^~~~~
```

So you cannot take `&f.level` and pass it to `scanf`, and you cannot `memcpy` an individual bitfield out. `scanf` cannot read a bitfield at all — for `level` you need a temporary `unsigned int`, scan into it, then assign.

Bitfields are a good fit for in-memory status flags, register-mirroring structures in device drivers, and protocol fields you pack yourself. For anything that touches a file, prefer explicit shifts.

---

## Padding and Alignment

The diagram in the first part showed `struct Point` as exactly two `int`s back to back. That is a special case. In general the compiler inserts **padding bytes** so that each member sits at an address its type can be read from efficiently.

The rule on x86-64:

| Type | Alignment | Size |
| --- | --- | --- |
| `char` | 1 | 1 |
| `short` | 2 | 2 |
| `int` | 4 | 4 |
| `long` | 8 | 8 |
| `double` | 8 | 8 |
| pointer | 8 | 8 |

Each member is placed at the next offset that is a multiple of its own alignment. The struct's total size is then rounded up to a multiple of its **strictest** member's alignment.

```c
#include <stdio.h>
#include <stddef.h>

struct Mixed {
    char  tag;        /* 1 byte */
    int   value;      /* 4 bytes */
    char  flag;       /* 1 byte */
    double weight;    /* 8 bytes */
};

int main(void) {
    printf("sizeof(struct Mixed)   = %zu bytes\n", sizeof(struct Mixed));
    printf("sum of the members    = %zu bytes\n",
           sizeof(char) + sizeof(int) + sizeof(char) + sizeof(double));

    printf("\noffsetof tag    = %zu\n", offsetof(struct Mixed, tag));
    printf("offsetof value  = %zu\n", offsetof(struct Mixed, value));
    printf("offsetof flag   = %zu\n", offsetof(struct Mixed, flag));
    printf("offsetof weight = %zu\n", offsetof(struct Mixed, weight));

    printf("\noffsets are always multiples of 8 here, because of the double.\n");

    return 0;
}
```

```text
sizeof(struct Mixed)   = 24 bytes
sum of the members    = 14 bytes

offsetof tag    = 0
offsetof value  = 4
offsetof flag   = 8
offsetof weight = 16

offsets are always multiples of 8 here, because of the double.
```

`offsetof` comes from `<stddef.h>` and gives the byte offset of a member from the start of the struct. Here is the whole 24 bytes:

```text
  offset 0                     24
  +-----+-------+-----+---------+-----------+-------+
  | tag |  PAD  |value|   PAD   |   weight  | PAD?  |
  |  1  |   3   |  4  |    5    |     8     |   0   |
  +-----+-------+-----+---------+-----------+-------+

   1 real    3 pad  4 real     5 pad      8 real     0

  1 + 3 + 4 + 5 + 8 = 21 bytes of layout, rounded up to 24 for the 8-byte alignment
  the 3 bytes at 1..3 exist so that `value` lands on a 4-byte boundary
  the 5 bytes at 9..13 exist so that `weight` lands on an 8-byte boundary
```

Note the second `flag`: it sits at offset 8, not 9, and wastes five bytes instead of one. That is the cost of the following `double` dragging the layout up to an 8-byte grid.

### Reordering Removes Padding

```c
#include <stdio.h>
#include <stddef.h>

struct Bad {
    char  tag;
    int   value;
    char  flag;
    double weight;
};

struct Good {
    double weight;
    int   value;
    char  tag;
    char  flag;
};

int main(void) {
    printf("sizeof(struct Bad)  = %zu bytes (14 bytes of data, 10 bytes of padding)\n",
           sizeof(struct Bad));
    printf("sizeof(struct Good) = %zu bytes (14 bytes of data, 2 bytes of padding)\n",
           sizeof(struct Good));

    printf("\nBad:\n");
    printf("  offsetof tag    = %zu\n", offsetof(struct Bad, tag));
    printf("  offsetof value  = %zu\n", offsetof(struct Bad, value));
    printf("  offsetof flag   = %zu\n", offsetof(struct Bad, flag));
    printf("  offsetof weight = %zu\n", offsetof(struct Bad, weight));

    printf("\nGood:\n");
    printf("  offsetof weight = %zu\n", offsetof(struct Good, weight));
    printf("  offsetof value  = %zu\n", offsetof(struct Good, value));
    printf("  offsetof tag    = %zu\n", offsetof(struct Good, tag));
    printf("  offsetof flag   = %zu\n", offsetof(struct Good, flag));

    return 0;
}
```

```text
sizeof(struct Bad)  = 24 bytes (14 bytes of data, 10 bytes of padding)
sizeof(struct Good) = 16 bytes (14 bytes of data, 2 bytes of padding)

Bad:
  offsetof tag    = 0
  offsetof value  = 4
  offsetof flag   = 8
  offsetof weight = 16

Good:
  offsetof weight = 0
  offsetof value  = 8
  offsetof tag    = 12
  offsetof flag   = 13
```

Same data, 8 bytes saved per instance, from nothing but reordering the members. **Rule: declare members from largest alignment to smallest.**

### Padding Multiplies

For a single struct the waste is a few bytes. In an array it is per element:

```c
#include <stdio.h>
#include <stddef.h>

struct Row {
    char  tag;
    int   value;
    char  flag;
};

int main(void) {
    printf("sizeof(struct Row) = %zu bytes, but only %zu hold data\n",
           sizeof(struct Row), sizeof(char) + sizeof(int) + sizeof(char));

    printf("\nrow  tag   value  flag\n");
    printf("---  -----  -----  -----\n");

    struct Row rows[3] = {
        {'a', 1, 1},
        {'b', 2, 0},
        {'c', 3, 1}
    };

    for (int i = 0; i < 3; i++) {
        printf("%3d  %5c  %5d  %5d\n",
               i, rows[i].tag, rows[i].value, rows[i].flag);
    }

    printf("\n%d rows cost %zu bytes instead of %zu\n",
           3, sizeof rows, 3 * (sizeof(char) + sizeof(int) + sizeof(char)));

    return 0;
}
```

```text
sizeof(struct Row) = 12 bytes, but only 6 hold data

row  tag   value  flag
---  -----  -----  -----
  0      a      1      1
  1      b      2      0
  2      c      3      1

3 rows cost 36 bytes instead of 18
```

Twice the memory for the same data. In a program holding a million of these rows you waste a megabyte.

Two related details:

- `sizeof` a struct is fixed by the compiler for that target. Never hardcode a size in a protocol or an assertion.
- A struct is **never** smaller than its largest member. If `sizeof` comes out smaller than one of the types inside, something is wrong with the program.

---

## Anonymous Structs and Unions (C11)

C11 lets a struct or union member have no name at all. Its members are then reached directly on the parent:

```text
  struct Event            struct Tagged
  +----------------+      +-------+-----------------------+
  | id = 1         |      | type | i (4) | d (8)         |
  +----------------+      +-------+-----------------------+
  | x = 10         |        ^
  +----------------+      reached as t.i or t.d, no extra name
  | y = 20         |
  +----------------+      the union is 8 bytes, sized by the double
```

```c
#include <stdio.h>

/* C11 anonymous struct: no tag, no member name.
   v.x reaches the x member directly. */
struct Event {
    int id;
    struct {
        int x;
        int y;
    };                  /* anonymous struct member */
};

/* C11 anonymous union: the members live at the top level of the struct */
struct Tagged {
    int type;           /* 0 = int, 1 = double */
    union {
        int    i;
        double d;
    };                  /* anonymous union member */
};

int main(void) {
    struct Event e = {.id = 1, .x = 10, .y = 20};

    printf("e.id = %d, e.x = %d, e.y = %d\n", e.id, e.x, e.y);
    printf("no member name to write: it is just e.x\n");

    struct Tagged t = {.type = 1, .d = 2.5};

    printf("\ntype = %d, d = %.1f, i = %d\n", t.type, t.d, t.i);

    t.type = 0;
    t.i = 7;
    printf("type = %d, i = %d\n", t.type, t.i);

    return 0;
}
```

```text
e.id = 1, e.x = 10, e.y = 20
no member name to write: it is just e.x

type = 1, d = 2.5, i = 0
type = 0, i = 7
```

The convenience is real but narrow: an anonymous union works well as the tag-and-payload combination from the previous section. For anything you want to pass around or nest, use a **named** union member — `v.value.i` reads better than `v.i` when there are other integers in the struct.

One rule with anonymous members: **you cannot have two of the same type.** Two anonymous structs both containing an `int x` would make `v.x` ambiguous, so it does not compile. Give one of them a name.

---

## Named Types for APIs vs Anonymous Types for Locals

The choice of how to declare a struct comes down to one question: does anyone else need to name this type?

| Situation | Declaration |
| --- | --- |
| Used in one function only | anonymous, declared inside the function |
| Used across several functions in one file | named type at file scope |
| Used in a header by several `.c` files | `typedef struct Name { ... } Name;` |
| A value in a tagged union | named union member, so `.value.i` stays readable |
| A struct you never pass around | anonymous member is fine |

An anonymous local struct, kept out of the way:

```c
#include <stdio.h>

typedef struct {
    int   width;
    int   height;
} Size;

/* A struct declared inside a function is local to it: no tag, no typedef,
   no chance of colliding with anything in another file. */
static long area_of(int w, int h) {
    struct {                      /* local, private, unnamed type */
        int w;
        int h;
    } s = {w, h};

    return (long)s.w * (long)s.h;
}

int main(void) {
    Size s = {3, 4};

    printf("area = %ld\n", area_of(s.width, s.height));

    return 0;
}
```

```text
area = 12
```

That last example is included to show what **not** to do — a struct declared inside a function when a `typedef` at file scope says the same thing more clearly. The rule is simple: anonymous and local types are fine for genuinely private, single-use shapes; anything reused gets a name at file scope.

---

## Checklist

```text
[ ] Union size == size of the largest member, not the sum
[ ] Only one union member is valid at a time; always keep a tag
[ ] Never trust a union read without a matching switch on the tag
[ ] Bitfields truncate on assignment: validate the range yourself
[ ] Never fread/fwrite a struct containing bitfields
[ ] sizeof a struct can exceed the sum of its members: check with offsetof
[ ] Order members largest-alignment-first to reduce padding
[ ] Anonymous struct/union members are C11, and only one per type
```

---

#### [Previous](./02-part.md) | [Next](./exercises.md)
