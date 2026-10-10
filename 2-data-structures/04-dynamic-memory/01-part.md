# Stack vs Heap and `malloc`/`free`

This is the chapter the whole course has been building toward. A local variable is not "a variable in memory somewhere" — it is a **block of bytes in a specific region**, reclaimed at a specific moment, by a rule you did not write. Now you choose the region, and you become responsible for giving the bytes back.

---

## Two Regions, Two Sets of Rules

When a program runs, the memory it uses comes from two places that behave completely differently.

```text
  address
  high      +-----------------------------+
           |  ...                        |
           |  stack   grows DOWN          |   local variables
           |  [main's frame]              |   automatic, freed for you
           |  [print_total's frame]       |   fast, small (a few MB)
           |  [main's local array]        |   LIFO
           +-----------------------------+
           |      free space              |
           |      (unmapped)              |
           +-----------------------------+
           |  heap    grows UP            |   malloc / calloc / realloc
           |  [block A: 40 bytes]         |   manual, you MUST free
           |  [block B: 12 bytes]         |   slower, as large as the machine
           |  [block C: 4096 bytes]       |   no order, no tracking
           +-----------------------------+
  low
```

| | **Stack** | **Heap** |
| --- | --- | --- |
| Who allocates | the compiler, for you | you, with `malloc` |
| When it dies | when the block ends | when **you** call `free` |
| Size | small: typically 1–8 MB | large: limited by RAM and address space |
| Speed | very fast (just move a pointer) | slower (a real allocator, with bookkeeping) |
| Order | LIFO — last in, first out | none — any block, any time |
| Failure mode | stack overflow, crash | silent leak, corruption, or `NULL` |
| Typical use | every local variable, small arrays | data of unknown size, big buffers, anything that outlives its function |

### The Stack in Detail

Every time you enter a block, the compiler reserves a region for its locals. When the block ends, that region is reclaimed — not cleared, just made available again.

```text
  int main(void) {                 <- frame 1 created
      int n = 5;                   <- 4 bytes at frame1 + 0
      int total = 0;               <- 4 bytes at frame1 + 4
      int data[100];               <- 400 bytes at frame1 + 8

      print_total(data, 100);      <- frame 2 created on top
  }

  void print_total(int *a, int count) {   <- frame 2: 16 bytes
      int i;
      int sum = 0;
  }                                 <- frame 2 popped, 16 bytes recycled
  }                                 <- frame 1 popped, 408 bytes recycled
```

Two consequences you can feel immediately:

- **A big local array is dangerous.** `int data[1000000];` is 4 MB on the stack, and if the stack is 8 MB you get one of those. Large data belongs on the heap.
- **A local does not outlive its block.** Returning the address of a local variable hands the caller a pointer to memory that is already recycled:

```c
int *bad(void) {
    int x = 42;
    return &x;   /* x is dead the moment this returns */
}
```

That compiles, runs, and is a use-after-free. The heap exists precisely so you can have data that outlives its creator.

### The Heap in Detail

The heap is a large pool the operating system hands to the process. `malloc` carves a block out of it and gives you back its address; `free` puts the block back.

```text
  the heap before your malloc(20)

  +--------+--------+--------+--------+--------+--------+
  | free   | free   | in use | free   | free   | free   |
  +--------+--------+--------+--------+--------+--------+

  after malloc(20) -- the allocator finds a free hole big enough
                    and marks exactly 20 bytes as yours

  +--------+--------+--------+--------+--------+--------+
  | free   | free   | YOURS  | free   | free   | free   |
  +--------+--------+--------+--------+--------+--------+
                    ^ 20 bytes
                    |
                    +-- the only bytes you may touch
```

Three things to notice in that picture:

1. **You get exactly the bytes you asked for.** Not rounded up, not a struct, not a fence. Ask for 20 and 20 is what you get.
2. **Nothing around it is yours.** The bytes before and after belong to other blocks. Writing one byte past the end does not fail loudly — it corrupts someone else's data, which is the worst possible outcome.
3. **The block has no metadata you can see.** There is no length field. The allocator keeps a table outside your block, and **your program does not know how big it is**. This is why every array you allocate must carry its own length in a separate variable.

---

## `malloc`

`malloc` takes one argument — a number of **bytes** — and returns a `void *`, a pointer to uninitialized memory.

```c
void *malloc(size_t size);
```

Because it returns `void *` (a pointer to "some unspecified type"), you must **cast** it to the type you want:

```c
int *values = (int *) malloc(10 * sizeof(int));
```

### The Rule That Prevents Most Bugs

> **`malloc` counts bytes. Always write the size as `count * sizeof(type)`.**

```c
/* RIGHT -- the compiler does the arithmetic */
int *a = malloc(n * sizeof(int));
Point *p = malloc(sizeof(Point));
char *s = malloc(n * sizeof *s);       /* also correct, and self-maintaining */

/* WRONG -- n means n BYTES */
int *a = malloc(n);
Point *p = malloc(8);
```

`malloc(n)` for an array of `int` gives you a block for **n bytes**, which holds `n / sizeof(int)` ints — two and a half of them, if `n` is 10. Everything you write past that point is heap corruption.

Here is the mistake, running for real:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *values = malloc(10);      /* WRONG: 10 BYTES, not 10 ints */
    int i;

    for (i = 0; i < 10; i++) {
        values[i] = i * 2;
    }

    for (i = 0; i < 10; i++) {
        printf("values[%d] = %d\n", i, values[i]);
    }

    free(values);
    return 0;
}
```

Compiled with the normal flags it does not even reach the print loop:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g malloc10.c -o malloc10
./malloc10
```

```text
malloc(): corrupted top size
Aborted (core dumped)
```

Under the sanitizer, the exact line and the exact block:

```text
=================================================================
==377350==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x6df77cde0018 at pc 0x57c5a720d3b8 bp 0x7fff00ac8e10 sp 0x7fff00ac8e00
WRITE of size 4 at 0x6df77cde0018 thread T0
    #0 0x57c5a720d3b7 in main /tmp/malloc10.c:9
    #1 0x71d77de2a600 in __libc_start_call_main ../sysdeps/nptl/libc-start_call_main.h:59
    #2 0x71d77de2a717 in __libc_start_main_impl ../csu/libc-start.c:360

0x6df77cde001a is located 0 bytes after 10-byte region [0x6df77cde0010,0x6df77cde001a)
allocated by thread T0 here:
    #0 0x71d77eb2b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x57c5a720d2e1 in main /tmp/malloc10.c:5

SUMMARY: AddressSanitizer: heap-buffer-overflow /tmp/malloc10.c:9 in main
```

Read `10-byte region` next to `values[2]`. The block held two whole `int`s and two bytes. `values[2]` starts at offset 8, and writing it touched offset 8–11, which is past the end.

The correct version of the program:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int n = 5;
    int *values;
    int i;
    int sum = 0;

    values = malloc(n * sizeof(int));
    if (values == NULL) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    printf("allocated %zu bytes\n", n * sizeof(int));
    printf("each element is %zu bytes\n\n", sizeof(int));

    for (i = 0; i < n; i++) {
        values[i] = (i + 1) * 10;
    }

    printf("  offset  value\n");
    printf("  ------  -----\n");
    for (i = 0; i < n; i++) {
        printf("  %6zu  %5d\n", (size_t)(&values[i] - values) * sizeof(int), values[i]);
    }

    for (i = 0; i < n; i++) {
        sum += values[i];
    }
    printf("\nsum = %d\n", sum);

    free(values);
    values = NULL;

    return 0;
}
```

```text
allocated 20 bytes
each element is 4 bytes

  offset  value
  ------  -----
       0     10
       4     20
       8     30
      12     40
      16     50

sum = 150
```

`malloc(n * sizeof(int))` asked for 20 bytes, got 20 bytes, and the block holds exactly 5 ints at offsets 0, 4, 8, 12, 16.

### The Memory Is Garbage

`malloc` does not clear anything. A fresh block holds whatever the last user of those bytes left there:

```c
int *values = malloc(100 * sizeof(int));
/* values[0] .. values[99] are INDETERMINATE, not zero */
```

Printing an uninitialized `int` gives you a random number. **Always initialize before reading.** `calloc` and `memset` do this for you, and the next part covers both.

### Always Check for NULL

`malloc` returns `NULL` when it cannot satisfy the request — an absurd size, or a genuinely exhausted machine. Dereferencing `NULL` is undefined behavior and, in practice, a segmentation fault.

```c
int *values = malloc(n * sizeof(int));
if (values == NULL) {
    fprintf(stderr, "out of memory\n");
    return 1;
}
```

There is no way to make the check disappear. Do it anyway.

For small, bounded allocations the check is arguably decorative, but for anything whose size comes from input, a file, or a loop, it is the difference between a clean error message and a crash.

---

## `free`

`free` returns a block to the heap. It takes the pointer `malloc` gave you, and nothing else.

```c
void free(void *ptr);
```

### Five Rules

| Rule | Why |
| --- | --- |
| Free only what `malloc`/`calloc`/`realloc` returned | Freeing a stack address, a string literal, or a pointer to the middle of a block corrupts the allocator |
| Free each block exactly **once** | A second `free` on the same block is undefined behavior and usually aborts |
| `free(NULL)` is safe and does nothing | It lets you write cleanup without an `if` |
| After `free`, the pointer is **dangling** — never read it, write through it, or compare it | The memory may already belong to someone else |
| `free` does not change your variable | The pointer still holds the old address; set it to `NULL` yourself |

That last pair is the one that catches everybody:

```c
free(values);
printf("%d\n", values[0]);   /* use after free */
if (values == NULL) { ... }  /* FALSE: values still holds the old address */
```

`free(values)` does **not** set `values` to `NULL`, because `free` receives a copy of the address. Only your own assignment can do that:

```c
free(values);
values = NULL;
```

Setting a pointer to `NULL` after freeing it does not make the freed memory safe — it makes the *next mistake* a clean crash instead of silent corruption.

### The Pairing Discipline

Every `malloc` gets exactly one `free`, in the same scope if possible:

```c
int *values = malloc(n * sizeof(int));
if (values == NULL) {
    return 1;
}

/* ... use it ... */

free(values);
```

Real code is full of early returns and `goto`s between the two. The three ways to keep the pair honest:

| Technique | How |
| --- | --- |
| Single exit point | do the cleanup at the end of the function, once |
| `goto cleanup` | `goto` to a label at the bottom where the `free` lives |
| Owner object | a struct with a `free` function; the container frees what it holds |

```c
/* single exit point: every path ends at the free */
int process(void) {
    int *data = malloc(100 * sizeof(int));
    int result = 0;

    if (data == NULL) {
        return -1;
    }

    if (!load(data)) {
        result = -1;
    } else {
        result = save(data);
    }

    free(data);
    return result;
}
```

---

## The Same Program, Leaking

Here is the correct program above with a single line removed. **This is what a leak looks like, and it is the single most common bug in C.**

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int n = 5;
    int *values;
    int i;
    int sum = 0;

    values = malloc(n * sizeof(int));
    if (values == NULL) {
        return 1;
    }

    for (i = 0; i < n; i++) {
        values[i] = (i + 1) * 10;
    }

    for (i = 0; i < n; i++) {
        sum += values[i];
    }
    printf("sum = %d\n", sum);

    /* BUG: no free(values). The program ends and the block is never returned. */

    return 0;
}
```

Compiled and run the ordinary way, it is perfect:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g leak1.c -o leak1
./leak1
```

```text
sum = 150
```

Exit status 0. No warning. Correct output. Under the sanitizer:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer leak1.c -o leak1_san
./leak1_san
```

```text
=================================================================
==377436==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 20 byte(s) in 1 object(s) allocated from:
    #0 0x71a053d2b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x643f428e52d8 in main /tmp/leak1.c:10
    #2 0x71a05302a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #3 0x71a05302a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #4 0x643f428e51e4 in _start (/tmp/leak1+0x11e4) (BuildId: e05f36ed769cf333d334c8e427b2a6cff8831d3b)

SUMMARY: AddressSanitizer: 20 byte(s) leaked in 1 allocation(s).
```

`Direct leak of 20 byte(s)` and the frame `main /tmp/leak1.c:10` — the `malloc` line, in the function that leaked it. That is the whole report: **what** leaked, **how much**, and **where it came from**.

### Why a Leak Matters

- **Memory is never returned to the OS.** When your program ends, the OS reclaims everything, so a leak in a short-lived program is harmless. When the program is a server, a loop, or a game, the leak accumulates until the process dies.
- **The symptom appears far away.** The allocation is in function A; the out-of-memory crash happens ten thousand iterations later in a function that never allocated anything.
- **Leaked memory is not necessarily lost forever.** It may be reused by a later `malloc`, which is why leaks often stay invisible until the program's shape changes.
- **In a loop it is guaranteed.** A leak inside a `for` over 1,000,000 items allocates 1,000,000 blocks.

The habit that catches them: **build with the sanitizers and run the program, every time.** The build command for the rest of this module is:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer file.c -o file
```

---

## What the Heap Looks Like in a Real Program

One `malloc`, one `free`, and the block's whole life:

```text
  1. int *values = malloc(5 * sizeof(int));

     +---------------+
     |  values = 0x..|------------------------+
     +---------------+                        |
                                              v
     +--------+--------+--------+--------+--------+
     |   ??   |   ??   |   ??   |   ??   |   ??   |   uninitialized: garbage
     +--------+--------+--------+--------+--------+

  2. for (i = 0; i < 5; i++) values[i] = ...;

     +--------+--------+--------+--------+--------+
     |   10   |   20   |   30   |   40   |   50   |   yours to read
     +--------+--------+--------+--------+--------+

  3. sum = ...;  printf(...);

     unchanged

  4. free(values);

     +--------+--------+--------+--------+--------+
     | free   | free   | free   | free   | free   |   back in the pool
     +--------+--------+--------+--------+--------+
     values still points at it  <- dangling
```

Step 4 is the one that catches people. `values` is a variable like any other; `free` took a **copy** of the address and did nothing to `values`. Only `values = NULL;` after the `free` clears it.

---

## Checklist

```text
[ ] Know which region a variable lives in before you declare it
[ ] Big data (> a few KB) belongs on the heap, not the stack
[ ] A local does not outlive its block; never return &local
[ ] malloc takes BYTES: always n * sizeof(type)
[ ] Cast the void * to the type you want
[ ] Check every malloc against NULL
[ ] malloc memory is garbage: initialize before reading
[ ] One malloc, one free, in the same scope where possible
[ ] Only free what malloc returned, and only once
[ ] free(NULL) is safe; after free, set the pointer to NULL yourself
[ ] Build with -fsanitize=address,undefined and run
```

---

#### [Previous](../03-structs/exercises.md) | [Next](./02-part.md)
