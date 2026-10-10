# Memory Tools: valgrind and Sanitizers

`-Wall` catches what the compiler can prove. It cannot prove anything about what happens at runtime. This part is about the tools that watch the program while it runs: **valgrind** for leaks and invalid access, **sanitizers** that are compiled into the binary, and gcc's built-in **static analyzer**.

---

## valgrind

### Install

```bash
# Ubuntu / Debian
sudo apt install valgrind

# Fedora
sudo dnf install valgrind

# macOS
brew install valgrind
```

Check it:

```bash
valgrind --version
```

```text
valgrind-3.26.0
```

If that command fails with `valgrind: command not found`, it is not installed. Everything in this section works exactly the same without it — read the reports and know what they mean, install it when you can.

valgrind does not need special compilation flags. It runs your normal binary:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g leak.c -o leak
valgrind --leak-check=full ./leak
```

`-g` still matters: without debug symbols, valgrind reports addresses and function names from the dynamic table instead of `file.c:15`.

valgrind is roughly 20 to 50 times slower than a normal run. That is fine for a test program and painful for anything heavy.

### A Leaking Program

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static char **create_names(int count) {
    char **names = malloc((size_t)count * sizeof(char *));
    int i;

    if (names == NULL) {
        return NULL;
    }

    for (i = 0; i < count; i++) {
        names[i] = malloc(6 * sizeof(char));
        if (names[i] == NULL) {
            return NULL;
        }
        strcpy(names[i], "name!");
    }
    return names;
}

int main(void) {
    char **names = create_names(3);
    int i;

    if (names == NULL) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    for (i = 0; i < 3; i++) {
        printf("names[%d] = %s\n", i, names[i]);
    }

    free(names);
    return 0;
}
```

Run it normally:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g leak.c -o leak
./leak
```

```text
names[0] = name!
names[1] = name!
names[2] = name!
```

It works perfectly. The leak is invisible from the outside — the program printed everything you asked and exited with status 0.

### The valgrind Report

```bash
valgrind --leak-check=full ./leak
```

```text
==352840== Memcheck, a memory error detector
==352840== Copyright (C) 2002-2024, and GNU GPL'd, by Julian Seward et al.
==352840== Using Valgrind-3.26.0 and LibVEX; rerun with -h for copyright info
==352840== Command: ./leak
==352840==
names[0] = name!
names[1] = name!
names[2] = name!
==352840==
==352840== HEAP SUMMARY:
==352840==     in use at exit: 18 bytes in 3 blocks
==352840==   total heap usage: 5 allocs, 2 frees, 4,138 bytes allocated
==352840==
==352840== 18 bytes in 3 blocks are definitely lost in loss record 1 of 1
==352840==    at 0x4850858: malloc (vg_replace_malloc.c:447)
==352840==    by 0x4001206: create_names (leak.c:14)
==352840==    by 0x400127B: main (leak.c:24)
==352840==
==352840== LEAK SUMMARY:
==352840==    definitely lost: 18 bytes in 3 blocks
==352840==    indirectly lost: 0 bytes in 0 blocks
==352840==      possibly lost: 0 bytes in 0 blocks
==352840==    still reachable: 0 bytes in 0 blocks
==352840==         suppressed: 0 bytes in 0 blocks
==352840==
==352840== For lists of detected and suppressed errors, rerun with: -s
==352840== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

The program frees the outer `names` array but never the three strings inside it. The report points at `leak.c:14`, which is `names[i] = malloc(6 * sizeof(char));`, and shows the call chain that reached it.

### Reading the Blocks

| Block | Meaning | Your problem? |
| --- | --- | --- |
| `definitely lost` | no pointer to the block exists anywhere valgrind can see. Nobody can ever reach it again | **yes, always a real leak** |
| `indirectly lost` | the block is only reachable *through* another block that is itself definitely lost. It would have been freed if the parent had been freed | counted with its parent; fix the parent |
| `possibly lost` | an interior pointer exists, so valgrind cannot be sure. Usually a pointer that only differs from the real one by its base address | often real, worth a look |
| `still reachable` | a pointer exists, in a global or a static, so the block could be used later | usually fine. A cache you never free on purpose |
| `suppressed` | errors you told valgrind to ignore with a suppression file | nothing |

The line `in use at exit` is the total across all categories. `total heap usage: 5 allocs, 2 frees` is a good sanity check: 5 allocations happened, only 2 were ever freed.

### Seeing `still reachable` on Purpose

```c
#include <stdio.h>
#include <stdlib.h>

int *g_cache = NULL;

int main(void) {
    g_cache = malloc(8 * sizeof(int));
    g_cache[0] = 1;
    printf("cached = %d\n", g_cache[0]);
    return 0;
}
```

```bash
valgrind --leak-check=full --show-leak-kinds=all ./reach
```

```text
==353305== Memcheck, a memory error detector
==353305== Copyright (C) 2002-2024, and GNU GPL'd, by Julian Seward et al.
==353305== Using Valgrind-3.26.0 and LibVEX; rerun with: -h for copyright info
==353305== Command: ./reach
==353305==
cached = 1
==353305==
==353305== HEAP SUMMARY:
==353305==     in use at exit: 32 bytes in 1 blocks
==353305==   total heap usage: 2 allocs, 1 frees, 4,128 bytes allocated
==353305==
==353305== 32 bytes in 1 blocks are still reachable in loss record 1 of 1
==353305==    at 0x4850858: malloc (vg_replace_malloc.c:447)
==353305==    by 0x400117A: main (reach.c:7)
==353305==
==353305== LEAK SUMMARY:
==353305==    definitely lost: 0 bytes in 0 blocks
==353305==    indirectly lost: 0 bytes in 0 blocks
==353305==      possibly lost: 0 bytes in 0 blocks
==353305==    still reachable: 32 bytes in 1 blocks
==353305==         suppressed: 0 bytes in 0 blocks
==353305==
==353305== For lists of detected and suppressed errors, rerun with: -s
==353305== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

`ERROR SUMMARY: 0 errors`. The block is not counted as an error because `g_cache` can still reach it. That is the difference between a leak and an intentional cache, and valgrind distinguishes them for you.

### Catching an Invalid Read

valgrind also catches out-of-bounds access, not just leaks:

```c
#include <stdio.h>
#include <stdlib.h>

static int sum_all(int *values, int count) {
    int i;
    int total = 0;

    for (i = 0; i <= count; i++) {
        total += values[i];
    }
    return total;
}

int main(void) {
    int *data = malloc(5 * sizeof(int));
    int i;

    for (i = 0; i < 5; i++) {
        data[i] = (i + 1) * 10;
    }

    printf("sum = %d\n", sum_all(data, 5));

    free(data);
    return 0;
}
```

```bash
valgrind ./sum
```

```text
==353154== Memcheck, a memory error detector
==353154== Copyright (C) 2002-2024, and GNU GPL'd, by Julian Seward et al.
==353154== Using Valgrind-3.26.0 and LibVEX; rerun with: -h for copyright info
==353154== Command: ./sum
==353154==
==353154== Invalid read of size 4
==353154==    at 0x40011BC: sum_all (sum.c:9)
==353154==    by 0x4001234: main (sum.c:22)
==353154==  Address 0x4a9e054 is 0 bytes after a block of size 20 alloc'd
==353154==    at 0x4850858: malloc (vg_replace_malloc.c:447)
==353154==    by 0x40011E7: main (sum.c:15)
==353154==
sum = 150
==353154==
==353154== HEAP SUMMARY:
==353154==     in use at exit: 0 bytes in 0 blocks
==353154==   total heap usage: 2 allocs, 2 frees, 4,116 bytes allocated
==353154==
==353154== All heap blocks were freed -- no leaks are possible
==353154==
==353154== For lists of detected and suppressed errors, rerun with: -s
==353154== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

Three lines matter:

- `Invalid read of size 4` at `sum.c:9` — the `total += values[i]` line, reading one `int` past the end.
- `Address 0x4a9e054 is 0 bytes after a block of size 20 alloc'd` — the exact boundary. The block is 20 bytes (5 ints), and the read is at the first byte after it.
- `by 0x40011E7: main (sum.c:15)` — who allocated it, so you can work out the intended length.

And note `sum = 150` still printed. The program produced the right answer *by luck*, because the byte after the block happened to hold something harmless. This is undefined behaviour doing what it always does: working, until it does not.

### Options Worth Knowing

| Option | Effect |
| --- | --- |
| `--leak-check=full` | the detailed leak breakdown per block |
| `--show-leak-kinds=all` | also show `still reachable` blocks, which are hidden by default |
| `--track-origins=yes` | where did the uninitialized value come from (slow) |
| `--error-exitcode=1` | exit nonzero when valgrind finds errors, for scripts and CI |

---

## AddressSanitizer and UndefinedBehaviorSanitizer

valgrind watches from outside, so it is slow and it sees the program as the CPU sees it. A **sanitizer** is compiled into the binary: the compiler inserts checks that trap the moment a bad access happens, and it prints a full report with the type of bug and the allocation site.

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g \
    -fsanitize=address,undefined -fno-omit-frame-pointer file.c -o file
```

| Flag | Full name | Catches |
| --- | --- | --- |
| `-fsanitize=address` | AddressSanitizer | heap and stack overflows, use after `free`, double `free`, leaks (via LeakSanitizer) |
| `-fsanitize=undefined` | UndefinedBehaviorSanitizer | signed overflow, shifts out of range, division by zero, bad casts, misaligned access |
| `-fno-omit-frame-pointer` | — | keeps frame pointers so the report can show a real stack trace |

Both run together by default and report independently. The program is 2 to 10 times slower, not 50, and the reports are far more precise.

### Catching a Buffer Overflow

The `strcpy` into an 8-byte buffer, from the previous chapter:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static void copy_name(char *dest) {
    strcpy(dest, "Wesley Bertipaglia");
}

int main(void) {
    char name[8];

    copy_name(name);
    printf("name = %s\n", name);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer over.c -o over
./over
```

```text
=================================================================
==351741==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7756098f0028 at pc 0x7b560cb28d2d bp 0x7ffcf22f5440 sp 0x7ffcf22f4be8
WRITE of size 19 at 0x7756098f0028 thread T0
    #0 0x7b560cb28d2c in memcpy ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors_memintrinsics.inc:117
    #1 0x6509767a8289 in copy_name /tmp/over.c:6
    #2 0x6509767a831f in main /tmp/over.c:12
    #3 0x7b560be2a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #4 0x7b560be2a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #5 0x6509767a8184 in _start (/tmp/over+0x1184) (BuildId: bc56ae3192aa6410ec2d79291ed9bb427a314d35)

Address 0x7756098f0028 is located in stack of thread T0 at offset 40 in frame
    #0 0x6509767a829c in main /tmp/over.c:9

  This frame has 1 object(s):
    [32, 40) 'name' (line 10) <== Memory access at offset 40 overflows this variable
HINT: this may be a false positive if your program uses some custom stack unwind mechanism, swapcontext or vfork
      (longjmp and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-buffer-overflow /tmp/over.c:6 in copy_name
```

The header line tells you everything: `stack-buffer-overflow`, a `WRITE of size 19`, at the first byte after `name`. Then the exact frame: `copy_name` at line 6 wrote 19 bytes (18 characters plus the `\0`) into an 8-byte array, called from `main` at line 12.

The `[32, 40) 'name'` line is the memory map: your array lives at offsets 32 to 39 of that stack frame, and the write began at offset 40 — the byte immediately after.

The fix, and a real check of the fix:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static int copy_name(char *dest, size_t size) {
    int written = snprintf(dest, size, "%s", "Wesley Bertipaglia");

    if (written < 0 || (size_t)written >= size) {
        fprintf(stderr, "error: name does not fit in %zu bytes\n", size);
        return -1;
    }
    return 0;
}

int main(void) {
    char small[8];
    char large[32];

    if (copy_name(small, sizeof(small)) != 0) {
        return 1;
    }
    if (copy_name(large, sizeof(large)) != 0) {
        return 1;
    }

    printf("large = %s\n", large);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer fixed.c -o fixed
./fixed
```

```text
error: name does not fit in 8 bytes
exit status: 1
```

`snprintf` truncates instead of overflowing, returns the length it *wanted* to write, and the comparison against `size` is how you detect that. Same program, no sanitizer output.

### Use After Free

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static void use_after_free(void) {
    char *name = malloc(16);

    strcpy(name, "Wesley");
    printf("allocated: %s\n", name);

    free(name);
    printf("after free: %s\n", name);
}

int main(void) {
    use_after_free();
    return 0;
}
```

Notice that gcc itself catches this one:

```text
use.c: In function ‘use_after_free’:
use.c:12:5: warning: pointer ‘name’ used after ‘free’ [-Wuse-after-free]
   12 |     printf("after free: %s\n", name);
      |     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
use.c:11:5: note: call to ‘free’ here
   11 |     free(name);
      |         ~~~~~~
```

And with the sanitizer:

```text
=================================================================
==352031==ERROR: AddressSanitizer: heap-use-after-free on address 0x6dc0fc7e0010 at pc 0x71a0fe4d0cae bp 0x7ffe1117b090 sp 0x7ffe1117b7a808
READ of size 2 at 0x6dc0fc7e0010 thread T0
    #0 0x71a0fe4d0cad in printf_common ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors_format.inc:567
    #1 0x71a0fe4f18d3 in vprintf ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors.inc:1731
    #2 0x71a0fe4f3aa4 in printf ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors.inc:1791
    #3 0x55611e2222d5 in use_after_free /tmp/use.c:12
    #4 0x55611e2222e5 in main /tmp/use.c:16
    #5 0x71a0fd82a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #6 0x71a0fd82a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #7 0x55611e222184 in _start (/tmp/use+0x1184) (BuildId: 71b63c1cc32018c5f22788fecc4e4c965b1edd99)

0x6dc0fc7e0010 is located 0 bytes inside of 16-byte region [0x6dc0fc7e0010,0x6dc0fc7e0020)
freed by thread T0 here:
    #0 0x71a0fe52a3ff in free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:51
    #1 0x55611e2222ba in use_after_free /tmp/use.c:11
```

The report has two halves: where the bad access happened (line 12) and where the memory was freed (line 11). That pairing is the whole point. This is why "set the pointer to `NULL` after `free`" matters — then a stray access is `NULL`, and the crash is immediate and obvious.

### Leaks, with Direct and Indirect

The same leaking program from the valgrind section, under ASan:

```text
=================================================================
==353254==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 18 byte(s) in 3 object(s) allocated from:
    #0 0x70decc32b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x5fdb92ace39c in create_names /tmp/leak.c:14
    #2 0x5fdb92ace596 in main /tmp/leak.c:24
    #3 0x70decb62a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #4 0x70decb62a700 in __libc_start_main_impl ../csu/libc-start.c:360
    #5 0x5fdb92ace244 in _start (/tmp/leak_asan+0x2244) (BuildId: c27c5ff911ba7084b0cdf887887af7bac128a553)

SUMMARY: AddressSanitizer: 18 byte(s) leaked in 3 allocation(s).
```

`18 byte(s) in 3 object(s)`: three `malloc` calls, six bytes each, all from the same line of the same function. One leak record covers all three because they came from the same place. Fixing that one line clears all three.

ASan also has a notion of indirect leak, and the clearest way to see it is with an array of pointers. Here is a program that builds a two-row table and never frees any of it:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static void show_table(int rows) {
    char **table = malloc((size_t)rows * sizeof(char *));
    int i;

    if (table == NULL) {
        fprintf(stderr, "out of memory\n");
        exit(1);
    }

    for (i = 0; i < rows; i++) {
        table[i] = malloc(8 * sizeof(char));
        if (table[i] == NULL) {
            exit(1);
        }
        strcpy(table[i], "cell");
        printf("table[%d] = %s\n", i, table[i]);
    }
}

int main(void) {
    show_table(2);
    return 0;
}
```

```text
table[0] = cell
table[1] = cell
```

Under valgrind:

```text
==362941== HEAP SUMMARY:
==362941==     in use at exit: 32 bytes in 3 blocks
==362941==   total heap usage: 4 allocs, 1 frees, 4,128 bytes allocated
==362941==
==362941== 32 (16 direct, 16 indirect) bytes in 1 blocks are definitely lost in loss record 2 of 2
==362941==    at 0x4850858: malloc (vg_replace_malloc.c:447)
==362941==    by 0x40011C9: show_table (table.c:6)
==362941==    by 0x40012CD: main (table.c:25)
==362941==
==362941== LEAK SUMMARY:
==362941==    definitely lost: 16 bytes in 1 blocks
==362941==    indirectly lost: 16 bytes in 2 blocks
==362941==      possibly lost: 0 bytes in 0 blocks
==362941==    still reachable: 0 bytes in 0 blocks
==362941==         suppressed: 0 bytes in 0 blocks
```

Read the first line of the loss record: `32 (16 direct, 16 indirect) bytes in 1 blocks`. One lost block — the `table` array itself — is 16 bytes. The two strings it pointed at are 16 bytes more, and they are only lost *because* `table` was lost.

Under ASan, the same program, with the distinction spelled out:

```text
=================================================================
==362961==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 16 byte(s) in 1 object(s) allocated from:
    #0 0x711c01b2b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x5a470ec5634b in show_table /tmp/table.c:6
    #2 0x5a470ec566bb in main /tmp/table.c:25
    #3 0x711c00e2a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #4 0x711c00e2a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #5 0x5a470ec56264 in _start (/tmp/table_asan+0x2264) (BuildId: 10353961cca9c8c4aac8aea3250e751bb3efb415)

Indirect leak of 16 byte(s) in 2 object(s) allocated from:
    #0 0x711c01b2b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x5a470ec5641c in show_table /tmp/table.c:15
    #2 0x5a470ec566bb in main /tmp/table.c:25
    #3 0x711c00e2a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #4 0x711c00e2a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #5 0x5a470ec56264 in _start (/tmp/table_asan+0x2264) (BuildId: 10353961cca9c8c4aac8aea3250e751bb3efb415)

SUMMARY: AddressSanitizer: 32 byte(s) leaked in 3 allocation(s).
```

The two records point at two different lines: `table.c:6` for the array, `table.c:15` for the strings. **Only line 6 is your bug.** Free the array and the two strings become freeable again, because the pointers to them still exist inside it. That is the meaning of "indirect": it is a consequence, not a separate mistake, and you fix it by fixing the parent.

The practical rule for a container of pointers: free every element first, then free the container. In that order.

```c
for (i = 0; i < rows; i++) {
    free(table[i]);
}
free(table);
```

---

### UndefinedBehaviorSanitizer

UBSan reports things that are technically legal C but mean nothing: signed overflow, shifts that are too large, division by zero. It prints a `runtime error:` line **and keeps going**, so you get the whole picture in one run.

Signed integer overflow, the classic:

```c
#include <stdio.h>

static int next_id(void) {
    static int counter = 0;
    int huge = 2147483647;

    counter++;
    return huge + counter;
}

int main(void) {
    int a = next_id();
    int b = next_id();

    printf("%d %d\n", a, b);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer id.c -o id
./id
```

```text
id.c:8:17: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
-2147483648 -2147483647
```

Two things in two lines: the exact expression that overflowed, and the nonsense results that followed. On a machine with different types, or with a different compiler, the second line would print something else. That non-portability is the definition of undefined behaviour.

A shift too large:

```c
#include <stdio.h>

int main(void) {
    int value = 1024;
    int shift = 40;

    printf("%d\n", value << shift);
    return 0;
}
```

```text
shift.c:7:26: runtime error: shift exponent 40 is too large for 32-bit type 'int'
262144
```

You may add these flags:

| Flag | Meaning |
| --- | --- |
| `-fno-sanitize-recover=all` | stop on the first UBSan error instead of continuing |
| `-fsanitize=leak` | LeakSanitizer on its own (it is already part of `address` on Linux) |
| `-fsanitize=address,undefined,leak` | everything at once |

UBSan is cheap enough to leave on in normal development. ASan's 2x to 10x slowdown makes it a testing build rather than a production one, but for a course-sized program it hardly matters — turn it on for every exercise.

---

## `gcc -fanalyzer`: A Built-In Static Analyzer

`fanalyzer` runs a whole-program analysis at compile time and needs nothing else installed:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fanalyzer file.c -o file
```

On the same `strcpy` overflow from earlier:

```text
fan.c: In function ‘copy_name’:
fan.c:6:5: warning: stack-based buffer overflow [CWE-121] [-Wanalyzer-out-of-bounds]
    6 |     strcpy(dest, "Wesley Bertipaglia");
      |     ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  ‘main’: events 1-3
    │
    │   10 | int main(void) {
    │      |     ^~~~
    │   11 |     char name[8];
    │      |          ~~~~
    │      |             |
    │      |             (2) capacity: 8 bytes
    │   13 |     copy_name(name, sizeof(name));
    │      |     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    │      |     |
    │      |     (3) calling ‘copy_name’ from ‘main’
    │
    └──> ‘copy_name’: events 4-5
           │
           │    5 | static void copy_name(char *dest, size_t size) {
           │      |             ^~~~~~~~~
           │      |             (4) entry to ‘copy_name’
           │    6 |     strcpy(dest, "Wesley Bertipaglia");
           │      |     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
           │      |     (5) ⚠️  out-of-bounds write from byte 8 till byte 18 but ‘name’ ends at byte 8
```

The `(1)`, `(2)`, `(3)` numbering is the analyzer's path from `main` to the bad write: it entered `main`, found a capacity of 8 bytes for `name`, called `copy_name`, and then proved that the string literal is 19 bytes long and does not fit. It ends with an ASCII diagram of the two objects:

```text
  │‘W’│‘e’│‘s’│‘l’│‘e’│‘y’│                │‘a’ │‘g’ │‘l’ │‘i’ │‘a’ │NUL │
  ├───┴───┴───┴───┴───┴───┴────────────────┴────┴────┴────┴────┴────┴────┤
  │                  string literal (type: ‘char[19]’)                   │
  └──────────────────────────────────────────────────────────────────────┘
    │   │   │   │   │   │  │   │      │      │    │    │    │    │    │
    v   v   v   v   v   v  v   v      v      v    v    v    v    v    v
  ┌───┬──────────────────────┬───┐┌──────────────────────────────────────┐
  │[0]│         ...          │[7]││                                      │
  ├───┴──────────────────────┴───┤│          after valid range           │
  │   ‘name’ (type: ‘char[8]’)   ││                                      │
  └──────────────────────────────┘└──────────────────────────────────────┘
  ├──────────────┬───────────────┤├──────────────────┬───────────────────┤
                 │                                   │
        ╭────────┴────────╮              ╭───────────┴───────────╮
        │capacity: 8 bytes│              │⚠️  overflow of 11 bytes│
        ╰─────────────────╯              ╰───────────────────────╯
```

It also found `NULL` dereferences, double frees, and dead stores. It is a static analyzer, so it can only reason about paths it can follow — a bug that depends on a runtime value it cannot predict may not be found. But it is free, it needs no separate tool, and it catches the classic beginner bugs. Run it on everything:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fanalyzer -Werror file.c -o file
```

---

## Which Tool, When

| Symptom | Tool |
| --- | --- |
| wrong value printed, no crash | `gdb` first, then `-fanalyzer` |
| `Segmentation fault` | `gdb` for the stack trace |
| heap or stack overflow | ASan |
| use after `free`, double `free` | ASan (and `-Wall`, which often catches the simple ones) |
| memory grows and never comes back | valgrind `--leak-check=full`, or ASan's LeakSanitizer |
| negative or absurd numbers from arithmetic | UBSan |
| anything suspicious, and you want a second opinion | `-fanalyzer` |

If you can only pick one habit, make it this: **build with the sanitizers and run the program.** Nearly every bug in this module stops being a mystery and becomes a line number.

---

## The Debugging Workflow

This is the procedure. Follow it in order.

```text
1. REPRODUCE      Make it fail on demand, reliably.
                  A bug you cannot trigger is a bug you cannot verify.

2. MINIMIZE       Cut the program down until the bug still happens.
                  300 lines -> 15 lines means 15 candidates, not 300.

3. COMPILE HARD   gcc -std=c11 -Wall -Wextra -pedantic -g file.c -o file
                  Then add, in this order:
                    -O2                          (catches uninitialized reads)
                    -fanalyzer                   (proves some overflows)
                    -fsanitize=address,undefined (catches the rest at runtime)

4. HYPOTHESIZE    Say what you think is happening, in a sentence,
                  before touching anything. "values[i] reads past the end
                  because the loop uses <= instead of <."

5. VERIFY         gdb. break the suspect line, print every variable.
                  print the loop variable at every iteration and look for
                  the value it never takes. After a crash: bt, frame 1,
                  info locals.

6. FIX            Change the cause, not the symptom. Do not silence a
                  warning; do not add a sleep; do not add a bounds check
                  that hides a loop bug.

7. TEST           Re-run with the fix. Then run it again with the bad
                  input that caused it. Then add the case to a file you
                  run every time.
```

Two rules that matter more than the rest:

> **Change one thing at a time.** If you fix three things and the bug goes away, you learned nothing.

> **A bug you fixed and did not write a test for is a bug that will come back.** The test is the deliverable, not the fix.

A session that goes well looks like this:

```text
$ ./avg
average = 22.50            <- reproduce

$ gdb ./avg                <- verify
(gdb) break 8
(gdb) run
(gdb) print i
$6 = 1                     <- i never reaches 0; hypothesis confirmed

$ sed -i 's/i = 1;/i = 0;/' avg.c
$ gcc -std=c11 -Wall -Wextra -pedantic -g avg.c -o avg && ./avg
average = 25.00            <- fixed
```

Fifteen minutes, and the next one of the same shape takes thirty seconds.

---

## Checklist

```text
[ ] gcc -std=c11 -Wall -Wextra -pedantic -g, zero warnings
[ ] gcc -O2 once, looking for uninitialized warnings
[ ] gcc -fanalyzer once per program
[ ] gcc -fsanitize=address,undefined -fno-omit-frame-pointer, then RUN it
[ ] valgrind --leak-check=full --show-leak-kinds=all ./app before calling it done
[ ] After every crash: bt, then frame 1, then info locals
[ ] Did I write a test for the bug I just fixed?
```

---

#### [Previous](./02-part.md) | [Next](./exercises.md)