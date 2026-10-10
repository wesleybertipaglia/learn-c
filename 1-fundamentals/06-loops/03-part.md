# Jump Statements: `break`, `continue` and `goto`

Three keywords can move control out of the middle of a loop. Two of them (`break` and `continue`) are everyday tools. The third (`goto`) is legal, compiles cleanly, and is almost always the wrong answer.

---

## `break`: Leave the Loop Immediately

`break` jumps to the statement right after the loop:

```c
#include <stdio.h>

int main(void) {
    int sum = 0;

    for (int i = 1; i <= 10; i++) {
        if (i == 3) {
            continue;
        }
        if (i == 7) {
            break;
        }
        sum = sum + i;
    }

    printf("sum = %d\n", sum);

    return 0;
}
```

```text
sum = 18
```

Trace it: `i` runs 1 to 6 (`3` is skipped by `continue`, `7` ends the loop). The sum is `1 + 2 + 4 + 5 + 6 = 18`.

`break` works in `while`, `do-while` and `for` alike, and inside `switch` too.

---

## `continue`: Skip to the Next Iteration

`continue` abandons the rest of the current iteration and starts the next one.

The pairing with `if` is the usual shape: skip the work you do not want to do.

```c
for (int i = 1; i <= 10; i++) {
    if (i % 2 != 0) {
        continue;   /* odd: do nothing else */
    }
    printf("%d ", i);
}
```

Prints `2 4 6 8 10`.

---

## The `continue` Trap in `while` and `do-while`

In a `for` loop, `continue` still runs the update — that is part of what makes `continue` safe there. In `while` and `do-while`, `continue` jumps straight to the **condition**, skipping the update entirely. If the update is the only thing that moves the counter, the loop never terminates:

```c
#include <stdio.h>

int main(void) {
    int i = 0;

    while (i < 5) {
        if (i == 2) {
            continue;   /* jumps to the condition, i++ is never reached */
        }
        printf("i=%d\n", i);
        i++;
    }

    return 0;
}
```

```text
i=0
i=1
```

...and then `i` stays `2` forever, printing nothing, until you kill the program with `Ctrl+C`.

This compiles with **zero warnings** under `-Wall -Wextra -pedantic`. The fix is to move the update above the `continue`:

```c
    while (i < 5) {
        i++;              /* update first */
        if (i == 2) {
            continue;
        }
        printf("i=%d\n", i);
    }
```

Or, better, use a `for` loop where the update is in the header and cannot be jumped over:

```c
    for (int i = 0; i < 5; i++) {
        if (i == 2) {
            continue;
        }
        printf("i=%d\n", i);
    }
```

> Rule of thumb: in a `while` loop, never put `continue` before the line that advances the counter.

---

## Nested Loops and `break`

A `break` only breaks the **innermost** loop it sits in. The outer loop keeps going.

```c
#include <stdio.h>

int main(void) {
    for (int a = 0; a < 3; a++) {
        for (int b = 0; b < 3; b++) {
            if (b == 2) {
                break;
            }
            printf("a=%d b=%d\n", a, b);
        }
    }

    return 0;
}
```

```text
a=0 b=0
a=0 b=1
a=1 b=0
a=1 b=1
a=2 b=0
a=2 b=1
```

`a` reached `2` even though `b` never got past `1`. If you need to leave both loops at once, you need a flag or a helper function (see below) — `break` alone cannot do it.

---

## Example: Multiplication Table With an Inner `break`

A multiplication table has to stop each row at a fixed column. A `break` in the inner loop does exactly that, and the outer loop must keep going:

```c
#include <stdio.h>

int main(void) {
    for (int row = 1; row <= 3; row++) {
        for (int col = 1; col <= 5; col++) {
            if (col > 3) {
                break;   /* leaves only the inner loop */
            }
            printf("%d ", row * col);
        }
        printf("\n");
    }

    return 0;
}
```

```text
1 2 3 
2 4 6 
3 6 9 
```

(Each line ends with a space, because `printf("%d ", ...)` prints one after every value, including the last.)

For comparison, the same table without the inner limit:

```c
for (int row = 1; row <= 3; row++) {
    for (int col = 1; col <= 3; col++) {
        printf("%d ", row * col);
    }
    printf("\n");
}
```

```text
1 2 3 
2 4 6 
3 6 9 
```

Same result, simpler. This is worth noticing: a `break` that only re-creates the natural end of the loop is dead weight.

---

## Infinite Loops and `break` as the Way Out

`for (;;)` and `while (1)` are infinite by construction, so `break` is not an escape hatch — it is the *only* exit. Every exit path you leave out is a bug waiting for a user to find it.

```c
#include <stdio.h>

int main(void) {
    for (;;) {
        int line;

        printf("Enter a number (0 to quit): ");

        if (scanf("%d", &line) != 1) {
            int c;
            while ((c = getchar()) != '\n' && c != EOF) {
            }
            printf("not a number, try again\n");
            continue;
        }

        if (line == 0) {
            break;
        }

        printf("  doubled: %d\n", line * 2);
    }

    printf("bye\n");

    return 0;
}
```

```text
Enter a number (0 to quit):   doubled: 10
Enter a number (0 to quit): bye
```

Two exits exist here: `break` for the user's sentinel, and `return 0` when the program is done. Any infinite loop you write needs an equivalent pair.

### Guard Clauses Are `break` in Disguise

A very common use of `break` is leaving a loop early because one input was invalid, before any real work happens:

```c
for (int i = 0; i < 100; i++) {
    int value;
    scanf("%d", &value);

    if (value < 0) {
        printf("negative value, stopping\n");
        break;
    }

    process(value);
}
```

This is better than nesting the rest of the loop inside `if (value >= 0) { ... }` because the happy path stays at the left margin.

---

## `goto`: Legal, Compiles, Almost Always Wrong

`goto` jumps to a **label** anywhere in the same function:

```c
#include <stdio.h>

int main(void) {
    int total = 0;
    int i = 0;
    int limit = 100;

again:
    if (i > limit) {
        goto done;
    }

    total += i;
    i++;

    goto again;

done:
    printf("total = %d\n", total);

    return 0;
}
```

```text
total = 5050
```

It works. It compiles with no warnings. And the loop above is exactly equivalent to:

```c
int total = 0;
for (int i = 0; i <= 100; i++) {
    total += i;
}
printf("total = %d\n", total);
```

The `for` version is shorter, cannot jump into the middle of a loop, and cannot skip an initialization. That is the whole argument against `goto`:

| Problem | Why it matters |
| --- | --- |
| Jumps can go anywhere | Any statement can be skipped, including initialization, so the compiler cannot reason about your variables |
| Spaghetti | Enough labels and control flow stops being traceable by a human |
| No compile error | Nothing warns you; the damage shows up at runtime |
| Already covered | `break` handles loops, `switch` handles multi-way branching |

Where it used to be genuinely necessary — jumping to the end of a deeply nested block, or recovering from an error deep inside a parser — `break`, `return`, `goto` alternatives like `break` out of a `switch`, or just restructuring the function all work. And if you find yourself in a function so tangled that `goto` seems reasonable, the actual problem is usually that the function is doing too many things and should be split.

**Use `break` or `continue`.** `goto` in new C code is a bug in review.

---

## Which One Do I Need?

| Situation | Use |
| --- | --- |
| Leave the loop now | `break` |
| Skip this iteration, keep looping | `continue` |
| Leave a loop plus everything around it | `return` |
| Choose between many fixed cases | `switch` (Conditionals chapter) |
| Anything else | restructure |

---

#### [Previous](./02-part.md) | [Next](./exercises.md)