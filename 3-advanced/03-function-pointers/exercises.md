# Exercise List 3 - Function Pointers

Six exercises that cover the whole chapter: `qsort` comparators, struct sorting through callbacks, dispatch tables, `map`/`filter` with `void *context`, menu dispatch without a giant `switch`, and `bsearch` with a correct comparator. Compile everything with:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

Zero warnings is the bar. A warning on a function-pointer line is a wrong signature, and calling through a wrong signature is undefined behaviour — even if it "seems to work".

Global rules for all six exercises:

- `typedef` every function-pointer type. No raw `int (*)(...)` in parameters or arrays.
- Initialize every function pointer (`= func` or `= NULL`) and check `!= NULL` before calling.
- Comparators return `-1` / `0` / `1` with branches. `return a - b` is forbidden (overflow).
- `qsort`/`bsearch` comparators use `const void *` with a cast to `const RealType *`.

---

## Exercise 1 — Int Comparator + `qsort`

Write a safe `int` comparator and sort one array ascending and a copy descending.

```c
#include <stdio.h>
#include <stdlib.h>

typedef int (*QsortCmp)(const void *a, const void *b);

int cmp_asc(const void *pa, const void *pb) {
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

int cmp_desc(const void *pa, const void *pb) {
    const int *a = (const int *)pa;
    const int *b = (const int *)pb;
    if (*a > *b) {
        return -1;
    }
    if (*a < *b) {
        return 1;
    }
    return 0;
}

void print_ints(const int *arr, size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}

int main(void) {
    int a[7] = {5, -2, 9, 0, 3, -8, 1};
    int b[7] = {5, -2, 9, 0, 3, -8, 1};

    qsort(a, 7, sizeof(a[0]), cmp_asc);
    qsort(b, 7, sizeof(b[0]), cmp_desc);

    printf("asc:  ");
    print_ints(a, 7);
    printf("desc: ");
    print_ints(b, 7);

    return 0;
}
```

```text
asc:  -8 -2 0 1 3 5 9
desc: 9 5 3 1 0 -2 -8
```

Requirements:

1. Keep the exact signature `(const void *, const void *) -> int`. Any deviation is undefined behaviour when `qsort` calls back.
2. Cast to `const int *`, dereference, then branch. Never compare the `void *` parameters directly and never `return *a - *b`.
3. Prove the overflow bug in a comment: state what `cmp_bad(&INT_MAX, &-1)` returns with subtraction (`INT_MIN`, negative — wrong) versus the branch version (`1` — right). Include `<limits.h>` reasoning, not a live overflow.
4. Pass `sizeof(a[0])` as the element size. Add a comment explaining why `sizeof(int *)` would scramble an `int` array here.
5. `typedef` the comparator type (`QsortCmp`) and use it if you store the comparator in a variable.

---

## Exercise 2 — Sort Structs by Field via Callbacks

Sort the same `Student` array by age, by name, and by score descending — three comparators, three `qsort` calls, one array type.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    char name[32];
    int age;
    double score;
} Student;

int cmp_by_age(const void *pa, const void *pb) {
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

int cmp_by_name(const void *pa, const void *pb) {
    const Student *a = (const Student *)pa;
    const Student *b = (const Student *)pb;
    return strcmp(a->name, b->name);
}

int cmp_by_score_desc(const void *pa, const void *pb) {
    const Student *a = (const Student *)pa;
    const Student *b = (const Student *)pb;
    if (a->score > b->score) {
        return -1;
    }
    if (a->score < b->score) {
        return 1;
    }
    return 0;
}

void print_group(const Student *g, int n) {
    for (int i = 0; i < n; i++) {
        printf("%-8s age %d score %.1f\n", g[i].name, g[i].age, g[i].score);
    }
}

int main(void) {
    Student group[4] = {
        {"Bruno", 22, 8.5},
        {"Ana", 19, 9.2},
        {"Carla", 22, 7.0},
        {"Diego", 20, 9.2},
    };

    qsort(group, 4, sizeof(group[0]), cmp_by_age);
    printf("-- by age --\n");
    print_group(group, 4);

    qsort(group, 4, sizeof(group[0]), cmp_by_name);
    printf("-- by name --\n");
    print_group(group, 4);

    qsort(group, 4, sizeof(group[0]), cmp_by_score_desc);
    printf("-- by score desc --\n");
    print_group(group, 4);

    return 0;
}
```

```text
-- by age --
Ana      age 19 score 9.2
Diego    age 20 score 9.2
Bruno    age 22 score 8.5
Carla    age 22 score 7.0
-- by name --
Ana      age 19 score 9.2
Bruno    age 22 score 8.5
Carla    age 22 score 7.0
Diego    age 20 score 9.2
-- by score desc --
Ana      age 19 score 9.2
Diego    age 20 score 9.2
Bruno    age 22 score 8.5
Carla    age 22 score 7.0
```

Requirements:

1. One comparator per field. The age comparator branches on `int`; the name comparator delegates to `strcmp`; the score comparator branches on `double` — no subtraction anywhere.
2. Comment why `return (int)(a->score - b->score);` would be wrong twice: truncation toward zero *and* precision loss on large `double` values.
3. Re-sort the same array three times and print after each pass, proving the strategy pattern: the data and the sort call stay fixed, only the callback changes.
4. Keep `const` in every cast (`const Student *`). The comparator must not modify elements.
5. Note in a comment that `qsort` is not stable: `Bruno`/`Carla` (both age 22) and `Ana`/`Diego` (both 9.2) may come out in either relative order.

---

## Exercise 3 — Calculator with Array of Operations

Build a four-operation calculator as a dispatch table. No `switch` on the operator.

```c
#include <stdio>

typedef int (*BinaryOp)(int a, int b);

int op_add(int a, int b) {
    return a + b;
}

int op_sub(int a, int b) {
    return a - b;
}

int op_mul(int a, int b) {
    return a * b;
}

int op_div(int a, int b) {
    if (b == 0) {
        fprintf(stderr, "error: division by zero\n");
        return 0;
    }
    return a / b;
}

int main(void) {
    BinaryOp ops[4] = {op_add, op_sub, op_mul, op_div};
    const char *names[4] = {"add", "sub", "mul", "div"};
    const char symbols[4] = {'+', '-', '*', '/'};

    int x = 20;
    int y = 4;

    for (int i = 0; i < 4; i++) {
        if (ops[i] == NULL) {
            fprintf(stderr, "missing op %d\n", i);
            continue;
        }
        printf("%s: %d %c %d = %d\n", names[i], x, symbols[i], y, ops[i](x, y));
    }

    /* simulate one user choice */
    int choice = 3;
    if (choice >= 0 && choice < 4 && ops[choice] != NULL) {
        printf("choice %d -> %d\n", choice, ops[choice](x, y));
    } else {
        fprintf(stderr, "bad choice %d\n", choice);
    }

    /* division by zero goes through the table too */
    printf("div by zero: %d\n", ops[3](10, 0));

    return 0;
}
```

```text
add: 20 + 4 = 24
sub: 20 - 4 = 16
mul: 20 * 4 = 80
div: 20 / 4 = 5
choice 3 -> 5
error: division by zero
div by zero: 0
```

Requirements:

1. Declare the table as `BinaryOp ops[4]`, not raw syntax. Every entry must match `(int, int) -> int` exactly.
2. Bounds-check `choice` (`0 <= choice < 4`) *and* `NULL`-check `ops[choice]` before calling. Demonstrate the rejection path with an out-of-range choice in a comment or second run.
3. Handle division by zero inside `op_div` (print to `stderr`, return `0`). The table caller must not need a special case.
4. Keep a parallel `names[]`/`symbols[]` array and comment why: a function pointer carries no name, so the table needs metadata alongside.
5. Write the forbidden `switch` version in a comment (four cases) with one sentence explaining what the table does better: adding an operation is one table entry, not a new case plus a new branch to test.

---

## Exercise 4 — `map` / `filter` with Callback + Context

Write two generic higher-order functions driven by callbacks that each receive a `void *context`:

```c
void map(int *arr, int n, int (*fn)(int value, void *ctx), void *ctx);
int filter(const int *in, int n, int *out, int (*keep)(int value, void *ctx), void *ctx);
```

Full program skeleton:

```c
#include <stdio.h>

typedef int (*MapFn)(int value, void *ctx);
typedef int (*FilterFn)(int value, void *ctx);

void map(int *arr, int n, MapFn fn, void *ctx) {
    if (fn == NULL) {
        return;
    }
    for (int i = 0; i < n; i++) {
        arr[i] = fn(arr[i], ctx);
    }
}

int filter(const int *in, int n, int *out, FilterFn keep, void *ctx) {
    int count = 0;
    if (keep == NULL) {
        return 0;
    }
    for (int i = 0; i < n; i++) {
        if (keep(in[i], ctx)) {
            out[count] = in[i];
            count++;
        }
    }
    return count;
}

int scale(int value, void *ctx) {
    int *factor = (int *)ctx;
    return value * (*factor);
}

int add_offset(int value, void *ctx) {
    int *off = (int *)ctx;
    return value + (*off);
}

int above_threshold(int value, void *ctx) {
    int *limit = (int *)ctx;
    return value > (*limit);
}

int main(void) {
    int data[6] = {1, 2, 3, 4, 5, 6};
    int factor = 10;
    int limit = 25;
    int out[6] = {0};

    map(data, 6, scale, &factor);
    printf("scaled: %d %d %d %d %d %d\n",
           data[0], data[1], data[2], data[3], data[4], data[5]);

    int kept = filter(data, 6, out, above_threshold, &limit);
    printf("kept %d:", kept);
    for (int i = 0; i < kept; i++) {
        printf(" %d", out[i]);
    }
    printf("\n");

    return 0;
}
```

```text
scaled: 10 20 30 40 50 60
kept 4: 30 40 50 60
```

Requirements:

1. `map` transforms in place; `filter` writes matches into `out` and returns the count. Both take the callback *and* a `void *ctx`, and both return immediately (doing nothing) when the callback is `NULL`.
2. Each callback casts `ctx` back to exactly the type that was passed (`int *` here). Comment what goes wrong if the caller passes an `int` but the callback casts to `double *`: garbage values, no warning.
3. Demonstrate two different contexts with the same callback shape: run `scale` with `factor = 10` and then `add_offset` with an offset, printing both results.
4. Prove no globals: `factor` and `limit` must be locals in `main`, passed by address. State in a comment why a global `factor` would break the moment two scalings are in flight.
5. Guard the `filter` output bounds: `out` must hold at least `n` ints, and `kept <= n` always. Comment that contract above `filter`.

---

## Exercise 5 — Menu Dispatch Without a Giant `switch`

Replace a five-way `switch` menu with a table of `void (*)(void)` actions plus a parallel names array.

```c
#include <stdio.h>

typedef void (*MenuAction)(void);

void action_new(void) {
    printf("new file created\n");
}

void action_open(void) {
    printf("file opened\n");
}

void action_save(void) {
    printf("file saved\n");
}

void action_quit(void) {
    printf("bye!\n");
}

void action_unknown(void) {
    printf("unknown option\n");
}

int main(void) {
    MenuAction actions[4] = {action_new, action_open, action_save, action_quit};
    const char *names[4] = {"new", "open", "save", "quit"};

    /* simulated user input: 0..4, where 4 is invalid */
    int inputs[5] = {0, 2, 3, 1, 9};

    for (int k = 0; k < 5; k++) {
        int choice = inputs[k];
        printf("> choice %d: ", choice);
        if (choice >= 0 && choice < 4 && actions[choice] != NULL) {
            printf("[%s] ", names[choice]);
            actions[choice]();
        } else {
            action_unknown();
        }
    }

    return 0;
}
```

```text
> choice 0: [new] new file created
> choice 2: [save] file saved
> choice 3: [quit] bye!
> choice 1: [open] file opened
> choice 9: unknown option
```

Requirements:

1. Table type is `MenuAction` (`void (*)(void)`). No action takes input or returns output — all context comes from outside, keeping the dispatch line to `actions[choice]()`.
2. Validate every choice: range-check *and* `NULL`-check before calling. Route misses to `action_unknown`, never to a raw `else printf`.
3. Include the `names[]` array and print it on each dispatch. Comment that the table cannot print itself, so names live next to pointers.
4. Paste the replaced `switch` (5 cases) in a comment and count its lines versus the two-line dispatch. State the scaling argument: the 20th menu item is one more table row, not one more `case` plus fallthrough risk.
5. Initialize the table fully (`= {action_new, ...}`) or to `NULL` for unassigned slots. No uninitialized `MenuAction` entries; calling one is a wild jump.

---

## Exercise 6 — `bsearch` with a Correct Comparator

Sort an array with `qsort`, then find keys with `bsearch` using a shared, overflow-safe comparator. Handle the miss (`NULL`) every time.

```c
#include <stdio.h>
#include <stdlib.h>

typedef int (*QsortCmp)(const void *a, const void *b);

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
    int data[8] = {9, -3, 42, 0, 17, -99, 8, 5};

    qsort(data, 8, sizeof(data[0]), cmp_int);

    printf("sorted:");
    for (int i = 0; i < 8; i++) {
        printf(" %d", data[i]);
    }
    printf("\n");

    int keys[3] = {17, -99, 100};
    for (int i = 0; i < 3; i++) {
        int *hit = (int *)bsearch(&keys[i], data, 8,
                                  sizeof(data[0]), cmp_int);
        if (hit != NULL) {
            printf("key %d found at index %ld\n",
                   keys[i], (long)(hit - data));
        } else {
            printf("key %d not found\n", keys[i]);
        }
    }

    return 0;
}
```

```text
sorted: -99 -3 0 5 8 9 17 42
key 17 found at index 6
key -99 found at index 0
key 100 not found
```

Requirements:

1. Use one shared comparator for both `qsort` and `bsearch`. Comment why two inconsistent comparators (e.g. ascending sort + descending search) make `bsearch` return arbitrary results.
2. Sort before searching — prove it by printing the sorted array first. Searching the unsorted original is the planted bug: describe its symptom (random hits/misses, no error) in a comment.
3. Pass the key by address (`&keys[i]`) and cast the `bsearch` return from `void *` to `int *`. Check `!= NULL` before dereferencing or doing `hit - data`, every iteration.
4. Keep the comparator subtraction-free with `const int *` casts. Quote the forbidden line in a comment: `/* FORBIDDEN: return *a - *b; -- overflows on INT_MAX vs -1 */`.
5. Compute the index as `(long)(hit - data)` only on a hit, and explain in one comment line why pointer subtraction gives the index (elements are contiguous `int`s, difference is in elements, not bytes).

---

## Checklist Before You Run

| Check | Why |
| --- | --- |
| Every function-pointer type has a `typedef` | Call sites stay readable; mismatches surface at compile time |
| Every function pointer initialized; `NULL` checked before call | Unchecked indirect calls segfault |
| Comparators branch to `-1` / `0` / `1`; no `a - b` | Subtraction overflows and mis-sorts extremes |
| `qsort`/`bsearch` comparators take `const void *` and cast to `const T *` | Any other signature is undefined behaviour |
| `qsort`/`bsearch` get `sizeof(arr[0])` | Wrong size corrupts the array silently |
| `bsearch` only on a sorted array, same comparator | Unsorted input gives arbitrary answers, never an error |
| `bsearch` return checked for `NULL` | Miss is `NULL`; dereference crashes |
| Dispatch/menu index range-checked | Out-of-bounds table call jumps to garbage |
| Compiles clean with `-std=c11 -Wall -Wextra -pedantic -g` | Every warning here is a crash or UB |

---

#### [Previous](./02-part.md) | [Next](../04-file-io/01-part.md)
