# Pointers and Arrays

An array and a pointer look like different things in C, but they are closer than you would guess. This part explains why `arr[i]` is not really subscripting anything, and why that detail produces a family of real bugs.

---

## `arr[i]` Is `*(arr + i)`

This is not a metaphor. It is what the compiler does.

```text
   what you write          what it means
   -----------              -------------
   arr[i]        ==>        *(arr + i)
   arr           ==>        &arr[0]
```

The bracket syntax was a convenience invented for people who did not have pointers. It is *defined* in terms of them:

```c
#include <stdio.h>

int main(void) {
    int arr[5] = {10, 20, 30, 40, 50};
    int *p = arr;

    printf("arr[0] = %d\n", arr[0]);
    printf("*(arr + 0) = %d\n", *(arr + 0));
    printf("arr[2] = %d\n", arr[2]);
    printf("*(arr + 2) = %d\n", *(arr + 2));
    printf("*p = %d\n", *p);
    printf("arr == &arr[0] ? %s\n", (arr == &arr[0]) ? "yes" : "no");

    printf("addresses and byte offsets:\n");
    for (int i = 0; i < 5; i++) {
        printf("  &arr[%d] = %p   offset = %2ld bytes   value = %d\n",
               i, (void *)&arr[i], (long)((char *)&arr[i] - (char *)&arr),
               arr[i]);
    }

    printf("sizeof(arr) = %zu bytes\n", sizeof(arr));
    printf("sizeof(arr) / sizeof(arr[0]) = %zu elements\n",
           sizeof(arr) / sizeof(arr[0]));

    return 0;
}
```

```text
arr[0] = 10
*(arr + 0) = 10
arr[2] = 30
*(arr + 2) = 30
*p = 10
arr == &arr[0] ? yes
addresses and byte offsets:
  &arr[0] = 0x7ffd13ce2090   offset =  0 bytes   value = 10
  &arr[1] = 0x7ffd13ce2094   offset =  4 bytes   value = 20
  &arr[2] = 0x7ffd13ce2098   offset =  8 bytes   value = 30
  &arr[3] = 0x7ffd13ce209c   offset = 12 bytes   value = 40
  &arr[4] = 0x7ffd13ce20a0   offset = 16 bytes   value = 50
sizeof(arr) = 20 bytes
sizeof(arr) / sizeof(arr[0]) = 5 elements
```

### Why "Decay" Is the Word Used

An array **cannot** be copied, passed to a function, or returned. In almost every expression context, the array name is automatically converted to a pointer to its first element. That conversion is called **decay**, and it is why:

```c
int *p = arr;    /* the same address as &arr[0] */
printf("%p", arr);   /* prints &arr[0] */
```

The important exception: decay does **not** happen on `sizeof`, on `&`, or when the array is the operand of `sizeof` or is used to initialize another array. That exception is what makes `sizeof(arr) == 20` above correct.

---

## The Layout of an `int[5]`

```text
  int arr[5] = {10, 20, 30, 40, 50};

  high address
  +---------+   &arr[4] = 0x...a0    offset 16
  |   50    |
  +---------+   &arr[3] = 0x...9c    offset 12
  |   40    |
  +---------+   &arr[2] = 0x...98    offset  8
  |   30    |
  +---------+   &arr[1] = 0x...94    offset  4
  |   20    |
  +---------+   &arr[0] = 0x...90    offset  0   <-- arr, p
  |   10    |
  +---------+
  low address
```

Each element is exactly 4 bytes and there are no gaps. So `arr + 3` is not "element 3" — it is **the address 12 bytes past `arr`**, and dereferencing it lands on `40`.

---

## Pointer Arithmetic Moves by Whole Elements

This is the part that trips everyone up. `p + 1` does **not** add 1 byte. It adds `sizeof(*p)` bytes:

```text
  p + 1  ==  (char *)p + sizeof(int)   ==  p + 4 bytes
  p + n  ==  (char *)p + n * sizeof(int)

  char *c;   c + 1  ==  c + 1 byte      (sizeof(char) == 1)
  double *d; d + 1  ==  d + 8 bytes     (sizeof(double) == 8)
```

The compiler knows the type of the pointer and scales the addition for you. That is the whole feature. It is also why adding pointers of different types is an error:

```c
int    *p = arr;
double *q = NULL;

p = q;    /* error: incompatible pointer types */
p + q;    /* error: invalid operands to binary + */
```

And comparing addresses is only meaningful within one array:

```c
if (p < q) { }   /* only meaningful if p and q point into the same array */
```

---

## Moving a Pointer Around

```c
#include <stdio.h>

int main(void) {
    int arr[5] = {10, 20, 30, 40, 50};
    int vals[5] = {10, 20, 30, 40, 50};
    int *p = arr;

    printf("start:        p = %p, *p = %d\n", (void *)p, *p);
    p++;
    printf("after p++:    p = %p, *p = %d\n", (void *)p, *p);
    ++p;
    printf("after ++p:    p = %p, *p = %d\n", (void *)p, *p);
    p += 2;
    printf("after p += 2: p = %p, *p = %d\n", (void *)p, *p);
    p--;
    printf("after p--:    p = %p, *p = %d\n", (void *)p, *p);

    p = vals + 3;
    printf("\n(*p)++  increments the VALUE at p; p does not move\n");
    printf("before: *p = %d\n", *p);
    (*p)++;
    printf("after:  *p = %d, p is still at %p\n", *p, (void *)p);

    printf("\n*p++    uses the value at p, then moves p by one int\n");
    printf("before: *p = %d\n", *p);
    printf("used:   %d\n", *p++);
    printf("after:  p points at %d\n", *p);

    printf("\nwalking the array with pointer arithmetic:\n");
    for (p = arr; p < arr + 5; p++) {
        printf("  *p = %d, address = %p\n", *p, (void *)p);
    }

    printf("\nsame thing with subscripts:\n");
    for (int i = 0; i < 5; i++) {
        printf("  arr[%d] = %d\n", i, arr[i]);
    }

    return 0;
}
```

```text
start:        p = 0x7ffd209f7740, *p = 10
after p++:    p = 0x7ffd209f7744, *p = 20
after ++p:    p = 0x7ffd209f7748, *p = 30
after p += 2: p = 0x7ffd209f7750, *p = 50
after p--:    p = 0x7ffd209f774c, *p = 40

(*p)++  increments the VALUE at p; p does not move
before: *p = 40
after:  *p = 41, p is still at 0x7ffd209f776c

*p++    uses the value at p, then moves p by one int
before: *p = 41
used:   41
after:  p points at 50

walking the array with pointer arithmetic:
  *p = 10, address = 0x7ffd209f7740
  *p = 20, address = 0x7ffd209f7744
  *p = 30, address = 0x7ffd209f7748
  *p = 40, address = 0x7ffd209f774c
  *p = 50, address = 0x7ffd209f7750

same thing with subscripts:
  arr[0] = 10
  arr[1] = 20
  arr[2] = 30
  arr[3] = 40
  arr[4] = 50
```

### `p++`, `++p`, `*p++`, `(*p)++`

| Expression | Moves the pointer? | Changes the value it points at? | Value of the expression |
| --- | --- | --- | --- |
| `p++` | yes, after | no | the **old** pointer |
| `++p` | yes, before | no | the **new** pointer |
| `*p++` | yes, after | no | the **old** value at `p` |
| `(*p)++` | **no** | yes, +1 | the **new** value at `p` |

`p++` and `++p` differ only in whether the side effect happens before or after the value of the expression is produced. As statements they are identical. `*p++` is read as `*(p++)` — postfix `++` binds tighter than `*` — so it means "give me what is here, then step forward". `(*p)++` needs the parentheses or it means something entirely different: it increments the number stored in the array.

```text
  *p++          (*p)++

  read *p,       change *p,
  then p += 1    p stays put
```

Confusing? Yes. Common? Extremely. Prefer `p = p + 1;` when the difference matters.

---

## The Classic Bug: An Already-Advanced Pointer

Now we can explain the mistake that appears in almost every C course: mixing a moving cursor with an offset.

**Wrong.** Here `p` is walked to the *end* of the array, and then the loop adds offsets to it:

```c
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;
int *end = arr + 5;
int i;

/* walk p to the end of the array */
while (p < end) {
    p++;
}

for (i = 0; i < 5; i++) {
    printf("%d ", *(p + i));   /* BUG: reads arr[5] .. arr[9], past the end */
}
```

**Right.** Keep one cursor. Either let `p` walk, or let `i` walk — not both:

```c
/* option 1: subscript only */
for (i = 0; i < 5; i++) {
    printf("%d ", arr[i]);
}

/* option 2: pointer only, reset to the beginning first */
p = arr;
for (i = 0; i < 5; i++) {
    printf("%d ", *(p + i));
}

/* option 3: pointer only, advancing instead of offsetting */
for (i = 0, p = arr; i < 5; i++, p++) {
    printf("%d ", *p);
}
```

`base + offset` only makes sense when `base` points at the **first element**. Once `p` has been advanced, the pointer *is* the position — adding to it moves away from the array, not into it.

The same mistake written differently:

```c
int *p = arr;
print(*p);      /* prints 10 */
p++;            /* p now points at arr[1] */
print(*(p + 0)); /* prints 20, not 10 — offset 0 counts from here */
```

If `p` points at `arr[2]` and you write `*(p + i)` for `i = 0..4`, you read `arr[2]`, `arr[3]`, `arr[4]`, and then **two elements past the end of the array**. Nothing there is a valid object any more. The bug lives in `p`'s value *before* the loop, which is why it is invisible when you read the loop alone.

The same mistake written with a `for` loop that walks the array and then a second loop that offsets from wherever it stopped:

```c
p = arr;
for (i = 0; i < 5; i++) {
    printf("%d ", *p);
    p++;                 /* p ends up one past the last element */
}

for (i = 0; i < 5; i++) {
    printf("%d ", *(p + i));  /* WRONG: offsets counted from past-the-end */
}
```

**Bug and fix side by side:**

```c
/* WRONG: two moving cursors, offset from a moving base */
p = arr;
while (*p) { printf("%d ", *p); p++; }        /* p is at the end now */
for (i = 0; i < 5; i++) { printf("%d ", *(p + i)); }

/* RIGHT: one cursor. Choose subscript or pointer, not both */
for (i = 0; i < 5; i++) { printf("%d ", arr[i]); }     /* subscript loop only */

for (p = arr; p < arr + 5; p++) { printf("%d ", *p); } /* pointer loop only */
```

---

## Pointer Arithmetic Has Rules, Not Just Syntax

Adding 1 to a pointer that points into a 5-element array, landing outside it, is undefined behaviour. Not "probably works", not "works until Tuesday" — undefined. The compiler is entitled to assume you did not do it and to optimize accordingly.

```text
  &arr[0] ──▶  &arr[1] ──▶  &arr[2] ──▶  &arr[3] ──▶  &arr[4] ──▶ ???
     ▲            ▲            ▲            ▲            ▲          ▲
     │            │            │            │            │          │
     ok           ok           ok           ok           ok       NOT OK
     p+0          p+1          p+2          p+3          p+4      p+5
```

What is and is not allowed:

```c
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr + 2;      /* legal: points at arr[2]          */
int *q = arr + 5;      /* legal: one past the end          */
                       /* comparing p < q is legal         */
int *r = arr + 6;      /* UNDEFINED: past one-past-the-end */
arr[7] = 0;            /* UNDEFINED: out of bounds write  */
*(p + 10) = 0;         /* UNDEFINED                        */
```

The "one past the end" pointer is the reason C's `for` loops work so conveniently:

```c
int *end = arr + 5;        /* legal to create */
for (p = arr; p != end; p++) { ... }   /* legal to compare */
```

But you may not dereference it. `p != end` is fine; `*p` when `p == end` is not.

And a rule worth stating loudly: **pointer arithmetic between two different arrays is meaningless**, even if both are `int *`. `arr_b[0] - arr_a[0]` compiles, runs, and produces a number that means nothing.

---

## The `sizeof` Trap

Because array parameters decay to pointers, `sizeof` on a parameter gives you the size of a **pointer**, not of the array:

```c
#include <stdio.h>

void print_all(const int *arr, size_t n) {
    printf("inside print_all, the caller passed %zu elements:\n", n);
    for (size_t i = 0; i < n; i++) {
        printf("  arr[%zu] = %d\n", i, arr[i]);
    }
}

void print_all_wrong(int arr[], size_t n) {
    printf("inside print_all_wrong, n = %zu but sizeof(arr) = %zu\n",
           n, sizeof(arr));
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

    print_all(arr, 5);
    print_all_wrong(arr, 5);

    printf("sum = %d\n", sum(arr, 5));

    return 0;
}
```

Note how `int arr[]` in the parameter list is *exactly* `int *arr` — the `[]` is documentation, nothing more. gcc warns about it:

```text
p4_decay.c:13:21: warning: 'sizeof' on array function parameter 'arr' will
                         return size of 'int *' [-Wsizeof-array-argument]
   13 |            n, sizeof(arr));
      |                     ^
   11 | void print_all_wrong(int arr[], size_t n) {
      |                      ~~~~~~~~~
```

And the program output proves the trap:

```text
inside print_all, the caller passed 5 elements:
  arr[0] = 10
  arr[1] = 20
  arr[2] = 30
  arr[3] = 40
  arr[4] = 50
inside print_all_wrong, n = 5 but sizeof(arr) = 8
sum = 150
```

`n = 5` but `sizeof(arr) = 8` — 8 bytes is the pointer, not the 20 bytes of data. A loop written as `for (i = 0; i < sizeof(arr)/sizeof(arr[0]); i++)` inside a function reads 1 element on a 64-bit machine instead of 5. Use `-Wsizeof-array-argument` (part of `-Wall`) and never write that idiom in a parameter.

### Consequences of Decay

Because decay happens everywhere except `sizeof`, `&`, and array initializers:

```c
int arr[5];
int *p1 = arr;             /* fine: decay to &arr[0]        */
size_t n = sizeof(arr);    /* fine: 20, no decay             */
int *p2 = &arr;            /* type int (*)[5], NOT int *    */
                          /* int *p3 = &arr;  -> warning    */
```

`&arr` gives the address of the *whole array*, whose type is `int (*)[5]`. Assigning it to `int *` is a mismatch that gcc flags. Do not use `&arr` unless you specifically want to talk about arrays as objects (as in multidimensional arrays).

---

## Array vs Pointer

| | Array | Pointer |
| --- | --- | --- |
| What is stored | N elements of data | one address |
| Size | `N * sizeof(T)` | one machine word (8 bytes on 64-bit) |
| Can be copied? | no — decays | yes |
| Can be passed/returned? | no — decays | yes |
| Knows its length? | yes (`sizeof arr / sizeof arr[0]`) | no |
| Arithmetic | no, unless it decays | yes, in element units |
| Dereference with `*` | no | yes |
| Can be `NULL`? | no | yes |
| `&x` on the name | gives `T (*)[N]` | gives `T **` |

Similarities worth remembering:

- `arr[i]` and `*(arr + i)` are the same thing.
- `arr` in an expression is `&arr[0]`.
- An array's first element and a pointer to it are interchangeable in most code.
- Both are addresses under the hood; the array is just the bytes that live there.

The decisive difference: **an array owns memory, a pointer merely refers to it.** When an array goes out of scope, its memory is gone and any pointer to it is dangling. When a pointer goes out of scope, the memory it pointed at is untouched — unless you freed it, and then you are the one responsible.

---

## Which One Should You Write?

For a beginner, subscripts. Always. They are checked against the array bounds in your head, and they do not require tracking a position separately.

Reach for pointer arithmetic when:

- You are writing a low-level loop over a buffer and need speed (rare at the start; the compiler optimizes `arr[i]` into pointer arithmetic anyway).
- You are writing functions like `strlen`, `memcpy`, or anything that processes a stream of bytes without knowing its length.
- You are inside the standard library.

```c
/* clearer, and exactly as fast after optimization */
for (int i = 0; i < n; i++) {
    printf("%d\n", arr[i]);
}
```

---

#### [Previous](./03-part.md) | [Next](./05-part.md)