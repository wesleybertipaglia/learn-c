# Searching and Sorting by Hand

Searching asks "where is it". Sorting asks "put everything in order". Both are done here **by hand** — no `bsearch`, no `qsort` — because the point is the loop invariant: the promise that stays true on every iteration and makes the code correct.

The compile command for this part:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer search.c -o search
```

---

## The Swap Helper

Every sort below exchanges two elements. One helper, used everywhere:

```c
static void swap(int *a, int *b) {
    int tmp = *a;
    *a = *b;
    *b = tmp;
}
```

No return value, no allocation, no failure mode. Callers pass addresses of array slots: `swap(&a[i], &a[j])`.

---

## Linear Search: Read Every Slot

Walk the array front to back. Return the first index holding `target`, or `-1` when it is absent.

```c
#include <stdio.h>
#include <stddef.h>

/* OWNERSHIP: borrows `a`; returns index of first `target`, or -1. */
static int linear_search(const int *a, size_t n, int target) {
    for (size_t i = 0; i < n; i++) {
        if (a[i] == target) {
            return (int)i;
        }
    }
    return -1;
}

int main(void) {
    const int a[] = { 4, 2, 7, 1, 9 };
    const size_t n = sizeof(a) / sizeof(a[0]);

    printf("find 7 -> %d\n", linear_search(a, n, 7));
    printf("find 5 -> %d\n", linear_search(a, n, 5));
    printf("find in empty -> %d\n", linear_search(a, 0, 7));
    return 0;
}
```

```text
find 7 -> 2
find 5 -> -1
find in empty -> -1
```

The array parameter is `const int *a`: search only reads. Indices and counts are `size_t`; the return is `int` because `-1` must be representable — that mismatch is deliberate, and callers must check for `-1` before indexing. Empty input (`n == 0`) returns `-1` without touching `a[0]`.

```text
  a = [4, 2, 7, 1, 9], target = 7: i=0 miss, i=1 miss, i=2 MATCH -> return 2
```

Cost: up to `n` comparisons. No precondition on order — it works on any array. That generality is also its ceiling: without order, no algorithm can do better than looking everywhere in the worst case.

| Case | Comparisons |
| --- | --- |
| Found at front | 1 |
| Found at end | n |
| Absent | n |
| Empty array | 0 |

---

## Binary Search: Halve a Sorted Interval

If the array is sorted ascending, each comparison discards half the candidates. Keep a live half-open interval `[lo, hi)` of indices that may still hold the target.

**Invariant:** if `target` is anywhere in the array, it is inside `a[lo..hi)`. The loop shrinks the interval while preserving the invariant, and stops when the interval is empty (`lo == hi`) or the target is found.

### Iterative version (prefer this one)

```c
#include <stdio.h>
#include <stddef.h>

/* OWNERSHIP: borrows `a`, which MUST be sorted ascending.
   Returns the index of `target`, or -1 when absent. */
static int binary_search(const int *a, size_t n, int target) {
    size_t lo = 0;
    size_t hi = n;             /* EXCLUSIVE upper bound: live range is [lo, hi) */

    while (lo < hi) {
        /* Overflow-safe midpoint: never computes (lo + hi). */
        size_t mid = lo + (hi - lo) / 2;

        if (a[mid] == target) {
            return (int)mid;
        }
        if (a[mid] < target) {
            lo = mid + 1;      /* target must be right of mid */
        } else {
            hi = mid;          /* target must be at or left of mid */
        }
    }
    return -1;
}

int main(void) {
    const int a[] = { 1, 2, 4, 7, 9, 12 };
    const size_t n = sizeof(a) / sizeof(a[0]);

    printf("find 7 -> %d\n", binary_search(a, n, 7));
    printf("find 1 -> %d\n", binary_search(a, n, 1));
    printf("find 12 -> %d\n", binary_search(a, n, 12));
    printf("find 5 -> %d\n", binary_search(a, n, 5));
    printf("find in empty -> %d\n", binary_search(a, 0, 7));
    return 0;
}
```

```text
find 7 -> 3
find 1 -> 0
find 12 -> 5
find 5 -> -1
find in empty -> -1
```

Traces in `[1, 2, 4, 7, 9, 12]`:

```text
  find 7: [0, 6): mid = 3, a[3] = 7 == target -> return 3
  find 5 (absent):
    [0, 6): mid = 3, a[3] = 7 > 5  -> live = [0, 3)
    [0, 3): mid = 1, a[1] = 2 < 5  -> live = [2, 3)
    [2, 3): mid = 2, a[2] = 4 < 5  -> live = [3, 3) = empty -> return -1
```

At each step the interval strictly shrinks (`lo` moves up or `hi` moves down), so the loop must terminate. The `[lo, hi)` half-open form is chosen deliberately: `hi = n` is legal even for an empty array (where `lo == hi == 0` and the loop never runs), and no `hi = n - 1` underflows when `n == 0`.

### Recursive version (same invariant, frames instead of a loop)

```c
/* Helper: search the half-open range [lo, hi). */
static int binary_search_rec(const int *a, size_t lo, size_t hi, int target) {
    if (lo >= hi) {            /* base case: empty range, absent */
        return -1;
    }
    size_t mid = lo + (hi - lo) / 2;
    if (a[mid] == target) {
        return (int)mid;
    }
    if (a[mid] < target) {
        return binary_search_rec(a, mid + 1, hi, target);
    }
    return binary_search_rec(a, lo, mid, target);
}

/* OWNERSHIP: borrows `a`, which MUST be sorted ascending. */
static int binary_search_r(const int *a, size_t n, int target) {
    return binary_search_rec(a, 0, n, target);
}
```

Base case: empty range. Recursive step: one of the two halves. Depth is O(log n) — about 20 frames for a million elements — so recursion is safe here. Prefer the iterative version anyway: same logic, one frame.

### The overflow-safe midpoint

The naive formula is a real bug on large arrays:

```c
size_t mid_bad = (lo + hi) / 2;        /* BUG when lo + hi overflows */
size_t mid = lo + (hi - lo) / 2;       /* safe: hi - lo <= n, never overflows */
```

With 32-bit `unsigned` indices and an array near 4 billion elements, `lo + hi` wraps around and `mid_bad` points outside the live range — an out-of-bounds read. With `size_t` on 64-bit the array would need to be impossibly large to trigger it, but the habit is what matters: **never add two indices**. `lo + (hi - lo) / 2` is correct at every width, so write it every time.

| Formula | Overflow when | Verdict |
| --- | --- | --- |
| `(lo + hi) / 2` | `lo + hi` exceeds index type | never use |
| `lo + (hi - lo) / 2` | never (`hi - lo <= n`) | always use |

### Precondition: sorted input

Binary search on an unsorted array returns garbage with full confidence:

```text
  a = [7, 1, 9, 2, 4] (NOT sorted), target = 2 (present at index 3)

  [0, 5): mid = 2, a[2] = 9 > 2 -> live = [0, 2)
  [0, 2): mid = 1, a[1] = 1 < 2 -> live = [2, 2) -> return -1 (WRONG: a[3] == 2)
```

No sanitizer catches this — every access is in bounds. The invariant "target, if present, is in `[lo, hi)`" is false from the first step on unsorted input, so the discarded half may hold the answer. Sorted input is a contract, not a suggestion. Document it on the function.

---

## Bubble Sort: Sink the Largest, One Pass at a Time

Repeatedly walk the array swapping adjacent out-of-order pairs. After each full pass, the largest unsorted element has "bubbled" to its final position at the end.

```c
#include <stdio.h>
#include <stddef.h>

static void swap(int *a, int *b) {
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

/* OWNERSHIP: MUTATES `a` in place; sorts ascending. */
static void bubble_sort(int *a, size_t n) {
    if (n < 2) {
        return;
    }
    for (size_t end = n; end > 1; end--) {
        /* Invariant after each outer pass: a[end..n) is sorted and final. */
        for (size_t i = 0; i + 1 < end; i++) {
            if (a[i] > a[i + 1]) {
                swap(&a[i], &a[i + 1]);
            }
        }
    }
}

static void print_array(const int *a, size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", a[i]);
    }
    printf("\n");
}

int main(void) {
    int a[] = { 5, 1, 4, 2, 8 };
    const size_t n = sizeof(a) / sizeof(a[0]);

    bubble_sort(a, n);
    print_array(a, n);
    return 0;
}
```

```text
1 2 4 5 8
```

Full trace of `[5, 1, 4, 2, 8]`, showing every comparison (`*` marks a swap):

```text
  pass 1 (end=5):  [5, 1, 4, 2, 8]
    5>1* -> [1, 5, 4, 2, 8]
    5>4* -> [1, 4, 5, 2, 8]
    5>2* -> [1, 4, 2, 5, 8]
    5<8   -> no swap
    result: [1, 4, 2, 5, 8]      8 is final

  pass 2 (end=4):  [1, 4, 2, 5 | 8]
    1<4   -> no swap
    4>2* -> [1, 2, 4, 5 | 8]
    4<5   -> no swap
    result: [1, 2, 4, 5 | 8]     5 is final

  passes 3-4 confirm the order with no swaps. Done.
```

Two details worth noticing:

- The inner condition is `i + 1 < end`, not `i < end`. Comparing `a[end]` against `a[end + 1]` would read past the array (or past the unsorted region). The `+ 1` form keeps every access in bounds with no subtraction that could underflow `size_t`.
- The `n < 2` early return handles empty and one-element arrays, and also protects `end > 1` bookkeeping. Sorting nothing is a no-op, not an error.

---

## Insertion Sort: Grow a Sorted Prefix

Keep a sorted prefix `a[0..i)`. Take element `a[i]`, shift larger prefix elements one slot right, and drop it into the gap.

```c
/* OWNERSHIP: MUTATES `a` in place; sorts ascending. */
static void insertion_sort(int *a, size_t n) {
    for (size_t i = 1; i < n; i++) {
        /* Invariant: a[0..i) is sorted. Insert a[i] into it. */
        int key = a[i];
        size_t j = i;
        while (j > 0 && a[j - 1] > key) {
            a[j] = a[j - 1];
            j--;
        }
        a[j] = key;
    }
}
```

Trace on `[5, 1, 4, 2, 8]` (`|` marks the prefix boundary, `[ ]` the gap):

```text
  start:        [5 | 1, 4, 2, 8]     prefix [5] is trivially sorted
  i=1 key=1:    shift 5 -> [1, 5 | 4, 2, 8]
  i=2 key=4:    shift 5, 1<4 stop -> [1, 4, 5 | 2, 8]
  i=3 key=2:    shift 5, shift 4 -> [1, 2, 4, 5 | 8]
  i=4 key=8:    5<8, no shifts -> [1, 2, 4, 5, 8 |]   done
```

```text
  sorted: 1 2 4 5 8
```

Why `j > 0` comes first in `while (j > 0 && a[j - 1] > key)`: C evaluates `&&` left to right and stops at the first false. When `j == 0`, the first test fails and `a[j - 1]` — which would be `a[SIZE_MAX]`, a giant out-of-bounds read — is never evaluated. Reversing the two conditions compiles identically and reads out of bounds on every insertion at the front. Under the sanitizer:

```text
=================================================================
==413112==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000000 (pc 0x5612aa0432c1 ...)
READ of size 4 at 0x000000000000 thread T0
    #0 0x5612aa0432c1 in insertion_sort /tmp/sort.c:14

SUMMARY: AddressSanitizer: SEGV /tmp/sort.c:14 in insertion_sort
```

Short-circuit order is a correctness property, not a style preference.

Bubble vs insertion at a glance:

| | Bubble sort | Insertion sort |
| --- | --- | --- |
| Idea | sink the largest to the end, per pass | insert the next element into the sorted prefix |
| Sorted input cost | O(n^2) comparisons (this version; O(n) with an early-exit flag) | O(n) — inner loop never shifts |
| Nearly-sorted input | slow (still scans everything) | fast (few shifts) |
| Code shape | two nested `for` loops + swap | one `for` + shifting `while` + key |

Both are O(n^2) worst case. Neither is used for large data. Both are worth writing once, because the invariant discipline — "what is true after each pass" — is the same discipline binary search and mergesort demand.

---

## The Complete Program

Linear search, both binary searches, both sorts, one `main` proving they agree:

```c
#include <stdio.h>
#include <stddef.h>

static void swap(int *a, int *b) {
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

static int linear_search(const int *a, size_t n, int target) {
    for (size_t i = 0; i < n; i++) {
        if (a[i] == target) {
            return (int)i;
        }
    }
    return -1;
}

static int binary_search(const int *a, size_t n, int target) {
    size_t lo = 0;
    size_t hi = n;
    while (lo < hi) {
        size_t mid = lo + (hi - lo) / 2;
        if (a[mid] == target) {
            return (int)mid;
        }
        if (a[mid] < target) {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    return -1;
}

static void bubble_sort(int *a, size_t n) {
    if (n < 2) {
        return;
    }
    for (size_t end = n; end > 1; end--) {
        for (size_t i = 0; i + 1 < end; i++) {
            if (a[i] > a[i + 1]) {
                swap(&a[i], &a[i + 1]);
            }
        }
    }
}

static void insertion_sort(int *a, size_t n) {
    for (size_t i = 1; i < n; i++) {
        int key = a[i];
        size_t j = i;
        while (j > 0 && a[j - 1] > key) {
            a[j] = a[j - 1];
            j--;
        }
        a[j] = key;
    }
}

static void print_array(const int *a, size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", a[i]);
    }
    printf("\n");
}

int main(void) {
    int b[] = { 5, 1, 4, 2, 8 };
    const size_t n = sizeof(b) / sizeof(b[0]);
    int c[] = { 5, 1, 4, 2, 8 };

    bubble_sort(b, n);
    insertion_sort(c, n);
    print_array(b, n);
    print_array(c, n);

    printf("linear 4 -> %d\n", linear_search(b, n, 4));
    printf("binary 4 -> %d\n", binary_search(b, n, 4));
    printf("binary 6 -> %d\n", binary_search(b, n, 6));
    return 0;
}
```

```text
1 2 4 5 8
1 2 4 5 8
linear 4 -> 2
binary 4 -> 2
binary 6 -> -1
```

---

## Common Bugs

| Bug | Symptom | Fix |
| --- | --- | --- |
| `(lo + hi) / 2` midpoint | out-of-bounds read on huge arrays; wrong `mid` after wraparound | `lo + (hi - lo) / 2`, always |
| `hi = n - 1` with `n == 0` | `size_t` underflow to `SIZE_MAX`, loop reads everywhere | half-open `[lo, hi)` with `hi = n` |
| Binary search on unsorted input | plausible but wrong answers, no sanitizer report | sort first; document the precondition |
| `while (a[j-1] > key && j > 0)` | SEGV at `j == 0` (reads `a[SIZE_MAX]`) | test `j > 0` first |
| Inner bubble condition `i < end` with `a[i+1]` | one-past-the-end read on the last comparison | `i + 1 < end` |
| Forgetting `-1` check before indexing | `a[linear_search(...)]` reads `a[-1]` as `a[SIZE_MAX]` | check `== -1` first, always |
| Sorting with an unsigned loop `for (i = n-1; ...)` at `n == 0` | underflow, giant loop | early return for `n < 2` |

The `a[-1]` case deserves one concrete report, because it compiles without a warning. `linear_search` returns `int`, and `-1` converts to `SIZE_MAX` when used as a `size_t` index:

```c
int pos = linear_search(a, n, 99);
printf("%d\n", a[pos]);   /* BUG when pos == -1: a[-1] is out of bounds */
```

```text
=================================================================
==413255==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000000 (pc 0x5612aa043310 ...)
READ of size 4 at 0x000000000000 thread T0
    #0 0x5612aa043310 in main /tmp/search.c:41

SUMMARY: AddressSanitizer: SEGV /tmp/search.c:41 in main
```

Guard every use:

```c
int pos = linear_search(a, n, 99);
if (pos < 0) {
    printf("absent\n");
} else {
    printf("a[%d] = %d\n", pos, a[(size_t)pos]);
}
```

---

## Checklist

```text
[ ] swap is one static helper, used by every sort (no duplicated exchange code)
[ ] read-only array parameters are const int *a; mutated ones are int *a
[ ] all indices and counts are size_t, printed with %zu
[ ] search misses return -1 and every caller checks for -1 before indexing
[ ] midpoint is always lo + (hi - lo) / 2, never (lo + hi) / 2
[ ] binary search uses half-open [lo, hi) with hi = n (empty array safe)
[ ] binary search documents the sorted-input precondition in a comment
[ ] insertion-sort while tests j > 0 BEFORE touching a[j - 1]
[ ] bubble inner loop uses i + 1 < end; both sorts return early for n < 2
[ ] builds with -fsanitize=address,undefined, runs with no diagnostics
```

---

#### [Previous](./01-part.md) | [Next](./03-part.md)
