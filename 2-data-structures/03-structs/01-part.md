# Structs

So far you have grouped data with arrays: a run of values of the **same** type. A `struct` groups values of **different** types into a single thing with a name.

There is one thing to say up front, because it changes how you read every line that follows: **C has no classes.** There is no `class`, no `struct` method, no inheritance, no private section. A `struct` is a plain memory layout, and the functions that work on it are ordinary functions with ordinary parameters. Everything else in this chapter is that fact taken seriously.

---

## Defining a Struct

```c
struct Point {
    int x;
    int y;
};
```

Read it out loud: *a struct tag `Point`, containing an `int` member `x` and an `int` member `y`.* Nothing is created by that declaration — no memory is reserved, no variable exists. It only gives a name to a shape.

Now create a variable of that shape:

```c
struct Point p;
```

and access members with the dot operator:

```c
p.x = 10;
p.y = 20;
```

Full program:

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

int main(void) {
    struct Point p;

    p.x = 10;
    p.y = 20;

    printf("p.x = %d\n", p.x);
    printf("p.y = %d\n", p.y);
    printf("sum = %d\n", p.x + p.y);

    return 0;
}
```

```text
p.x = 10
p.y = 20
sum = 30
```

The order matters: `p.x = 10;` before `p.x` exists is a compile error, because `struct Point p;` did not initialize anything. Use `= {0}` if you want zeros.

Two variables of the same struct are two separate objects:

```c
struct Point a = {1, 2};
struct Point b = {3, 4};
```

`a` and `b` have no more relationship than two `int`s.

---

## Initializing

The brace form matches the member order in the definition:

```c
struct Point p = {3, 7};     /* x = 3, y = 7 */
```

It is positional. Reorder the struct definition and every initializer in your program silently changes meaning, so for anything with more than two members, use **designated initializers** (C99):

```c
struct Point p = {.x = 3, .y = 7};
struct Point q = {.y = 7, .x = 3};   /* same thing, order does not matter */
```

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

struct Rectangle {
    int width;
    int height;
};

int main(void) {
    struct Point p = {3, 7};            /* positional: x = 3, y = 7            */
    struct Point q = {.y = 7, .x = 3};  /* designated: order does not matter */
    struct Rectangle r = {.height = 2, .width = 5};

    printf("p = (%d, %d)\n", p.x, p.y);
    printf("q = (%d, %d)\n", q.x, q.y);
    printf("r = %d x %d\n", r.width, r.height);

    struct Point empty = {0};
    printf("empty = (%d, %d)\n", empty.x, empty.y);

    return 0;
}
```

```text
p = (3, 7)
q = (3, 7)
r = 5 x 2
empty = (0, 0)
```

### Two Rules About Missing Members

Any member you do not mention is set to **zero**. This is not a special case — the compiler zero-fills the whole object and then writes what you gave it:

```c
struct Point p = {.x = 5};   /* p.y is 0, not garbage */
```

And assigning one struct to another copies every member:

```c
struct Point b = a;   /* a full member-by-member copy */
```

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

int main(void) {
    struct Point a = {.x = 5};
    struct Point b = a;   /* assignment copies every member */

    printf("a    = (%d, %d)\n", a.x, a.y);
    printf("b    = (%d, %d)\n", b.x, b.y);

    b.y = 99;
    printf("after b.y = 99:\n");
    printf("a    = (%d, %d)  <- untouched\n", a.x, a.y);
    printf("b    = (%d, %d)\n", b.x, b.y);

    return 0;
}
```

```text
a    = (5, 0)
b    = (5, 0)
after b.y = 99:
a    = (5, 0)  <- untouched
b    = (5, 99)
```

Note that `a.y` was `0` even though the initializer never mentioned it. `struct Point a;` alone would have left it as garbage; `struct Point a = {0};` does not.

---

## Reading With `scanf`

`scanf` needs an address. For a struct member, that address is `&p.x` — the `&` applies to the member, not to the struct:

```c
struct Point p;
scanf("%d %d", &p.x, &p.y);
```

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

int main(void) {
    struct Point p;

    printf("Enter x and y: ");
    if (scanf("%d %d", &p.x, &p.y) != 2) {
        printf("invalid input\n");
        return 1;
    }

    printf("read (%d, %d)\n", p.x, p.y);

    int sum = p.x + p.y;
    printf("sum = %d\n", sum);

    return 0;
}
```

```text
Enter x and y: 3 4
read (3, 4)
sum = 7
```

`p.x + p.y` is plain arithmetic on two `int`s. Once the values are *out* of the struct, the struct is out of the picture.

For a `char *` member you pass the member directly, because the member already holds a pointer:

```c
struct Person {
    char name[32];
    int  age;
};

struct Person p;
scanf("%31s %d", p.name, &p.age);   /* p.name, not &p.name */
```

---

## A Struct in Memory

Members are stored **in declaration order, one after another**, with no gaps you control. `p` is eight bytes of memory that happen to have two names:

```text
  struct Point p;      sizeof(struct Point) == 8 on x86-64

  offset  0                                                          offset 7
  +---------------+---------------+
  |     x = 10    |     y = 20    |
  +---------------+---------------+
  &p.x                          &p.y

  p.x  <--- p, 4 bytes --->  p.y
```

Proof, runnable:

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

int main(void) {
    struct Point p = {10, 20};

    printf("sizeof(struct Point) = %zu bytes\n", sizeof(struct Point));
    printf("sizeof p.x           = %zu bytes\n", sizeof p.x);
    printf("sizeof p.y           = %zu bytes\n", sizeof p.y);

    unsigned long base  = (unsigned long)(size_t)&p;
    unsigned long addrx = (unsigned long)(size_t)&p.x;
    unsigned long addry = (unsigned long)(size_t)&p.y;

    printf("offset of x = %lu\n", addrx - base);
    printf("offset of y = %lu\n", addry - base);

    return 0;
}
```

```text
sizeof(struct Point) = 8 bytes
sizeof p.x           = 4 bytes
sizeof p.y           = 4 bytes
offset of x = 0
offset of y = 4
```

This is why `p.x + 1` and `p.y` are not the same address, and why `&p.x + 1` happens to equal `&p.y` here. What the diagram does **not** show is padding: a struct with mixed types usually has gaps. The third part of this chapter takes that apart with `offsetof`.

---

## `typedef`: Dropping the Word `struct`

Without a typedef, every use of the type spells out `struct`:

```c
struct Point p;            /* declare */
struct Point origin;       /* declare */
struct Point make(void);   /* return a value */
void shift(struct Point *p);   /* parameter */
```

That is four times you type a word that carries no information. `typedef` creates a plain alias so the tag is not needed:

```c
typedef struct {
    int x;
    int y;
} Point;
```

```c
#include <stdio.h>

struct Point {      /* the tag is "struct Point" */
    int x;
    int y;
};

typedef struct {    /* the alias is just "Point" */
    int x;
    int y;
} Point;

int main(void) {
    struct Point tagged = {1, 2};
    Point aliased = {3, 4};

    printf("tagged  = (%d, %d)  declared as struct Point tagged\n", tagged.x, tagged.y);
    printf("aliased = (%d, %d)  declared as Point aliased\n", aliased.x, aliased.y);

    struct Point same = tagged;   /* one type, spelled two ways */
    printf("same    = (%d, %d)\n", same.x, same.y);

    printf("sizeof(Point)        = %zu\n", sizeof(Point));
    printf("sizeof(struct Point) = %zu\n", sizeof(struct Point));

    return 0;
}
```

```text
tagged  = (1, 2)  declared as struct Point tagged
aliased = (3, 4)  declared as Point aliased
same    = (1, 2)
sizeof(Point)        = 8
sizeof(struct Point) = 8
```

`Point` and `struct Point` are the same type when they come from one typedef. The form that keeps both is:

```c
typedef struct Point {
    int x;
    int y;
} Point;
```

Now the tag exists (needed for self-referential structs, and for forward declarations in a header) *and* the alias exists. Use this form for any type that other files will include.

### The Tradeoff

| | `struct Point` | `typedef ... Point` |
| --- | --- | --- |
| Typing | verbose at every use | short: `Point p;` |
| Risk of a name clash | low, names live in the `struct` namespace | real: `Point` is a plain identifier |
| Two headers defining `Point` | no clash | a redefinition error |
| Self-reference | `struct Node *next;` | `struct Node *next;` (still needs the tag) |
| Readable in small programs | fine | better |

House rules that hold up in practice:

- Small, single-file program: `typedef struct { ... } Point;` is fine.
- Anything in a header included by several files: `typedef struct Point { ... } Point;`.
- Keep the tag even with a typedef. It costs nothing and it is the only thing that can self-reference.

---

## Passing Structs to Functions

C has exactly one parameter-passing mechanism: by value. For a struct, "by value" means **the whole struct is copied**.

```c
void move_by_value(struct Point p, int dx, int dy) {
    p.x += dx;
    p.y += dy;
}
```

Inside, `p` is a complete, separate copy. When the function returns, the copy is discarded.

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

static void move_by_value(struct Point p, int dx, int dy) {
    p.x += dx;
    p.y += dy;
    printf("  inside move_by_value: p = (%d, %d)\n", p.x, p.y);
}

static void move_by_pointer(struct Point *p, int dx, int dy) {
    p->x += dx;
    p->y += dy;
    printf("  inside move_by_pointer: p = (%d, %d)\n", p->x, p->y);
}

int main(void) {
    struct Point a = {10, 20};

    printf("before: a = (%d, %d)\n", a.x, a.y);
    move_by_value(a, 5, 5);
    printf("after move_by_value:   a = (%d, %d)\n", a.x, a.y);

    move_by_pointer(&a, 5, 5);
    printf("after move_by_pointer: a = (%d, %d)\n", a.x, a.y);

    /* p->x is exactly (*p).x */
    struct Point *p = &a;
    p->x += 1;
    (*p).x += 1;
    printf("after p->x += 1 and (*p).x += 1: a.x = %d\n", a.x);

    return 0;
}
```

```text
before: a = (10, 20)
  inside move_by_value: p = (15, 25)
after move_by_value:   a = (10, 20)
  inside move_by_pointer: p = (15, 25)
after move_by_pointer: a = (15, 25)
after p->x += 1 and (*p).x += 1: a.x = 17
```

Read the two "after" lines together. The by-value call did the same arithmetic, printed the same numbers inside the function, and changed nothing outside it. That is not a bug in C; that is C.

```text
   caller's memory                     function's memory

   +--------------------+              +--------------------+
   | a.x = 10           |  copy        | p.x = 10           |
   | a.y = 20           |  ------->    | p.y = 20           |
   +--------------------+              +--------------------+
          ^                                       |
          |                                       | p.x += 5; p.y += 5
          |                                       v
          |                              +--------------------+
          |                              | p.x = 15           |  changed, then thrown away
          |                              | p.y = 25           |
          +---- still (10, 20) ---------+--------------------+
                    no effect
```

### By Pointer: `&p` and `->`

To modify the caller's struct, pass its address. Then members are reached with `->`:

```c
void move_by_pointer(struct Point *p, int dx, int dy) {
    p->x += dx;
    p->y += dy;
}
```

The two operators:

| Expression | Use when | Means |
| --- | --- | --- |
| `p.x` | `p` is a `struct Point` | member `x` of `p` |
| `p->x` | `p` is a `struct Point *` | member `x` of the struct `p` points to |
| `(*p).x` | same as above, written out | dereference, then member |

`p->x` is **exactly** `(*p).x`. The arrow is not an operator with its own meaning; it is shorthand the language provides so you do not have to parenthesize the dereference. The last two lines of the example above prove it: `p->x += 1` and `(*p).x += 1` moved `a.x` from 15 to 17.

The rule of thumb: **`.` for structs, `->` for pointers to structs.** If you are ever unsure, write `(*p).x` — it is always correct.

### Which Should You Pass?

| The function | Pass |
| --- | --- |
| only reads the struct | by value, or `const struct Point *` for a big struct |
| modifies the struct | `struct Point *p`, called as `f(&p)` |
| modifies nothing and the struct is large | `const struct Point *p` — no copy at all |

A `Point` with two `int`s is 8 bytes, so copying it is cheaper than the indirection. A struct with a 4 KB buffer inside should be passed by `const` pointer even for reading.

Read-only pointer parameters are a promise the compiler checks:

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

static void print_point(const struct Point *p) {
    printf("(%d, %d)\n", p->x, p->y);
    /* p->x = 0;   would not compile: p points to a const struct Point */
}

int main(void) {
    struct Point p = {7, 8};

    print_point(&p);

    return 0;
}
```

```text
(7, 8)
```

Uncomment that line and the compiler stops you:

```text
error: assignment of member 'x' in read-only object
   10 |     p->x = 0;
      |          ^
```

---

## Checklist

```text
[ ] struct defines a shape; a variable declaration creates the object
[ ] struct Point is a different type from Point unless a typedef says otherwise
[ ] Prefer designated initializers; any member you skip is zero
[ ] scanf("%d", &p.x) — the & goes on the member
[ ] Members are laid out in declaration order, contiguously
[ ] Pass a struct by value only if the caller must not see the change
[ ] Otherwise pass struct Point * and use ->
[ ] Add const to every struct pointer you do not modify
```

---

#### [Previous](../02-strings/exercises.md) | [Next](./02-part.md)
