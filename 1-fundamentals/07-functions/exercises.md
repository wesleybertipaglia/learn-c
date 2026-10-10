# Exercise List 7 - Functions

Six exercises on function design, argument passing and recursion. Compile everything with:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

A useful habit for this list: add `-Wshadow` as well. Several of these exercises are exactly the kind where a name silently shadows another.

---

## Exercise 1 — `max` and `swap`

Write two functions in one file:

```c
int  max(int a, int b);
void swap(int *a, int *b);
```

`max` returns the larger of two values. `swap` exchanges the contents of the caller's two variables.

```text
max(3, 9) = 9
after swap: x=2 y=1
```

**Why one takes a pointer.** Comment this in your own words, then check it against this reasoning: arguments are passed by value, so `max` receives copies and can compare them freely without affecting the caller. `swap` is defined by its effect on the caller's variables, and a by-value parameter would only ever modify a copy that is thrown away. So its parameters must be pointers and the caller must pass addresses.

Write the prototype for `swap` before you write the definition, and never write a prototype for `max` with different types than the definition.

---

## Exercise 2 — Is Even

Write `int is_even(int n)` that returns whether `n` is even. Test it against positive, negative and zero values.

```text
4 is even
7 is odd
0 is even
-3 is odd
10 is even
```

Two details that separate a correct answer from a lucky one:

- Use `n % 2 == 0` rather than `n / 2 * 2 == n`. For negative `n`, integer division truncates toward zero, which makes the second version quietly give the wrong answer for some inputs. (`%` is defined for negative dividends, so the first version is safe — review the Operators chapter.)
- The name `is_even` promises a boolean, so return a comparison result (`n % 2 == 0`) rather than `if (...) return 1; else return 0;`. Any non-zero `int` is true, so `return n % 2 == 0;` is complete.

---

## Exercise 3 — Quotient and Remainder With Out-Parameters

Write `void integer_divide(int numerator, int denominator, int *quotient, int *remainder)` that produces both halves of an integer division in one call.

```text
  47 /    5 =    9 remainder    2  (q*d+r = 47)
 100 /    7 =   14 remainder    2  (q*d+r = 100)
 -47 /    5 =   -9 remainder   -2  (q*d+r = -47)
  47 /   -5 =   -9 remainder    2  (q*d+r = 47)
```

**Integers only.** Document the convention you chose for negatives as a comment at the top of the function. The output above uses C's native behaviour: truncation toward zero, so the remainder carries the sign of the numerator. If you prefer the mathematical convention (remainder always non-negative), add the adjustment from the Operators chapter — but say which one you implemented and why.

Three things to get right:

1. Print `q * d + r` next to each result. If it equals the numerator, your arithmetic is consistent. This is the property that defines integer division, and printing it makes any error visible immediately.
2. Decide what happens when `denominator` is `0` and handle it. Dividing by zero is undefined behaviour — a crash with no message — so check first and return or report instead.
3. Initialize `q` and `r` to `0` at the declaration even though the function overwrites them. It costs nothing and makes the data flow obvious.

---

## Exercise 4 — Recursion: Countdown and Digit Sum

Write two recursive functions:

```c
void print_countdown(int n);
int  sum_digits(int n);
```

`print_countdown(5)` prints 5, 4, 3, 2, 1 and returns. `sum_digits` returns the sum of an integer's digits, handling negative input by recursing on the absolute value.

```text
5
4
3
2
1
liftoff!

sum_digits(12345) = 15
sum_digits(-908) = 17
sum_digits(0) = 0
```

Working solutions:

```c
void print_countdown(int n) {
    if (n <= 0) {
        return;
    }
    printf("%d\n", n);
    print_countdown(n - 1);
}

int sum_digits(int n) {
    if (n < 0) {
        return sum_digits(-n);
    }
    if (n < 10) {
        return n;
    }
    return n % 10 + sum_digits(n / 10);
}
```

Check your base cases against these:

| Case | Why it matters |
| --- | --- |
| `n <= 0` in `print_countdown`, not `n == 0` | A negative `n` would otherwise recurse downward forever |
| `n < 0` handled in `sum_digits` | `n % 10` and `n / 10` on a negative `n` give the wrong digits |
| `n < 10` in `sum_digits`, not `n == 0` | `0` must return `0`, and a single digit is its own base case |

Then ask the questions from part 4: does every call move closer to the base case (yes, `n - 1` and `n / 10` both shrink `n`), and does the function return on every path?

---

## Exercise 5 — A `static` Helper Across Two Files

Split a program into two `.c` files so that a helper is `static` in one of them, then demonstrate that the other file cannot reach it.

`stats.h`:

```c
#ifndef STATS_H
#define STATS_H

int average_clamped(const int *values, int count);

#endif
```

`stats.c`:

```c
#include "stats.h"

static int clamp(int value, int low, int high) {
    /* static: a private helper of this file only. Another .c file cannot call
       it even if it declares a prototype, because the linker will not find the
       symbol. */
    if (value < low) {
        return low;
    }
    if (value > high) {
        return high;
    }
    return value;
}

int average_clamped(const int *values, int count) {
    int total = 0;

    for (int i = 0; i < count; i++) {
        total += clamp(values[i], 0, 100);
    }

    return count > 0 ? total / count : 0;
}
```

`main.c`:

```c
#include <stdio.h>

#include "stats.h"

int main(void) {
    int values[5];

    values[0] = 80;
    values[1] = 150;
    values[2] = -20;
    values[3] = 100;
    values[4] = 50;

    printf("average clamped = %d\n", average_clamped(values, 5));

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c stats.c -o app
./app
```

```text
average clamped = 66
```

The values clamp to `80, 100, 0, 100, 50`, which total `330` over `5` items.

Now write a fourth file that declares its own prototype for `clamp` and calls it:

```c
#include <stdio.h>

#include "stats.h"

int clamp(int value, int low, int high);

int main(void) {
    /* trying to reach the private helper of stats.c */
    printf("%d\n", clamp(150, 0, 100));

    (void)average_clamped;

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g cheat.c stats.c -o cheat
```

```text
/usr/bin/x86_64-linux-gnu-ld.bfd: cheat.o: in function ‘main':
cheat.c:(.text+0x18): undefined reference to `clamp'
collect2: error: ld returned 1 exit status
```

**Comment in your source** why this fails: the compiler is satisfied by the prototype, but `static` gave `clamp` internal linkage, so the symbol was never emitted into `stats.o` and the linker has nothing to resolve against. (The exact wording of the linker message differs between systems and linkers; the content does not.)

Then remove `static` from `clamp` and rebuild — the call now succeeds. That contrast is the whole point of the keyword, so do not skip it.

---

## Exercise 6 — Numbering Command Line Arguments

Write a program that reads `argc`/`argv` and prints each argument with its position, skipping `argv[0]`.

```text
$ ./args alpha beta "gamma delta"
this program received 3 argument(s):
1: alpha
2: beta
3: gamma delta
```

```text
$ ./args
this program received 0 argument(s):
```

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("this program received %d argument(s):\n", argc - 1);

    for (int i = 1; i < argc; i++) {
        printf("%d: %s\n", i, argv[i]);
    }

    return 0;
}
```

Details worth checking:

- `argc - 1`, not `argc`, because `argv[0]` is the program name.
- `"gamma delta"` arrives as **one** argument. The shell splits the command line, not your program.
- The loop starts at `1` and the condition is `i < argc`. Reading `argv[argc]` would be reading the null terminator slot, which is legal but prints nothing useful.

Then extend it: accept exactly two arguments, convert them with `strtol` from `<stdlib.h>`, and print their sum. When `argc` is anything other than 3, print a usage line to `stderr` using `argv[0]` and `return 1`, so a shell can detect the failure:

```bash
./sum 3 4
sum = 7
./sum 3
usage: ./sum <first> <second>
echo $?      # 1
```

---

## Checklist Before You Run

| Check | Why |
| --- | --- |
| Every prototype matches its definition exactly | A mismatch is a hard-to-find link or runtime bug |
| Every `int`-returning path returns a value | Missing `return` is undefined behaviour, not a warning |
| Out-parameters are passed as `&variable` | Passing the value modifies a copy |
| Nothing is a global unless it truly must be | Globals cannot be tested or reasoned about |
| Helpers are `static` | No name collisions, no accidental external use |
| Base cases cover zero and negative input | Recursion without them never terminates |
| Compiles clean with `-Wall -Wextra -pedantic` | Every warning is a real defect |

---

#### [Previous](./04-part.md) | [Next](../08-pointers/01-part.md)