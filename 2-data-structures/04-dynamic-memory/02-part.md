# `calloc`, `realloc` and Ownership

The previous part gave you `malloc` and `free`. Three functions complete the set: `calloc` (zeroed memory), `realloc` (change a block's size), and the one function you should never call — `free` on something you do not own. This part is mostly about **who frees what**, because that question is where the real bugs live.

---

## `calloc`: Memory That Starts at Zero

```c
void *calloc(size_t count, size_t size);
```

`calloc` takes the count and the size **separately** and returns memory with every byte set to zero.

```c
int *values = calloc(10, sizeof(int));   /* 10 ints, all zero */
char *buffer = calloc(256, 1);           /* 256 bytes, all zero */
```

| | `malloc(n * size)` | `calloc(n, size)` |
| --- | --- | --- |
| arguments | one: total bytes | two: count, then size |
| initial content | **indeterminate** — leftover garbage | **all zero** |
| overflow check | you must get `n * size` right yourself | the multiplication is done inside the library |
| speed | one pass | one pass (plus the zeroing) |

### `malloc` Really Does Return Garbage

This is worth seeing rather than believing. The program below fills a block with `0xAB`, frees it, and then asks the allocator for the same size again:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SIZE 32

static int count_nonzero(const unsigned char *p, size_t n) {
    int count = 0;
    size_t i;

    for (i = 0; i < n; i++) {
        if (p[i] != 0) {
            count++;
        }
    }
    return count;
}

int main(void) {
    unsigned char *dirty;
    unsigned char *from_malloc;
    unsigned char *from_calloc;
    unsigned char *from_memset;

    /* dirty the heap first, so a later malloc gets memory that was just used */
    dirty = malloc(SIZE);
    if (dirty == NULL) {
        return 1;
    }
    memset(dirty, 0xAB, SIZE);
    free(dirty);

    from_malloc = malloc(SIZE);
    from_calloc = calloc(SIZE, 1);
    from_memset = malloc(SIZE);

    if (from_malloc == NULL || from_calloc == NULL || from_memset == NULL) {
        return 1;
    }
    memset(from_memset, 0, SIZE);

    printf("block reused after being filled with 0xAB, %d bytes:\n\n", SIZE);
    printf("malloc(%d, 1)              -> %2d of %d bytes non-zero\n",
           SIZE, count_nonzero(from_malloc, SIZE), SIZE);
    printf("calloc(%d, 1)              -> %2d of %d bytes non-zero\n",
           SIZE, count_nonzero(from_calloc, SIZE), SIZE);
    printf("malloc(%d, 1) + memset 0   -> %2d of %d bytes non-zero\n",
           SIZE, count_nonzero(from_memset, SIZE), SIZE);

    free(from_malloc);
    free(from_calloc);
    free(from_memset);

    return 0;
}
```

```text
block reused after being filled with 0xAB, 32 bytes:

malloc(32, 1)              -> 21 of 32 bytes non-zero
calloc(32, 1)              ->  0 of 32 bytes non-zero
malloc(32, 1) + memset 0   ->  0 of 32 bytes non-zero
```

The exact number in the first line depends on the allocator and the machine — it is the point that matters that it is **not zero**. `calloc` and `memset` are identical in result here; `calloc` is just the library doing it for you.

### The Overflow Argument for `calloc`

`calloc(n, size)` checks whether `n * size` overflows before multiplying. `malloc(n * size)` computes the product in your code, where a huge `n` silently wraps around:

```c
size_t n = huge_value_from_input;
int *a = malloc(n * sizeof(int));       /* n * 4 may wrap to a small number */
int *b = calloc(n, sizeof(int));        /* returns NULL instead */
```

With `calloc` you get `NULL` and a clean error message. With `malloc` you get a 16-byte block and a heap overflow the moment you fill it.

`memset` is the manual equivalent, and it is the right tool when you want zeroed memory from an existing block (after a `realloc`, for instance):

```c
memset(values, 0, n * sizeof(int));   /* the same as calloc(n, sizeof(int)) */
```

---

## The Function Table

| Function | Signature | Initializes to | Can move the block? | Frees the old block? |
| --- | --- | --- | --- | --- |
| `malloc` | `void *malloc(size_t size)` | nothing (garbage) | n/a | n/a |
| `calloc` | `void *calloc(size_t n, size_t size)` | all zero | n/a | n/a |
| `realloc` | `void *realloc(void *p, size_t size)` | nothing (old contents kept up to the smaller size) | **yes, possibly** | yes, the old one |
| `free` | `void free(void *p)` | n/a | n/a | the block you pass |

Rules that go with the table:

- `realloc(NULL, n)` is exactly `malloc(n)`.
- `free(NULL)` does nothing.
- `realloc(p, 0)` is implementation-defined: it may free and return `NULL`, or return a unique non-null pointer. Do not rely on either — never call it with 0 to mean "free".

---

## `realloc`: Changing a Block's Size

```c
void *realloc(void *ptr, size_t size);
```

`realloc` resizes the block `ptr` points at, and returns a pointer to the resized block. The contents up to the **smaller** of the old and new sizes are preserved.

The one thing you must always remember: **the block may move.**

```text
  before realloc                        after realloc (moved)

  +---------------+                     +---------------+
  | A: block X    |  realloc            | B: block X    |   <- the same bytes,
  | B: block Y    |  ===============>   |     ...       |      copied to a new
  | C: block Z    |                     | C: block Y    |      address. The OLD
  +---------------+                     | D: block Z    |      block X is gone.
                                        +---------------+

  p ----> X                           old p --> X   (dangling)
  p ----> X   (unchanged)             p ----> Y'  (new)
```

The allocator only moves the block when it has to: when the following bytes are already in use, or when the new size needs a different memory region.

### Always Use the Returned Pointer

```c
int *tmp = realloc(values, new_count * sizeof(int));
if (tmp == NULL) {
    /* handle failure; `values` is still valid and still ours */
}
values = tmp;
```

Three separate reasons this matters:

1. **A failed `realloc` returns `NULL` and leaves the old block untouched.** If you write `values = realloc(values, huge);` and it fails, `values` becomes `NULL` and the old block is now unreachable: a leak, on top of the failure.
2. **A successful `realloc` that moved the block leaves every other copy of the old pointer dangling.** Any alias, any pointer stored in a struct, any pointer passed to another function — all of them now point at freed memory.
3. **The old block is freed by `realloc` itself.** You must not `free` it again.

### The Failure Mode in Full

```c
#include <stdio.h>
#include <stdlib.h>

/* Ownership: the List owns the block that `data` points at. */
typedef struct {
    int  *data;
    size_t count;
} List;

static int list_init(List *l, size_t capacity) {
    l->data = malloc(capacity * sizeof(int));
    if (l->data == NULL) {
        l->count = 0;
        return -1;
    }
    l->count = 0;
    return 0;
}

/* BUG: 'old' keeps the pre-realloc address. If realloc moved the block,
   'old' is now dangling, and the writes below use freed memory. */
static int list_push_buggy(List *l, int value) {
    int *old = l->data;

    l->data = realloc(l->data, (l->count + 1) * sizeof(int));
    if (l->data == NULL) {
        return -1;
    }
    l->count++;

    old[l->count - 1] = value;   /* writes into the old, possibly freed block */
    return l->count - 1;
}

int main(void) {
    List l;
    int i;

    if (list_init(&l, 4) != 0) {
        return 1;
    }

    for (i = 0; i < 6; i++) {
        list_push_buggy(&l, i * 11);
        printf("pushed %2d, count = %zu, data = %p\n",
               i * 11, l.count, (void *)l.data);
    }

    free(l.data);
    return 0;
}
```

Compiled with the normal flags, this "works" — the allocator happened to extend the block in place, so `old` and `l->data` stayed equal and the writes landed in live memory:

```text
pushed  0, count = 1, data = 0x56741f034010
pushed 11, count = 2, data = 0x56741f034010
pushed 22, count = 3, data = 0x56741f034010
pushed 33, count = 4, data = 0x56741f034010
pushed 44, count = 5, data = 0x56741f034010
pushed 55, count = 6, data = 0x56741f034010
```

The address never changes, so the bug is invisible. Change the compile flags and it is not:

```text
=================================================================
==379154==ERROR: AddressSanitizer: heap-use-after-free on address 0x75ff19de0010 at pc 0x56e661e40881 bp 0x7ffe2fb91240 sp 0x7ffe2fb91230
WRITE of size 4 at 0x75ff19de0010 thread T0
    #0 0x56e661e40880 in list_push_buggy /tmp/realloc_old.c:31
    #1 0x56e661e409e0 in main /tmp/realloc_old.c:44

0x75ff19de0010 is located 0 bytes inside of 16-byte region [0x75ff19de0010,0x75ff19de0020)
freed by thread T0 here:
    #0 0x79df1bb2a6cf in realloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:81
    #1 0x56e661e4062e in list_push_buggy /tmp/realloc_old.c:25

previously allocated by thread T0 here:
    #0 0x79df1bb2b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x56e661e4036d in list_init /tmp/realloc_old.c:11
    #2 0x56e661e40986 in main /tmp/realloc_old.c:39

SUMMARY: AddressSanitizer: heap-use-after-free /tmp/realloc_old.c:31 in list_push_buggy
```

Three sections in that report, and together they tell the whole story: the bad write at line 31, the `realloc` that freed the block at line 25, and the `malloc` that created it at line 11. Under the normal build the same program printed six successful pushes. **This is why the sanitizers are not optional.**

### Shrinking

A `realloc` to a smaller size keeps the surviving data and hands the tail back to the allocator. Shrinking usually happens in place, so the address typically does not change — but "usually" is not a guarantee, so the returned pointer is still the only one you may use:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *values = malloc(10 * sizeof(int));
    size_t i;

    if (values == NULL) {
        return 1;
    }

    for (i = 0; i < 10; i++) {
        values[i] = (int)(i * 3);
    }

    printf("before shrink: ");
    for (i = 0; i < 10; i++) {
        printf("%d ", values[i]);
    }
    printf("\n");

    /* realloc to a smaller size. The data is kept, up to the new size.
       Shrinking often happens in place, so the address may not change. */
    int *tmp = realloc(values, 4 * sizeof(int));
    if (tmp == NULL) {
        free(values);
        return 1;
    }
    values = tmp;

    printf("after  shrink: ");
    for (i = 0; i < 4; i++) {
        printf("%d ", values[i]);
    }
    printf("\n");

    printf("the allocator may keep the remaining 6 ints for itself\n");

    free(values);
    return 0;
}
```

```text
before shrink: 0 3 6 9 12 15 18 21 24 27 
after  shrink: 0 3 6 9 
the allocator may keep the remaining 6 ints for itself
```

### The Correct Growth Loop

Here is the pattern you will use for the rest of the course. Push 1 through 10 into an array that starts empty and grows as needed:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *values = NULL;
    size_t capacity = 0;
    size_t count = 0;
    int i;

    for (i = 1; i <= 10; i++) {
        if (count == capacity) {
            size_t new_capacity = (capacity == 0) ? 4 : capacity * 2;
            int *tmp = realloc(values, new_capacity * sizeof(int));

            if (tmp == NULL) {
                free(values);
                return 1;
            }
            values = tmp;
            capacity = new_capacity;
            printf("grew to capacity %zu\n", capacity);
        }

        values[count] = i * i;
        count++;
    }

    printf("\n");
    for (i = 0; i < (int)count; i++) {
        printf("%d^2 = %d\n", i + 1, values[i]);
    }

    free(values);
    return 0;
}
```

```text
grew to capacity 4
grew to capacity 8
grew to capacity 16

1^2 = 1
2^2 = 4
3^2 = 9
4^2 = 16
5^2 = 25
6^2 = 36
7^2 = 49
8^2 = 64
9^2 = 81
10^2 = 100
```

Three reallocations for ten items. Note the `free(values)` in the failure branch: if the first `realloc` fails, `values` is still `NULL`, and `free(NULL)` is a no-op — so that path is correct without an `if`.

---

## Ownership: Who Frees What

Every dynamic pointer has exactly one **owner**: the code that is responsible for calling `free` on it. Getting this wrong is the most common source of double frees and leaks in real C, more than typos.

### The Four Rules

1. **Whoever calls `malloc` owns the result.** Not the callee, not the caller — the function whose line contains the `malloc`.
2. **A function that receives a pointer does not own it**, unless its documentation says otherwise. It may use it, and it must not `free` it.
3. **A function that allocates and returns a pointer transfers ownership to the caller.** The caller must `free` it, and only the caller.
4. **Document ownership in a comment on every pointer-returning function.** If you cannot write the comment, the design is unclear.

```text
  malloc in f()  ------  returns  ------>  g() owns it  ---- free()
        (f gave up ownership)                     (g took it)

  malloc in f()  ------  returns  ------>  h(&p)  ---->  h borrows, g still owns

  malloc in f()  ------  returns  ------>  caller ignores it  ----  LEAK
```

### Rule 2: Borrowing

```c
/* Borrows: reads the buffer, never frees it, never keeps it. */
static void print_scores(const int *scores, size_t n) {
    size_t i;

    for (i = 0; i < n; i++) {
        printf("%d ", scores[i]);
    }
    printf("\n");
}
```

The `const` is the compiler-checked half of the promise. The comment is the other half.

### Rule 3: Transferring

```c
/* Allocates a copy of `src` and returns it.
   OWNERSHIP: the caller owns the result and must free() it.
   Returns NULL on allocation failure. */
static int *dup_array(const int *src, size_t n) {
    int *copy = malloc(n * sizeof(int));
    size_t i;

    if (copy == NULL) {
        return NULL;
    }
    for (i = 0; i < n; i++) {
        copy[i] = src[i];
    }
    return copy;
}
```

The caller now holds the obligation:

```c
int *copy = dup_array(values, n);
if (copy == NULL) {
    return 1;
}
/* use copy */
free(copy);
```

### Rule 4: The comment

Every pointer-returning function gets a comment. This is not decoration; it is the only thing that tells the next reader who has to write the `free`:

```c
/* OWNERSHIP: returns a heap copy; the caller must free() it. */
static int *dup_array(const int *src, size_t n);

/* OWNERSHIP: borrows `data`; does not free it, does not keep it. */
static int sum_of(const int *data, size_t n);

/* OWNERSHIP: takes ownership of `data`; always frees it, even on error.
   After this call the caller's pointer is dangling. */
static int consume(int *data, size_t n);
```

Two functions that both free the same pointer are a **double free**. Two functions that both think the other one frees it are a **leak**. The ownership comment exists to make both impossible.

### An Owning Struct

When a struct holds a heap pointer, the struct is usually the owner, and it gets a matching free function:

```c
typedef struct {
    int  *data;
    size_t count;
    size_t capacity;
} IntVec;

static void intvec_free(IntVec *v) {
    if (v == NULL) {
        return;
    }
    free(v->data);
    v->data = NULL;   /* so a second call is harmless */
    v->count = 0;
    v->capacity = 0;
}
```

Setting `v->data = NULL` inside `free` is what makes the destructor safe to call twice — a common pattern for cleanup code that may run on several error paths.

---

## The Five Bugs

| Bug | What happens | Symptom | Fix |
| --- | --- | --- | --- |
| **Leak** | allocated, never freed | memory grows; slow death; LeakSanitizer report | one `free` per `malloc`, same scope |
| **Double free** | freed twice | `free(): double free detected` abort, or silent corruption | one owner only; `NULL` after freeing |
| **Use after free** | read or write through a dangling pointer | garbage values, corruption, or a segfault | never use a pointer after `free`; `NULL` it |
| **Buffer overflow** | write past the end of a block | corruption of another block, or a segfault | `malloc(n * sizeof(type))`; check every index |
| **Invalid free** | free a non-heap pointer, or a pointer into the middle of a block | `free(): invalid pointer` abort | free exactly what `malloc` returned |

Two of these deserve a sentence each, because they look reasonable and are not:

**Invalid free.** `free` takes the address the allocator knows. Freeing `&data[2]` frees "the block starting at `data[2]`" — which the allocator has never heard of, and which is usually a middle chunk of someone else's allocation. The process aborts, or worse.

**Overflow vs underflow.** An overflow writes past the end; an underflow writes before the start, usually through a negative index. Same family, same fix: check the index against the count you stored, because the block itself does not know its own length.

---

## Checklist

```text
[ ] calloc zeroes; malloc does not
[ ] calloc(n, size) checks the multiplication for overflow
[ ] memset(p, 0, n * sizeof(int)) is the manual equivalent of calloc
[ ] realloc may move the block: use ONLY the returned pointer
[ ] A failed realloc leaves the old block valid — do not lose it
[ ] Do not keep the old pointer across a realloc; it dangles
[ ] Ownership: the malloc caller owns; receivers borrow; returners transfer
[ ] Every pointer-returning function documents ownership in a comment
[ ] An owning struct gets a free function that NULLs the pointer
[ ] Build with -fsanitize=address,undefined and run
```

---

#### [Previous](./01-part.md) | [Next](./03-part.md)
