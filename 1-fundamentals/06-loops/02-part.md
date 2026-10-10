# The `for` Loop

`for` is the loop you will use every day. It does exactly what `while` does — the initialization, condition and update are just written in one place instead of scattered across three lines.

```c
for (init; condition; update) {
    body;
}
```

Read it as a sentence: *initialize once, then while the condition holds, run the body and apply the update.*

---

## Anatomy of the Header

```c
for (int i = 0; i < 10; i++) {
/*   ^^^^^^^^^^^^   ^^^^^^^   ^^^
     initialize     condition  update */
}
```

The three parts are separated by semicolons and run in this order on every single entry into the loop:

1. `int i = 0` — runs **once**, before anything else.
2. `i < 10` — tested **before every iteration**. False means exit.
3. `i++` — runs **after every completed iteration**, never after the exit test.

That last detail catches people: `i` ends the loop at `10`, and the update runs after the body, so the last body execution is the one where `i == 9`.

---

## Equivalent to `while`

These two programs print exactly the same thing:

```c
int i = 0;              /* init    */
while (i < 5) {         /* cond    */
    printf("%d ", i);
    i = i + 1;          /* update  */
}
```

```c
for (i = 0; i < 5; i++) {
    printf("%d ", i);
}
```

```text
0 1 2 3 4
```

`for` does not add any capability. It puts the three parts next to each other so a reader can see the loop's shape in one line.

---

## Full Example

```c
#include <stdio.h>

int main(void) {
    int i;
    int count = 0;

    for (i = 0; i < 5; i++) {
        count = i + 1;
    }
    printf("last value printed: %d\n", count);

    for (i = 0; i < 5; i++) {
        printf("%d ", i);
    }
    printf("\ni after the loop = %d\n", i);

    for (i = 10; i > 0; i--) {
        printf("%d ", i);
    }
    printf("\n");

    return 0;
}
```

```text
last value printed: 5
0 1 2 3 4
i after the loop = 5
10 9 8 7 6 5 4 3 2 1
```

`i` is declared outside the loops, so it is still readable after them. That is the traditional way to write C: declare once, use as needed.

---

## Variants

### `for (;;)` — the Loop With No Header

All three parts can be left empty:

```c
for (;;) {
    /* same as while (1) */
}
```

The condition is then treated as always true, so the loop only ends through `break` or `return`. Use it when the exit condition cannot be expressed up front — reading input until end of file, or waiting for a condition that several different pieces of code can set.

### Iterating an `int`

The common case. Note both directions:

```c
for (int i = 0; i < 5; i++)      { /* forwards  */ }
for (int i = 4; i >= 0; i--)     { /* backwards */ }
```

### Iterating a `double`

Nothing in `for` is limited to integers:

```c
#include <stdio.h>

int main(void) {
    for (double x = 0.0; x < 1.0; x += 0.25) {
        printf("x = %.2f\n", x);
    }

    return 0;
}
```

```text
x = 0.00
x = 0.25
x = 0.50
x = 0.75
```

This works, but accumulating floating point is a different story: adding `0.1` a hundred times does not give exactly `10.0`. When the value matters, prefer counting with an `int` and computing the step.

### Iterating a String

A C string ends with a `\0` terminator. Comparing against it is the idiomatic way to walk every character, including embedded spaces:

```c
#include <stdio.h>

int main(void) {
    char word[] = "loop";

    for (int i = 0; word[i] != '\0'; i++) {
        printf("%c", word[i]);
    }
    printf("\n");

    return 0;
}
```

```text
loop
```

The Strings chapter will use `%s` with `printf` for most things, but the `\0` loop still appears regularly, including in `strlen`-free code that has to stop early.

### Iterating Backwards

To walk a string in reverse you need its last index. A string literal of length `L` has indices `0` through `L - 1`, because the last real character sits just before the `'\0'`:

```c
#include <stdio.h>

int main(void) {
    char word[] = "cato";   /* 4 characters: indices 0,1,2,3 */

    for (int i = 3; i >= 0; i--) {
        printf("%c", word[i]);
    }
    printf("\n");

    return 0;
}
```

```text
otac
```

Hardcoding `3` ties the loop to this one string. For anything real, get the length with `strlen` from `<string.h>` and start at `len - 1`.

### Multiple Variables

The header can declare and update as many variables as you like. This loop shrinks a range from both ends:

```c
#include <stdio.h>

int main(void) {
    int n = 5;

    for (int i = 0, j = n; i < j; i++, j--) {
        printf("i=%d j=%d\n", i, j);
    }

    return 0;
}
```

```text
i=0 j=5
i=1 j=4
i=2 j=3
```

Declarations and updates are matched up by position: `i++,` initializes `i` and `j--` updates `j`. Two counters that must move together belong in one header, where the reader sees the relationship immediately.

---

## Off-by-One: `<` vs `<=`

The most common loop bug in C is a boundary that is off by one.

Given `n = 5`:

```c
for (int i = 0; i < n; i++)   { /* 5 iterations: i = 0,1,2,3,4 */ }
for (int i = 0; i <= n; i++)  { /* 6 iterations: i = 0,1,2,3,4,5 */ }
```

For arrays this is not a style question — it is a memory bug. An array of `n` elements has valid indices `0` to `n - 1`:

```c
#include <stdio.h>

int main(void) {
    int n = 5;

    for (int i = 0; i <= n; i++) {
        printf("%d ", i);
    }
    printf("\n");

    return 0;
}
```

```text
0 1 2 3 4 5 
```

(trailing space, from `printf("%d ", i)`)

Six values for five slots. If `i` were used as an index, the last iteration would read one element past the end.

The rule: when the loop variable counts **items**, use `<`. When it counts **array indices**, use `<` as well, because the array has exactly `n` indices. Reach for `<=` only when the last iteration must be genuinely inclusive, like printing the numbers 1 through 10 inclusive.

```c
for (int i = 1; i <= 10; i++) { /* prints 1..10 — this is what <= is for */ }
```

---

## Loop-Only Scope (C99 and Later)

A variable declared in the `for` header exists **only inside the loop**:

```c
for (int i = 0; i < 3; i++) {
    printf("%d\n", i);   /* fine */
}

/* printf("%d\n", i);  ← error: 'i' was declared in the inner scope */
```

```text
l2scope.c: In function ‘main’:
l2scope.c:8:20: error: ‘i’ undeclared (first use in this function)
    8 |     printf("%d\n", i);
      |                    ^
```

Compare with the earlier example, where `i` was declared before the loop and was still `5` afterwards. The rule is not "`for` variables die" — the rule is **scope follows the braces**. Anything inside `{ }` disappears at the closing brace.

```text
    for (int i = 0; ...) {   <- i is born here, inside the braces
        ...
    }                       <- i dies here
```

Both styles are valid C11. Declare in the header when the counter is only needed inside the loop. Declare before the loop when you need its final value afterwards — counting how many iterations ran, searching and remembering which index matched, or writing code that must also compile as C89.

---

## When to Use Which

| Situation | Use |
| --- | --- |
| Counting from a start to an end | `for` |
| Walking an array or a string | `for` |
| Any loop with a counter | `for` |
| Waiting for input or a condition computed elsewhere | `while` or `for (;;)` |
| Body must run at least once | `do-while` |

Default to `for`. Switch to `while` when the loop's exit depends on something you only discover inside the body, and to `do-while` when one mandatory execution comes first.

---

#### [Previous](./01-part.md) | [Next](./03-part.md)