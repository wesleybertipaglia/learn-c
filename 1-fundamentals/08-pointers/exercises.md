# Exercise List 8 - Pointers

Six exercises that cover the whole chapter: addresses, declaration and dereference, pointers in functions, pointer arithmetic, and character pointers. Compile everything with:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

Zero warnings is the bar. Several exercises ask you to write a bug on purpose — quote the warning in a comment, then fix it in the same file.

---

## Exercise 1 — Addresses, `&` and `%p`

Write a program that prints the value, the address and the size of four variables, then explains the `scanf` bug in a comment.

```c
#include <stdio.h>

int main(void) {
    int x = 42;
    int y = 7;
    char letter = 'A';
    double pi = 3.14;

    printf("x      = %d, address = %p\n", x, (void *)&x);
    printf("y      = %d, address = %p\n", y, (void *)&y);
    printf("letter = %c, address = %p\n", letter, (void *)&letter);
    printf("pi     = %.2f, address = %p\n", pi, (void *)&pi);

    printf("size of int    = %zu bytes\n", sizeof(int));
    printf("size of char   = %zu bytes\n", sizeof(char));
    printf("size of double = %zu bytes\n", sizeof(double));

    return 0;
}
```

```text
x      = 42, address = 0x7ffd977ac9a8
y      = 7, address = 0x7ffd977ac9ac
letter = A, address = 0x7ffd977ac9a7
pi     = 3.14, address = 0x7ffd977ac9b0
size of int    = 4 bytes
size of char   = 1 bytes
size of double = 8 bytes
```

Your exact addresses will differ on every run — that is expected, and part of the exercise. Run it twice and note that the values stay the same while the addresses move.

Requirements:

1. Cast every address to `(void *)` for `%p`. Compiling without the cast must produce no new warnings once you add it.
2. Add a comment answering, in your own words: why does `scanf("%d", &age)` need `&`, and what does `scanf("%d", age)` hand to `scanf` instead?
3. Demonstrate the lvalue rule: add these two lines, commented out, with the compiler error quoted above them:
   ```c
   /* error: lvalue required as unary '&' operand */
   /* int *p = &5; */
   /* int *q = &(x + y); */
   ```
4. Never hardcode an address, store one in a file, or compare addresses across runs.

---

## Exercise 2 — Declare, Initialize, Dereference, Check for `NULL`

Write a program that holds one `int` behind two pointers and proves they alias each other, then proves `NULL` is safe to compare and fatal to follow.

```c
#include <stdio.h>
#include <stddef.h>

int main(void) {
    int x = 42;
    int *p = &x;
    int *q = &x;
    int *empty = NULL;

    printf("x = %d\n", x);
    printf("*p = %d, *q = %d\n", *p, *q);

    *p = 10;
    printf("after *p = 10: x = %d, *q = %d\n", x, *q);

    if (empty == NULL) {
        printf("empty is NULL, will not dereference\n");
    } else {
        printf("*empty = %d\n", *empty);
    }

    printf("sizeof(int) = %zu, sizeof(int *) = %zu\n",
           sizeof(int), sizeof(int *));

    return 0;
}
```

```text
x = 42
*p = 42, *q = 42
after *p = 10: x = 10, *q = 10
empty is NULL, will not dereference
sizeof(int) = 4, sizeof(int *) = 8
```

Requirements:

1. Initialize every pointer at declaration — `= &x` or `= NULL`. No bare `int *p;` anywhere in the file.
2. Show the aliasing effect: writing through `*p` must be visible through `*q` and `x`. Comment one sentence explaining why.
3. Guard every dereference of a pointer that could be `NULL` with an explicit `== NULL` check. The `else` branch above is the pattern.
4. Demonstrate Bug 1 from part 2 on purpose, then fix it. Keep both versions in comments:
   ```c
   /* WRONG: p = x;   -- error: assignment to 'int *' from 'int' */
   /* RIGHT: p = &x; */
   ```
   Paste the real gcc error you get for the wrong version.
5. Answer in a comment: why is `sizeof(int *)` 8 on your machine even though `sizeof(int)` is 4? (One sentence: pointer size follows the architecture, not the pointee type.)

Do not run a `NULL` dereference "to see what happens". The result is a segfault and it teaches you nothing you cannot learn from the `if` above.

---

## Exercise 3 — `swap`, Out-Parameters and "Pointer by Value Is Still by Value"

Write three functions in one file:

```c
void swap_by_address(int *a, int *b);
void zero_out(int *p);
void set_to_null(int *p);
int find_min_max(int a, int b, int *max_out);
```

Full program skeleton:

```c
#include <stdio.h>

void swap_by_address(int *a, int *b) {
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

void zero_out(int *p) {
    *p = 0;
}

void set_to_null(int *p) {
    p = NULL;   /* only rebinds the local copy */
}

int find_min_max(int a, int b, int *max_out) {
    *max_out = (a > b) ? a : b;
    return (a < b) ? a : b;
}

int main(void) {
    int x = 10;
    int y = 20;
    int max;

    printf("before: x = %d, y = %d\n", x, y);
    swap_by_address(&x, &y);
    printf("after swap: x = %d, y = %d\n", x, y);

    zero_out(&x);
    printf("after zero_out(&x): x = %d\n", x);

    int *p = &y;
    set_to_null(p);
    printf("after set_to_null(p): p still points at y ? %s\n",
           (p == &y) ? "yes" : "no");

    printf("min = %d\n", find_min_max(3, 9, &max));
    printf("max = %d\n", max);

    return 0;
}
```

```text
before: x = 10, y = 20
after swap: x = 20, y = 10
after zero_out(&x): x = 0
after set_to_null(p): p still points at y ? yes
min = 3
max = 9
```

Requirements:

1. Call `swap_by_address(&x, &y)` — with `&` on both arguments. Then write the broken `swap_by_value(x, y)` next to it in a comment and state what it prints (nothing changes) and why (it swaps local copies).
2. State the rule in a comment above `set_to_null` / `zero_out`: `p = something` changes only the function's copy; `*p = something` changes the caller's data.
3. Name every out-parameter `*_out` (`max_out`) and document that the caller must pass `&variable`.
4. Add `const` where the function only reads: if you write a `print_all(const int *arr, size_t n)` helper for testing, verify that adding `arr[0] = 1;` inside it fails to compile, and quote that error in a comment.

---

## Exercise 4 — Arrays Are Not Pointers (But They Decay Into Them)

Write a program that walks the same array three ways — subscript, offset from base, advancing pointer — and proves `sizeof` lies inside functions.

```c
#include <stdio.h>

void print_all(const int *arr, size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("  arr[%zu] = %d\n", i, arr[i]);
    }
}

/* WRONG on purpose: sizeof on a parameter is sizeof(pointer) */
void print_all_wrong(const int *arr, size_t n) {
    printf("n = %zu but sizeof(arr) = %zu\n", n, sizeof(arr));
    (void)arr;
}

int sum(const int *arr, size_t n) {
    int total = 0;
    for (size_t i = 0; i < n; i++) {
        total += arr[i];
    }
    return total;
}

int main(void) {
    int arr[5] = {10, 20, 30, 40, 50};
    int *p;

    printf("arr[2] = %d, *(arr + 2) = %d\n", arr[2], *(arr + 2));
    printf("arr == &arr[0] ? %s\n", (arr == &arr[0]) ? "yes" : "no");

    /* option 1: subscript only */
    for (int i = 0; i < 5; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");

    /* option 2: offset from a fresh base */
    p = arr;
    for (int i = 0; i < 5; i++) {
        printf("%d ", *(p + i));
    }
    printf("\n");

    /* option 3: advancing cursor */
    for (p = arr; p < arr + 5; p++) {
        printf("%d ", *p);
    }
    printf("\n");

    print_all(arr, 5);
    print_all_wrong(arr, 5);
    printf("sum = %d\n", sum(arr, 5));

    return 0;
}
```

```text
arr[2] = 30, *(arr + 2) = 30
arr == &arr[0] ? yes
10 20 30 40 50
10 20 30 40 50
10 20 30 40 50
  arr[0] = 10
  arr[1] = 20
  arr[2] = 30
  arr[3] = 40
  arr[4] = 50
n = 5 but sizeof(arr) = 8
sum = 150
```

Requirements:

1. Prove `arr[i] == *(arr + i)` and `arr == &arr[0]` with printed output, not just a comment.
2. Reproduce the already-advanced-pointer bug once, commented out, with one sentence explaining why it reads past the end:
   ```c
   /* BUG: p walked to arr + 5 above; *(p + i) now reads arr[5]..arr[9] */
   /* for (int i = 0; i < 5; i++) { printf("%d ", *(p + i)); } */
   ```
   The fix is one of the three options above: pick one cursor, reset `p = arr` first.
3. Explain `*p++` vs `(*p)++` in a comment with a one-line demo of each. `*p++` moves the pointer; `(*p)++` changes the value and keeps the pointer still.
4. Keep the `print_all_wrong` function and its output (`sizeof(arr) = 8`). Comment why: array parameters decay to `int *`, so `sizeof` sees a pointer, never the 20 bytes of data. Never write `sizeof(arr)/sizeof(arr[0])` inside a function.
5. Stay inside bounds: `arr + 5` (one past the end) may be created and compared but never dereferenced. `arr + 6` may not even be created.

---

## Exercise 5 — Writable Strings vs Read-Only Literals

Write a program that uses `const char *` for everything it only reads and a real `char` array for everything it writes.

```c
#include <stdio.h>
#include <string.h>

void print_label(const char *label) {
    printf("%s\n", label);
}

int main(void) {
    const char *name = "Wesley";
    char buffer[8] = "Wesley";
    char grid[3][20];
    const char *names[] = {"Ana", "Bruno", "Carla"};

    printf("name = %s\n", name);
    printf("literal size = %zu bytes (including \\0)\n", sizeof("Wesley"));

    printf("buffer = %s (%zu bytes, writable)\n", buffer, sizeof(buffer));
    buffer[0] = 'J';
    printf("after buffer[0] = 'J': %s\n", buffer);

    strcpy(grid[0], "first");
    strcpy(grid[1], "second");
    strcpy(grid[2], "third");
    printf("grid[0] = %s\n", grid[0]);
    printf("grid[2] = %s\n", grid[2]);
    printf("sizeof(grid[0]) = %zu, sizeof(grid) = %zu\n",
           sizeof(grid[0]), sizeof(grid));

    for (int i = 0; i < 3; i++) {
        print_label(names[i]);
    }

    return 0;
}
```

```text
name = Wesley
literal size = 7 bytes (including \0)
buffer = Wesley (8 bytes, writable)
after buffer[0] = 'J': Jesley
grid[0] = first
grid[2] = third
sizeof(grid[0]) = 20, sizeof(grid) = 60
Ana
Bruno
Carla
```

Requirements:

1. Declare every string literal as `const char *`, never `char *`. Keep the wrong version in a comment with the fix side by side:
   ```c
   /* WRONG: char *name = "Wesley";       -- promises writability you do not have */
   /* RIGHT: const char *name = "Wesley"; -- read-only, and the compiler enforces it */
   ```
2. Prove the compiler enforces it: add `name[0] = 'J';` once, commented out, with the real error quoted above it (`error: assignment of read-only location '*name'`). The writable version is `buffer[0] = 'J';` on the array copy.
3. Take every string parameter as `const char *`. Demonstrate that it accepts both `buffer` and a literal, and that passing a `const char *` to a `char *` parameter warns about discarding `const`.
4. Use `strcpy` into `grid[row]`, never `grid[row] = "..."`. Quote the error for the assignment version in a comment (`error: assignment to expression with array type`).
5. Comment the memory picture in two lines: `char *names[]` is 3 pointers to read-only literals; `char grid[3][20]` is one writable 60-byte block with fixed 20-byte rows.

Do not execute a write through a literal pointer "to see what happens". It segfaults on most systems and silently corrupts on others — both outcomes are undefined behaviour.

---

## Exercise 6 — Pointer to Pointer: Rebind the Caller's Pointer

Write a function that can do what nothing in Exercise 3 could: change *which* variable the caller's pointer points at.

```c
#include <stdio.h>

void point_at(int **target, int *where) {
    *target = where;
}

void set_value(int **target, int value) {
    **target = value;
}

int main(void) {
    int x = 42;
    int y = 99;
    int *p = &x;
    int **pp = &p;

    printf("*p = %d, **pp = %d\n", *p, **pp);

    *pp = &y;
    printf("after *pp = &y: *p = %d\n", *p);

    set_value(&p, 7);
    printf("after set_value(&p, 7): y = %d, **pp = %d\n", y, **pp);

    point_at(&p, &x);
    printf("after point_at(&p, &x): *p = %d\n", *p);

    return 0;
}
```

```text
*p = 42, **pp = 42
after *pp = &y: *p = 99
after set_value(&p, 7): y = 7, **pp = 7
after point_at(&p, &x): *p = 42
```

Requirements:

1. Read the declarators right to left in a comment: `int *p` is "pointer to int", `int **pp` is "pointer to pointer to int", `int *arr[3]` is "array of 3 pointers to int".
2. Demonstrate the dangling-pointer bug once, commented out, with the explanation:
   ```c
   /* WRONG inside a function taking int **target: */
   /* *target = &value;  -- 'value' dies at return, caller gets a dangling pointer */
   /* RIGHT: **target = value; -- writes through into memory the caller owns */
   ```
3. Explain in two sentences why `set_to_null(int *p)` from Exercise 3 could not do this job and why `point_at(int **target, ...)` can: the first receives a copy of the pointer, the second receives the address of the pointer and can rebind it.
4. Compile with `-Wall -Wextra -pedantic` and confirm there are no incompatible-pointer-type warnings. Any `int *` / `int **` mixup the compiler flags is a real bug here, not style.

---

## Checklist Before You Run

| Check | Why |
| --- | --- |
| Every pointer initialized at declaration (`= &x` or `= NULL`) | Uninitialized pointers hold garbage |
| `&x` where a pointer is expected, `*p` where the value is expected | Mixing them compiles and does the wrong thing |
| `NULL` compared before any dereference that might be invalid | Dereferencing `NULL` segfaults |
| Functions that modify the caller take a pointer; that rebind a pointer take `int **` | By-value copies cannot reach the caller otherwise |
| `const char *` for every string you only read | Turns a runtime segfault into a compile error |
| `sizeof(arr)` only at the array's definition site, never on a parameter | Parameters decayed; `sizeof` sees a pointer |
| Pointer arithmetic stays inside its own array (`arr + 5` comparable, never dereferenced) | Anything further is undefined behaviour |
| Compiles clean with `-Wall -Wextra -pedantic` | Every warning here is a crash or silent corruption |

---

#### [Previous](./05-part.md) | [Next](../09-errors/01-part.md)
