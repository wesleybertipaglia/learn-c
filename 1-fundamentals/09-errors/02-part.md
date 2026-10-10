# Assertions with `assert`

The status-return model handles **expected** failures: a file that is not there, a user typing letters into a number field. `assert` handles something else: **invariants that should be impossible.** It is a trap for your own bugs, not a shield for bad input.

---

## What `assert` Does

`assert` lives in `<assert.h>`:

```c
assert(expr);
```

- If `expr` is true, nothing happens and the program continues.
- If `expr` is false, the program prints a message naming the expression, the file and the line, then calls `abort()`.

```text
program: file.c:23: function_name: Assertion `expr' failed.
```

The line number is the point. It tells you exactly which of your thousands of assumptions was wrong, and the printed expression tells you which one, without you hunting through the source.

---

## A Runnable Example

```c
#include <stdio.h>
#include <assert.h>

static int average(int *values, int count) {
    int total = 0;
    int i;

    assert(count > 0);
    assert(values != NULL);

    for (i = 0; i < count; i++) {
        total += values[i];
    }
    return total / count;
}

int main(void) {
    int data[4] = {10, 20, 30, 40};

    printf("average = %d\n", average(data, 4));
    printf("now asking for the average of nothing...\n");
    printf("average = %d\n", average(data, 0));

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g average.c -o average
./average
```

```text
average = 25
now asking for the average of nothing...
average: average.c:8: average: Assertion `count > 0' failed.
Aborted (core dumped)
```

The shell reports the abort like this:

```text
Aborted (core dumped)
```

Read the message from the inside out:

| Part | Meaning |
| --- | --- |
| `average:` | the program name |
| `average.c:8:` | file and line — where the assumption is written |
| `average:` | the enclosing function |
| ``Assertion `count > 0' failed.`` | the exact expression, printed as you wrote it |

The program died on line 8, which is the `assert(count > 0)`. The value of `count` came from `main`, three frames away. Without the assertion this program would have divided by zero, and on most machines an integer division by zero does not even crash — it produces garbage.

---

## The Part Everyone Misses: `NDEBUG` Deletes Them

Here is the detail that makes `assert` different from an `if`:

> If the macro `NDEBUG` is defined when the file is compiled, every `assert` in it expands to nothing at all. Not "does less" — **nothing**. The expression is not evaluated, not even for its side effects.

`NDEBUG` is defined by default? No. It is defined by you, or by your build system:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g program.c -o program               # asserts ON
gcc -std=c11 -Wall -Wextra -pedantic -g -DNDEBUG program.c -o program   # asserts GONE
```

or inside the file, before including `<assert.h>`:

```c
#define NDEBUG
#include <assert.h>
```

You can see the difference in the preprocessed output:

```bash
gcc -std=c11 -E program.c | grep -c __assert_fail   # 3
gcc -std=c11 -E -DNDEBUG program.c | grep -c __assert_fail   # 0
```

Three calls survive without `NDEBUG`. Zero with it.

A warning tells you the same story from the other direction. This program has nothing but assertions:

```c
#include <assert.h>

int main(void) {
    int x = 5;

    assert(x == 5);
    assert(x > 0);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -DNDEBUG program.c -o program
```

```text
program.c: In function ‘main’:
program.c:4:9: warning: unused variable ‘x’ [-Wunused-variable]
    4 |     int x = 5;
      |         ^~~~~~
```

`x` is now unused, because the only code that read it was deleted. If a release build starts warning about variables that a debug build uses happily, `-DNDEBUG` is why.

### The Release Build of the Same Program

Take a program whose assertions pass, and compile it both ways:

```c
#include <stdio.h>
#include <assert.h>

static void withdraw(double *balance, double amount) {
    assert(balance != NULL);
    assert(amount > 0.0);
    *balance -= amount;
    assert(*balance >= 0.0);
}

int main(void) {
    double balance = 100.0;

    printf("start: %.2f\n", balance);
    withdraw(&balance, 30.0);
    printf("after withdrawing 30.00: %.2f\n", balance);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g withdraw.c -o withdraw
./withdraw
gcc -std=c11 -Wall -Wextra -pedantic -g -DNDEBUG withdraw.c -o withdraw_release
./withdraw_release
```

```text
start: 100.00
after withdrawing 30.00: 70.00
```

Identical output. The release build simply no longer contains the three checks.

And the release build of the *broken* program from the previous section?

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -DNDEBUG average.c -o average_release
./average_release
```

```text
average = 25
now asking for the average of nothing...
Floating point exception (core dumped)
```

The helpful message with the file and line is gone. You get a bare crash with no explanation. This is the whole trade-off in one command: assertions are free in development and worth nothing in production unless you keep them.

---

## Why This Matters: `assert` Is Not Input Validation

The rule that follows from the previous section:

> **Never use `assert` to check anything that comes from outside your program.**

If you write:

```c
int count;
scanf("%d", &count);
assert(count > 0);
```

then a user typing `-1` aborts your program in the debug build and skips the check entirely in the release build. Neither behaviour is acceptable. This is the wrong tool:

```c
/* WRONG: vanishes with -DNDEBUG */
assert(count > 0);

/* RIGHT: always runs, in every build */
if (count <= 0) {
    fprintf(stderr, "error: count must be positive\n");
    return -1;
}
```

Assertions are for facts **you** are responsible for:

| Good use of `assert` | Wrong use of `assert` |
| --- | --- |
| the index is inside the array | the index the user typed is inside the array |
| the malloc result is not NULL after you already checked | a `scanf` succeeded |
| the queue is not empty before `dequeue` | a file exists (it may be deleted at runtime) |
| this branch is only reached with `size > 0` | the password matches |

The first column is code you wrote. The second column is code an outside party controls, and outside parties change their minds between your build and your user's afternoon.

---

## Side Effects Disappear Too

Because the expression is not evaluated when `NDEBUG` is defined, this is a real bug waiting to happen:

```c
/* WRONG in a -DNDEBUG build: the function is never called */
assert(consume_token() == TOKEN_OK);
```

In the debug build your token is consumed. In the release build the call vanishes, and the program behaves differently than the one you tested. If an expression has a side effect, it does not belong in an `assert`.

---

## What `assert` Costs, and What It Buys

A failing `assert` calls `abort()`, which raises `SIGABRT`. That is deliberate:

| | `assert` | returning a status |
| --- | --- | --- |
| caller must do something | yes, check the return | yes, check the return |
| survives `-DNDEBUG` | **no** | yes |
| survives optimization | yes | yes |
| good for | internal invariants, unreachable branches, postconditions | anything that can genuinely fail at runtime |

When a program in the field aborts instead of degrading gracefully, you want a **status return**, not an assertion. An assertion firing in production means you shipped a bug you could have handled; it is a bug report against yourself.

---

## Checklist

```text
[ ] Does every assert check something only my own code can guarantee?
[ ] Is the expression free of side effects?
[ ] Would removing this check turn a clean error message into a crash?
      if yes, it is not an assert — it is an if
[ ] Am I building with -Wall -Wextra? (assert uses them for its message)
```

---

#### [Previous](./01-part.md) | [Next](./03-part.md)