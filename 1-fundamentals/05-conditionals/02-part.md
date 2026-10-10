# The `switch` Statement

When you are comparing one value against a list of constants, `switch` is clearer and faster than a chain of `if`/`else if`.

## Syntax

```c
switch (expression) {
    case constant1:
        statements;
        break;
    case constant2:
        statements;
        break;
    default:
        statements;
        break;
}
```

The `switch` expression is evaluated once, and its value is compared against each `case` label. The first match starts executing, and execution **continues to the end of the switch** unless a `break` stops it.

```c
#include <stdio.h>

int main(void) {
    int day;

    printf("Day of the week (1-7): ");
    if (scanf("%d", &day) != 1) {
        return 1;
    }

    switch (day) {
        case 1:
            printf("Monday\n");
            break;
        case 2:
            printf("Tuesday\n");
            break;
        case 3:
            printf("Wednesday\n");
            break;
        case 4:
            printf("Thursday\n");
            break;
        case 5:
            printf("Friday\n");
            break;
        case 6:
        case 7:
            printf("Weekend\n");
            break;
        default:
            printf("Invalid day\n");
            break;
    }

    return 0;
}
```

Note `case 6:` and `case 7:` stacked together with no statements between them. That is the idiomatic way to share a body between several values.

---

## What Can Be Switched

The expression and every `case` label must be **integer types**: `int`, `char`, or an enumerated type. Not `float`, not `double`, not `long long` portably, not a string.

```c
#include <stdio.h>

enum Color { RED, GREEN, BLUE };

int main(void) {
    enum Color c = GREEN;
    char letter = 'q';

    switch (c) {                    /* enums work */
        case RED:   printf("red\n");   break;
        case GREEN: printf("green\n"); break;
        case BLUE:  printf("blue\n");  break;
    }

    switch (letter) {                /* chars work */
        case 'q':
        case 'Q':
            printf("quit\n");
            break;
        default:
            printf("other\n");
            break;
    }

    return 0;
}
```

### `case` Labels Must Be Compile-Time Constants

```c
int limit = 10;

switch (n) {
    case limit:          /* ERROR: limit is not a constant expression */
        break;
}
```

```text
error: case label does not reduce to a constant expression
```

The label must be a constant known at compile time: a literal, an enum constant, or a `const int` initialized with a literal.

```c
#define MAX 100
const int limit = 10;   /* this IS a constant expression in C */

switch (n) {
    case MAX:           /* ok: macro */
        break;
    case limit:         /* ok: const int with a constant initializer */
        break;
}
```

If you need to compare against a runtime value, use `if`:

```c
if (n == limit) {
    /* works */
}
```

### `switch` Does Not Work on Strings or Floats

```c
char name[] = "ana";

switch (name) {            /* ERROR: array decays to a pointer */
    case "ana":            /* ERROR: not an integer constant */
}
```

```text
error: case label does not reduce to a constant expression
error: switch quantity is not an integer
```

A string literal is an array of `char`, not an integer, and comparing arrays compares addresses. To branch on a string you must use `strcmp` inside an `if`:

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char choice[16];

    printf("Command: ");
    if (fgets(choice, sizeof(choice), stdin) == NULL) {
        return 1;
    }

    if (strcmp(choice, "start\n") == 0 || strcmp(choice, "start") == 0) {
        printf("starting\n");
    } else if (strcmp(choice, "stop") == 0) {
        printf("stopping\n");
    } else {
        printf("unknown\n");
    }

    return 0;
}
```

Floats fail for a different reason: `case` labels are integer constants, so there is no way to write `case 1.5:`. And comparing floats with `==` is unreliable anyway (see part 3). Use `if` with a tolerance.

---

## `break` and the Fallthrough Bug

Without `break`, execution keeps going into the next case. This is called **fallthrough**. Sometimes it is what you want, but usually it is a bug.

Here is a menu with a missing `break`:

```c
#include <stdio.h>

int main(void) {
    char choice;

    printf("Choose a, b or c: ");
    if (scanf(" %c", &choice) != 1) {
        return 1;
    }

    switch (choice) {
        case 'a':
            printf("Apples: 3.50\n");
            /* MISSING break! */
        case 'b':
            printf("Bananas: 2.10\n");
            break;
        case 'c':
            printf("Cherries: 8.00\n");
            break;
        default:
            printf("Invalid option\n");
            break;
    }

    return 0;
}
```

Note that this example is intentionally broken: compiling it produces `warning: this statement may fall through [-Wimplicit-fallthrough=]`. That warning is the compiler doing its job.

Choosing `a` prints two lines:

```text
Choose a, b or c: a
Apples: 3.50
Bananas: 2.10
```

The program reports the price of apples **and** bananas. The user sees two items in their cart when they bought one. With `break` restored, `a` prints only the apples line.

Note the leading space in `scanf(" %c", &choice)`: without it, `scanf` skips whitespace for `%d` but not for `%c`, so pressing Enter after a previous input would read the newline and match `default`. This is a real habit to build for character input.

### When Fallthrough Is Correct

Stacking labels (no statements between them) and deliberate grouping are both legitimate:

```c
#include <stdio.h>

int main(void) {
    char grade;

    printf("Grade: ");
    if (scanf(" %c", &grade) != 1) {
        return 1;
    }

    switch (grade) {
        case 'A':
        case 'B':
        case 'C':
            printf("Passed\n");
            break;
        case 'D':
        case 'F':
            printf("Failed\n");
            break;
        default:
            printf("Unknown grade\n");
            break;
    }

    return 0;
}
```

If you intentionally want to run several cases in sequence, say so with a comment naming the target case — GCC recognizes the comment and stops warning:

```c
case 'q':
    printf("Quitting\n");
    /* fall through */
case 'x':
    return 0;
```

---

## `default`

`default` runs when no `case` matches. It is optional, but recommended: without it, an unexpected value silently does nothing.

`default` can be placed anywhere in the switch, though convention puts it last.

---

## Complete Example: A Menu Program

```c
#include <stdio.h>

int main(void) {
    int    choice;
    double balance = 150.00;

    while (1) {
        printf("\n=== ATM ===\n");
        printf("1. Balance\n");
        printf("2. Deposit\n");
        printf("3. Withdraw\n");
        printf("0. Exit\n");
        printf("Choice: ");

        if (scanf("%d", &choice) != 1) {
            return 1;
        }

        switch (choice) {
            case 1:
                printf("Balance: %.2f\n", balance);
                break;

            case 2: {
                double amount;
                printf("Amount to deposit: ");
                if (scanf("%lf", &amount) != 1 || amount <= 0.0) {
                    printf("Invalid amount\n");
                    break;
                }
                balance += amount;
                printf("New balance: %.2f\n", balance);
                break;
            }

            case 3: {
                double amount;
                printf("Amount to withdraw: ");
                if (scanf("%lf", &amount) != 1 || amount <= 0.0) {
                    printf("Invalid amount\n");
                    break;
                }
                if (amount > balance) {
                    printf("Insufficient funds\n");
                    break;
                }
                balance -= amount;
                printf("New balance: %.2f\n", balance);
                break;
            }

            case 0:
                printf("Goodbye\n");
                return 0;

            default:
                printf("Invalid option: %d\n", choice);
                break;
        }
    }
}
```

```text
=== ATM ===
1. Balance
2. Deposit
3. Withdraw
0. Exit
Choice: 2
Amount to deposit: 100.50
New balance: 250.50

=== ATM ===
...
Choice: 0
Goodbye
```

Two details to notice:

- The `case 2:` and `case 3:` bodies declare variables, so they need their own `{ }` block. Without the braces, every declaration in the whole switch would be in one scope, and you could not reuse a name.
- `default` handles garbage input and loops back instead of crashing.

---

## `switch` vs `if`/`else`

| Situation | Use |
| --- | --- |
| One value against many integer constants | `switch` |
| Ranges (`score >= 90`) | `if` |
| Comparing two variables | `if` |
| Comparing strings | `if` with `strcmp` |
| Comparing floats | `if`, with a tolerance |
| Two or three simple cases | either; `if` is fine |

```c
/* switch: one variable, many exact constants */
switch (status) {
    case 200: printf("OK\n");    break;
    case 404: printf("Not found\n"); break;
    case 500: printf("Server error\n"); break;
}

/* if: ranges and arbitrary expressions */
if (status >= 200 && status < 300) {
    printf("Success family\n");
} else if (status >= 500) {
    printf("Server error family\n");
}
```

---

#### [Previous](./01-part.md) | [Next](./03-part.md)