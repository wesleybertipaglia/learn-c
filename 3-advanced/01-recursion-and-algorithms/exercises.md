# Exercise List 1 - Recursion and Algorithms

Six exercises covering the three parts: recursion, hand-written search and sort, then complexity and divide and conquer. Every program must compile with `gcc -std=c11 -Wall -Wextra -pedantic -g file.c -o file` with **zero** warnings, then also compile and run clean under:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer file.c -o file_san
```

Rules for all six: `static` helpers, `const` on read-only pointer parameters, `size_t` for indices and counts, every `malloc` checked against `NULL`, no VLAs, and no `qsort`/`bsearch` anywhere — the sorting and searching are implemented by hand.

---

## Exercise 1: Recursive Factorial and Fibonacci

Write two recursions with explicit base cases: `factorial` and `fibonacci`.

```c
#include <stdio.h>

/* Returns n!. Base cases: 0! = 1, 1! = 1. */
static unsigned long long factorial(unsigned int n) {
    /* TODO: base case, then n * factorial(n - 1). */
    (void)n;
    return 0ULL;   /* replace */
}

/* Returns fib(n): fib(0) = 0, fib(1) = 1, fib(n) = fib(n-1) + fib(n-2). */
static unsigned long long fib(unsigned int n) {
    /* TODO: two base cases, then the double recursive call. */
    (void)n;
    return 0ULL;   /* replace */
}

int main(void) {
    /* TODO: print factorial(0..10) and fib(0..10).
       TODO: also print factorial(5) from an iterative version and
       assert both agree. */
    return 0;
}
```

Expected output (after the TODOs are done):

```text
0! = 1
5! = 120
10! = 3628800
fib(0) = 0
fib(1) = 1
fib(10) = 55
iter factorial(5) = 120
```

Requirements:

- State each base case in a comment before coding it. `factorial` needs one (`n == 0`); `fib` needs two (`n == 0`, `n == 1`). Delete one `fib` base case on purpose, run `fib(3)`, and record what happens (infinite mutual recursion into negative-equivalent wraparound for unsigned input, ending in stack overflow). Then restore it.
- Write `factorial_iter` with a loop and compare outputs for `0..10`. Explain in a comment why the loop is the better production choice (O(1) stack vs O(n) frames) even though both print the same numbers.
- Keep inputs small: `n <= 20` for factorial (64-bit holds `20!`), `n <= 30` for naive `fib` (the call count doubles each step — exercise 6 measures exactly why). Document both limits in comments.
- Call `fib` with `n = 0` and `n = 1` directly and confirm the base cases return without further calls. Draw the call tree for `fib(4)` on paper (9 calls total) before running it.
- Extra credit: count calls with a `static` counter inside `fib` (reset between runs) and print `calls(10)`, `calls(20)`. Confirm the count roughly doubles per increment of `n`.

---

## Exercise 2: Binary Search With a Safe Midpoint

Implement iterative binary search over a sorted array, with the overflow-safe midpoint and the half-open `[lo, hi)` invariant.

```c
#include <stdio.h>
#include <stddef.h>

/* OWNERSHIP: borrows `a`, which MUST be sorted ascending.
   Returns the index of `target`, or -1 when absent. */
static int binary_search(const int *a, size_t n, int target) {
    /* TODO: lo = 0, hi = n; while (lo < hi): mid = lo + (hi - lo) / 2;
       compare a[mid] vs target, shrink to [mid+1, hi) or [lo, mid). */
    (void)a;
    (void)n;
    (void)target;
    return -1;   /* replace */
}

int main(void) {
    const int a[] = { 1, 2, 4, 7, 9, 12 };
    const size_t n = sizeof(a) / sizeof(a[0]);

    /* TODO: search for 7 (middle), 1 (first), 12 (last), 5 (absent),
       and in an empty range (n = 0). Print each result. */
    (void)a;
    (void)n;

    return 0;
}
```

Expected output:

```text
find 7 -> 3
find 1 -> 0
find 12 -> 5
find 5 -> -1
find in empty -> -1
```

Requirements:

- The midpoint must be `lo + (hi - lo) / 2`. Write a comment explaining why `(lo + hi) / 2` can overflow and why the safe form cannot (`hi - lo <= n` always fits).
- Use the half-open interval `[lo, hi)` with `hi = n`. Explain in a comment why `hi = n - 1` underflows `size_t` when `n == 0` and why your form needs no special case for the empty array.
- State the invariant in a comment above the loop: "if `target` is present, it is in `a[lo..hi)`." Argue in one sentence per branch why each update preserves it.
- Test all five cases above plus a one-element array (`{42}`, found and absent). Record which case each branch (`==`, `<`, `>`) covers.
- Deliberately run the search on an unsorted array (`{7, 1, 9, 2, 4}`, target `2`) and record the wrong answer. Explain in a comment why no sanitizer reports it (every access is in bounds; the invariant was false from the start).
- Extra credit: write the recursive helper `binary_search_rec(a, lo, hi, target)` with base case `lo >= hi` and confirm identical results. State its depth as O(log n).

---

## Exercise 3: Bubble Sort and Comparison Counting

Implement bubble sort by hand and count exactly how many comparisons it performs.

```c
#include <stdio.h>
#include <stddef.h>

static size_t g_comparisons = 0;

static void swap(int *a, int *b) {
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

/* OWNERSHIP: MUTATES `a` in place; sorts ascending. Counts comparisons. */
static void bubble_sort(int *a, size_t n) {
    /* TODO: early return for n < 2. Outer end = n down to 2,
       inner i with i + 1 < end, count each comparison, swap when a[i] > a[i+1]. */
    (void)a;
    (void)n;
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

    /* TODO: sort, print the result, print g_comparisons.
       TODO: re-run on an already-sorted copy and on a reversed copy;
       print all three counts. */
    (void)a;
    (void)n;

    return 0;
}
```

Expected output:

```text
sorted: 1 2 4 5 8
comparisons (n=5): 10
sorted-input comparisons: 10
reversed-input comparisons: 10
```

Requirements:

- The count for `n = 5` must be exactly `10` (`4 + 3 + 2 + 1`). Derive `n*(n-1)/2` in a comment and verify `10 == 5*4/2` in the program output.
- The inner condition must be `i + 1 < end`. Explain in a comment what `a[i + 1]` touches when `i == end` under the wrong condition, and confirm the sanitizer reports it on the first run if written wrongly.
- Early-return for `n < 2` is mandatory. Test `bubble_sort(a, 0)` and a one-element array; both must be silent no-ops.
- Show that this version does the same 10 comparisons on sorted, random, and reversed input (no early-exit flag). Explain in a comment why that proves the comparison count depends only on `n`, not on order — and why that is an argument for insertion sort on nearly-sorted data (next exercise).
- Verify the result is sorted with a `static int is_sorted(const int *a, size_t n)` helper. Call it after every sort; abort to `stderr` when it fails.
- Extra credit: add the early-exit flag (`swapped` per pass, stop when a pass swaps nothing) and report the sorted-input count drops to `n - 1 = 4`. Keep the flag version clearly separated from the required version.

---

## Exercise 4: Insertion Sort

Implement insertion sort by hand: grow the sorted prefix `a[0..i)`, insert `a[i]` by shifting.

```c
#include <stdio.h>
#include <stddef.h>

/* OWNERSHIP: MUTATES `a` in place; sorts ascending. */
static void insertion_sort(int *a, size_t n) {
    /* TODO: for i in [1, n): key = a[i]; j = i;
       while (j > 0 && a[j-1] > key) { a[j] = a[j-1]; j--; }
       a[j] = key; */
    (void)a;
    (void)n;
}

static void print_array(const int *a, size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", a[i]);
    }
    printf("\n");
}

/* OWNERSHIP: borrows `a`. Returns 1 when ascending, 0 otherwise. */
static int is_sorted(const int *a, size_t n) {
    for (size_t i = 1; i < n; i++) {
        if (a[i - 1] > a[i]) {
            return 0;
        }
    }
    return 1;
}

int main(void) {
    int a[] = { 5, 1, 4, 2, 8 };
    int sorted[] = { 1, 2, 3, 4, 5 };
    const size_t n = sizeof(a) / sizeof(a[0]);

    /* TODO: sort `a`, print it, assert is_sorted.
       TODO: sort `sorted` (already sorted), print shift behavior.
       TODO: sort a one-element and an empty array. */
    (void)a;
    (void)n;

    (void)sorted;
    return 0;
}
```

Expected output:

```text
sorted: 1 2 4 5 8
already sorted stays: 1 2 3 4 5
one element: 42
empty: (no output, no crash)
```

Requirements:

- The `while` condition order (`j > 0` first) is load-bearing. Explain in a comment why `a[j - 1] > key && j > 0` reads `a[SIZE_MAX]` when `j == 0`, and paste the sanitizer SEGV report from one run with the conditions swapped (then swap them back).
- Trace the first two insertions of `[5, 1, 4, 2, 8]` on paper (`i = 1, key = 1`; `i = 2, key = 4`) showing each shift, before coding. The trace is the implementation plan.
- Count shifts with a global counter and report three numbers: random input, already-sorted input (expect 0 shifts), reversed input (expect `n*(n-1)/2` shifts). Explain in a comment why insertion sort is O(n) on sorted input while the exercise-3 bubble sort is O(n^2) on the same input.
- Handle `n == 0` (loop never runs) and `n == 1` (loop never runs) with no special-case branch. Explain in a comment why none is needed (`i` starts at 1).
- No `qsort`, no `memcpy` tricks for the shifting — the `while` loop moves elements one slot at a time so every read and write is visible.
- Extra credit: make the sort stable by construction. Test with an array of structs sorted by key (`{key, id}` pairs with duplicate keys) and confirm equal keys keep their original order. State in a comment which operator (`>` vs `>=`) guarantees it and why.

---

## Exercise 5: Mergesort or Recursive Linked-List Reverse

Pick **one** of the two tracks. Both practice the same skill — a recursion whose combine step must touch exactly the right pointers — at different depths (O(log n) vs O(n)).

### Track A: Mergesort

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

/* OWNERSHIP: borrows a[lo..hi) (halves sorted); merges into a[lo..hi)
   using tmp[lo..hi) as scratch. */
static void merge(int *a, int *tmp, size_t lo, size_t mid, size_t hi) {
    /* TODO: two cursors + copy rest + copy back [lo, hi). */
    (void)a;
    (void)tmp;
    (void)lo;
    (void)mid;
    (void)hi;
}

static void mergesort_rec(int *a, int *tmp, size_t lo, size_t hi) {
    /* TODO: base hi - lo < 2; mid = lo + (hi - lo) / 2; recurse; merge. */
    (void)a;
    (void)tmp;
    (void)lo;
    (void)hi;
}

/* OWNERSHIP: MUTATES `a`; 0 on success, -1 when malloc fails. */
static int mergesort(int *a, size_t n) {
    /* TODO: n < 2 -> 0; malloc tmp; NULL check; recurse; free; return. */
    (void)a;
    (void)n;
    return -1;   /* replace */
}

int main(void) {
    int a[] = { 5, 1, 4, 2, 8, 7, 3, 6 };
    const size_t n = sizeof(a) / sizeof(a[0]);

    /* TODO: mergesort, check return, print result, verify sorted. */
    (void)a;
    (void)n;

    return 0;
}
```

Track A expected output:

```text
1 2 3 4 5 6 7 8
sorted: yes
```

### Track B: Recursive Linked-List Reverse

```c
#include <stdio.h>
#include <stdlib.h>

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

/* OWNERSHIP: takes ownership of `head`; returns new head (caller owns it). */
static Node *reverse_rec(Node *head) {
    /* TODO: base head == NULL || head->next == NULL -> return head.
       rest = reverse_rec(head->next); head->next->next = head;
       head->next = NULL; return rest. */
    (void)head;
    return NULL;   /* replace */
}

static void print_list(const struct Node *head) {
    for (const struct Node *p = head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}

static void destroy_list(Node *head) {
    while (head != NULL) {
        Node *next = head->next;
        free(head);
        head = next;
    }
}

int main(void) {
    /* TODO: build 10 -> 20 -> 30, print, reverse_rec, print again,
       reverse back, print, destroy. */

    return 0;
}
```

Track B expected output:

```text
before: 10 -> 20 -> 30 -> NULL
after: 30 -> 20 -> 10 -> NULL
back: 10 -> 20 -> 30 -> NULL
```

Requirements (both tracks):

- Track A: exactly one `malloc` and one `free` of the scratch buffer, in `mergesort` (not in the recursion). The `NULL` check must leave the caller's array untouched and return `-1`; `main` must check it. Delete the `free` once, run under the sanitizer, and paste the `Direct leak of 400 byte(s)`-style report for your `n`. Then restore it.
- Track A: copy back exactly `[lo, hi)`. Copying the whole array each merge still sorts but wastes O(n log n) moves — explain in a comment why the range form is the correct contract. Use `<=` on ties for stability and test with duplicates (`{3, 1, 3, 2}` -> `{1, 2, 3, 3}`).
- Track B: the two rewire lines in order (`head->next->next = head` before `head->next = NULL`). Swap them once, record the sanitizer SEGV (`READ of size 8 at 0x0`), explain which pointer was destroyed first. Then restore.
- Track B: test `reverse_rec(NULL)` (expect `NULL`) and a one-node list (expect it unchanged). Explain in a comment why no extra branch is needed when the base case covers both.
- Both tracks: state the maximum depth in a comment (Track A: O(log n) frames, safe at any size; Track B: O(n) frames, one per node — fine for 3 nodes, dangerous at a million). Name which track you chose and why the other was deferred.
- Extra credit: do the other track as well and compare depth behavior in one comment block.

---

## Exercise 6: Measure and Compare

Count operations (not wall-clock guesses) for linear search, binary search, and one O(n^2) sort across doubling input sizes, and confirm the shapes O(n), O(log n), O(n^2).

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

static size_t g_cmp = 0;

static void swap(int *a, int *b) {
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

/* TODO: linear_search counting each a[i] == target test in g_cmp. */
/* TODO: binary_search counting each a[mid] == target test in g_cmp. */
/* TODO: bubble_sort (or insertion_sort) counting each element
   comparison in g_cmp. */

static void fill_sorted(int *a, size_t n) {
    for (size_t i = 0; i < n; i++) {
        a[i] = (int)(i * 2);   /* even numbers: every odd target is absent */
    }
}

int main(void) {
    /* TODO: for n in {1000, 2000, 4000, 8000}:
       malloc array, fill_sorted, time/count worst-case linear search
       (absent target), worst-case binary search (absent target),
       and bubble-sort comparisons on a reversed copy.
       Print one table row per n. Free everything. */

    return 0;
}
```

Expected output (counts; yours must show the same ratios):

```text
n        linear   binary   bubble
1000     1000     10       499500
2000     2000     11       1999000
4000     4000     12       7998000
8000     8000     13       31996000
ratios (2x input): linear ~2x, binary +1, bubble ~4x
```

Requirements:

- Drive sizes `{1000, 2000, 4000, 8000}` with `malloc` (checked for `NULL`, `free`d each round). No VLAs — the arrays are heap-allocated precisely so the sanitizer watches every access.
- Worst-case inputs, all three: search for an absent target (`-1`: odd number in an evens array) so every candidate is examined; sort a reversed array so every pair is tested. Explain in a comment why best-case inputs (found-at-front, already-sorted) would hide the growth shape.
- Reset `g_cmp = 0` before each measurement and print raw counts, not only Big-O labels. Confirm the ratios by division in the output or comments: linear `2000/1000 = 2`, binary `11 - 10 = +1`, bubble `1999000/499500 ~= 4`.
- Include the complexity table from part 03 (O(log n) / O(n) / O(n^2) rows for your four sizes) as a comment and mark where your measured counts land in it.
- State the conclusion in a comment block: at `n = 8000` bubble does ~32000000 comparisons vs binary search's 13 — and project `n = 10^6` from the table (binary ~20, bubble ~5*10^11). One sentence on why the O(n^2) sort must be replaced above a few thousand elements.
- Extra credit: add `clock()` timing around each measurement (`time.h`, `CLOCKS_PER_SEC`) and show time ratios tracking count ratios. Note in a comment why counts are the stable metric (timing varies with machine load; counts do not).

---

## Sanitize Everything

Do this for **every** exercise above, and for every exercise from now on:

1. Compile with warnings as errors and with both sanitizers:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -Werror -g \
       -fsanitize=address,undefined -fno-omit-frame-pointer main.c -o main_san
   ```

2. Run it several times, including the empty and one-element inputs. Boundary bugs hide at size 0 and 1.
3. Fix everything the sanitizer reports. It is not a suggestion.
4. Re-run until the output contains no sanitizer diagnostics at all. Silence is the passing state.
5. Then compile the plain version and confirm identical behavior:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
   ```

Report, for each program, in a comment at the top of the file:

```text
[ ] compiles with -Werror, zero warnings
[ ] no sanitizer diagnostics
[ ] every malloc has a matching free
[ ] every free argument came from malloc (no stack frees, no double frees)
[ ] no pointer used after free
[ ] every index checked before use (no walk past bounds, no -1 used as index)
[ ] no qsort/bsearch/VLA used; sorts and searches are hand-written
```

That checklist is the habit the whole module is trying to build. A sort that passes it may still order two elements wrong — but it will not read past the array finding out.

---

#### [Previous](./03-part.md) | [Next](../02-bit-manipulation/01-part.md)
