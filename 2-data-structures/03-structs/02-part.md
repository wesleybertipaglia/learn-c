# Working with Many Structs

A single struct is a shape. This part is about containers of them: arrays of structs, structs inside structs, pointers to structs, and the one shape that changes everything — a struct that points at itself.

---

## Arrays of Structs

An array of structs is exactly what you expect, and the indexing composes with the dot operator:

```c
struct Point points[5];

points[0].x = 10;
points[2].y = 40;
```

`points[i]` is a whole `struct Point`, so `points[i].x` reads a member of the i-th one.

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

int main(void) {
    struct Point points[5];

    printf("sizeof points       = %zu bytes\n", sizeof points);
    printf("sizeof points[0]    = %zu bytes\n", sizeof points[0]);
    printf("number of elements  = %zu\n", sizeof points / sizeof points[0]);

    for (int i = 0; i < 5; i++) {
        points[i].x = i * 10;
        points[i].y = i * 10 + 1;
    }

    for (int i = 0; i < 5; i++) {
        printf("points[%d] = (%d, %d)\n", i, points[i].x, points[i].y);
    }

    /* the count idiom, once per program */
    size_t count = sizeof points / sizeof points[0];
    int total = 0;
    for (size_t i = 0; i < count; i++) {
        total += points[i].x;
    }
    printf("sum of x = %d\n", total);

    return 0;
}
```

```text
sizeof points       = 40 bytes
sizeof points[0]    = 8 bytes
number of elements  = 5
points[0] = (0, 1)
points[1] = (10, 11)
points[2] = (20, 21)
points[3] = (30, 31)
points[4] = (40, 41)
sum of x = 100
```

`sizeof points / sizeof points[0]` is the element count. It only works where the array is a real array — see the next part, where a pointer has no length of its own.

### A Complete Example: Students

This is the shape of almost every "collection of records" program you will write. Note how every function takes `const Student *` plus a `count`: the array decays to a pointer, so the length has to travel alongside it.

```c
#include <stdio.h>

#define MAX_STUDENTS 5

typedef struct {
    char name[32];
    int  age;
    double grade;
} Student;

static void print_student(const Student *s) {
    printf("%-10s %3d     %.1f\n", s->name, s->age, s->grade);
}

/* Prints the whole array. The array decays to a pointer, so the size must
   travel with it. */
static void print_all(const Student *students, int count) {
    printf("%-10s %3s     %s\n", "NAME", "AGE", "GRADE");
    printf("-----------------------------\n");
    for (int i = 0; i < count; i++) {
        print_student(&students[i]);
    }
}

/* Returns the index of the highest grade, or -1 if the array is empty. */
static int best_index(const Student *students, int count) {
    if (count <= 0) {
        return -1;
    }

    int best = 0;
    for (int i = 1; i < count; i++) {
        if (students[i].grade > students[best].grade) {
            best = i;
        }
    }
    return best;
}

static double average(const Student *students, int count) {
    if (count <= 0) {
        return 0.0;
    }

    double total = 0.0;
    for (int i = 0; i < count; i++) {
        total += students[i].grade;
    }
    return total / (double)count;
}

int main(void) {
    Student students[MAX_STUDENTS] = {
        {"Ana",    19, 8.5},
        {"Bruno",  21, 9.7},
        {"Carla",  20, 7.0},
        {"Diego",  22, 9.7},
        {"Eva",    18, 6.5}
    };

    int count = MAX_STUDENTS;

    print_all(students, count);

    int best = best_index(students, count);
    printf("\nbest    : %s with %.1f (index %d)\n",
           students[best].name, students[best].grade, best);

    printf("average : %.2f\n", average(students, count));

    /* a pass-by-pointer function CAN change the array */
    students[0].grade = 10.0;
    printf("\nafter students[0].grade = 10.0, average is %.2f\n",
           average(students, count));

    return 0;
}
```

```text
NAME       AGE     GRADE
-----------------------------
Ana         19     8.5
Bruno       21     9.7
Carla       20     7.0
Diego       22     9.7
Eva         18     6.5

best    : Bruno with 9.7 (index 1)
average : 8.28

after students[0].grade = 10.0, average is 8.58
```

`best_index` returns `-1` for an empty array rather than pretending it found something. Every one of these functions guards against `count <= 0` — a function that loops `count` times must decide what zero means.

---

## Nested Structs

A struct may contain another struct **by value**:

```c
struct Circle {
    struct Point center;   /* the Point lives inside the Circle */
    int radius;
};
```

Access it with a second dot, chained: `c.center.x`.

The consequence is that assigning copies:

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

struct Circle {
    struct Point center;   /* the Point lives inside the Circle, by value */
    int radius;
};

int main(void) {
    struct Point center = {10, 20};

    struct Circle c = {{0, 0}, 5};
    c.center = center;      /* copies the two ints into the nested member */

    printf("c.center = (%d, %d), radius = %d\n",
           c.center.x, c.center.y, c.radius);
    printf("sizeof(struct Circle) = %zu bytes\n", sizeof(struct Circle));
    printf("center is still (%d, %d) -> the nested struct was a copy\n",
           center.x, center.y);

    center.x = 999;
    printf("after center.x = 999: c.center.x = %d\n", c.center.x);

    return 0;
}
```

```text
c.center = (10, 20), radius = 5
sizeof(struct Circle) = 12 bytes
center is still (10, 20) -> the nested struct was a copy
after center.x = 999: c.center.x = 10
```

```text
  struct Circle c;                       12 bytes

  +------------------------+---------------+
  |  c.center.x = 10       |  c.radius = 5 |
  +------------------------+---------------+
       \_____ two more ints, copied ___/
```

### A Struct Containing a Pointer to Another Struct

The alternative is to store the address:

```c
struct Marker {
    struct Point   origin;   /* by value: a copy lives here */
    struct Point * anchor;   /* by pointer: only an address lives here */
};
```

Now the two share one `Point`:

```c
#include <stdio.h>

typedef struct {
    int x;
    int y;
} Point;

typedef struct {
    Point   origin;    /* by value: the Point is INSIDE this struct */
    Point * anchor;    /* by pointer: this struct only holds an address */
} Marker;

int main(void) {
    Point anchor = {100, 100};

    Marker m;
    m.origin.x = 0;
    m.origin.y = 0;
    m.anchor = &anchor;

    printf("origin = (%d, %d)  -- a copy inside Marker\n", m.origin.x, m.origin.y);
    printf("anchor = (%d, %d)  -- the original, reached through a pointer\n",
           m.anchor->x, m.anchor->y);
    printf("sizeof(Marker) = %zu bytes (Point + address), not %zu\n",
           sizeof(Marker), sizeof(Marker) + sizeof(Point));

    anchor.x = 555;
    printf("\nafter anchor.x = 555:\n");
    printf("  anchor->x  = %d   <- shared, both see the change\n", m.anchor->x);
    printf("  origin.x   = %d   <- copied, unchanged\n", m.origin.x);

    /* Marker never allocated anything, so nobody needs to free anything here.
       The stack variable `anchor` owns its own memory. */

    return 0;
}
```

```text
origin = (0, 0)  -- a copy inside Marker
anchor = (100, 100)  -- the original, reached through a pointer
sizeof(Marker) = 16 bytes (Point + address), not 24

after anchor.x = 555:
  anchor->x  = 555   <- shared, both see the change
  origin.x   = 0   <- copied, unchanged
```

The tradeoff in one table:

| | By value `Point origin` | By pointer `Point *anchor` |
| --- | --- | --- |
| Size of the containing struct | grows by `sizeof(Point)` | grows by `sizeof(Point *)` |
| Copying the container | copies the nested data too | copies only an address |
| Changes visible to both | no, they are separate | yes, one shared object |
| Lifetime | automatic, dies with the container | must outlive the container |
| Risk | none | dangling pointer if the target dies first |

Default to **by value**. Reach for a pointer when the nested struct is large, when the two must see the same data, or when there is a cycle (a parent pointing at a child that points back).

---

## Pointers to Structs and Dynamic Allocation

A local `struct Point p;` lives on the stack and disappears at the end of the block. When you need the struct to outlive the function that created it, allocate it:

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int x;
    int y;
} Point;

/* This function allocates, so the caller owns the result and must free it. */
static Point *make_point(int x, int y) {
    Point *p = malloc(sizeof(Point));
    if (p == NULL) {
        return NULL;
    }
    p->x = x;
    p->y = y;
    return p;
}

static void translate(Point *p, int dx, int dy) {
    p->x += dx;
    p->y += dy;
}

int main(void) {
    Point *p = make_point(10, 20);
    if (p == NULL) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    printf("p = (%d, %d)\n", p->x, p->y);

    translate(p, 5, -3);
    printf("after translate: p = (%d, %d)\n", p->x, p->y);

    /* (*p).x and p->x are the same expression */
    printf("(*p).x = %d, p->x = %d\n", (*p).x, p->x);

    free(p);   /* paired with the malloc inside make_point */
    p = NULL;

    return 0;
}
```

```text
p = (10, 20)
after translate: p = (15, 17)
(*p).x = 15, p->x = 15
```

### Always Write `malloc(sizeof(Point))`

The number `8` is what `struct Point` happens to be **on your machine, today**. Add a `double`, compile for a 32-bit target, or move to a compiler with different alignment, and the answer changes. The only correct call is the one that asks the compiler:

```c
Point *p = malloc(sizeof(Point));   /* always this */
```

Never a hand-computed size. Here is what the wrong version actually does — the compiler catches it, and so does the sanitizer:

```c
Point *p = malloc(4);    /* WRONG: Point is 8 bytes */

p->x = 1;
p->y = 2;                /* writes past the end of the block */
```

```text
warning: allocation of insufficient size '4' for type 'Point' with size '8' [-Walloc-size]
   11 |     Point *p = malloc(4);
      |                ~~~~~~~^

==370956==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x78fd33fe0014
WRITE of size 4 at 0x78fd33fe0014 thread T0
    #0 0x6373dfc91373 in main /tmp/malloc_bad.c:14
0x78fd33fe0014 is located 0 bytes after 4-byte region [0x78fd33fe0010,0x78fd33fe0014)
allocated by thread T0 here:
    #0 0x7cdd35d2b60f in malloc
    #1 0x6373dfc9127f in main /tmp/malloc_bad.c:11

SUMMARY: AddressSanitizer: heap-buffer-overflow /tmp/malloc_bad.c:14 in main
```

`malloc(4)` gave you 4 bytes. `p->y` starts at offset 4, which is the first byte outside the block.

That is the same trap as `malloc(n)` for an array of `int`s, one level up. The rule covers both:

```c
Point  *p = malloc(sizeof(Point));       /* one struct          */
int    *a = malloc(n * sizeof(int));      /* n ints, never malloc(n) */
char   *s = malloc(strlen(x) + 1);        /* n bytes: a real byte count */
```

The next part of this module is entirely about `malloc`, `free` and the mistakes between them.

---

## Returning Structs

A function may return a struct by value. It works exactly like returning an `int` — the struct is copied into the return slot and then copied into the variable you assigned to. For a small struct this is free in practice; for a big one it is a real memcpy each time.

```c
#include <stdio.h>

typedef struct {
    int x;
    int y;
} Point;

typedef struct {
    int id;
    char name[64];
    double scores[16];
} Student;

/* A small struct: 8 bytes, copying it is cheaper than a pointer. */
static Point midpoint(Point a, Point b) {
    Point m;

    m.x = (a.x + b.x) / 2;
    m.y = (a.y + b.y) / 2;

    return m;
}

/* A big struct: every one of these copies the whole thing, twice. */
static Student promote(Student s) {
    s.id += 1;
    return s;
}

int main(void) {
    Point a = {0, 0};
    Point b = {10, 20};

    Point m = midpoint(a, b);
    printf("midpoint = (%d, %d)\n", m.x, m.y);
    printf("sizeof(Point)   = %zu bytes\n", sizeof(Point));

    Student s = {7, "Ana", {8.5, 9.0}};
    Student t = promote(s);

    printf("s.id = %d, t.id = %d  <- the copy was modified\n", s.id, t.id);
    printf("sizeof(Student) = %zu bytes per copy\n", sizeof(Student));

    return 0;
}
```

```text
midpoint = (5, 10)
sizeof(Point)   = 8 bytes
s.id = 7, t.id = 8  <- the copy was modified
sizeof(Student) = 200 bytes per copy
```

Read `promote` carefully. It takes 200 bytes by value and returns 200 bytes by value — 400 bytes copied to change one `int`. `s.id` is still 7 because the function only ever saw a copy.

Rules that follow from this:

| The struct is | Then |
| --- | --- |
| a few bytes (a pair of coords, a key) | return by value, pass by value |
| larger than a cache line or so | return via an out-parameter: `int fill(Student *out)` and return `int` as success/failure |
| going to be stored in a container | allocate it once and pass `Student *` around |

Returning a struct is the one place where by-value parameters are the right default, because it avoids making the caller manage memory for the result.

---

## Self-Referential Structs

A struct may contain a pointer to its own type. This is legal as soon as the member is a **pointer**, because a pointer is an address (a fixed size) and not a copy of the object:

```c
typedef struct Node {
    int value;
    struct Node *next;   /* the tag is required: a Node cannot contain a Node */
} Node;
```

Two details that trip everyone up:

1. **The tag `struct Node` is mandatory.** `Node *next;` inside the definition does not compile, because the typedef name is not finished until the closing brace. Use `struct Node *next;`.
2. **The size is bounded.** `sizeof(Node)` is `sizeof(int) + sizeof(pointer)` — no matter how long the chain gets. 16 bytes on x86-64, whether the list holds 3 nodes or 3 million.

```text
  Node *head
     |
     v
  +---------------+      +---------------+      +---------------+
  | value = 3     |  ->  | value = 2     |  ->  | value = 1     | -> NULL
  | next  = ------+      | next  = ------+      | next  = NULL  |
  +---------------+      +---------------+      +---------------+
   16 bytes                16 bytes                16 bytes
   allocated               allocated               allocated
   separately             separately             separately
```

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
    int value;
    struct Node *next;   /* the tag is required here: a Node cannot contain a Node */
} Node;

/* Ownership: this function allocates one node, so the caller owns the result
   and must free it (or hand it to a function that frees the whole list). */
static Node *list_push(Node *head, int value) {
    Node *n = malloc(sizeof(Node));
    if (n == NULL) {
        return head;   /* out of memory: return what we had */
    }

    n->value = value;
    n->next  = head;

    return n;   /* the new head */
}

static void list_free(Node *head) {
    while (head != NULL) {
        Node *next = head->next;
        free(head);
        head = next;
    }
}

int main(void) {
    Node *head = NULL;

    head = list_push(head, 1);
    head = list_push(head, 2);
    head = list_push(head, 3);

    for (Node *p = head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");

    printf("sizeof(Node) = %zu bytes (the 'next' member is an address, not a copy)\n",
           sizeof(Node));

    list_free(head);

    return 0;
}
```

```text
3 -> 2 -> 1 -> NULL
sizeof(Node) = 16 bytes (the 'next' member is an address, not a copy)
```

Every node in that list was a separate `malloc`, so freeing them takes a walk down the chain. `list_push` returns the new head because the head moved — that is the standard idiom, and it is why every push reads `head = list_push(head, v)`.

This is the linked list. The next part covers `malloc` and `free` on their own, which is what you will need before building one for real.

---

## Checklist

```text
[ ] points[i].x composes indexing with member access
[ ] sizeof arr / sizeof arr[0] gives the element count of a real array
[ ] Every function taking an array also takes its length
[ ] A nested struct by value is copied; a nested pointer is shared
[ ] malloc(sizeof(Point)) always, never a hand-computed size
[ ] Check every malloc against NULL
[ ] Return small structs by value; pass large ones as const pointers
[ ] A struct may contain a pointer to itself, but never itself
```

---

#### [Previous](./01-part.md) | [Next](./03-part.md)
