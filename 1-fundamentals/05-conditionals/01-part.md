# The `if` Statement

Conditionals let your program choose. `if` runs a block when a condition is true, `else` covers the rest.

## Syntax

```c
if (condition) {
    /* runs when condition is nonzero */
} else {
    /* runs when condition is 0 */
}

if (condition_a) {
    /* first */
} else if (condition_b) {
    /* second, only if a was false */
} else {
    /* all other cases */
}
```

The condition is an expression of any integer type. Zero is false, anything else is true.

```c
#include <stdio.h>

int main(void) {
    int age;

    printf("Age: ");
    if (scanf("%d", &age) != 1) {
        return 1;
    }

    if (age >= 18) {
        printf("you are an adult\n");
    } else {
        printf("you are a minor\n");
    }

    return 0;
}
```

Only the first true branch runs. Once a branch is taken, the rest of the chain is skipped.

---

## Truthiness

You already saw this with the logical operators, and `if` uses the same rule:

| Expression | Result in `if` |
| --- | --- |
| `0`, `0L`, `0.0`, `'\0'` | false |
| `-1` | true |
| any nonzero number | true |
| a non-null pointer | true |
| `NULL` | false |

So `if (count)` means "if count is not zero", which is often what you want:

```c
if (count) {          /* means count != 0 */
    printf("%d items\n", count);
}
```

For a `char`, remember `'0'` is the digit zero with code 48, and is **true**:

```c
char c = '0';

if (c == '0') {       /* TRUE: 48 == 48 */
    printf("it is a zero digit\n");
}

if (c) {              /* also TRUE, but for the wrong reason */
    printf("c is nonzero\n");
}
```

Do not confuse "the character is zero" with "the character is false".

---

## Always Use Braces

This compiles:

```c
if (age >= 18)
    printf("adult\n");
else
    printf("minor\n");
```

And this one has a bug, invisible to the reader:

```c
if (age >= 18)
    printf("adult\n");
    printf("you may enter\n");   /* ALWAYS runs, regardless of age */
else
    printf("minor\n");
```

The braces around a single statement are optional, so the `printf` after it is a separate statement that always executes. `else` then attaches to the second `if`... except there is no second `if`, which makes this an outright syntax error in most arrangements. Either way, the lesson holds: **braces always**, even for one line.

A style worth adopting:

```c
if (age >= 18) {
    printf("adult\n");
} else {
    printf("minor\n");
}
```

Even if you only write one statement inside.

---

## The Dangling Else Problem

`else` always attaches to the **nearest unmatched `if`**, regardless of your indentation.

```c
#include <stdio.h>

int main(void) {
    int score = 50;
    int bonus = 0;

    /* BROKEN: the else binds to the inner if */
    if (score >= 60)
        if (bonus == 0)
            printf("pass, no bonus applied\n");
        else
            printf("failed\n");

    return 0;
}
```

What you wrote and what the compiler reads:

```text
you wrote          the compiler reads
if (a)             if (a)
    if (b)             if (b)
        x;                  x;
    else                  else
        y;                  y;
```

With `score = 50`, the outer `if` is false, so **nothing prints at all**. The `else` belongs to the inner `if`.

The fix is braces on the outer block:

```c
if (score >= 60) {
    if (bonus == 0) {
        printf("pass, no bonus applied\n");
    } else {
        printf("passed with a bonus\n");
    }
} else {
    printf("failed\n");
}
```

Now the meaning is unambiguous, and it matches your indentation.

The broken version above is in the file deliberately: compiling it produces the exact warning you want in your own code.

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g prog.c -o prog
```

```text
warning: suggest explicit braces to avoid ambiguous 'else' [-Wdangling-else]
```

---

## Complete Example: Grade Calculator

```c
#include <stdio.h>

int main(void) {
    double score;

    printf("Score (0 to 100): ");
    if (scanf("%lf", &score) != 1) {
        return 1;
    }

    if (score < 0.0 || score > 100.0) {
        printf("Invalid score: must be between 0 and 100\n");
    } else if (score >= 90.0) {
        printf("Grade: A\n");
    } else if (score >= 80.0) {
        printf("Grade: B\n");
    } else if (score >= 70.0) {
        printf("Grade: C\n");
    } else if (score >= 60.0) {
        printf("Grade: D\n");
    } else {
        printf("Grade: F\n");
    }

    return 0;
}
```

```text
Score (0 to 100): 73.5
Grade: C
```

Two details worth noticing:

- The chain is ordered from **highest threshold to lowest**, because the first true branch wins. Reversed, everyone would get an A.
- The range check comes first, so `score >= 90.0` can safely assume a valid score.

---

## Conditionals Are Statements, Not Expressions

`if (x > 0) y = 1;` is a statement, so it cannot be used where a value is needed. That is what `?:` is for, covered in the next part:

```c
int y = (x > 0) ? 1 : -1;      /* works: ?: is an expression */
```

And do not write `if (x = 5)`, which assigns instead of comparing:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g prog.c -o prog
```

```text
warning: suggest parentheses around assignment used as truth value [-Wparentheses]
```

---

#### [Previous](../04-operators/exercises.md) | [Next](./02-part.md)