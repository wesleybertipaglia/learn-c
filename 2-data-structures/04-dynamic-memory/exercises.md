# Exercise List 4 - Dynamic Memory

Six exercises on the heap, plus one instruction that applies to all of them. Every program must compile with `gcc -std=c11 -Wall -Wextra -pedantic -g file.c -o file` and produce **zero** warnings.

Write the programs as if the memory were yours, because it is.

---

## Exercise 1: Squares on the Heap

Ask the user for `N`, allocate an `int` array of that size **on the heap**, fill it with the first N squares, print it, and free it.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int n = 5;
    int *values;
    int i;

    printf("Enter how many numbers: ");
    if (scanf("%d", &n) != 1 || n < 0 || n > 100) {
        printf("invalid size\n");
        return 1;
    }

    /* n may be 0, and malloc(0) is allowed: it returns a unique pointer
       that must still be free()d. */
    values = malloc((size_t)n * sizeof(int));
    if (values == NULL) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    for (i = 0; i < n; i++) {
        values[i] = i * i;
    }

    printf("\n");
    for (i = 0; i < n; i++) {
        printf("%d^2 = %d\n", i, values[i]);
    }

    free(values);
    values = NULL;

    return 0;
}
```

```text
Enter how many numbers: 5

0^2 = 0
1^2 = 1
2^2 = 4
3^2 = 9
4^2 = 16
```

Requirements:

- Write a comment explaining what `sizeof(int)` contributes to the allocation size, and what happens if you drop it.
- Set `values = NULL;` right after the `free` and explain in comments what that changes.
- Reject a negative `N` with a clear message, and decide (in a comment) what your program should do for `N = 0`.
- Then, without `free`, run it and paste the LeakSanitizer report. With the `free` restored, run it again and confirm the report is gone.
- Extra credit: print how many bytes the block occupies and how many of them hold real data.

---

## Exercise 2: Duplicating an Array

Write `int *dup_array(const int *src, int n)` that returns a **heap copy** of `src`, and free it in `main`.

```c
#include <stdio.h>
#include <stdlib.h>

/* OWNERSHIP: borrows `src`; does not free it and does not keep it.
   OWNERSHIP: returns a heap copy; the CALLER must free() the result.
   Returns NULL on allocation failure or if src is NULL. */
static int *dup_array(const int *src, size_t n) {
    int *copy;
    size_t i;

    if (src == NULL) {
        return NULL;
    }

    copy = malloc(n * sizeof(int));
    if (copy == NULL) {
        return NULL;
    }

    for (i = 0; i < n; i++) {
        copy[i] = src[i];
    }
    return copy;
}

int main(void) {
    int original[5] = {10, 20, 30, 40, 50};
    int *copy;
    size_t i;

    copy = dup_array(original, 5);
    if (copy == NULL) {
        fprintf(stderr, "could not duplicate\n");
        return 1;
    }

    for (i = 0; i < 5; i++) {
        printf("original[%zu] = %2d   copy[%zu] = %2d\n", i, original[i], i, copy[i]);
    }

    copy[0] = 999;
    printf("\nafter copy[0] = 999:\n");
    printf("original[0] = %d  <- untouched: it is a copy, not an alias\n", original[0]);
    printf("copy[0]     = %d\n", copy[0]);

    free(copy);   /* the caller owns what dup_array returned */
    copy = NULL;

    return 0;
}
```

```text
original[0] = 10   copy[0] = 10
original[1] = 20   copy[1] = 20
original[2] = 30   copy[2] = 30
original[3] = 40   copy[3] = 40
original[4] = 50   copy[4] = 50

after copy[0] = 999:
original[0] = 10  <- untouched: it is a copy, not an alias
copy[0]     = 999
```

Requirements:

- The two `OWNERSHIP` comments above the function are not optional. Write your own version of them and explain, in comments, which of the two rules (borrow / transfer) applies to `src` and which applies to the result.
- Prove the copy is independent: modify `copy[0]` and show `original[0]` did not change. (The output above does this.)
- Write a second function `void free_copy(int **pp)` that takes the address of the pointer, frees `*pp`, and sets `*pp` to `NULL`. Call it with `free_copy(&copy)` and explain in comments why a single `int *` parameter could not do this.
- Add a `dup_array` that takes a `size_t n` and handles `n == 0` without calling `malloc` at all. Say in comments why you special-cased it.
- Extra credit: write `void print_array(const int *a, int n)` and reuse it in `main`.

---

## Exercise 3: Finishing the Vector

Here is a vector with `vector_push` missing. Implement it, including the doubling growth.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

typedef struct {
    int  *data;      /* the block, or NULL while empty          */
    size_t size;     /* how many elements are in use            */
    size_t capacity; /* how many elements fit before a realloc   */
} IntVec;

static void vector_init(IntVec *v) {
    v->data = NULL;
    v->size = 0;
    v->capacity = 0;
}

static void vector_free(IntVec *v) {
    if (v == NULL) {
        return;
    }
    free(v->data);
    v->data = NULL;
    v->size = 0;
    v->capacity = 0;
}

/* TODO: implement this. The growth policy is doubling, starting at 1.
   The two (void) casts below only silence -Wunused-parameter; delete them
   as soon as the parameters are used. */
static int vector_push(IntVec *v, int value) {
    (void)v;
    (void)value;
    return -1;
}

int main(void) {
    IntVec v;
    int i;

    vector_init(&v);

    for (i = 1; i <= 10; i++) {
        if (vector_push(&v, i * 10) != 0) {
            fprintf(stderr, "push failed at %d\n", i);
            vector_free(&v);
            return 1;
        }
    }

    for (i = 0; i < 10; i++) {
        printf("%d ", v.data[i]);
    }
    printf("\n");

    vector_free(&v);
    return 0;
}
```

Target output:

```text
10 20 30 40 50 60 70 80 90 100
```

Requirements:

- The signature is `static int vector_push(IntVec *v, int value)` returning 0 on success and -1 on failure. Everything else is yours.
- Print `size` and `capacity` after each push, so the growth is visible. You should see `capacity` go 1, 2, 4, 8, 16.
- Write in comments why `realloc` must go through a temporary pointer, and what your code would do if the `realloc` failed.
- Write in comments why the growth formula needs the `(capacity == 0) ? 1 :` special case, and what happens without it.
- After the first working version, print a table of "copies this push" like the one in the previous part, and confirm in comments that 16 pushes cost 15 element copies.
- Add `int vector_get(const IntVec *v, size_t index, int *out)` that rejects an index `>= size`, and use it in `main` instead of touching `v.data[i]` directly. Explain in comments why the check is not optional.
- Extra credit: add `vector_pop`, which shrinks `size` and never shrinks `capacity`. Explain in comments why that asymmetry is a deliberate choice.

---

## Exercise 4: Break It, Prove It, Fix It

This program has a leak and a use-after-free. Run it under the sanitizers, paste the **real** reports, then fix both.

```c
#include <stdio.h>
#include <stdlib.h>

/* OWNERSHIP: v is borrowed; this function does not free it and does not
   remember it. The caller keeps that responsibility. */
static int compute_total(const int *values, size_t n) {
    size_t i;
    int total = 0;

    for (i = 0; i < n; i++) {
        total += values[i];
    }
    return total;
}

int main(void) {
    int *values = malloc(4 * sizeof(int));
    int i;
    int total;

    if (values == NULL) {
        return 1;
    }

    for (i = 0; i < 4; i++) {
        values[i] = i * i;
    }

    total = compute_total(values, 4);
    printf("total = %d\n", total);
    fflush(stdout);

    /* BUG 1: `values` is never freed. The block is lost when main returns. */

    return 0;
}
```

The first report, from the leak:

```text
=================================================================
==385775==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 16 byte(s) in 1 object(s) allocated from:
    #0 0x7a7a75b2b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x5614e5d5240e in main /tmp/ex4c.c:17
    #2 0x7a7a74e2a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #3 0x7a7a74e2a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #4 0x5614e5d52224 in _start (/tmp/ex4c_asan+0x1224) (BuildId: 438966568f88efda06058ca54a04f6a97d768096)

SUMMARY: AddressSanitizer: 16 byte(s) leaked in 1 allocation(s).
```

Now add a use-after-free and catch that one too:

```c
    total = compute_total(values, 4);
    printf("total = %d\n", total);
    fflush(stdout);

    free(values);
    values = NULL;

    /* BUG 2: reading through a pointer that is already NULL. This is a null
       dereference rather than a use-after-free read, because the free was
       followed by NULL. Both are bugs; the sanitizer names the first. */
    for (i = 0; i < 4; i++) {
        total += values[i];
    }
```

```text
=================================================================
==384981==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000000 (pc 0x596f8ad334da bp 0x7fffb037b2f0 sp 0x7fffb037b2c0 T0)
==384981==The signal is caused by a READ memory access.
==384981==Hint: address points to the zero page.
    #0 0x596f8ad334da in main /tmp/ex4a.c:24
    #1 0x75104d62a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #2 0x75104d62a717 in __libc_start_main_impl ../csu/libc-start.c:360

SUMMARY: AddressSanitizer: SEGV /tmp/ex4a.c:24 in main
```

Requirements:

- Run it three ways: plain, with `-fsanitize=address`, and with `-fsanitize=address,undefined`. Paste what each one prints. Explain in comments why the plain build is the least useful of the three.
- Now remove the `values = NULL;`. Re-run and paste the new report. Explain in comments why the error changed from `SEGV on unknown address 0x0` to `heap-use-after-free`, and which one is the more informative message.
- Add a third bug: a `free` in a function that receives the pointer **as an argument**, then the same `free` in `main`. Paste the double-free report.
- Fix all three, then confirm with a clean sanitizer run. Write down the command you used and the fact that it printed nothing at all.
- Extra credit: wrap the allocation in a small struct with a `destroy` function, so the cleanup is in one place, and show that calling `destroy` twice is safe.

---

## Exercise 5: Shrinking an Over-Allocated Array

Start with 10 elements, then `realloc` down to 4 and free the result.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

int main(void) {
    int *values = malloc(10 * sizeof(int));
    size_t used = 10;
    size_t wanted = 4;
    size_t i;

    if (values == NULL) {
        return 1;
    }

    for (i = 0; i < used; i++) {
        values[i] = (int)(i * 2);
    }

    printf("before shrink: ");
    for (i = 0; i < used; i++) {
        printf("%d ", values[i]);
    }
    printf("\n");

    {
        int *shrunk = realloc(values, wanted * sizeof(int));
        if (shrunk == NULL) {
            printf("shrink failed, keeping the original block\n");
        } else {
            values = shrunk;   /* only the returned pointer may be stored */
            used = wanted;
        }
    }

    printf("after  shrink: ");
    for (i = 0; i < used; i++) {
        printf("%d ", values[i]);
    }
    printf("\n");

    free(values);
    values = NULL;

    return 0;
}
```

```text
before shrink: 0 2 4 6 8 10 12 14 16 18 
after  shrink: 0 2 4 6 
```

Requirements:

- Explain in comments why the `realloc` result goes into a **new** local variable instead of straight back into `values`. Then try the one-liner version `values = realloc(values, wanted * sizeof(int));` and explain, in comments, what is lost when the allocation fails.
- Print `&values[0]` before and after the shrink. Run it several times: does the address change? Write in comments why the answer is not guaranteed either way.
- Write `size_t shrink_to(int **block, size_t used, size_t wanted)` that takes the address of the pointer. Explain in comments why the double pointer is needed here and was not needed in the one-liner.
- Then over-allocate: start with 4, `realloc` up to 100, and confirm the first four values survived. Explain in comments what the standard guarantees about the contents after a growing `realloc`.
- Extra credit: use `realloc` to grow one element at a time and count the element copies, then repeat with doubling. Report both totals and explain the difference.

---

## Exercise 6: Merging Two Sorted Arrays

Implement `int *merge_sorted(const int *a, int n, const int *b, int m)` that mallocs the result.

```c
#include <stdio.h>
#include <stdlib.h>

/* OWNERSHIP: borrows both input arrays; never frees them.
   OWNERSHIP: returns a NEW heap block of n + m ints; the CALLER must
   free() it. Returns NULL on allocation failure. */
static int *merge_sorted(const int *a, int n, const int *b, int m) {
    int *out = malloc((size_t)(n + m) * sizeof(int));
    int i = 0;
    int j = 0;
    int k = 0;

    if (out == NULL) {
        return NULL;
    }

    while (i < n && j < m) {
        if (a[i] <= b[j]) {
            out[k++] = a[i++];
        } else {
            out[k++] = b[j++];
        }
    }
    while (i < n) {
        out[k++] = a[i++];
    }
    while (j < m) {
        out[k++] = b[j++];
    }

    return out;
}

int main(void) {
    int a[5] = {1, 4, 7, 9, 12};
    int b[4] = {2, 3, 8, 10};
    int n = 5;
    int m = 4;
    int *merged;
    int i;

    merged = merge_sorted(a, n, b, m);
    if (merged == NULL) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    for (i = 0; i < n + m; i++) {
        printf("%d ", merged[i]);
    }
    printf("\n");

    free(merged);
    return 0;
}
```

```text
1 2 3 4 7 8 9 10 12 
```

Requirements:

- Write the ownership contract as a comment, and restate it in `main` next to the `free`. The question the comment answers: who is allowed to call `free` on this pointer, and how many times?
- Handle `n == 0` and `m == 0` without special-casing the whole function. `malloc(0 * sizeof(int))` is legal; say in comments what your function does then, and what the caller must still do with the result.
- Add a `print` helper and a loop that merges 5 arrays of increasing size in a loop, freeing each result. Run it under the sanitizer to prove no leak accumulated.
- Write a version `int merge_sorted_into(const int *a, int n, const int *b, int m, int *out)` where the caller supplies the output buffer, and explain in comments how that changes the ownership story (it does not, but the function's contract does).
- Extra credit: allocate the result with `calloc` and explain in comments why that is pointless here but would not be in a function that only fills the first `n` of `n + m` slots.

---

## Exercise 7: Sanitize Everything

Do this for **every** exercise above, and for every exercise in the whole course from now on:

1. Compile with warnings as errors and with both sanitizers:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -Werror -g \
       -fsanitize=address,undefined -fno-omit-frame-pointer main.c -o main_san
   ```

2. Run it with several different inputs, not just one. A buffer overflow that only happens on the last iteration needs the last iteration to be reachable.
3. Fix everything the sanitizer reports. It is not a suggestion.
4. Re-run until the output contains no sanitizer diagnostics at all. Silence is the passing state.
5. Then compile the plain version and confirm it still behaves identically:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
   ```

6. If a program legitimately cannot be leak-free (a library that keeps state until exit), write down **why** in a comment, and check that the leaked bytes are bounded and documented.

Report, for each program, in a comment at the top of the file:

```text
[ ] compiles with -Werror, zero warnings
[ ] no sanitizer diagnostics
[ ] every malloc has a matching free
[ ] every free argument came from malloc/calloc/realloc
[ ] no pointer used after free
[ ] every index checked against the stored length
[ ] ownership documented on every pointer-returning function
```

That checklist is the habit the whole module is trying to build. A C program that passes it will not be the fastest, but it will still be working next month.

---

#### [Previous](./04-part.md) | [Next](../05-linked-lists/01-part.md)
