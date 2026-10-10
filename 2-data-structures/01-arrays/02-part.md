# Arrays as Parameters and Multidimensional Arrays

Pass an array to a function and it stops being an array. This part picks up where the previous one ended, explains the `sizeof` trap that catches every beginner, and then goes one dimension further.

---

## The Decay Rule, Again

You learned this in the Pointers chapter. It is repeated here because it is the single most important thing about arrays in C:

> In almost every expression, an array name is automatically converted to a pointer to its first element.

So this function:

```c
int sum(int values[], int n);
```

is **exactly** this function:

```c
int sum(int *values, int n);
```

The `[]` in the parameter list is documentation, not syntax. The compiler converts it to a pointer before your function body is even entered.

### Why That Forces You to Pass the Length

An array knows its size, but only through its type, and the type is thrown away at the function boundary. A bare `int *` carries no length — it is just an address. So the callee has no way to find out where the data ends, and you must tell it:

```c
int sum(int values[], int n);
```

Call it as `sum(values, 5)`. Always pass the length. There is no alternative and no exception.

### The `sizeof` Trap

Here is a program that shows the problem in full, including the real warning gcc emits:

```c
#include <stdio.h>

int sum(int values[], int n) {
    int total = 0;
    for (int i = 0; i < n; i++) {
        total += values[i];
    }
    return total;
}

int sum_wrong(int values[]) {
    int total = 0;
    /* BUG: n is not the array size, it is a pointer size */
    int n = (int)(sizeof(values) / sizeof(values[0]));
    for (int i = 0; i < n; i++) {
        total += values[i];
    }
    return total;
}

int sum_readonly(const int values[], int n) {
    int total = 0;
    for (int i = 0; i < n; i++) {
        total += values[i];
    }
    return total;
}

int main(void) {
    int values[5] = {10, 20, 30, 40, 50};

    printf("sum(values, 5)        = %d\n", sum(values, 5));
    printf("sum_readonly(values)  = %d\n", sum_readonly(values, 5));
    printf("sum_wrong(values)     = %d  (only 2 of 5 elements)\n", sum_wrong(values));

    printf("\nsizeof(values) inside main      = %zu bytes (the array)\n", sizeof(values));
    printf("sizeof(values[0]) inside main   = %zu bytes (an int)\n", sizeof(values[0]));

    return 0;
}
```

Compiling it produces this warning:

```text
a2_decay.c: In function 'sum_wrong':
a2_decay.c:14:25: warning: 'sizeof' on array function parameter 'values' will return size of 'int *' [-Wsizeof-array-argument]
   14 |     int n = (int)(sizeof(values) / sizeof(values[0]));
      |                         ^
   11 | int sum_wrong(int values[]) {
      |               ~~~~^~~~~~~~
```

And the program output proves the trap:

```text
sum(values, 5)        = 150
sum_readonly(values)  = 150
sum_wrong(values)     = 30  (only 2 of 5 elements)

sizeof(values) inside main      = 20 bytes (the array)
sizeof(values[0]) inside main   = 4 bytes (an int)
```

Inside `sum_wrong`, `sizeof(values)` is 8 (the pointer), and `sizeof(values[0])` is 4, so `n` becomes 2. The function silently summed the first two elements and called it a day. On a 32-bit machine it would have used `n = 1`. It would never be 5.

The compiler catches this one, which is rare. But note the trap survives whenever the array length and pointer size happen to divide evenly, and it survives in code that looks different. The rule is simple:

> **Never compute an array's length from `sizeof` inside a function that received it as a parameter.** Take the length as a second argument, every time.

### The Rules That Follow

```c
void f(int arr[5]);      /* identical to void f(int *arr) */
void g(int arr[1000]);   /* identical to void g(int *arr) */
void h(int arr[]);       /* identical to void h(int *arr) */
```

The `[5]` and `[1000]` are ignored completely. A function declared `void f(int arr[5])` happily accepts a 3-element array. Nothing is checked, ever.

What decay means in practice:

| You write inside a function | What actually happens |
| --- | --- |
| `sizeof(arr)` | 8 — the pointer |
| `arr[i]` | `*(arr + i)` — fine |
| `&arr[0]` | the parameter itself, already a pointer |
| `&arr` | `int (*)[5]` — no, this is an error |
| passing `arr` to another function | passes the pointer along |
| `arr = something` | error — arrays are not assignable |

You also cannot return an array. A function returning `int *` returns a pointer; the caller has no idea how long the data is, and the array it points at may already be dead.

---

## `const` Parameters

`const` on the parameter documents the contract: *this function will not modify your data.* It also lets the compiler catch a violation inside your own function body.

```c
int sum_readonly(const int values[], int n) { ... }
```

Reads like a promise to the caller, and the compiler enforces it. Two practical benefits:

1. **Proof to a reader** that the data is untouched. Useful when you are not sure whether a helper sorts in place.
2. **Better optimization.** The compiler knows nobody writes through the pointer.

For a function that does modify the array, drop the `const`:

```c
void double_all(int values[], int n);
```

`const` and non-`const` are two different function types for the linker, but they are the same shape, so mixing them up in a declaration is a bug worth knowing about. If you get a *conflicting types* error, this is usually why.

---

## Multidimensional Arrays

A 2D array is an array of arrays:

```c
int grid[3][4] = {
    { 1,  2,  3,  4},
    { 5,  6,  7,  8},
    { 9, 10, 11, 12}
};
```

`grid` has 3 elements; each element is an `int[4]`. You index with `grid[i][j]`, where `i` is the row and `j` is the column.

### The Memory Is One Flat Block

This is the fact that surprises people. There is no array of array headers, no pointers between rows. The whole thing is 12 ints, back to back:

```text
  int grid[3][4] = {{1,2,3,4},{5,6,7,8},{9,10,11,12}};
  48 bytes total, rows stored one after another

  high address
  0x...d30  +---------+   grid[2][0] = 9    offset 32
             |    9    |
  0x...d2c  +---------+   grid[1][3] = 8    offset 28
             |    8    |
  0x...d28  +---------+   grid[1][2] = 7    offset 24
             |    7    |
  0x...d24  +---------+   grid[1][1] = 6    offset 20
             |    6    |
  0x...d20  +---------+   grid[1][0] = 5    offset 16   <-- start of row 1
             |    5    |
  0x...d1c  +---------+   grid[0][3] = 4    offset 12
             |    4    |
  0x...d18  +---------+   grid[0][2] = 3    offset  8
             |    3    |
  0x...d14  +---------+   grid[0][1] = 2    offset  4
             |    2    |
  0x...d10  +---------+   grid[0][0] = 1    offset  0   <-- grid, &grid[0]
             |    1    |
             +---------+
  low address

  written as a grid:        laid out in memory:
    0   1   2   3            1  2  3  4  5  6
  0: 1   2   3   4    ==     7  8  9 10 11 12
  1: 5   6   7   8
  2: 9  10  11  12
```

Verified by printing the addresses:

```text
sizeof(grid)       = 48 bytes
sizeof(grid[0])    = 16 bytes (one row)
sizeof(grid[0][0]) = 4 bytes (one int)

addresses, flattened:
  grid[0][0] = 0x7fffabb86d10   offset =  0   value =  1
  grid[0][1] = 0x7fffabb86d14   offset =  4   value =  2
  grid[0][2] = 0x7fffabb86d18   offset =  8   value =  3
  grid[0][3] = 0x7fffabb86d1c   offset = 12   value =  4
  grid[1][0] = 0x7fffabb86d20   offset = 16   value =  5
  grid[1][1] = 0x7fffabb86d24   offset = 20   value =  6
  grid[1][2] = 0x7fffabb86d28   offset = 24   value =  7
  grid[1][3] = 0x7fffabb86d2c   offset = 28   value =  8
  grid[2][0] = 0x7fffabb86d30   offset = 32   value =  9
  grid[2][1] = 0x7fffabb86d34   offset = 36   value = 10
  grid[2][2] = 0x7fffabb86d38   offset = 40   value = 11
  grid[2][3] = 0x7fffabb86d3c   offset = 44   value = 12
```

Consequence: **a 2D array with `rows * cols` elements is exactly the same memory as a 1D array of `rows * cols` elements.** Nothing in memory records that a row ends after column 3. The compiler computes the offset for you:

```c
&grid[i][j]  ==  (char *)grid + (i * 4 + j) * sizeof(int)
```

which is why the offsets above are perfectly regular.

### `grid[i]` Is a Pointer to a Row

Because the elements of `grid` are `int[4]` arrays, `grid[i]` has type `int *` — a pointer to the first element of row `i`:

```c
grid       = 0x7ffd89f2f360
&grid[0]   = 0x7ffd89f2f360      same address: grid decays
&grid[1]   = 0x7ffd89f2f370      exactly one row (16 bytes) later
```

And `sizeof(grid[0])` is 16, the size of the row — *not* 4 and *not* 8, because `grid[0]` is still an array:

```text
sizeof(grid)    = 48      the whole 2D array
sizeof(grid[0]) = 16      one row: 4 ints
sizeof(row)     =  8      row declared as int (*)[4] -> a pointer
row + 1         = 0x7ffd89f2f370    pointer arithmetic scales by the row size
```

That last line is worth pausing on: a pointer of type `int (*)[4]` advances 16 bytes per `+ 1`, not 4. Pointer arithmetic always scales by the size of what the pointer points at.

### Parameters: Only the First Dimension Can Be Omitted

Since the parameter decays to a pointer, the row count is lost but the **row size must be known**, because the compiler needs it to compute `grid[i][j]`. So:

```c
void print_grid(const int grid[][COLS], int rows, int cols);  /* correct */
void print_grid(int grid[][4], int rows, int cols);            /* also correct */
void print_grid(const int grid[3][4], int rows, int cols);     /* compiles, [3] ignored */
void print_grid(int grid[]);          /* ERROR */
void print_grid(int grid[3][]);       /* ERROR */
```

The two errors:

```text
t10.c:1:12: error: array type has incomplete element type 'int[]'
    1 | void f(int grid[3][]) { (void)grid; }
      |            ^~~~
t10.c:1:12: note: declaration of 'grid' as multidimensional array must have bounds for all dimensions except the first
```

`int grid[]` in a parameter is a pointer to nothing in particular — `int *`. You cannot subscript it twice.

One wrinkle worth knowing: passing a mutable `int[3][4]` to a `const int (*)[4]` parameter is a qualifier mismatch that C only fixed in C23, and gcc with `-pedantic` reports it:

```text
warning: invalid use of pointers to arrays with different qualifiers in ISO C before C23 [-Wpedantic]
```

The clean workarounds are to make the caller's array `const`, or to drop the `const` from the parameter, or to cast explicitly. The simplest one is a `const` array at the call site:

```c
const int grid[ROWS][COLS] = { ... };
print_grid(grid, ROWS, COLS);   /* clean, no warning */
```

### A Complete 2D Example

```c
#include <stdio.h>

#define ROWS 3
#define COLS 4

void print_grid(const int grid[][COLS], int rows, int cols) {
    printf("    ");
    for (int j = 0; j < cols; j++) {
        printf("%5d", j);
    }
    printf("\n");

    for (int i = 0; i < rows; i++) {
        printf("%2d: ", i);
        for (int j = 0; j < cols; j++) {
            printf("%5d", grid[i][j]);
        }
        printf("\n");
    }
}

int grid_sum(const int grid[][COLS], int rows, int cols) {
    int total = 0;
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            total += grid[i][j];
        }
    }
    return total;
}

int main(void) {
    const int grid[ROWS][COLS] = {
        { 1,  2,  3,  4},
        { 5,  6,  7,  8},
        { 9, 10, 11, 12}
    };

    printf("sizeof(grid)       = %zu bytes\n", sizeof(grid));
    printf("sizeof(grid[0])    = %zu bytes (one row)\n", sizeof(grid[0]));
    printf("sizeof(grid[0][0]) = %zu bytes (one int)\n", sizeof(grid[0][0]));

    printf("\n");
    print_grid(grid, ROWS, COLS);
    printf("\ntotal = %d\n", grid_sum(grid, ROWS, COLS));

    return 0;
}
```

```text
sizeof(grid)       = 48 bytes
sizeof(grid[0])    = 16 bytes (one row)
sizeof(grid[0][0]) = 4 bytes (one int)

        0    1    2    3
 0:     1    2    3    4
 1:     5    6    7    8
 2:     9   10   11   12

total = 78
```

The `%5d` is a width specifier: print the number right-aligned in a field five characters wide. That is how you line up columns without any extra work.

### Iterating a 2D Array

Nested loops, outer over rows, inner over columns. This is the shape, and it is the same for 3D arrays with one more level:

```c
for (int i = 0; i < rows; i++) {          /* rows: outer */
    for (int j = 0; j < cols; j++) {      /* cols: inner */
        printf("%5d", grid[i][j]);
    }
    printf("\n");                         /* newline after each row */
}
```

Note that the inner loop must re-declare `j` on every pass of the outer loop. If you hoist `int j;` outside and forget `j = 0`, the second row prints nothing.

### Flattening

Because the memory is contiguous, you can also walk a 2D array with a single loop:

```c
int *flat = &grid[0][0];
for (int k = 0; k < ROWS * COLS; k++) {
    printf("%d ", flat[k]);
}
```

`&grid[0][0]` has type `int *` and points at the first byte of the whole thing. This is not usually what you want in readable code, but it shows there is nothing magic about the two dimensions — they are an indexing convenience.

---

## Passing a 2D Array to a Function That Modifies It

Drop the `const`, and the function can write. Here is a full example with both variants:

```c
#include <stdio.h>

#define ROWS 3
#define COLS 4

void print_grid(const int grid[][COLS], int rows, int cols) {
    printf("    ");
    for (int j = 0; j < cols; j++) {
        printf("%5d", j);
    }
    printf("\n");
    for (int i = 0; i < rows; i++) {
        printf("%2d: ", i);
        for (int j = 0; j < cols; j++) {
            printf("%5d", grid[i][j]);
        }
        printf("\n");
    }
}

void double_rows(int grid[][COLS], int rows, int cols) {
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            grid[i][j] *= 2;
        }
    }
}

int main(void) {
    int grid[ROWS][COLS] = {
        { 1,  2,  3,  4},
        { 5,  6,  7,  8},
        { 9, 10, 11, 12}
    };

    printf("before:\n");
    print_grid((const int (*)[COLS])grid, ROWS, COLS);

    double_rows(grid, ROWS, COLS);
    printf("\nafter doubling every element:\n");
    print_grid((const int (*)[COLS])grid, ROWS, COLS);

    return 0;
}
```

```text
before:
        0    1    2    3
 0:     1    2    3    4
 1:     5    6    7    8
 2:     9   10   11   12

after doubling every element:
        0    1    2    3
 0:     2    4    6    8
 1:    10   12   14   16
 2:    18   20   22   24
```

The caller sees its array changed in place, because there is only one copy — the pointer referred to it.

The cast `(const int (*)[COLS])grid` in the call is there only to silence the pre-C23 qualifier warning described above. If your array is declared `const` in `main`, you do not need it.

---

## Checklist

- An array parameter **is** a pointer. The `[]` and any number inside it are ignored.
- Always pass the length as a second parameter.
- Never use `sizeof` on an array parameter to get the element count.
- Use `const` on parameters that only read.
- 2D arrays are flat in memory: rows are contiguous, one after another.
- `grid[i][j]` is `*(*(grid + i) + j)`.
- `grid[i]` is a pointer to row `i`; `sizeof(grid[0])` is the row size.
- In a parameter, every dimension except the first must be given.
- Iterate with nested `for` loops, outer over rows.

Next part: the patterns and the mistakes that show up in every real program.

---

#### [Previous](./01-part.md) | [Next](./03-part.md)