# Complexity and Divide and Conquer

Part 01 built recursions. Part 02 searched and sorted by hand. This part asks the next question: **how fast**, as the input grows — and then uses that answer to justify one big recursive sort.

The compile command is unchanged:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer complex.c -o complex
```

---

## Counting Operations

Complexity starts as literal counting. Take linear search:

```c
static int linear_search(const int *a, size_t n, int target) {
    for (size_t i = 0; i < n; i++) {   /* n+1 condition checks */
        if (a[i] == target) {          /* up to n comparisons */
            return (int)i;
        }
    }
    return -1;
}
```

Worst case (target absent): `n` comparisons, `n + 1` loop-condition checks, `n` increments. Total work grows **proportionally to n**. Double the array, double the time. That is what "O(n)" means: the shape of the growth, ignoring constants.

Now one inner step of bubble sort:

```c
for (size_t end = n; end > 1; end--) {
    for (size_t i = 0; i + 1 < end; i++) {
        if (a[i] > a[i + 1]) {   /* one comparison per inner iteration */
            swap(&a[i], &a[i + 1]);
        }
    }
}
```

The inner loop runs `n - 1`, then `n - 2`, then `n - 3`, down to `1` times. Total comparisons:

```text
  (n-1) + (n-2) + ... + 1 = n*(n-1)/2
```

For `n = 5`: `4 + 3 + 2 + 1 = 10`. For `n = 1000`: `499500`. Double the array and the work roughly **quadruples**. That is "O(n^2)".

Binary search halves the interval each step:

```text
  n -> n/2 -> n/4 -> ... -> 1        about log2(n) steps
```

For `n = 1024`: 10 steps. For `n = 1048576` (a million): 20 steps. Double the array and the work grows by **one** comparison. That is "O(log n)".

---

## The Table, With Real Numbers

Assume one comparison costs 100 ns (generous for an `int` compare, pessimistic enough to feel real). `n*(n-1)/2` for O(n^2), `n` for O(n), `log2(n)` rounded up for O(log n):

| n | O(log n) steps | O(n) steps | O(n^2) steps | O(n) time | O(n^2) time |
| --- | --- | --- | --- | --- | --- |
| 10 | 4 | 10 | 45 | 1 us | 4.5 us |
| 100 | 7 | 100 | 4950 | 10 us | 0.5 ms |
| 1,000 | 10 | 1,000 | 499,500 | 0.1 ms | 50 ms |
| 10,000 | 14 | 10,000 | ~50M | 1 ms | 5 s |
| 1,000,000 | 20 | 1,000,000 | ~5*10^11 | 0.1 s | ~1.6 years |
| 10,000,000 | 24 | 10,000,000 | ~5*10^13 | 1 s | ~158 years |

Read the last two rows twice. At a million elements, linear work finishes in a blink and quadratic work outlives the programmer. That is why part 02's sorts are teaching tools: bubble and insertion sort are correct, but their O(n^2) shape disqualifies them above a few thousand elements.

A measured confirmation on one machine (your numbers will differ, the ratios will not):

```c
#include <stdio.h>
#include <time.h>

static size_t g_comparisons = 0;

static int counted_less(int a, int b) {
    g_comparisons++;
    return a < b;
}
```

```text
  n=1000:   bubble comparisons = 499500,   binary-search worst = 10
  n=2000:   bubble comparisons = 1999000 (~4x for 2x input: quadratic)
  n=4000:   bubble comparisons = 7998000 (~4x again)
  linear scan n=4000 worst = 4000 (~2x for 2x input: linear)
```

Doubling the input quadruples bubble sort's count and merely doubles linear search's. The table is not theory; it is what the counter prints.

What Big-O drops and why:

- **Constants:** O(2n) and O(n) are the same shape. Counting `n` vs `3n` comparisons stops mattering once `n` dominates.
- **Lower-order terms:** O(n^2 + n) is O(n^2). At `n = 10^6`, the `n` part is a rounding error.
- **Best vs worst vs average:** linear search is O(1) best case (front element), O(n) worst case (absent). Unless stated otherwise, complexity means **worst case** — the guarantee, not the luck.

| Complexity | Shape | Example in this course | 2x input costs |
| --- | --- | --- | --- |
| O(1) | constant | `swap`, array index | nothing extra |
| O(log n) | halving | binary search | +1 step |
| O(n) | scanning | linear search, `print_array`, merge step | 2x |
| O(n log n) | halve + linear merge, per level | mergesort (below) | slightly more than 2x |
| O(n^2) | all pairs | bubble, insertion sort | 4x |

---

## Divide and Conquer: The Idea

Divide and conquer solves a big problem with three moves:

1. **Divide** — split the input into smaller pieces.
2. **Conquer** — solve each piece (recursively, until trivially small).
3. **Combine** — merge the solved pieces into a full answer.

```text
  conquer [5, 1, 4, 2, 8]
    divide:   [5, 1, 4] | [2, 8]
    conquer:  sort each half -> [1, 4, 5] | [2, 8]
    combine:  merge the two sorted halves -> [1, 2, 4, 5, 8]
```

Binary search is divide and conquer without the combine step (it discards one half). Mergesort is the full pattern: split in half, sort each half, merge.

Depth is O(log n) halvings. Each level does O(n) total merge work across all its pieces. O(log n) levels times O(n) per level: **O(n log n)**. At `n = 10^6` that is ~20M operations instead of bubble sort's ~5*10^11 — the difference between instant and interminable.

---

## Mergesort: Halve, Sort, Merge

Requirements before reading the code: the merge step needs a temporary buffer (one `malloc`, one `free`, both in the same function), `size_t` indices, and a copy-back that moves exactly the merged range.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

/* OWNERSHIP: borrows `a[lo..hi)` (both halves sorted); writes the
   merged sorted run back into `a[lo..hi)`. Uses `tmp[lo..hi)` as
   scratch space borrowed from the caller. */
static void merge(int *a, int *tmp, size_t lo, size_t mid, size_t hi) {
    size_t i = lo;     /* cursor in left half  [lo, mid) */
    size_t j = mid;    /* cursor in right half [mid, hi) */
    size_t k = lo;     /* cursor in scratch    [lo, hi)  */

    while (i < mid && j < hi) {
        if (a[i] <= a[j]) {
            tmp[k] = a[i];
            i++;
        } else {
            tmp[k] = a[j];
            j++;
        }
        k++;
    }
    /* One half is exhausted. Copy the rest of the other half.
       Exactly one of these two loops runs. */
    while (i < mid) {
        tmp[k] = a[i];
        i++;
        k++;
    }
    while (j < hi) {
        tmp[k] = a[j];
        j++;
        k++;
    }
    /* Copy back ONLY the merged range. */
    for (size_t t = lo; t < hi; t++) {
        a[t] = tmp[t];
    }
}

static void mergesort_rec(int *a, int *tmp, size_t lo, size_t hi) {
    if (hi - lo < 2) {             /* base case: 0 or 1 elements, sorted */
        return;
    }
    size_t mid = lo + (hi - lo) / 2;   /* same overflow-safe midpoint */
    mergesort_rec(a, tmp, lo, mid);
    mergesort_rec(a, tmp, mid, hi);
    merge(a, tmp, lo, mid, hi);
}

/* OWNERSHIP: MUTATES `a` in place; sorts ascending.
   Returns 0 on success, -1 when the scratch buffer cannot be allocated
   (the array is left in an unspecified but in-bounds state). */
static int mergesort(int *a, size_t n) {
    int *tmp;
    if (n < 2) {
        return 0;
    }
    tmp = malloc(n * sizeof(int));
    if (tmp == NULL) {
        return -1;
    }
    mergesort_rec(a, tmp, 0, n);
    free(tmp);
    tmp = NULL;
    return 0;
}

static void print_array(const int *a, size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", a[i]);
    }
    printf("\n");
}

int main(void) {
    int a[] = { 5, 1, 4, 2, 8, 7, 3, 6 };
    const size_t n = sizeof(a) / sizeof(a[0]);

    if (mergesort(a, n) != 0) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }
    print_array(a, n);
    return 0;
}
```

```text
1 2 3 4 5 6 7 8
```

Trace on `[5, 1, 4, 2]` (addresses are indices, time goes down):

```text
  mergesort [0, 4): mid = 2
  +-- mergesort [0, 2): mid = 1
  |   +-- mergesort [0, 1): 1 element, base case, return
  |   +-- mergesort [1, 2): 1 element, base case, return
  |   +-- merge [0, 1, 2): [5] + [1] -> [1, 5]
  +-- mergesort [2, 4): mid = 3
  |   +-- mergesort [2, 3): base case
  |   +-- mergesort [3, 4): base case
  |   +-- merge [2, 3, 4): [4] + [2] -> [2, 4]
  +-- merge [0, 2, 4): [1, 5] + [2, 4] -> [1, 2, 4, 5]
```

```text
  the recursion tree for n = 8 (each row is one level, O(n) merge work):

  level 0: [5, 1, 4, 2, 8, 7, 3, 6]                       1 merge of 8
  level 1: [1, 4, 5] ... wait, halves: [5,1,4,2] [8,7,3,6] 2 merges of 4
  level 2: [5,1] [4,2] [8,7] [3,6]                        4 merges of 2
  level 3: [5] [1] [4] [2] [8] [7] [3] [6]                base cases
```

Three correctness properties, each load-bearing:

1. **One allocation, one free, same function.** `mergesort` owns `tmp` entirely. The recursive helper only borrows it. No per-call `malloc` (which would be O(n log n) allocations), no leak path: `free` sits on the single exit after the recursion.
2. **The copy-back range is exact.** `for (t = lo; t < hi; t++) a[t] = tmp[t]` moves only the merged run. Copying the whole array each time would still sort correctly but breaks no invariant — the real danger is the reverse: copying too little leaves stale values. `lo..hi` is the contract; honor it.
3. **`<=` keeps the sort stable.** `if (a[i] <= a[j])` takes from the left half on ties, so equal elements keep their original relative order. `<` would also sort, but silently unstabilize. Neither the compiler nor the sanitizer flags the difference — tests with duplicate values do.

Memory picture during `merge(a, tmp, 0, 2, 4)` with `a = [1, 5, 2, 4]`:

```text
  a:   [1, 5, 2, 4]      two sorted halves: [1, 5] and [2, 4]
  tmp: [_, _, _, _]      scratch, same length as a

  i=0(1) vs j=2(2): take 1 -> tmp=[1, _, _, _]
  i=1(5) vs j=2(2): take 2 -> tmp=[1, 2, _, _]
  i=1(5) vs j=3(4): take 4 -> tmp=[1, 2, 4, _]
  left rest: 5          -> tmp=[1, 2, 4, 5]
  copy back [0, 4):  a=[1, 2, 4, 5]
```

---

## Recursion on Linked Lists

A linked list is a recursive structure: a list is either empty (`NULL`) or a value followed by a smaller list. Functions that follow that shape read naturally as recursions.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

typedef struct Node Node;

struct Node {
    int value;
    Node *next;
};

static Node *push_front(Node *head, int value) {
    Node *node = malloc(sizeof(Node));
    if (node == NULL) {
        return NULL;
    }
    node->value = value;
    node->next = head;
    return node;
}

/* OWNERSHIP: borrows `head`. */
static size_t length_rec(const Node *head) {
    if (head == NULL) {            /* base case: empty list has length 0 */
        return 0;
    }
    return 1 + length_rec(head->next);   /* 1 for this node + rest */
}

/* OWNERSHIP: takes ownership of `head`; returns the NEW head of the
   reversed list (caller owns it). Reuses nodes: no malloc, no free. */
static Node *reverse_rec(Node *head) {
    if (head == NULL || head->next == NULL) {
        return head;               /* base case: empty or single node */
    }
    Node *rest = reverse_rec(head->next);   /* reverse everything after head */
    head->next->next = head;       /* the old second node now points at head */
    head->next = NULL;             /* head becomes the new tail */
    return rest;                   /* the old tail is the new head */
}

/* OWNERSHIP: borrows `head`. */
static void print_list(const Node *head) {
    for (const Node *p = head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}

/* OWNERSHIP: takes ownership of `head`. */
static void destroy_list(Node *head) {
    while (head != NULL) {
        Node *next = head->next;
        free(head);
        head = next;
    }
}

int main(void) {
    Node *head = NULL;

    head = push_front(head, 30);
    head = push_front(head, 20);
    head = push_front(head, 10);
    if (head == NULL) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    printf("length = %zu\n", length_rec(head));
    print_list(head);

    head = reverse_rec(head);
    print_list(head);

    destroy_list(head);
    head = NULL;
    return 0;
}
```

```text
length = 3
10 -> 20 -> 30 -> NULL
30 -> 20 -> 10 -> NULL
```

Trace `reverse_rec([10|*]->[20|*]->[30|NULL])`:

```text
  reverse_rec(10-node):
    rest = reverse_rec(20-node):
      rest = reverse_rec(30-node):  base case (next == NULL), return 30-node
      20-node->next->next = 20-node   (30-node now points at 20-node)
      20-node->next = NULL
      return 30-node
    10-node->next->next = 10-node     (20-node now points at 10-node)
    10-node->next = NULL
    return 30-node                    (30-node is the new head)

  result: [30|*]->[20|*]->[10|NULL]
```

The two lines after the recursive call are the whole algorithm, and their order is the whole bug surface. Setting `head->next = NULL` **before** `head->next->next = head` destroys the only pointer to the reversed rest — the list is severed and the tail is leaked:

```c
/* BUG: severs the list */
head->next = NULL;             /* the link to `rest` is gone */
head->next->next = head;       /* NULL->next: immediate SEGV */
```

```text
=================================================================
==414010==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000000 (pc 0x5612aa0432a8 ...)
READ of size 8 at 0x000000000000 thread T0
    #0 0x5612aa0432a8 in reverse_rec /tmp/llrec.c:31

SUMMARY: AddressSanitizer: SEGV /tmp/llrec.c:31 in reverse_rec
```

Depth warning, carried over from part 01: `length_rec` and `reverse_rec` both use O(n) frames — one per node. A million-node list overflows the stack where the iterative `for (p = head; ...)` loop does not. Recursive list functions are excellent for learning the structure and fine for short lists; for production-length lists, write the loop. Part 01's three-question test applies unchanged.

| Function | Base case | Smaller input | Depth |
| --- | --- | --- | --- |
| `length_rec` | `NULL` -> 0 | `head->next` | O(n) — loop preferred for long lists |
| `reverse_rec` | empty or single node | `head->next` | O(n) — loop preferred for long lists |
| `mergesort_rec` | fewer than 2 elements | halves `[lo,mid)`, `[mid,hi)` | O(log n) — always safe |
| `binary_search_rec` | empty range | one half | O(log n) — always safe |

---

## Common Bugs

| Bug | Symptom | Fix |
| --- | --- | --- |
| `malloc(n * sizeof(int))` unchecked | NULL dereference on OOM | `if (tmp == NULL) return -1;` and check the return |
| `free` missing for `tmp` | LeakSanitizer: `n * 4 byte(s)` leaked | one `malloc` + one `free` in `mergesort`, same function |
| Copy-back over the wrong range | silently wrong order after some merges | copy exactly `[lo, hi)` |
| `<` instead of `<=` in merge | sorts but unstable; duplicate-order tests fail | `a[i] <= a[j]` takes from the left on ties |
| Midpoint `(lo + hi) / 2` | wraparound on huge arrays | `lo + (hi - lo) / 2` (same rule as part 02) |
| Reverse lines swapped | immediate SEGV (`NULL->next`) | rewire `head->next->next` before clearing `head->next` |
| Deep list recursion (10^6 nodes) | `stack-overflow`, one frame per node | iterative loop for long lists |

The leak report for the missing `free(tmp)` is deterministic — 400 bytes for `n = 100`:

```text
=================================================================
==414122==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 400 byte(s) in 1 object(s) allocated from:
    #0 0x7a2b1d82b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:52
    #1 0x5612aa0432c1 in mergesort /tmp/complex.c:52

SUMMARY: AddressSanitizer: 400 byte(s) leaked in 1 allocation(s).
```

One allocation, one line, one missing `free`. The report names all three.

---

## Checklist

```text
[ ] operation counts measured or derived before Big-O claimed (n*(n-1)/2, log2 steps)
[ ] complexity table read for 2x-input cost: +1 step / 2x / 4x
[ ] mergesort: one malloc + NULL check + one free in the same function
[ ] mergesort: overflow-safe mid, exact [lo, hi) copy-back, <= for stability
[ ] list recursions: NULL base case handled, empty and one-node lists tested
[ ] list reverse rewires head->next->next BEFORE clearing head->next
[ ] depth checked: O(log n) recursion kept, O(n) recursion limited to short lists
[ ] helpers are static; read-only parameters are const; counts are size_t
[ ] builds with -fsanitize=address,undefined, runs with no diagnostics
```

---

#### [Previous](./02-part.md) | [Next](./exercises.md)
