# Callbacks and qsort

A callback is a function pointer you hand to someone else so *they* call *you* back. The C standard library does this twice: `qsort` sorts anything through your comparator, and `bsearch` finds anything through the same kind of comparator. Your code supplies the ordering; the library owns the algorithm.

This part covers the exact `qsort` comparator signature (`const void *`, correct casts, `<0` / `0` / `>0` without subtraction overflow), `bsearch`, generic callbacks with a `void *context` parameter, a strategy example (ascending vs descending via callback), and the classic `return a - b` overflow bug.

Compile everything with:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

---

## 1. The `qsort` Contract

```c
void qsort(void *base, size_t nmemb, size_t size,
           int (*compar)(const void *a, const void *b));
```

The library sorts `nmemb` elements of `size` bytes starting at `base`. It knows nothing about your data, so every time it needs to order two elements it calls your `compar`:

- return **negative** if `a` goes before `b`,
- return **zero** if they are equal,
- return **positive** if `a` goes after `b`.

Give the type a name once and reuse it:

```c
typedef int (*QsortCmp)(const void *a, const void *b);
```

Why `const void *`? Because `qsort` must accept *any* element type. `void *` erases the type; `const` promises your comparator will not modify the elements while comparing them.

Full integer example — the safe way:

```c
#include <stdio.h>
#include <stdlib.h>

typedef int (*QsortCmp)(const void *a, const void *b);

/* SAFE comparator: branches, no subtraction, no overflow */
int cmp_int(const void *pa, const void *pb) {
    const int *a = (const int *)pa;
    const int *b = (const int *)pb;

    if (*a < *b) {
        return -1;
    }
    if (*a > *b) {
        return 1;
    }
    return 0;
}

void print_ints(const int *arr, size_t n) {
    for (size_t i = 0; i < n; i++) printf("%d ", arr[i]);
    printf("\n");
}

int main(void) {
    int data[8] = {9, -3, 42, 0, 17, -99, 8, 5};

    printf("before: ");
    print_ints(data, 8);

    qsort(data, 8, sizeof(data[0]), cmp_int);

    printf("after:  ");
    print_ints(data, 8);

    return 0;
}
```

```text
before: 9 -3 42 0 17 -99 8 5
after:  -99 -3 0 5 8 9 17 42
```

The three steps inside every `qsort` comparator:

| Step | Code | Why |
| --- | --- | --- |
| Receive | `const void *pa` | Must match the `qsort` signature exactly |
| Cast | `(const int *)pa` | Restore the real type; keep `const` |
| Compare | `if (*a < *b) return -1; ...` | Compare values, never the pointers |

---

## 2. The Classic Bug: `return a - b` Overflows

This is the most copied comparator bug in C:

```c
/* WRONG: overflows, undefined behaviour on extreme values */
int cmp_int_bad(const void *pa, const void *pb) {
    const int *a = (const int *)pa;
    const int *b = (const int *)pb;
    return *a - *b;   /* BUG */
}
```

It works for small numbers and explodes on large ones. Signed integer overflow is undefined behaviour, and subtraction of two `int`s can overflow even when the *ordering* is perfectly well-defined:

```c
#include <stdio.h>
#include <stdlib.h>
#include <limits.h>

int cmp_int_bad(const void *pa, const void *pb) {
    const int *a = (const int *)pa;
    const int *b = (const int *)pb;
    return *a - *b;   /* BUG: overflow */
}

int cmp_int_good(const void *pa, const void *pb) {
    const int *a = (const int *)pa;
    const int *b = (const int *)pb;
    if (*a < *b) {
        return -1;
    }
    if (*a > *b) {
        return 1;
    }
    return 0;
}

int main(void) {
    int x = INT_MAX;   /*  2147483647 */
    int y = -1;

    printf("bad  cmp(INT_MAX, -1) = %d (says INT_MAX < -1!)\n",
           cmp_int_bad(&x, &y));
    printf("good cmp(INT_MAX, -1) = %d (correct: INT_MAX > -1)\n",
           cmp_int_good(&x, &y));

    return 0;
}
```

```text
bad  cmp(INT_MAX, -1) = -2147483648 (says INT_MAX < -1!)
good cmp(INT_MAX, -1) = 1 (correct: INT_MAX > -1)
```

`INT_MAX - (-1)` is `2147483648`, which does not fit in an `int`. It wraps to `INT_MIN` (negative), so `qsort` believes `INT_MAX` is the *smallest* value. With `-fwrapv` you get silent mis-sorting; without it the optimizer may assume overflow never happens and miscompile the surrounding code.

The rule, no exceptions:

```c
/* RIGHT for int, long, double, strings -- every type */
if (*a < *b) return -1;
if (*a > *b) return 1;
return 0;
```

The same ban applies to `long`, `short`, and pointer differences. For `double`, subtraction additionally breaks on `NaN` and infinity. For strings, subtraction of `char`s is meaningless — use `strcmp`, which already returns `<0` / `0` / `>0`.

---

## 3. Sorting Strings and Structs with `qsort`

Once the cast step clicks, every type is the same pattern. Only the highlighted lines change.

Sorting strings (array of `char *`):

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int cmp_str(const void *pa, const void *pb) {
    /* elements are (char *), so pointers-to-pointers */
    const char *const *a = (const char *const *)pa;
    const char *const *b = (const char *const *)pb;
    return strcmp(*a, *b);
}

int main(void) {
    const char *names[5] = {"Bruno", "Ana", "Carla", "ana", "Beatriz"};
    qsort(names, 5, sizeof(names[0]), cmp_str);
    for (int i = 0; i < 5; i++) printf("%s\n", names[i]);
    return 0;
}
```

```text
Ana
Beatriz
Bruno
Carla
ana
```

Uppercase sorts before lowercase because `'B' (66) < 'a' (97)` in ASCII. That is byte order, not dictionary order — document it if users will see the output.

Sorting structs by a field:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    char name[32];
    int age;
    double score;
} Student;

int cmp_student_by_age(const void *pa, const void *pb) {
    const Student *a = (const Student *)pa;
    const Student *b = (const Student *)pb;
    if (a->age < b->age) {
        return -1;
    }
    if (a->age > b->age) {
        return 1;
    }
    return 0;
}

int cmp_student_by_name(const void *pa, const void *pb) {
    const Student *a = (const Student *)pa;
    const Student *b = (const Student *)pb;
    return strcmp(a->name, b->name);
}

int main(void) {
    Student group[4] = {
        {"Bruno", 22, 8.5},
        {"Ana", 19, 9.2},
        {"Carla", 22, 7.0},
        {"Diego", 20, 9.2},
    };

    qsort(group, 4, sizeof(group[0]), cmp_student_by_age);
    printf("-- by age --\n");
    for (int i = 0; i < 4; i++) {
        printf("%-8s %d %.1f\n", group[i].name, group[i].age, group[i].score);
    }

    qsort(group, 4, sizeof(group[0]), cmp_student_by_name);
    printf("-- by name --\n");
    for (int i = 0; i < 4; i++) {
        printf("%-8s %d %.1f\n", group[i].name, group[i].age, group[i].score);
    }

    return 0;
}
```

```text
-- by age --
Ana      19 9.2
Diego    20 9.2
Bruno    22 8.5
Carla    22 7.0
-- by name --
Ana      19 9.2
Bruno    22 8.5
Carla    22 7.0
Diego    20 9.2
```

Same array, three orderings, zero changes to the sort call except the comparator. That is the strategy pattern in four lines: the *strategy* (which field, which direction) is a function pointer.

---

## 4. `bsearch`: Finding with the Same Comparator

```c
void *bsearch(const void *key, const void *base, size_t nmemb, size_t size,
              int (*compar)(const void *, const void *));
```

`bsearch` binary-searches a **sorted** array. It calls your comparator with `(key, element)` pairs and returns a pointer to a match, or `NULL` when nothing matches.

```c
#include <stdio.h>
#include <stdlib.h>

int cmp_int(const void *pa, const void *pb) {
    const int *a = (const int *)pa;
    const int *b = (const int *)pb;
    if (*a < *b) {
        return -1;
    }
    if (*a > *b) {
        return 1;
    }
    return 0;
}

int main(void) {
    int sorted[8] = {-99, -3, 0, 5, 8, 9, 17, 42};
    int key = 17;
    int *found = (int *)bsearch(&key, sorted, 8, sizeof(sorted[0]), cmp_int);
    if (found != NULL) {
        printf("found %d at index %ld\n", *found, (long)(found - sorted));
    } else {
        printf("%d not found\n", key);
    }
    key = 100;
    found = (int *)bsearch(&key, sorted, 8, sizeof(sorted[0]), cmp_int);
    if (found == NULL) printf("%d not found\n", key);
    return 0;
}
```

```text
found 17 at index 6
100 not found
```

Three `bsearch` rules:

1. **The array must already be sorted** with a consistent comparator. Unsorted input gives arbitrary results, never an error.
2. **The key must be the same type as the elements.** Pass `&key`, not `key`.
3. **Always `NULL`-check the result.** Miss returns `NULL`; dereferencing it segfaults.

| Function | Array must be | Returns | Miss means |
| --- | --- | --- | --- |
| `qsort` | anything | nothing (`void`) | n/a — it sorts |
| `bsearch` | sorted, same ordering | `void *` to match or `NULL` | `NULL` — check before dereference |
| `linear search` (your own loop) | anything | index or `-1` | `-1` — check before indexing |

Struct search works identically — build a key struct with only the searched field filled, and compare on that field.

---

## 5. Callbacks with `void *context`

`qsort` comparators take no extra parameters. Real-world callbacks usually need one more thing: state. The C convention is a `void *context` (sometimes called `userdata`, `closure`, or `arg`) that travels untouched through the generic code and arrives back at your function:

```c
#include <stdio.h>

typedef void (*EventHandler)(int event_id, void *context);

void on_event(int event_id, EventHandler handler, void *context) {
    if (handler != NULL) {
        handler(event_id, context);
    }
}

typedef struct {
    int count;
    const char *label;
} Counter;

void count_events(int event_id, void *context) {
    Counter *c = (Counter *)context;   /* restore the real type */
    c->count++;
    printf("[%s] event %d (total %d)\n", c->label, event_id, c->count);
    (void)event_id;
}

void log_events(int event_id, void *context) {
    const char *prefix = (const char *)context;
    printf("%s: got event %d\n", prefix, event_id);
}

int main(void) {
    Counter clicks = {0, "clicks"};
    const char *tag = "NET";

    on_event(1, count_events, &clicks);
    on_event(2, count_events, &clicks);
    on_event(3, log_events, (void *)tag);

    printf("final count = %d\n", clicks.count);

    return 0;
}
```

```text
[clicks] event 1 (total 1)
[clicks] event 2 (total 2)
NET: got event 3
final count = 2
```

The event processor (`on_event`) never knows what `context` is. It just forwards the pointer. Your handler casts it back to the real type. Same callback, different context, different behaviour — without a single global variable.

Why not use a global instead of `context`?

| Global variable | `void *context` |
| --- | --- |
| One shared state for the whole program | Each registration carries its own state |
| Breaks with two counters / threads | Two `Counter`s, two contexts, no interference |

Minimal strategy example — ascending or descending chosen at runtime:

```c
#include <stdio.h>

typedef int (*Comparator)(int a, int b);

int ascending(int a, int b) {
    if (a < b) return -1;
    if (a > b) return 1;
    return 0;
}

int descending(int a, int b) {
    if (a > b) return -1;
    if (a < b) return 1;
    return 0;
}

typedef struct {
    Comparator cmp;
} SortStrategy;

void sort_with_strategy(int *arr, int n, const SortStrategy *strategy) {
    if (strategy == NULL || strategy->cmp == NULL) {
        fprintf(stderr, "no strategy\n");
        return;
    }
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - 1 - i; j++) {
            if (strategy->cmp(arr[j], arr[j + 1]) > 0) {
                int tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
}

int main(void) {
    int a[5] = {5, 1, 4, 2, 3};
    int b[5] = {5, 1, 4, 2, 3};

    SortStrategy up = {ascending};
    SortStrategy down = {descending};

    sort_with_strategy(a, 5, &up);
    sort_with_strategy(b, 5, &down);

    printf("up:   %d %d %d %d %d\n", a[0], a[1], a[2], a[3], a[4]);
    printf("down: %d %d %d %d %d\n", b[0], b[1], b[2], b[3], b[4]);

    return 0;
}
```

```text
up:   1 2 3 4 5
down: 5 4 3 2 1
```

---

## Common Bugs

**Bug 1 — Subtracting in the comparator (overflow).**

```c
return *a - *b;   /* WRONG: UB on INT_MAX vs negative */
```

```c
/* RIGHT */
if (*a < *b) return -1;
if (*a > *b) return 1;
return 0;
```

Symptom: sorting looks correct on small tests and silently misplaces `INT_MIN`, `INT_MAX`, or large `long` values. Full demonstration in section 2.

**Bug 2 — Forgetting the cast (comparing `void *`).**

Always cast first, then dereference, then compare the values. Comparing the raw addresses sorts by memory location — nonsense that still compiles.

**Bug 3 — Wrong element size or unsorted input to `bsearch`.**

Always pass `sizeof(array[0])`, and always `qsort` before `bsearch` with a consistent comparator. Neither mistake warns — both give wrong answers.

**Bug 4 — Calling the callback without a `NULL` check.**

```c
void emit(int id, EventHandler h, void *ctx) {
    if (h != NULL) {
        h(id, ctx);
    }
}
```

---

## Checklist Before You Run

| Check | Why |
| --- | --- |
| Comparator signature is exactly `(const void *, const void *) -> int` | Any other signature is UB when `qsort` calls it |
| Cast to `const RealType *` first, then dereference | `void *` cannot be dereferenced or compared |
| Return `-1` / `0` / `1` via branches, never `a - b` | Subtraction overflows; overflow is UB |
| `qsort` gets `sizeof(arr[0])`, not `sizeof(ptr)` | Wrong size scrambles the array silently |
| Array sorted before `bsearch`, same ordering | Binary search on unsorted data is meaningless |
| `bsearch` result `NULL`-checked before use | Miss returns `NULL`; dereference segfaults |
| Callback checked `!= NULL` before invocation | Optional callbacks are legitimately `NULL` |
| `context` cast back to exactly the type that was passed in | Wrong cast reads garbage through `void *` |
| Compiles clean with `-std=c11 -Wall -Wextra -pedantic -g` | Signature mismatches hide behind warnings |

---

#### [Previous](./01-part.md) | [Next](./exercises.md)
