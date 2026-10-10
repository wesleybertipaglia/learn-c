# Array Patterns and Pitfalls

Every array program you will ever write is one of a handful of shapes: walk it, search it, count it, transform it. This part writes those shapes by hand, then lists the mistakes that produce almost every bug in array code.

---

## Finding the Maximum and the Minimum

The pattern: take the first element as the current best, then compare every other element against it.

```c
int find_max(const int values[], int n) {
    int best = values[0];
    for (int i = 1; i < n; i++) {
        if (values[i] > best) {
            best = values[i];
        }
    }
    return best;
}
```

Two details that matter:

- **Start the loop at 1, not 0.** Starting at 0 would compare `values[0]` with itself on the first pass, which is harmless but pointless. In other algorithms the equivalent mistake is fatal.
- **Seed with `values[0]`, not `0`.** If every value in the array is negative, seeding with `0` returns `0`, which is not in the array at all. Seeding with the first element makes the function correct for any input.

The minimum is the same code with `<` and `min`:

```c
int find_min(const int values[], int n) {
    int best = values[0];
    for (int i = 1; i < n; i++) {
        if (values[i] < best) {
            best = values[i];
        }
    }
    return best;
}
```

## Linear Search

Walk the array from the beginning, return as soon as you find the target, otherwise report failure with `-1`:

```c
int linear_search(const int values[], int n, int target) {
    for (int i = 0; i < n; i++) {
        if (values[i] == target) {
            return i;
        }
    }
    return -1;
}
```

Returning the **index** rather than a boolean is more useful, because the index tells you *where* you found it. If you need a boolean, test the result: `found = linear_search(...) >= 0`.

The loop can also stop early, which is the whole point of a linear search:

```c
if (values[i] == target) {
    return i;          /* half the array is never touched */
}
```

Return `-1` and not `0`, because index 0 is a perfectly valid answer and a caller doing `if (index)` would be wrong.

## Counting Occurrences

Same walk, but no early exit — you need to see everything:

```c
int count_occurrences(const int values[], int n, int target) {
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (values[i] == target) {
            count++;
        }
    }
    return count;
}
```

## Reversing In Place

Two cursors, one at each end, walking towards the middle, swapping as they go:

```c
void reverse_in_place(int values[], int n) {
    for (int i = 0; i < n / 2; i++) {
        int tmp = values[i];
        values[i] = values[n - 1 - i];
        values[n - 1 - i] = tmp;
    }
}
```

The temporary variable is not optional: `a = b; b = a;` overwrites the original `a` before it is copied.

`n / 2` handles odd lengths correctly — for `n = 5` it loops with `i = 0, 1`, swapping `(0,4)` and `(1,3)`, and the middle element is left alone. Using `n - 1 - i` instead of `n + 1 - i` matters for `i = 0`: the second form would produce `n`, which is out of bounds.

All of these in one program:

```c
#include <stdio.h>

#define SIZE 8

static int find_max(const int values[], int n) {
    int best = values[0];
    for (int i = 1; i < n; i++) {
        if (values[i] > best) {
            best = values[i];
        }
    }
    return best;
}

static int find_min(const int values[], int n) {
    int best = values[0];
    for (int i = 1; i < n; i++) {
        if (values[i] < best) {
            best = values[i];
        }
    }
    return best;
}

static int linear_search(const int values[], int n, int target) {
    for (int i = 0; i < n; i++) {
        if (values[i] == target) {
            return i;
        }
    }
    return -1;
}

static int count_occurrences(const int values[], int n, int target) {
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (values[i] == target) {
            count++;
        }
    }
    return count;
}

static void reverse_in_place(int values[], int n) {
    for (int i = 0; i < n / 2; i++) {
        int tmp = values[i];
        values[i] = values[n - 1 - i];
        values[n - 1 - i] = tmp;
    }
}

static void print(const char *label, const int values[], int n) {
    printf("%-12s", label);
    for (int i = 0; i < n; i++) {
        printf("%4d", values[i]);
    }
    printf("\n");
}

int main(void) {
    int values[SIZE] = {42, 17, 93, 8, 93, 0, 55, 42};

    print("original:", values, SIZE);
    printf("\n");
    printf("max                     = %d\n", find_max(values, SIZE));
    printf("min                     = %d\n", find_min(values, SIZE));
    printf("index of 93             = %d\n", linear_search(values, SIZE, 93));
    printf("index of 100            = %d  (-1 means not found)\n",
           linear_search(values, SIZE, 100));
    printf("how many 42             = %d\n", count_occurrences(values, SIZE, 42));
    printf("how many 93             = %d\n", count_occurrences(values, SIZE, 93));

    reverse_in_place(values, SIZE);
    print("\nreversed:", values, SIZE);

    return 0;
}
```

```text
original:     42  17  93   8  93   0  55  42

max                     = 93
min                     = 0
index of 93             = 2
index of 100            = -1  (-1 means not found)
how many 42             = 2
how many 93             = 2

reversed:    42  55   0  93   8  93  17  42
```

---

## Removing an Element

A fixed array cannot shrink. Removal therefore means: **shift the tail left by one and reduce the count.** The array still has its original capacity; the logical length just got smaller.

### Order-Preserving Removal

```c
int remove_at(int arr[], int n, int index) {
    if (index < 0 || index >= n) {
        return n;                       /* invalid index: nothing removed */
    }
    for (int i = index; i < n - 1; i++) {
        arr[i] = arr[i + 1];
    }
    return n - 1;
}
```

The loop reads from `index + 1` and writes to `index`, so it must run **left to right**. Running it backwards overwrites the element you are about to read.

### The O(1) Alternative

If the order does not matter, move the **last** element into the hole:

```c
int remove_at_fast(int arr[], int n, int index) {
    if (index < 0 || index >= n) {
        return n;
    }
    arr[index] = arr[n - 1];
    return n - 1;
}
```

One assignment instead of `n - index` of them. Order is destroyed, which is fine for a set of scores you are about to re-sort anyway.

Both, side by side:

```c
#include <stdio.h>

#define CAPACITY 10

static void print(const char *label, const int values[], int n) {
    printf("%-14s", label);
    printf("[ ");
    for (int i = 0; i < n; i++) {
        printf("%d", values[i]);
        if (i < n - 1) {
            printf(", ");
        }
    }
    printf(" ]  (n = %d)\n", n);
}

/* remove keeping order: shift left, then the count shrinks by one */
static int remove_at(int arr[], int n, int index) {
    if (index < 0 || index >= n) {
        return n;                       /* invalid index: nothing removed */
    }
    for (int i = index; i < n - 1; i++) {
        arr[i] = arr[i + 1];
    }
    return n - 1;
}

/* remove ignoring order: move the last element into the hole */
static int remove_at_fast(int arr[], int n, int index) {
    if (index < 0 || index >= n) {
        return n;
    }
    arr[index] = arr[n - 1];
    return n - 1;
}

int main(void) {
    int values[CAPACITY] = {5, 9, 13, 21, 34};
    int n = 5;

    print("start:", values, n);

    int removed = remove_at(values, n, 1);
    printf("removed 9\n");
    print("after:", values, removed);

    removed = remove_at(values, removed, 0);
    printf("removed 5\n");
    print("after:", values, removed);

    removed = remove_at(values, removed, 99);
    printf("tried to remove index 99\n");
    print("after:", values, removed);

    printf("\nsame removals with remove_at_fast (order not preserved):\n");

    int other[CAPACITY] = {5, 9, 13, 21, 34};
    int m = 5;
    print("start:", other, m);
    m = remove_at_fast(other, m, 1);
    printf("removed 9\n");
    print("after:", other, m);
    m = remove_at_fast(other, m, 0);
    printf("removed 5\n");
    print("after:", other, m);

    return 0;
}
```

```text
start:        [ 5, 9, 13, 21, 34 ]  (n = 5)
removed 9
after:        [ 5, 13, 21, 34 ]  (n = 4)
removed 5
after:        [ 13, 21, 34 ]  (n = 3)
tried to remove index 99
after:        [ 13, 21, 34 ]  (n = 3)

same removals with remove_at_fast (order not preserved):
start:        [ 5, 9, 13, 21, 34 ]  (n = 5)
removed 9
after:        [ 5, 34, 13, 21 ]  (n = 4)
removed 5
after:        [ 21, 34, 13 ]  (n = 3)
```

Notice that `remove_at` returns the **new length**, and nothing else tells the caller what happened. The old `n` is not modified, because C passes arguments by value. Two rules follow:

- The caller must store the return value: `n = remove_at(arr, n, 2);`
- Never assume the array itself got shorter. `sizeof(arr)` is unchanged; only `n` is smaller.

---

## Common Mistakes

### Off-by-One at the Ends

| Mistake | Consequence |
| --- | --- |
| `for (i = 0; i <= n; i++)` | one iteration too many: reads `arr[n]` |
| `for (i = 1; i < n; i++)` | never touches `arr[0]` |
| `for (i = n; i > 0; i--)` | reads `arr[n]` on the first pass |
| `index >= n` vs `index > n` in a bounds check | rejects a valid index, or accepts one too far |

The last one bites constantly. Valid indices are `0` through `n - 1`, so the correct rejection test is:

```c
if (index < 0 || index >= n) {   /* correct */
```

### Getting the Shift Direction Backwards

This is the single most common array bug. Removing an element means moving each later element **down** into the gap:

```c
for (int i = index; i < n - 1; i++) {
    arr[i] = arr[i + 1];       /* read to the right, write to the left */
}
```

The mirror image — reading left and writing right — destroys the array:

```c
int arr[6] = {10, 20, 30, 40, 50, 0};
int n = 5;

for (int i = 0; i < n; i++) {
    arr[i] = arr[i + 1];   /* WRONG */
}
```

```text
before: 10 20 30 40 50
after shifting the WRONG way: 20 30 40 50 0

after shifting the RIGHT way: 20 30 40 50
```

The wrong version overwrites `arr[0]` with `arr[1]` before `arr[1]` is read, so every original value is lost and a zero arrives at the end. The correct version shifts values *down* so that no value is overwritten before it is used.

The same error appears when inserting:

```c
for (int i = n; i > index; i--) {     /* insertion shifts UP, so it runs backwards */
    arr[i] = arr[i - 1];
}
arr[index] = new_value;
n++;
```

Note that insertion shifts right and therefore iterates right to left, and it requires one free slot — `n` must be less than the capacity.

### Forgetting to Update the Count

The most common logic bug in array code. After a removal, `n` is stale:

```c
int n = 5;
remove_at(values, n, 2);   /* the array changed, but n is still 5 */

/* now every loop over n reads one element too many */
```

Always capture the return value:

```c
n = remove_at(values, n, 2);
```

The same applies when a function "removes" something and forgets to decrement its own counter, or when a loop fills an array but the count variable is never incremented.

### The `arr[n++]` Idiom

Handy, compact, and the source of some very quiet bugs. It uses two facts:

- `n++` is **postfix**: it yields the *old* value of `n`, then increments it.
- Assignment happens after the right-hand side is evaluated.

So `stack[n++] = 10;` means "store 10 at `stack[n]`, then increase `n`". It is the standard way to build a stack or a list by hand:

```c
int stack[CAPACITY];
int n = 0;

stack[n++] = 10;
stack[n++] = 20;
stack[n++] = 30;
stack[n++] = 40;
stack[n++] = 50;

printf("%-10s", "stack:");
for (int i = 0; i < n; i++) {
    printf("%4d", stack[i]);
}
printf("   (n = %d)\n", n);
```

```text
stack:      10  20  30  40  50   (n = 5)
```

Each statement writes at the current `n` and then bumps it, so after five statements `n == 5` and the array is full. One more:

```text
now one too many: stack[5] with n == 5
the next store writes one element past the end of the array.
```

That is a heap or stack overflow, and it is entirely the programmer's job to prevent it — C will not check:

```c
if (n < CAPACITY) {
    stack[n++] = value;
} else {
    /* full: grow it, or refuse */
}
```

Where the idiom bites:

```c
arr[n] = arr[n++];      /* undefined behavior: modifies n and uses it without a sequence point */
arr[i++] = arr[i];      /* same problem */
```

If you need to use the old value and the new value of `n` in the same statement, split it into two statements.

### Other Ones Worth Knowing

| Mistake | What to do instead |
| --- | --- |
| Using the array length after the array decayed | keep `n` in a variable, pass it explicitly |
| `int *p = &arr;` | use `arr` or `&arr[0]`; `&arr` is `int (*)[5]` |
| Comparing pointers from different arrays | meaningless, even if it compiles |
| Modifying the loop variable inside the body | gives the compiler a reason to be suspicious |
| Passing `arr` to a function expecting `int *` and then using `sizeof` in the callee | pass the length |
| Assuming `arr[n-1]` is the largest value | nothing in memory records a maximum |
| `char buf[10] = "0123456789";` | exactly 10 chars plus the terminator does not fit: overflow |

---

## When an Array Is the Wrong Tool

An array is the right choice when:

- The size is known when you write the code.
- The data is a fixed shape: coordinates, a matrix, a lookup table, a board.
- You want the fastest possible access, and you are certain about the bounds.

It is the wrong choice when:

- **The size is only known at runtime.** Reading a file, parsing user input, a chat server with a variable number of clients — you cannot write `int data[n];` because `n` is not a constant.
- **The size changes often.** Inserting an element into the middle of an array means shifting everything after it, `O(n)` each time.
- **You do not want the whole thing in memory.** An array of a billion ints needs four gigabytes up front; a linked list stores only what exists.
- **You want to map it to a dictionary or a queue.** Those structures have rules about ordering and lookup that an array does not.

### C Has No Built-In Dynamic Array

There is no `std::vector`, no `ArrayList`, no `append()` on a built-in type. This is not an oversight you can work around with a library call: the language simply does not ship one.

What C ships instead is raw memory management:

| Tool | What it gives you | Cost |
| --- | --- | --- |
| `malloc(n * size)` | a block of `n` elements, size decided at runtime | you must check for failure |
| `realloc(p, new_size)` | resize, possibly copying | invalidates the old pointer |
| `free(p)` | give the memory back | freeing twice is undefined; leaking is a leak |
| `memcpy` / `memmove` | bulk copy | you compute the byte count yourself |

That is the whole toolkit. Everything dynamic in C — dynamic arrays, strings, hash tables, lists — is built from `malloc`, `realloc` and `free`.

**In the Dynamic Memory chapter (04 of this module) you will write a dynamic array from scratch**: a struct holding a pointer, a length and a capacity, with `push`, `pop`, `insert_at`, `remove_at` and `free_all`, growing by doubling when it runs out of room. Everything in this part reappears there — `n = remove_at(values, n, 2)` becomes a method that can actually grow the buffer instead of overflowing it.

For now, the constraint is honest and worth restating: **the size of an array is part of its type and must be a compile-time constant.** If your problem needs a size that is not, you need dynamic memory, and that is the next chapter after strings and structs.

---

## Summary

- Seed max and min with `values[0]` and start the loop at index 1.
- Search early-exits; count visits everything; both return indices or counts, not booleans.
- Reversing uses two cursors, `n / 2` iterations, and a temporary variable.
- Removing means shifting left and returning the new length. Order-preserving is `O(n)`; order-destroying is `O(1)`.
- Shift left when removing, shift right when inserting, and get the iteration direction right.
- Always update the count when the length changes.
- `arr[n++]` stores at `n` then increments; it is compact and easy to overflow.
- An array with a runtime size requires `malloc`, which is where the next chapters go.

---

#### [Previous](./02-part.md) | [Next](./exercises.md)