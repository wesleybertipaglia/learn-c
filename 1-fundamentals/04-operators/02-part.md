# Comparison and Logical Operators

Comparison operators produce 0 or 1. Logical operators combine conditions. Together they are the vocabulary of every decision in your programs.

## Comparison Operators

| Operator | Meaning | Example | Result |
| --- | --- | --- | --- |
| `==` | equal to | `5 == 5` | 1 |
| `!=` | not equal to | `5 != 5` | 0 |
| `<` | less than | `3 < 5` | 1 |
| `>` | greater than | `3 > 5` | 0 |
| `<=` | less than or equal | `5 <= 5` | 1 |
| `>=` | greater than or equal | `5 >= 6` | 0 |

The result of any comparison is an `int`: `1` for true, `0` for false. There is no separate boolean type in the operators themselves.

```c
#include <stdio.h>

int main(void) {
    int a = 10;
    int b = 20;

    printf("%d\n", a == b);    /* 0 */
    printf("%d\n", a != b);    /* 1 */
    printf("%d\n", a < b);     /* 1 */
    printf("%d\n", a >= b);    /* 0 */

    /* chaining does not work like mathematics */
    printf("%d\n", 1 < 2 < 3);  /* 1: (1 < 2) is 1, then 1 < 3 is 1 — coincidence! */
    printf("%d\n", 3 > 2 > 1);  /* 0: (3 > 2) is 1, then 1 > 1 is 0 */

    return 0;
}
```

The two chained comparisons above are deliberately wrong; GCC warns about both.

```text
warning: comparisons like 'X<=Y<=Z' do not have their mathematical meaning [-Wparentheses]
```

`1 < 2 < 3` happens to give 1, but only by luck: it evaluates `(1 < 2)` first, yielding `1`, then compares `1 < 3`. `3 > 2 > 1` shows the trap. You must use `&&`:

```c
if (a < b && b < c) {   /* correct */
}
```

---

## `=` Is Not `==`

The classic C bug:

```c
int x = 5;

if (x = 5) {          /* assigns 5 to x, and the condition is 5, which is true */
    printf("always true\n");
}

if (x == 5) {         /* compares x with 5 */
    printf("true\n");
}
```

- `=` is **assignment**. It stores the right-hand value into the left-hand variable and yields the stored value.
- `==` is **comparison**. It asks a question and yields 0 or 1.

GCC warns about the first version, which is the only reason it survives:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g prog.c -o prog
```

```text
warning: suggest parentheses around assignment used as truth value [-Wparentheses]
```

And this is worse, because it compiles without a warning and silently stores into the wrong variable:

```c
int x = 0;
int y;

if (x == y = 5) {   /* parses as x == (y = 5): legal, assigns 5 to y */
}
```

Fix it at the root of your typing habits: write `if (x == 5)` with **both** `=` signs, and if you always write the two spaces of `==`, the typo stands out immediately.

---

## Logical Operators

| Operator | Meaning | True when |
| --- | --- | --- |
| `&&` | logical AND | both operands are true |
| `\|\|` | logical OR | at least one operand is true |
| `!` | logical NOT | flips the operand |

```c
#include <stdio.h>

int main(void) {
    int age = 20;
    int has_ticket = 1;

    printf("%d\n", age >= 18 && has_ticket);
    printf("%d\n", age >= 18 || has_ticket);
    printf("%d\n", !(age >= 18));

    if (age >= 18 && has_ticket) {
        printf("allowed\n");
    }

    return 0;
}
```

These operators operate on **truthiness**, not on booleans. Only the value of each operand matters, not the value itself.

---

## Short-Circuit Evaluation

`&&` and `||` **do not evaluate their right operand if the result is already known**. This is not an optimization; it is guaranteed by the standard, and it is the safe way to write expressions that could be dangerous.

```c
#include <stdio.h>

int main(void) {
    int a = 10;
    int b = 0;

    if (b != 0 && a / b > 2) {
        printf("division is bigger than 2\n");
    } else {
        printf("not evaluated, no crash\n");
    }

    return 0;
}
```

Without the `b != 0 &&` guard, `a / b` would be integer division by zero and the program would die with `SIGFPE`. Because `b != 0` is false, `&&` never evaluates `a / b`.

The same trick works with `||`:

```c
char *name = NULL;

/* the left side is false, so name is never dereferenced */
if (name == NULL || name[0] == '\0') {
    printf("empty\n");
}
```

Write `p != NULL && *p == '\0'`, never `*p == '\0' && p != NULL` — the second version dereferences `p` before checking it.

> Order matters, and only one order is safe.

---

## Truthiness: What Counts as True

In C, `0` is false and **everything else is true**. There is no exception.

```c
#include <stdio.h>

int main(void) {
    int        a = 0;
    int        b = -1;
    int        c = 256;
    char       letter = 'A';
    double     d = 0.0001;
    void      *p = NULL;

    printf("%d %d %d\n", !a, !b, !c);          /* 1 0 0 */
    printf("%d\n", !!b);                       /* 1: b is nonzero */
    printf("%d\n", !!a);                       /* 0 */

    if (letter)  { printf("letter is true\n"); }
    if (d)       { printf("0.0001 is true, even though it is nearly zero\n"); }
    if (p)       { printf("p is true\n"); }
    else         { printf("NULL is false\n"); }

    return 0;
}
```

```text
1 0 0
1
0
letter is true
0.0001 is true, even though it is nearly zero
NULL is false
```

Things to internalize:

- `-1` is true. Only exactly `0` is false.
- A non-null pointer is true. A null pointer (`NULL`, which is conceptually `(void *)0`) is false.
- Any nonzero `char` is true, including a space or a newline.
- A very small nonzero `double` is true. Do not test floats with `if (d)` expecting "is it zero" to work reliably.

### `!` Returns 0 or 1, Not "the opposite number"

```c
int n = 42;
printf("%d\n", !n);   /* 0, not "minus 42" */
```

That is why `!!x` is the idiomatic "normalize to 0 or 1".

---

## When to Parenthesize

Comparisons bind tighter than `&&`, which binds tighter than `||`. The logic is correct without parentheses, but hard to read. Add them:

```c
/* correct but hard to read */
if (x > 0 && y > 0 || z == 0) {
}

/* same meaning, obvious */
if ((x > 0 && y > 0) || (z == 0)) {
}
```

For chains, parentheses are mandatory: `a < b == c` parses as `(a < b) == c`, which is almost never what you meant. Always write `(a < b && b < c)`.

---

## A Complete Example

```c
#include <stdio.h>

int main(void) {
    int age;
    int has_id;
    int banned;

    printf("Age: ");
    if (scanf("%d", &age) != 1) {
        return 1;
    }
    printf("Has ID (1/0): ");
    if (scanf("%d", &has_id) != 1) {
        return 1;
    }
    printf("Banned (1/0): ");
    if (scanf("%d", &banned) != 1) {
        return 1;
    }

    if (banned) {
        printf("ACCESS DENIED: banned\n");
    } else if (!has_id) {
        printf("ACCESS DENIED: no ID\n");
    } else if (age < 18) {
        printf("ACCESS DENIED: minors only with a guardian\n");
    } else if (age >= 65) {
        printf("ACCESS GRANTED: senior discount applies\n");
    } else {
        printf("ACCESS GRANTED\n");
    }

    return 0;
}
```

Note how `else if` chains are evaluated in order and stop at the first true branch. That structure is covered in the next chapter.

---

#### [Previous](./01-part.md) | [Next](./03-part.md)