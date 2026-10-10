# Building a Dynamic Array

Arrays have a fixed size chosen before the program runs. `malloc` gives you a block of a size you choose at run time, and `realloc` lets you change that choice. Put the two together and you can build the one container C does not hand you: a growable array. Every dynamic array in every language is this structure with a different type parameter.

---

## The Problem

```c
int data[100];              /* 100, forever, decided by whoever wrote it */
int *data = malloc(100 * sizeof(int));   /* 100, decided at run time */
```

Both fail the same requirement: the number of items is not known in advance. You cannot make a C array grow. What you can do is keep the array on the heap and ask for more when you need it.

---

## The Vector Pattern

Three fields in a struct, and every growing-array implementation is a variation on this:

```c
typedef struct {
    int  *data;      /* the block, or NULL when empty     */
    size_t size;     /* how many elements are in use       */
    size_t capacity; /* how many elements the block holds  */
} IntVec;
```

```text
  the struct                       the block it points at

  +-------------------+  data     +-----+-----+-----+-----+-----+
  | size = 3          |---------->|  10 |  20 |  30 |     |     |
  | capacity = 8      |           +-----+-----+-----+-----+-----+
  +-------------------+              ^  <---- size ---->^
    the block is bigger
    than the data
```

| Field | Meaning |
| --- | --- |
| `data` | the block; `NULL` while the vector is empty |
| `size` | how many elements are **in use** |
| `capacity` | how many elements **fit** before a realloc is needed |

Keeping `size` and `capacity` separate is the entire trick. If they were one number, every push would need a `realloc`, which copies the whole array — that is O(n) per push, O(n²) overall. With a capacity that grows ahead of the size, most pushes are free.

`data`, `size` and `capacity` also have to travel together. A bare `int *` has no idea how long it is. Wrapping them in a struct makes it impossible to pass one without the other two.

---

## The Complete Implementation

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

typedef struct {
    int  *data;     /* the block malloc/realloc manages; may be NULL when empty */
    size_t size;    /* how many elements are actually in use                        */
    size_t capacity;/* how many elements fit in the block before a realloc        */
} IntVec;

/* OWNERSHIP: v owns its own block. No allocation here, so nothing can fail. */
static void vector_init(IntVec *v) {
    v->data = NULL;
    v->size = 0;
    v->capacity = 0;
}

/* Frees the block and resets the vector. Safe to call twice. */
static void vector_free(IntVec *v) {
    if (v == NULL) {
        return;
    }
    free(v->data);
    v->data = NULL;      /* so a second free(NULL) is a no-op */
    v->size = 0;
    v->capacity = 0;
}

/* Appends one value, growing the block by doubling when it is full.
   Returns 0 on success, -1 if the allocation failed. */
static int vector_push(IntVec *v, int value) {
    if (v->size == v->capacity) {
        size_t new_capacity = (v->capacity == 0) ? 1 : v->capacity * 2;
        int *tmp = realloc(v->data, new_capacity * sizeof(int));

        if (tmp == NULL) {
            return -1;      /* the old block is untouched and still ours */
        }
        v->data = tmp;      /* only the RETURNED pointer may be stored */
        v->capacity = new_capacity;
    }

    v->data[v->size] = value;
    v->size++;
    return 0;
}

/* Borrows the vector. Returns 0 and writes to *out; returns -1 on a bad index. */
static int vector_get(const IntVec *v, size_t index, int *out) {
    if (index >= v->size) {
        return -1;
    }
    *out = v->data[index];
    return 0;
}

int main(void) {
    IntVec v;
    size_t i;
    int value;

    vector_init(&v);

    for (i = 1; i <= 10; i++) {
        if (vector_push(&v, (int)(i * 10)) != 0) {
            fprintf(stderr, "out of memory\n");
            vector_free(&v);
            return 1;
        }
        printf("after push %2zu: size = %zu, capacity = %zu\n",
               i, v.size, v.capacity);
    }

    printf("\n");
    for (i = 0; i < v.size; i++) {
        if (vector_get(&v, i, &value) == 0) {
            printf("%zu: %d\n", i, value);
        }
    }

    if (vector_get(&v, 99, &value) != 0) {
        printf("\nindex 99 rejected: the vector only holds %zu\n", v.size);
    }

    vector_free(&v);

    return 0;
}
```

```text
after push  1: size = 1, capacity = 1
after push  2: size = 2, capacity = 2
after push  3: size = 3, capacity = 4
after push  4: size = 4, capacity = 4
after push  5: size = 5, capacity = 8
after push  6: size = 6, capacity = 8
after push  7: size = 7, capacity = 8
after push  8: size = 8, capacity = 8
after push  9: size = 9, capacity = 16
after push 10: size = 10, capacity = 16

0: 10
1: 20
2: 30
3: 40
4: 50
5: 60
6: 70
7: 80
8: 90
9: 100

index 99 rejected: the vector only holds 10
```

Ten pushes, three reallocations, and the block is now oversized (16 slots for 10 values) — the price of never reallocating on every push.

Note the ownership lines in the comments. `vector_init` and `vector_free` deal with the vector's own memory; `vector_push` manages the block and may grow it; `vector_get` only reads. That separation is what makes the type safe to hand to other code.

---

## Why the Capacity Starts at 1

`new_capacity = (v->capacity == 0) ? 1 : v->capacity * 2` — the special case is not optional. Here is what happens without it:

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

typedef struct {
    int  *data;
    size_t size;
    size_t capacity;
} IntVec;

static void vector_init(IntVec *v) {
    v->data = NULL;
    v->size = 0;
    v->capacity = 0;
}

static int vector_push_buggy(IntVec *v, int value) {
    if (v->size == v->capacity) {
        v->data = realloc(v->data, v->capacity * 2 * sizeof(int));
        if (v->data == NULL) {
            return -1;
        }
        v->capacity = v->capacity * 2;
    }

    v->data[v->size] = value;
    v->size++;
    return 0;
}

int main(void) {
    IntVec v;

    vector_init(&v);

    printf("first push with capacity 0 -> capacity*2 == 0 -> realloc(p, 0)\n");

    if (vector_push_buggy(&v, 42) != 0) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    printf("pushed 42, size = %zu, capacity = %zu, data = %p\n",
           v.size, v.capacity, (void *)v.data);

    v.data[v.size - 1] = 42;   /* may be writing through a NULL-ish pointer */

    free(v.data);
    return 0;
}
```

The first push asks for `0 * 2 * sizeof(int) == 0` bytes. `realloc(p, 0)` is implementation-defined, and the result here is a 1-byte block — so the `int` write at the end lands outside it:

```text
first push with capacity 0 -> capacity*2 == 0 -> realloc(p, 0)
pushed 42, size = 1, capacity = 0, data = 0x5b08e264f020
```

```text
=================================================================
==381848==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x7830ccbe0010 at pc 0x5d6fe6103933 bp 0x7ffdea401d60 sp 0x7ffdea401d50
WRITE of size 4 at 0x7830ccbe0010 thread T0
    #0 0x5d6fe6103932 in vector_push_buggy /tmp/vector_bug.c:26
    #1 0x5d6fe6103ac0 in main /tmp/vector_bug.c:38

0x7830ccbe0011 is located 0 bytes after 1-byte region [0x7830ccbe0010,0x7830ccbe0011)
allocated by thread T0 here:
    #0 0x7c10ce92a6cf in realloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:81
    #1 0x5d6fe6103671 in vector_push_buggy /tmp/vector_bug.c:19

SUMMARY: AddressSanitizer: heap-buffer-overflow /tmp/vector_bug.c:26 in vector_push_buggy
```

`1-byte region`, written with a `WRITE of size 4`. The bug is one missing special case in the growth formula, and it only appears on the very first push — after that `capacity` is nonzero and the arithmetic behaves.

---

## The `realloc` Temporary-Pointer Pitfall

The correct line in `vector_push` is:

```c
int *tmp = realloc(v->data, new_capacity * sizeof(int));
if (tmp == NULL) {
    return -1;
}
v->data = tmp;
```

Not this:

```c
/* WRONG */
v->data = realloc(v->data, new_capacity * sizeof(int));
if (v->data == NULL) {
    return -1;      /* v->data is now NULL and the OLD block is leaked */
}
```

`realloc` is the **only** heap function with this trap, and the reason is that it has two possible outcomes that both involve the old pointer:

| `realloc` outcome | Old block | Correct action |
| --- | --- | --- |
| succeeds, moves | **freed** | use the new pointer; the old one dangles |
| succeeds, in place | still alive, now bigger | use the new pointer (same address) |
| fails | **still alive and still yours** | keep the old pointer, report the error |

Writing `p = realloc(p, ...)` handles the first two rows correctly and the third catastrophically: on failure `p` is overwritten with `NULL` and the surviving block has no pointer anywhere in your program. It is not just a leak — the data is unreachable and unrecoverable.

The temporary-pointer version is correct in all three rows, because `v->data` is only overwritten once a valid pointer is in hand.

```text
  WRONG: p = realloc(p, n)          RIGHT: tmp = realloc(p, n)
                                     if (tmp == NULL) return -1;
  +-------+                        +-------+          +-------+
  | p = A |                        | p = A |  realloc | p = A |  realloc
  +-------+  realloc               +-------+ -------> +-------+
  | p = B | ------->              | tmp=NULL|<------- +-------+
  +-------+  fail                        |
                                   keep p, the block survived
```

Use the temporary pointer form **everywhere**, without exception. The cost is one extra variable; the alternative is a leak you will never find.

---

## Why Doubling, and What It Costs

`capacity * 2` is called **amortized** growth. It is not a constant-time guarantee, but it is a constant-time *average*, and the reason is a counting argument. This program measures it:

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

int main(void) {
    int *data = NULL;
    size_t capacity = 0;
    size_t total_copies = 0;
    size_t reallocs = 0;
    size_t i;

    printf("push   capacity   copies this push   total copies\n");
    printf("----   --------   ----------------   ------------\n");

    for (i = 1; i <= 16; i++) {
        size_t copies = 0;

        if (i > capacity) {
            size_t new_capacity = (capacity == 0) ? 1 : capacity * 2;
            int *tmp = realloc(data, new_capacity * sizeof(int));

            if (tmp == NULL) {
                free(data);
                return 1;
            }
            data = tmp;
            copies = capacity;
            capacity = new_capacity;
            reallocs++;
        }

        data[i - 1] = (int)i;
        total_copies += copies;
        printf("%4zu   %8zu   %16zu   %12zu\n", i, capacity, copies, total_copies);
    }

    printf("\n%zu pushes caused %zu reallocs and %zu element copies\n",
           (size_t)16, reallocs, total_copies);
    printf("that is %.1f copies per push, and the capacity never exceeds 2n\n",
           (double)total_copies / 16.0);

    free(data);
    return 0;
}
```

```text
push   capacity   copies this push   total copies
----   --------   ----------------   ------------
   1          1                  0              0
   2          2                  1              1
   3          4                  2              3
   4          4                  0              3
   5          8                  4              7
   6          8                  0              7
   7          8                  0              7
   8          8                  0              7
   9         16                  8             15
  10         16                  0             15
  11         16                  0             15
  12         16                  0             15
  13         16                  0             15
  14         16                  0             15
  15         16                  0             15
  16         16                  0             15

16 pushes caused 5 reallocs and 15 element copies
that is 0.9 copies per push, and the capacity never exceeds 2n
```

Sixteen pushes, five reallocations, **fifteen** element copies in total. (The first "realloc" is the 0-to-1 growth, which is really the first `malloc`.) Every realloc copies everything that is already there, and each one is immediately followed by a run of free pushes:

```text
  capacity  1    2    4    4    8    8    8    8    16   16   16 ...
  copies    -    X    X    .    X    .    .    .    X    .    .  ...
             |    |    |         |              |              |
             |    |    |         |              |              +-- 8 free pushes
             |    |    |         |              +----------------- copy 8 elements
             |    |    |         +-------------------------------- free pushes
             |    |    +------------------------------------------- copy 4 elements
             |    +------------------------------------------------ copy 2 elements
             +----------------------------------------------------- copy 1 element
```

The sum is a geometric series: for 16 elements the copies are 1 + 2 + 4 + 8 = 15, which is `n - 1`. For any n, the total work to build the vector is **O(n)**, and the cost per push is **less than one element copy** on average — even though a single unlucky push copies the whole array.

That is the **amortized O(1)** guarantee. It holds because:

- After the k-th doubling the capacity is 2^k, so the next 2^k pushes need no realloc at all.
- A given element is therefore copied at most once per doubling, so at most log₂(n) times over the vector's whole life.
- Summing 1 + 2 + 4 + ... + 2^(k-1) gives 2^k − 1 = n − 1.

This is why every real dynamic array doubles. Growing by a constant (+10) instead gives O(n²) total work. Growing by 1.5x needs more reallocations than doubling for the same memory headroom.

The costs you pay:

- **Memory.** Up to 2x the space of the data, because capacity is never shrunk on its own.
- **One O(n) realloc occasionally.** A push can stall while the array is copied. (This is why `realloc` is allowed to return a different address: it may not be able to extend in place.)
- **A realloc you do not control.** To reclaim the memory you call `realloc` down to the size you want — with the usual temporary-pointer rule.

---

## The `realloc` Temp-Pointer Rule, Applied to Shrinking

```c
int *shrunk = realloc(data, wanted * sizeof(int));
if (shrunk == NULL) {
    /* the original block is still valid: keep using `data` */
} else {
    data = shrunk;
    capacity = wanted;
}
free(data);
```

Three things worth noting: the failure branch leaves `data` usable, the success branch is the only place `data` changes, and the `free` happens on the pointer that survived.

---

## This Is a `List<T>`

If you write C# or Java or Go, you have used this. The mapping is exact:

| C | C# / Java |
| --- | --- |
| `IntVec` | `List<int>` |
| `vector_init(&v)` | `var list = new List<int>();` |
| `vector_push(&v, x)` | `list.Add(x);` |
| `vector_get(&v, i, &out)` | `list[i]` (after a bounds check) |
| `v.size` | `list.Count` |
| `vector_free(&v)` | the garbage collector, or `Dispose()` |
| `v.capacity` | internal to the runtime; you never see it |

The differences are all consequences of C not having a runtime:

| | C | C# |
| --- | --- | --- |
| Index out of range | your own `index >= size` check | throws `IndexOutOfRangeException` |
| Indexing syntax | `v.data[i]` — you get the raw block | `v[i]` |
| Memory | freed by you, by hand | collected when unreachable |
| Allocating a lot | each block is a real `malloc` | pooled and reused by the GC |

The "you get the raw block" line is the one that catches people coming from a managed language. `v.data` is the address of a heap block, and **you may index it with any `int` you like**. `vector_get` exists precisely so that the safe path is also the easy path.

---

## Checklist

```text
[ ] data + size + capacity always travel together in one struct
[ ] data is NULL when the vector is empty, not a 1-element array
[ ] capacity starts at 1, never 0
[ ] Grow by doubling: amortized O(1) per push
[ ] realloc through a temporary pointer, always
[ ] On realloc failure, the old block is still valid - keep the pointer
[ ] Use only the pointer realloc returned
[ ] free sets data to NULL, so the destructor is safe to call twice
[ ] Every access goes through a function that checks against size
[ ] A push that reallocates invalidates every index pointer into the block
[ ] Run under -fsanitize=address,undefined
```

---

#### [Previous](./03-part.md) | [Next](./exercises.md)
