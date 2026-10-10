# Memory Bugs in Detail

Each bug in this part follows the same shape: the code, the real AddressSanitizer report, and the fix. Read the reports — the `ERROR:` line names the bug, the frames name the line, and the `freed by` / `allocated by` sections name the two other lines that matter.

The build command for every program here:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer file.c -o file
```

---

## (a) Buffer Overflow

Writing past the end of a block. This is the most common heap bug there is, and the classic cause is `malloc(n)` where `n * sizeof(type)` was meant.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int n = 5;
    int *values = malloc(n * sizeof(int));
    int i;

    if (values == NULL) {
        return 1;
    }

    for (i = 0; i < n; i++) {
        values[i] = i * 7;
    }

    values[n] = 999;   /* BUG: index n is one past the end */

    printf("last value = %d\n", values[n - 1]);

    free(values);
    return 0;
}
```

The ordinary build says nothing at all and prints the right answer, because `values[n - 1]` is the last **valid** element and the bad write went into a `printf` buffer's slack space:

```text
last value = 28
```

The sanitizer stops the program at the offending write:

```text
=================================================================
==380119==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x7562939e0054 at pc 0x614c9cc314b0 bp 0x7ffea7bcd1e0 sp 0x7ffea7bcd1d0
WRITE of size 4 at 0x7562939e0054 thread T0
    #0 0x614c9cc314af in main /tmp/overflow1.c:17
    #1 0x793294a2a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #2 0x793294a2a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #3 0x614c9cc31204 in _start (/tmp/overflow1_asan+0x1204) (BuildId: ca104dccad7ba033ff48f2b4d760ac967b5a64a8)

0x7562939e0054 is located 0 bytes after 20-byte region [0x7562939e0040,0x7562939e0054)
allocated by thread T0 here:
    #0 0x79329572b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x614c9cc312ef in main /tmp/overflow1.c:6
    #2 0x793294a2a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #3 0x793294a2a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #4 0x614c9cc31204 in _start (/tmp/overflow1_asan+0x1204) (BuildId: ca104dccad7ba033ff48f2b4d760ac967b5a64a8)

SUMMARY: AddressSanitizer: heap-buffer-overflow /tmp/overflow1.c:17 in main
```

Three lines to read: `WRITE of size 4` (an `int` was written), line 17 (the assignment), and `0 bytes after 20-byte region` (the address is the first byte past a 20-byte block). The block itself was allocated at line 6.

**The fix is not "be more careful".** It is to make the array carry its own length and check the index:

```c
if (i >= 0 && (size_t)i < n) {
    values[i] = 999;
}
```

Or, for a string, use `snprintf` and compare its return against the buffer size, exactly as in the Debugging chapter. A heap block does not know its own length, so *some* object has to know it — usually the `size` field of a container struct.

---

## (b) Use After Free

Reading or writing through a pointer whose block has been returned to the heap.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int n = 5;
    int *values = malloc(n * sizeof(int));
    int i;
    int total = 0;

    if (values == NULL) {
        return 1;
    }

    for (i = 0; i < n; i++) {
        values[i] = i * 7;
    }

    free(values);   /* values is now DANGLING: it still holds the old address */

    /* BUG: reading through a freed pointer is undefined behavior */
    for (i = 0; i < n; i++) {
        total += values[i];
    }

    printf("total = %d\n", total);

    return 0;
}
```

GCC catches this one, because the `free` and the use are in the same function:

```text
warning: pointer 'values' may be used after 'free' [-Wuse-after-free]
   22 |         total += values[i];
      |                        ^
/tmp/uaf2.c:18:5: note: call to 'free' here
   18 |     free(values);
      |     ^~~~~~~~~~~~
```

But the program still compiles, still runs, and still exits 0. The value it prints is whatever happens to be in those bytes now:

```text
total = -354804762
```

Run it again and you get a different number, because between runs the allocator handed the same block to a different program:

```text
total = -1646628827
total = -766718474
```

**A use-after-free that prints a number is worse than one that crashes.** A crash is a bug report; a wrong number is a wrong answer you may ship. Under the sanitizer:

```text
=================================================================
==380357==ERROR: AddressSanitizer: heap-use-after-free on address 0x721d75be0040 at pc 0x5ac1d6a2d4cf bp 0x7ffd448b4e90 sp 0x7ffd448b4e80
READ of size 4 at 0x721d75be0040 thread T0
    #0 0x5ac1d6a2d4ce in main /tmp/uaf2.c:22
    #1 0x75ed76c2a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #2 0x75ed76c2a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #3 0x5ac1d6a2d204 in _start (/tmp/uaf2_asan+0x1204) (BuildId: 5aa6c1afbf73148cfcfb943b99fa70f771ad32d0)

0x721d75be0040 is located 0 bytes inside of 20-byte region [0x721d75be0040,0x721d75be0054)
freed by thread T0 here:
    #0 0x75ed7792a3ff in free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:51
    #1 0x5ac1d6a2d421 in main /tmp/uaf2.c:18
    #2 0x75ed76c2a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #3 0x75ed76c2a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #4 0x5ac1d6a2d204 in _start (/tmp/uaf2_asan+0x1204) (BuildId: 5aa6c1afbf73148cfcfb943b99fa70f771ad32d0)

previously allocated by thread T0 here:
    #0 0x75ed7792b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x5ac1d6a2d2ef in main /tmp/uaf2.c:6
```

`READ of size 4` at line 22, `freed by` at line 18, `allocated by` at line 6. The whole lifetime of the block in three lines.

**The fix is `values = NULL;` right after the `free`.** That does not make the access safe — it makes the *mistake* safe, because the next use is a null-pointer dereference, which crashes immediately and obviously instead of silently returning garbage:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int n = 5;
    int *values = malloc(n * sizeof(int));
    int i;
    int total = 0;

    if (values == NULL) {
        return 1;
    }

    for (i = 0; i < n; i++) {
        values[i] = i * 7;
    }

    free(values);
    values = NULL;   /* the fix: make the mistake impossible to survive */

    if (values != NULL) {          /* the compiler now forces this check */
        for (i = 0; i < n; i++) {
            total += values[i];
        }
    }

    printf("total = %d (the loop was skipped)\n", total);

    return 0;
}
```

```text
total = 0 (the loop was skipped)
```

---

## (c) Use After Free Through `realloc`

The same bug, but the `free` was done by `realloc` and you never wrote it yourself, which is why it is easy to miss.

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

The normal build looks perfect — the address never changes, so the bug never fires:

```text
pushed  0, count = 1, data = 0x56741f034010
pushed 11, count = 2, data = 0x56741f034010
pushed 22, count = 3, data = 0x56741f034010
pushed 33, count = 4, data = 0x56741f034010
pushed 44, count = 5, data = 0x56741f034010
pushed 55, count = 6, data = 0x56741f034010
```

The sanitizer, which always relocates, catches the very first push:

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

Note `freed by ... in realloc`. The program never called `free`, which is why this class of bug hides from a grep for `free`.

**The fix is to never keep a second copy of the pointer across a `realloc`.** Use the returned pointer and nothing else:

```c
int *tmp = realloc(l->data, (l->count + 1) * sizeof(int));
if (tmp == NULL) {
    return -1;              /* l->data is still valid: we kept it */
}
l->data = tmp;              /* one pointer, the new one */
l->data[l->count++] = value;
```

---

## (d) Double Free

Two owners for one block. The classic shape is an alias: a second pointer to the same memory, and two cleanup functions that both "helpfully" free it.

```c
#include <stdio.h>
#include <stdlib.h>

static void cleanup(int *data) {
    free(data);
    printf("cleaned up\n");
}

int main(void) {
    int *values = malloc(5 * sizeof(int));
    int *alias;

    if (values == NULL) {
        return 1;
    }

    values[0] = 42;
    alias = values;   /* a second name for the same block */

    cleanup(values);  /* the owner frees it */
    cleanup(alias);   /* BUG: alias is the same block, freed a second time */

    printf("never reached\n");

    return 0;
}
```

glibc detects it and aborts:

```text
free(): double free detected in tcache 2
Aborted (core dumped)
```

The sanitizer says the same thing with locations:

```text
=================================================================
==380576==ERROR: AddressSanitizer: attempting double-free on 0x772b723e0040 in thread T0:
    #0 0x7afb7412a3ff in free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:51
    #1 0x5b5c3c55c264 in cleanup /tmp/doublefree.c:5
    #2 0x5b5c3c55c32a in main /tmp/doublefree.c:21

0x772b723e0040 is located 0 bytes inside of 20-byte region [0x772b723e0040,0x772b723e0054)
freed by thread T0 here:
    #0 0x7afb7412a3ff in free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:51
    #1 0x5b5c3c55c264 in cleanup /tmp/doublefree.c:5
    #2 0x5b5c3c55c31e in main /tmp/doublefree.c:20

previously allocated by thread T0 here:
    #0 0x7afb7412b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x5b5c3c55c28c in main /tmp/doublefree.c:10
```

Two `freed by` blocks, at line 20 and line 21. That is the diagnosis in one glance.

A double free is not always a crash. It can silently corrupt the allocator's free lists, and the program keeps running until something unrelated breaks hours later. The abort is the good outcome.

**The fix is one owner, and NULL out every alias you can reach.** In practice:

- Pass ownership explicitly: either the caller frees, or the callee frees, never both.
- If a function takes ownership, document it (`/* OWNERSHIP: takes ownership, always frees */`) and set the caller's pointer to `NULL` — which means taking a `int **`, since a pointer parameter is only a copy.
- Make cleanup idempotent: a `vector_free` that sets `v->data = NULL` is safe to call twice; a `free(v->data)` without the `NULL` is not.

---

## Dangling Pointers Beyond `free`

A **dangling pointer** is any pointer to a block that no longer exists. It is not a separate bug: it is what every use-after-free has as its cause.

### The `free; p = NULL;` Pattern

```c
int *p = malloc(10 * sizeof(int));
/* ... use p ... */

free(p);
p = NULL;   /* from here on, p is honest about there being nothing there */
```

Read the "why" carefully, because it is subtle:

- Setting `p = NULL` does **not** make the freed memory available again. It is gone.
- What it does is remove the *trap*. Without it, a stray use reads garbage and the program misbehaves in a way that looks like a logic error. With it, the same stray use is a null dereference: an immediate, obvious, unmistakable crash at the exact line.
- It also makes `if (p == NULL)` a meaningful check, which it is not otherwise — remember `free(p)` does not modify `p`.

```text
  p holds a real address        p is NULL
  +---------------------+       +---------+
  | p = 0x5555..a010    |  -->  | p = 0   |
  +---------------------+ free  +---------+
   |                                        |
   |  *p is a use-after-free                |  *p is a null dereference
   |  reads garbage, may crash              |  crashes HERE, every time,
   |  the symptom is far from the cause     |  blaming the exact line
   |  the address may even be valid again   |
```

So the rule is: **`free(p)` is always followed by `p = NULL`**, in the same statement pair, with nothing in between. The one exception is when the variable goes out of scope immediately, where it makes no difference.

### The Dangerous Case: A Pointer Into the Middle

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *values = malloc(5 * sizeof(int));
    int *middle;
    int i;

    if (values == NULL) {
        return 1;
    }

    for (i = 0; i < 5; i++) {
        values[i] = i + 1;
    }

    middle = &values[2];   /* points into the MIDDLE of the block */
    printf("values[2] = %d\n", *middle);

    free(values);
    values = NULL;         /* the block is gone, but `middle` still points into it */

    printf("after free, *middle = %d  <- reading freed memory\n", *middle);
    printf("middle == NULL ? %s  <- the free never touched it\n",
           (middle == NULL) ? "yes" : "no");

    return 0;
}
```

GCC warns again, and the program still runs and still exits 0:

```text
warning: pointer 'middle' used after 'free' [-Wuse-after-free]
   23 |     printf("after free, *middle = %d  <- reading freed memory\n", *middle);
      |                                     ~~~~~~~~~
/tmp/middle.c:20:5: note: call to 'free' here
   20 |     free(values);
      |     ^~~~~~~~~~~~
```

```text
values[2] = 3
after free, *middle = 365142072  <- reading freed memory
middle == NULL ? no  <- the free never touched it
```

Setting `values = NULL` did nothing for `middle`, because `middle` holds a **different address** — 8 bytes into the block, not the start. The sanitizer shows exactly that offset:

```text
=================================================================
==380722==ERROR: AddressSanitizer: heap-use-after-free on address 0x701ac1fe0048 at pc 0x5ca1e1fcb4ff bp 0x7ffccebdbdb0 sp 0x7ffccebdbda0
READ of size 4 at 0x701ac1fe0048 thread T0
    #0 0x5ca1e1fcb4fe in main /tmp/middle.c:23
    #1 0x73eac302a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #2 0x73eac302a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #3 0x5ca1e1fcb1e4 in _start (/tmp/middle_asan+0x21e4) (BuildId: 3695b8a4cca688a30049843f176b9f9628a880fb)

0x701ac1fe0048 is located 8 bytes inside of 20-byte region [0x701ac1fe0040,0x701ac1fe0054)
freed by thread T0 here:
    #0 0x73eac3d2a3ff in free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:51
    #1 0x5ca1e1fcb497 in main /tmp/middle.c:20
```

Compare with the earlier report, which said `0 bytes inside`. Here it says **8 bytes inside** — that is `&values[2]` in a 20-byte block of `int`s. The middle pointer is the reason `free(middle)` is an *invalid* free, and the reason `values = NULL` cannot save it.

**The rule that follows: keep exactly one pointer to the start of a block, and treat every other pointer into it as a temporary that dies with the block.** If you need a stable "current element" pointer across a realloc, recompute it from the index after the realloc (`&data[i]`, not a saved `&data[i]`).

---

## Why These Bugs Are So Nasty

Four properties make heap bugs worse than ordinary logic errors.

**1. The program often still works.** The `malloc(4)` for a 16-byte struct, the use-after-free that printed `-354804762`, the realloc that happened not to move — all of them ran to completion, produced output, and exited 0. The bug is latent, not absent. It becomes visible when the input changes, the heap layout changes, or the compiler changes.

**2. The failure appears far from the cause.** A leak in a function called once at startup shows up as an out-of-memory crash in a loop that ran two million iterations later. A one-byte overflow corrupts a neighbouring block, and the wrong value shows up three subsystems away. This is why sanitizer reports print the *allocation site* as well as the failure site — the allocation is usually the place to look.

**3. It may only happen on someone else's machine.** The realloc example is the clearest case: the block was extended in place on this run, so the bug never fired. Change the allocator's state, the malloc size, the thread count, or the libc version, and the block moves — and now it fails in production, on a user's laptop, with no way to reproduce it from the source. Memory bugs are why C programs are tested on many machines and why the sanitizers exist.

**4. They are undefined behaviour, so the compiler may help or hinder.** A double free that happens to be caught is the *lucky* case. The standard permits the compiler to assume UB never happens, which means at `-O2` it may delete the very check that would have detected it, hoist a load out of a loop, or reorder a store past a `free`. The bugs that survive to release builds are rarely the ones you wrote deliberately.

---

## The Defensive Checklist

```text
[ ] malloc(n * sizeof(type)); never malloc(n) for a non-byte type
[ ] Check the result of every malloc, calloc and realloc against NULL
[ ] On realloc failure, keep the old pointer: the block is still valid
[ ] Use ONLY the pointer realloc returns; never keep the old one
[ ] One malloc, one free, in the same scope where possible
[ ] free(p) is always followed by p = NULL
[ ] free only pointers malloc/calloc/realloc returned, exactly once
[ ] free is never called on a stack address, a string literal, or a
    pointer into the middle of a block
[ ] A struct that owns a pointer has a free function that NULLs the field
[ ] Every index into a heap array is checked against the stored length
[ ] Reading a char buffer uses a width-limited scanf or snprintf
[ ] Every pointer-returning function documents ownership in a comment
[ ] Never return &local: the block dies with the frame
[ ] gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined
      -fno-omit-frame-pointer, then RUN IT
[ ] Run it again with different inputs, sizes and allocation orders
```

That last pair of lines is the whole discipline. There is no substitute for executing the program under a sanitizer, and the check is nearly free.

---

#### [Previous](./02-part.md) | [Next](./04-part.md)
