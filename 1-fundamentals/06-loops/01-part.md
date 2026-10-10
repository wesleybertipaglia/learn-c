# Loops: `while` and `do-while`

A loop repeats a block of code while a condition holds. In C you have exactly three loop constructs — `while`, `do-while` and `for` — and they are interchangeable in power: anything one can do, the others can do too. Choosing between them is a matter of readability, not capability.

This part covers `while` and `do-while`. The `for` loop, which is what you will use 90% of the time, is in the next part.

---

## What a Loop Is

Every loop is the same three pieces wearing different clothes:

| Part | Question it answers |
| --- | --- |
| **Initialization** | where do we start? |
| **Condition** | when do we stop? |
| **Update** | how do we move toward the answer? |

If any of the three is missing or wrong, the loop runs forever. That is not an error the compiler can catch in most cases, so you learn to check each one by reading.

---

## The `while` Loop

`while` tests the condition **before** every iteration. If the condition is false the very first time, the body never runs.

```c
#include <stdio.h>

int main(void) {
    int i = 5;

    printf("Countdown:\n");
    while (i > 0) {
        printf("%d\n", i);
        i = i - 1;
    }
    printf("Liftoff!\n");

    return 0;
}
```

```text
Countdown:
5
4
3
2
1
Liftoff!
```

Line by line, the loop repeats with `i = 5`, then `4`, then `3`, `2`, `1`. When `i` becomes `0` the condition `i > 0` fails and control falls to the line after the closing brace.

### Zero Iterations Is Normal

```c
int i = 10;

while (i > 0) {
    /* i is 10, condition false, body never runs */
}
```

This matters when you build the value inside the loop. If the body never runs, the accumulator keeps its initial value, not some garbage.

---

## Forgetting the Update: The Infinite Loop

Remove the update and the condition never changes, so the loop runs until the program is killed:

```c
#include <stdio.h>

int main(void) {
    int i = 1;

    printf("BUG: the update is missing\n");

    while (i <= 5) {
        printf("%d\n", i);
        /* forgot i = i + 1; */
    }

    return 0;
}
```

```text
BUG: the update is missing
1
1
1
1
1
1
1
...
```

It never stops. Press `Ctrl+C` to kill it.

The important detail: **this compiles with zero warnings** under `-Wall -Wextra -pedantic`. The compiler sees a valid `while` whose condition happens to be permanently true. Nothing is wrong syntactically, so there is nothing to warn about. Only you can see the missing update.

Nor can any warning flag save you. gcc has no option that reports "this loop can never terminate" for a condition that depends on a variable: as far as the compiler can see, some other path in the program might change that variable, and `for (;;) { }` with no `break` is accepted silently too.

---

## The `do-while` Loop

`do-while` runs the body **first** and tests the condition afterwards. It is the only loop in C whose body always executes at least once.

```c
#include <stdio.h>

int main(void) {
    int i = 3;

    printf("DO-WHILE countdown:\n");

    do {
        printf("%d\n", i);
        i = i - 1;
    } while (i > 0);

    printf("Liftoff!\n");

    return 0;
}
```

```text
DO-WHILE countdown:
3
2
1
Liftoff!
```

### The Trailing Semicolon Is Mandatory

`do-while` ends with a semicolon after the closing parenthesis:

```c
    } while (i > 0);
                  ^ this semicolon is required
```

Leave it out and the parser swallows the next statement looking for it. Here is the classic bug:

```c
#include <stdio.h>

int main(void) {
    int i = 3;

    do {
        printf("%d\n", i);
        i = i - 1;
    }

    return 0;
}
```

```text
l1miss.c: In function ‘main’:
l1miss.c:11:5: error: expected ‘while’ before ‘return’
   11 |     return 0;
      |     ^~~~~~
```

The message points at `return`, which is innocent. The real problem is on the previous line.

---

## `while` or `do-while`?

| Situation | Use |
| --- | --- |
| The loop might not need to run at all | `while` |
| The body must run once before any decision | `do-while` |
| You have a value and just want to keep halving it | `do-while` |
| You are waiting for input that may never arrive | `while` |

A `do-while` is the natural fit when the loop body itself produces the "should I continue?" answer. This program halves `n` until it is 1:

```c
#include <stdio.h>

int main(void) {
    int n = 10;
    int rounds = 0;

    do {
        printf("round %d, n = %d\n", rounds, n);
        n = n / 2;
        rounds++;
    } while (n > 1);

    printf("stopped after %d rounds\n", rounds);

    return 0;
}
```

```text
round 0, n = 10
round 1, n = 5
round 2, n = 2
stopped after 3 rounds
```

A `while` version would need the body duplicated or a dummy first iteration to guarantee the print happens. That duplication is a sign you want `do-while`.

---

## Example: Summing 1 to N

The classic accumulation loop. Three parts, all in the same block of code:

```c
#include <stdio.h>

int main(void) {
    int i;
    int n = 5;
    int sum = 0;

    i = 1;                        /* initialization */
    while (i <= n) {              /* condition */
        sum = sum + i;
        i = i + 1;                /* update */
    }

    printf("sum of 1..%d = %d\n", n, sum);

    return 0;
}
```

```text
sum of 1..5 = 15
```

Notice `sum = 0` before the loop. Delete it and the program still runs — and still prints `15` at `-O0`, because the leftover stack value happened to be zero. But the code is now undefined behaviour, and `gcc -O2` says so:

```text
wv.c: In function ‘main’:
wv.c:10:13: warning: ‘sum’ is used uninitialized [-Wuninitialized]
   10 |         sum = sum + i;
      |         ~~~~^~~~~~~~~
wv.c:6:9: note: ‘sum’ was declared here
    6 |     int sum;
      |       ^~~
```

That is the worst kind of bug: it passes the test, passes review, and fails in production. Initializing the accumulator is part of the initialization step, and forgetting it is the second most common loop bug after a missing update.

---

## Example: Reading Until a Sentinel Value

Interactive programs need a way out. A **sentinel** is a value the user types to mean "I am done" — commonly `0`, `-1`, or an empty string.

```c
#include <stdio.h>

int main(void) {
    int n;

    printf("Enter numbers, 0 to stop:\n");

    while (scanf("%d", &n) == 1 && n != 0) {
        printf("read %d, its double is %d\n", n, n * 2);
    }

    printf("done\n");

    return 0;
}
```

Input:

```text
5
-3
0
```

Output:

```text
Enter numbers, 0 to stop:
read 5, its double is 10
read -3, its double is -6
done
```

Two conditions joined by `&&`, and both matter:

- `scanf(...) == 1` — did we actually read an integer? See the next section.
- `n != 0` — is the user done?

Also note that a **negative** number is not a sentinel here: `-3` is processed normally. Pick your sentinel carefully, and document it in your program's prompt.

---

## Robust Input: `while (scanf(...) != 1)`

`scanf` returns the number of values it successfully converted. If the user types `hello` when you asked for an `int`, it returns `0`, converts nothing, and **leaves the bad text in the input buffer**. The next `scanf` will read the same garbage and fail again, forever:

```c
int n;

scanf("%d", &n);   /* user types "hello" -> returns 0, n untouched */
scanf("%d", &n);   /* returns 0 again: "hello" is still in the buffer */
```

The fix is a loop that checks the return value and throws away whatever it could not use:

```c
#include <stdio.h>

int main(void) {
    int n;
    int sum = 0;

    printf("Enter numbers, 0 to stop. Anything else is ignored.\n");

    while (1) {
        printf("> ");
        if (scanf("%d", &n) != 1) {
            int c;
            while ((c = getchar()) != '\n' && c != EOF) {
                /* throw away the rest of the bad line */
            }
            printf("not a number, ignored\n");
            continue;
        }

        if (n == 0) {
            break;
        }

        sum = sum + n;
        printf("got %d, running sum = %d\n", n, sum);
    }

    printf("total = %d\n", sum);

    return 0;
}
```

Input:

```text
hello
42
7 x
0
```

Output:

```text
Enter numbers, 0 to stop. Anything else is ignored.
> not a number, ignored
> got 42, running sum = 42
> got 7, running sum = 49
> not a number, ignored
> total = 49
```

Read the inner `getchar` loop carefully: it consumes characters up to and including the newline, which is exactly what clears the offending line. Without it you would spin forever on the same text. The `c != EOF` check matters when input is redirected from a file and there is no trailing newline — without it, a program that reads to end-of-file loops on `EOF` forever, since `EOF` never equals `'\n'`.

> This pattern is correct but noisy. In real programs you often see `fgets` used to read a whole line and then converted with `sscanf` or `strtol`, which is easier to reason about. You now know why that exists.

---

## Checklist for Every Loop You Write

Before you run a program with a loop, answer these four questions:

1. Is the counter initialized to the correct starting value?
2. Is the condition the correct comparison — `i < n` or `i <= n`?
3. Does the update move in the direction the condition expects?
4. Is the loop guaranteed to terminate?

---

#### [Previous](../05-conditionals/exercises.md) | [Next](./02-part.md)