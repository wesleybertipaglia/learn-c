# Arrays

A single variable holds one value. Real problems need many values of the same type: ten exam scores, fifty sensor readings, a row of pixels. C's answer to that is the **array** — a block of memory holding a fixed number of elements of one type, side by side, with no gaps and no metadata.

An array is the first real data structure in this course, and also the first place where C stops protecting you. Once you index past the end, the program does not complain. It does something arbitrary.

---

## Declaring an Array

```c
int numbers[5];
```

Read it as: *an array named `numbers`, of `int`, holding 5 elements.* Three facts follow directly:

- The size is part of the type. `int[5]` is a different type from `int[10]`, and they are not interchangeable.
- The size **must be a compile-time constant**. `int numbers[5];` is fine; `int numbers[n];` is not, unless `n` is a constant or you opt into VLAs (below).
- All five elements exist immediately, whether you write to them or not. The memory is allocated when the declaration is reached and released at the end of the block.

### The Size Must Be Known at Compile Time

```c
#define SIZE 5

int a[SIZE];          /* fine: SIZE is a constant expression */
enum { MAX = 10 };
int b[MAX];           /* fine: enum constants are constant expressions */

int n = 5;
int c[n];             /* ERROR: not a constant expression */
int d[n + 1];         /* ERROR: same reason */
```

The reason is that C lays arrays out in the generated code: there is no room in the program text to say "wait and see how big `n` turns out to be". For a *local* array the size has to be known when the function is compiled. When the size is only known at runtime you need dynamic memory, which is the subject of chapter 4 of this module.

### Aside: Variable Length Arrays

C99 added a feature called a **VLA** (variable length array): a local array whose size is a runtime value.

```c
int n = 5;
int vla[n];          /* legal in C99 and later */
```

Three warnings before you use one:

1. `-Wvla` is not in `-Wall`, but VLAs are an **optional** feature in C11. A compiler may not support them, and then your program does not build.
2. VLAs are not allowed at file scope, and cannot be `static`.
3. The size is not stored with the array. You cannot ask a VLA how big it is; you have to keep the length in a separate variable yourself, exactly like a pointer. If you lose that variable, the length is gone.

For a beginner: use a fixed-size array, or dynamic memory. A VLA buys you very little and costs you portability.

---

## Initialization

An array declared with no initializer contains **indeterminate values**. The memory is real, it belongs to your program, but nobody ever wrote anything into it, so you are reading whatever the machine left there.

Here is a program that shows it. It first fills the stack with recognizable values using a recursive function, then reads an array that was never written:

```c
#include <stdio.h>
#include <stdint.h>

static int sink;

static void dirty(int depth) {
    volatile int junk[64];
    uintptr_t here = (uintptr_t)&junk[0];

    for (int i = 0; i < 64; i++) {
        junk[i] = 0x5a5a0000 + (int)((here >> 4) & 0xfffu) + depth * 64 + i;
    }

    sink += junk[63];
    if (depth > 0) {
        dirty(depth - 1);
    }
}

static void show(const int *values, int n) {
    for (int i = 0; i < n; i++) {
        printf("  numbers[%d] = %d (0x%08x)\n", i, values[i], values[i]);
    }
}

static void read_back(void) {
    int numbers[5];                 /* no initializer: contents are undefined */
    const int *volatile p = numbers;

    show(p, 5);
}

int main(void) {
    dirty(3);

    printf("an array declared with no initializer, after the stack was used:\n");
    read_back();

    printf("\nrun it again: the values change, because they are whatever\n");
    printf("happened to be in that memory.\n");

    return 0;
}
```

```text
an array declared with no initializer, after the stack was used:
  numbers[0] = 0 (0x00000000)
  numbers[1] = 0 (0x00000000)
  numbers[2] = 1199234760 (0x477adec8)
  numbers[3] = 32765 (0x00007ffd)
  numbers[4] = 1 (0x00000001)

run it again: the values change, because they are whatever
happened to be in that memory.
```

A second run of the same binary:

```text
an array declared with no initializer, after the stack was used:
  numbers[0] = 0 (0x00000000)
  numbers[1] = 0 (0x00000000)
  numbers[2] = 1070794472 (0x3fd306e8)
  numbers[3] = 32764 (0x00007ffc)
  numbers[4] = 1 (0x00000001)
```

Same program, same machine, different numbers. That is what "indeterminate" means. Worse, `numbers[2]` in the first run is a garbage value, and comparing or branching on it is **undefined behavior** in the strict reading of the standard: the compiler may assume an uninitialized read never happens and optimize your `if` away entirely.

Note also the practical detail: on a fresh process the stack may be zeroed by the operating system, so a quick test can show a tidy row of zeros and convince you the feature is harmless. It is not. Add a function call before the declaration and the zeros disappear, as above.

### The Four Initialization Forms

| Form | What happens |
| --- | --- |
| `int a[5];` | indeterminate values — **never do this** |
| `int a[5] = {0};` | every element set to 0 |
| `int a[5] = {10, 20, 30};` | the first three set, the rest set to 0 |
| `int a[] = {1, 2, 3, 4};` | size inferred from the initializer: 4 |

The rule to remember: **any element you do not mention in the initializer list becomes 0.** That is why `{10, 20, 30}` on a 5-element array is safe, and why `char buf[256] = {0}` is the standard way to get a zeroed buffer.

A complete example:

```c
#include <stdio.h>

int main(void) {
    int numbers[5] = {10, 20, 30};
    int zeroes[5] = {0};
    int inferred[] = {1, 2, 3, 4, 5, 6};
    char zeros[4] = {0};

    printf("int numbers[5] = {10, 20, 30}\n");
    for (int i = 0; i < 5; i++) {
        printf("  numbers[%d] = %d\n", i, numbers[i]);
    }

    printf("\nint zeroes[5] = {0}\n");
    for (int i = 0; i < 5; i++) {
        printf("  zeroes[%d] = %d\n", i, zeroes[i]);
    }

    printf("\nint inferred[] = {1, 2, 3, 4, 5, 6}\n");
    printf("  sizeof(inferred) = %zu bytes -> %zu elements\n",
           sizeof(inferred), sizeof(inferred) / sizeof(inferred[0]));

    printf("\nchar zeros[4] = {0} prints as four zero bytes:\n");
    printf("  bytes:");
    for (int i = 0; i < 4; i++) {
        printf(" %d", (int)(unsigned char)zeros[i]);
    }
    printf("\n");

    printf("\nsizes:\n");
    printf("  sizeof(numbers)     = %zu bytes\n", sizeof(numbers));
    printf("  sizeof(numbers[0])  = %zu bytes\n", sizeof(numbers[0]));
    printf("  sizeof(numbers) / sizeof(numbers[0]) = %zu elements\n",
           sizeof(numbers) / sizeof(numbers[0]));

    printf("\naddress of each element:\n");
    for (int i = 0; i < 5; i++) {
        printf("  &numbers[%d] = %p   offset = %2ld bytes   value = %d\n",
               i, (void *)&numbers[i],
               (long)((char *)&numbers[i] - (char *)&numbers),
               numbers[i]);
    }

    return 0;
}
```

```text
int numbers[5] = {10, 20, 30}
  numbers[0] = 10
  numbers[1] = 20
  numbers[2] = 30
  numbers[3] = 0
  numbers[4] = 0

int zeroes[5] = {0}
  zeroes[0] = 0
  zeroes[1] = 0
  zeroes[2] = 0
  zeroes[3] = 0
  zeroes[4] = 0

int inferred[] = {1, 2, 3, 4, 5, 6}
  sizeof(inferred) = 24 bytes -> 6 elements

char zeros[4] = {0} prints as four zero bytes:
  bytes: 0 0 0 0

sizes:
  sizeof(numbers)     = 20 bytes
  sizeof(numbers[0])  = 4 bytes
  sizeof(numbers) / sizeof(numbers[0]) = 5 elements

address of each element:
  &numbers[0] = 0x7ffd15fc1a30   offset =  0 bytes   value = 10
  &numbers[1] = 0x7ffd15fc1a34   offset =  4 bytes   value = 20
  &numbers[2] = 0x7ffd15fc1a38   offset =  8 bytes   value = 30
  &numbers[3] = 0x7ffd15fc1a3c   offset = 12 bytes   value = 0
  &numbers[4] = 0x7ffd15fc1a40   offset = 16 bytes   value = 0
```

---

## Memory Layout

Here is `int numbers[5] = {10, 20, 30};` as it actually sits in memory. Five `int`s, four bytes each, no gaps, lowest address first:

```text
  int numbers[5] = {10, 20, 30};
  20 bytes total

  high address
  +---------+   &numbers[4] = 0x...810   offset 16   value 0
  |    0    |
  +---------+   &numbers[3] = 0x...80c   offset 12   value 0
  |    0    |
  +---------+   &numbers[2] = 0x...808   offset  8   value 30
  |   30    |
  +---------+   &numbers[1] = 0x...804   offset  4   value 20
  |   20    |
  +---------+   &numbers[0] = 0x...800   offset  0   value 10   <-- numbers
  |   10    |
  +---------+
  low address
```

Three things to take from the picture:

1. **Element `i` lives at `base + i * 4`.** That is exactly why `&numbers[i] - &numbers[0] == i`.
2. **The whole array is one object.** There is no header, no length field, no null terminator (that is a string, part 1 of the next chapter). Nothing in memory records the length — only your source code knows it.
3. **The elements are in order, and there is no gap between them.** `char` arrays have exactly this layout too, one byte per element, which is what makes them useful for text.

Compare with the real addresses printed above: `0x7ffd15fc1a30`, `0x7ffd15fc1a34`, `0x7ffd15fc1a38` — exactly 4 bytes apart. (The absolute values change every run; the spacing never does.)

---

## `sizeof` on an Array Versus on an Element

`sizeof` is an operator evaluated by the compiler, not a function. Its result depends on its argument's **type**.

```text
  sizeof(numbers)      = 20 bytes   -> the whole array
  sizeof(numbers[0])   =  4 bytes   -> one int
  sizeof numbers        = 20 bytes   -> parentheses optional, same thing
```

Which means the number of elements is always computable, for a real array:

```c
size_t count = sizeof(numbers) / sizeof(numbers[0]);
```

This is worth committing to memory, with one giant caveat that is the subject of the next part: it works **only where the array is still an array**. As soon as it decays to a pointer, `sizeof` gives you 8.

### Useful Array Facts

```c
int numbers[5] = {10, 20, 30, 40, 50};

int  first   = numbers[0];          /* the first element      */
int  *ptr    = numbers;              /* a pointer to the first */
int  *end    = numbers + 5;          /* one past the last: legal to create */
int   last   = *(end - 1);           /* 50                     */
int   middle = numbers[2];           /* 30                     */
```

---

## Indexing

```c
int value = numbers[2];
```

is exactly the pointer expression `*(numbers + 2)` — you saw this in the Pointers chapter, and the two forms are interchangeable:

```c
numbers[2]  ==  *(numbers + 2)  ==  *(2 + numbers)  ==  2[numbers]
```

That last one is real C and a good party trick, because addition is commutative and subscripting is defined in terms of addition. Never write it.

Two details beginners trip over:

- Indices are **zero-based**: the first element is `numbers[0]`, the fifth is `numbers[4]`.
- The bracket takes an `int` expression, evaluated at runtime. `numbers[i]` works for any `i` you compute — including an `i` that is out of range. Nothing checks it.

---

## Out-of-Bounds Access Is Undefined Behavior

An array of 5 elements has valid indices `0` to `4`. Everything else is undefined behavior: reading, writing, or even *forming* an address more than one past the end.

```c
#include <stdio.h>

#define SIZE 5

int main(void) {
    int numbers[SIZE] = {10, 20, 30, 40, 50};

    printf("valid indices are 0 to %d\n", SIZE - 1);
    for (int i = 0; i < SIZE; i++) {
        printf("  numbers[%d] = %d\n", i, numbers[i]);
    }

    printf("\nnow one past the end:\n");
    printf("  numbers[%d] = %d\n", SIZE, numbers[SIZE]);

    return 0;
}
```

Compiled with `gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address`, this aborts with a real report:

```text
=================================================================
==370617==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7212c39f0034 at pc 0x583a7828c522 bp 0x7ffffddb1cd0 sp 0x7ffffddb1cc0
READ of size 4 at 0x7212c39f0034 thread T0
    #0 0x583a7828c521 in main a1_oob.c:14
    #1 0x7612c5a2a600 in __libc_start_call_main ../sysdeps/nptl/libc-start.c:59

Address 0x7212c39f0034 is located in stack of thread T0 at offset 52 in frame
    #0 0x583a7828c278 in main a1_oob.c:5

  This frame has 1 object(s):
    [32, 52) 'numbers' (line 6) <== Memory access at offset 52 overflows this variable
SUMMARY: AddressSanitizer: stack-buffer-overflow a1_oob.c:14 in main
```

Read that report carefully, because it is a picture of the bug: the array occupies bytes `[32, 52)` — twenty bytes, five ints — and the access was at offset 52, which is the first byte *after* the object.

Without AddressSanitizer the same program usually just prints some number and carries on. Sometimes it segfaults. Sometimes the compiler deletes your `if` because, per the standard, an out-of-bounds access cannot happen, so the check is dead code. All three outcomes are legal.

### Writing Out of Bounds Is Worse

Reading garbage is bad. Writing out of bounds **corrupts something else** — another local variable, a saved pointer, a return address. That is the mechanism behind most security vulnerabilities in C, and it is why the course spends a chapter on the Dynamic Memory toolchain.

```c
int a[3] = {1, 2, 3};
int b[3] = {4, 5, 6};

a[5] = 99;   /* UNDEFINED: clobbers memory past a[] */

/* if the two arrays are adjacent, b now holds your 99 */
```

---

## Looping Over an Array

The standard pattern is a counter from 0 up to the length, exclusive:

```c
for (int i = 0; i < n; i++) {
    printf("%d ", values[i]);
}
```

```text
  i < n   (correct, 5 values)
  scores[0] = 72
  scores[1] = 91
  scores[2] = 65
  scores[3] = 88
  scores[4] = 50
```

### Why `i < n` and Not `i <= n`

`<=` reads naturally as "go through all of them", and that is exactly why it is the wrong habit. For an array of `n` elements the last valid index is `n - 1`, so the loop must stop when `i` reaches `n`:

```c
for (int i = 0; i < n; i++) { }   /* i takes 0..n-1   : correct, n iterations */
for (int i = 0; i <= n; i++) { }  /* i takes 0..n     : WRONG, n+1 iterations */
```

The second version performs one extra iteration that reads `values[n]` — one element past the end. Here is the observable result:

```text
i <= n  (one step too far, 6 reads)
  scores[0] = 72
  scores[1] = 91
  scores[2] = 65
  scores[3] = 88
  scores[4] = 50
  scores[5] = 0
```

It "works" and prints a plausible-looking extra value, which is what makes it dangerous. On a different build it would print garbage, or crash, or vanish under optimization. The compiler does not warn about it either, because `i <= n` is perfectly valid logic — it just isn't the logic you meant.

Memorize the shape: **`i < n`**. When the count is exclusive, the `<` is not an off-by-one bug. Writing `<=` when the last valid index is `n - 1` is the bug.

### The Pointer Form

Since an array name decays to a pointer, this is the same loop:

```c
for (int *p = values; p < values + n; p++) {
    printf("%d ", *p);
}
```

It has one advantage worth knowing: `p < values + n` compares against a pointer, so the compiler does not need to recompute the multiplication in `values[i]`. Prefer the subscript form for clarity until you are writing code where that matters.

### Common Loop Bugs

| Bug | What happens |
| --- | --- |
| `for (i = 0; i <= n; i++)` | one read past the end |
| `for (i = 1; i < n; i++)` | skips the first element |
| `for (i = 0; i < n; i--)` | starts at 0 and counts down: reads `values[0]` then `values[-1]` |
| loop bound is a `<=` but the array has size `n + 1` | fine, but confusing enough to hide a bug |
| reusing `i` for two nested loops | the inner loop destroys the outer counter |

---

## A Complete Example

```c
#include <stdio.h>

#define SCORE_COUNT 5

int main(void) {
    int scores[SCORE_COUNT] = {72, 91, 65, 88, 50};
    int total = 0;

    for (int i = 0; i < SCORE_COUNT; i++) {
        total += scores[i];
    }

    printf("scores: ");
    for (int i = 0; i < SCORE_COUNT; i++) {
        printf("%d%s", scores[i], (i == SCORE_COUNT - 1) ? "\n" : " ");
    }

    printf("total: %d\n", total);
    printf("mean:  %.2f\n", (double)total / SCORE_COUNT);

    int highest = scores[0];
    for (int i = 1; i < SCORE_COUNT; i++) {
        if (scores[i] > highest) {
            highest = scores[i];
        }
    }
    printf("highest: %d\n", highest);

    printf("element count from sizeof: %zu\n",
           sizeof(scores) / sizeof(scores[0]));

    return 0;
}
```

```text
scores: 72 91 65 88 50
total: 366
mean:  73.20
highest: 91
element count from sizeof: 5
```

---

## Summary

- `int numbers[5];` allocates five ints, contiguous, size known at compile time.
- The size must be a constant expression. VLAs allow runtime sizes but are optional in C11 and fragile.
- No initializer means **indeterminate** values. Always use `{0}`, `{10, 20, 30}`, or a full list.
- Unmentioned elements in an initializer are set to zero.
- `sizeof(arr)` is the whole array; `sizeof(arr[0])` is one element; the quotient is the count.
- Elements are at `base + i * sizeof(T)`. `arr[i]` is `*(arr + i)`.
- Valid indices are `0` to `n - 1`. `arr[n]` is undefined behavior — the compiler will not warn, ASan will catch it.
- Loop with `i < n`, not `i <= n`.
- Nothing in the memory stores the length. Keep it in a variable.

Next part: passing arrays to functions, which is where the array stops being an array.

---

#### [Previous](../1-fundamentals/challenge.md) | [Next](./02-part.md)