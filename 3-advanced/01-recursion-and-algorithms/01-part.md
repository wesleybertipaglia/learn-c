# Recursion

A loop repeats by jumping back to the top. Recursion repeats by calling itself with a smaller problem. Both repeat work, but the mechanism is different: a loop reuses one stack frame, recursion builds a new stack frame per call.

Every recursive function has exactly two parts:

1. **Base case** — the input so small the answer is immediate, with no further call.
2. **Recursive step** — the function calls itself on a strictly smaller input, then combines the result.

If the input never reaches the base case, the calls never stop. That is not a wrong answer. It is a crash.

```c
/* shape of every recursion:
   if (input is trivial) return direct answer;   // base case
   return combine(current, f(smaller input));     // recursive step
 */
```

| Part | Job | Missing it causes |
| --- | --- | --- |
| Base case | stop the calls | infinite recursion, stack overflow |
| Recursive step | shrink the problem | wrong answer, or no progress toward base |

---

## The Simplest Recursion: Countdown

Count down from `n` to `0`, printing each number. The base case is `n == 0`. The recursive step prints `n`, then counts down from `n - 1`.

```c
#include <stdio.h>

static void countdown(unsigned int n) {
    if (n == 0) {              /* base case: nothing smaller to do */
        printf("0\n");
        return;
    }
    printf("%u\n", n);         /* work at this level */
    countdown(n - 1);          /* recursive step: strictly smaller */
}

int main(void) {
    countdown(3);
    return 0;
}
```

```text
3
2
1
0
```

Read the execution order carefully. `countdown(3)` prints `3` **before** calling `countdown(2)`. Work done before the call happens on the way **down**; work done after the call happens on the way **up**. This program does all its printing on the way down.

Swap the two lines and the order reverses:

```c
static void countdown_up(unsigned int n) {
    if (n == 0) {
        printf("0\n");
        return;
    }
    countdown_up(n - 1);       /* recurse first ... */
    printf("%u\n", n);         /* ... then print on the way back up */
}
```

```text
0
1
2
3
```

Same calls, same base case, different output. Where the work sits relative to the recursive call is part of the algorithm.

---

## Stack Frames: What a Call Costs

Each call gets its own **stack frame**: room for its parameters, its local variables, and its return address. Frames stack on top of each other and are destroyed in reverse order.

Calling `countdown(2)` builds this stack over time:

```text
  time -->

  frame for countdown(2)          frame for countdown(1)          frame for countdown(0)
  +-------------------+           +-------------------+           +-------------------+
  | n = 2             |  calls    | n = 2             |  calls    | n = 2             |
  +-------------------+  ---->    +-------------------+  ---->    +-------------------+
                                          | n = 1             |           | n = 1             |
                                          +-------------------+           +-------------------+
                                                                          | n = 0 (base)      |
                                                                          +-------------------+

  stack grows DOWNWARD in the diagram, one frame per call
  returns unwind in reverse: countdown(0) returns, then (1), then (2)
```

Three rules fall out of the picture:

1. **Depth equals the number of live calls.** `countdown(1000)` holds 1001 frames at its deepest point.
2. **Locals are private.** The `n` in one frame is a different variable from the `n` in the frame below it. Assigning to one never touches the other.
3. **The stack is finite.** Typical default is 8 MB on Linux. A few hundred thousand frames of any real size exhaust it.

That third rule is why the base case is load-bearing. Every missing base case is a program that keeps pushing frames until the operating system kills it.

Compare with a loop:

```text
  recursion:  [frame][frame][frame][frame] ... one per call, freed on return
  loop:       [      one frame, reused     ] ... counter updated in place
```

A loop uses O(1) stack space. Recursion uses O(depth) stack space. For `countdown`, depth equals `n`.

---

## Factorial: The Canonical Example

`n! = n * (n-1) * ... * 1`, with `0! = 1` by definition. The definition is already recursive: `n! = n * (n-1)!`.

```c
#include <stdio.h>

static unsigned long long factorial(unsigned int n) {
    if (n == 0) {                /* base case: 0! is 1 */
        return 1ULL;
    }
    return (unsigned long long)n * factorial(n - 1);   /* recursive step */
}

int main(void) {
    for (unsigned int i = 0; i <= 10; i++) {
        printf("%u! = %llu\n", i, factorial(i));
    }
    return 0;
}
```

```text
0! = 1
1! = 1
2! = 2
3! = 6
4! = 24
5! = 120
6! = 720
7! = 5040
8! = 40320
9! = 362880
10! = 3628800
```

Trace `factorial(3)` to see the down-and-up pattern:

```text
  factorial(3)
  = 3 * factorial(2)
        = 2 * factorial(1)
              = 1 * factorial(0)
                    = 1              <-- base case, no more calls
              = 1 * 1 = 1
        = 2 * 1 = 2
  = 3 * 2 = 6
```

The multiplications happen on the way **up**, after each call returns. At the deepest point there are 4 live frames (`3`, `2`, `1`, `0`). Each frame waits, holding its own `n`, while the frame above it computes.

The iterative version does the same arithmetic with one frame:

```c
static unsigned long long factorial_iter(unsigned int n) {
    unsigned long long acc = 1ULL;
    for (unsigned int i = 2; i <= n; i++) {
        acc *= i;
    }
    return acc;
}
```

Same answers, O(1) stack instead of O(n). For factorial, the loop is strictly better — there is no branching subproblem that recursion simplifies. Keep that observation; the "Recursion vs Iteration" section returns to it.

Overflow note: `13!` already exceeds 32-bit `int` range, and `21!` exceeds 64-bit `unsigned long long`. That is arithmetic overflow, not a recursion bug. With `-fsanitize=address,undefined`, signed overflow is reported; unsigned overflow wraps silently. Use `unsigned long long` and small test inputs (`n <= 12` for 32-bit, `n <= 20` for 64-bit), and treat larger inputs as out of scope for this chapter.

---

## Sum of Digits: Shrinking Without Counting Down

Recursion does not have to subtract 1. It has to make **progress** toward the base case by any measure that strictly decreases. For digits, divide by 10:

```c
#include <stdio.h>

static unsigned int sum_digits(unsigned int n) {
    if (n < 10) {                /* base case: one digit, it IS the sum */
        return n;
    }
    return (n % 10) + sum_digits(n / 10);   /* last digit + sum of the rest */
}

int main(void) {
    const unsigned int values[] = { 7, 42, 1234, 9999 };
    const size_t count = sizeof(values) / sizeof(values[0]);
    for (size_t i = 0; i < count; i++) {
        printf("sum_digits(%u) = %u\n", values[i], sum_digits(values[i]));
    }
    return 0;
}
```

```text
sum_digits(7) = 7
sum_digits(42) = 6
sum_digits(1234) = 10
sum_digits(9999) = 36
```

Trace `sum_digits(1234)`:

```text
  sum_digits(1234) = 4 + sum_digits(123)
                   = 4 + (3 + sum_digits(12))
                   = 4 + (3 + (2 + sum_digits(1)))
                   = 4 + (3 + (2 + 1))       <-- base case: 1 < 10
                   = 10
```

The "size" that shrinks is the number of digits, not the value by 1. Depth is O(number of digits), which is O(log n) — tiny. This is a recursion that cannot overflow the stack for any `unsigned int` input: at most 10 frames.

Generalize the pattern:

| Function | Base case | Smaller input | Combine step |
| --- | --- | --- | --- |
| `countdown(n)` | `n == 0` | `n - 1` | print, then recurse |
| `factorial(n)` | `n == 0` | `n - 1` | `n * f(n-1)` |
| `sum_digits(n)` | `n < 10` | `n / 10` | `(n % 10) + f(n/10)` |

Before writing any recursive function, fill in that row: what is trivial, what is smaller, how do the pieces combine. If any cell is blank, the function is not ready to write.

---

## Infinite Recursion and Stack Overflow

Delete the base case and every input recurses forever:

```c
#include <stdio.h>

/* BUG: no base case. Do NOT run this without a limit. */
static void boom(unsigned int n) {
    printf("%u\n", n);
    boom(n + 1);   /* grows AWAY from any stopping point */
}

int main(void) {
    boom(0);
    return 0;
}
```

Each call pushes a frame and never returns. After a few hundred thousand frames the stack is exhausted and the process dies with `SIGSEGV`. Under AddressSanitizer the report names the cause directly:

```text
=================================================================
==412831==ERROR: AddressSanitizer: stack-overflow on address 0x7ffc2e1a9ff8 (pc 0x5612aa0431d8 bp 0x7ffc2e1aa000 sp 0x7ffc2e1a9000 T0)
    #0 0x5612aa0431d8 in boom /tmp/recur.c:6
    #1 0x5612aa0431f2 in boom /tmp/recur.c:8
    #2 0x5612aa0431f2 in boom /tmp/recur.c:8
    #3 0x5612aa0431f2 in boom /tmp/recur.c:8
    [... hundreds of identical frames, all in boom ...]
    #425381 0x5612aa0431f2 in boom /tmp/recur.c:8
    #425382 0x5612aa04323a in main /tmp/recur.c:12

SUMMARY: AddressSanitizer: stack-overflow /tmp/recur.c:6 in boom
```

How to read it:

- `stack-overflow` (not `SEGV`, not `heap-use-after-free`) means the stack hit its limit.
- Hundreds of identical frames in one function mean uncontrolled self-calls. The fix is never at the crash line. The fix is the missing or unreachable base case.
- `boom(n + 1)` grows away from zero, so even a `n == 0` check placed after the call would never trigger. The base-case test must come **first** and the argument must move **toward** it.

Two subtler variants of the same bug:

1. **Base case exists but is skipped.** `if (n == 0) return;` placed *after* the recursive call never runs in time. Order matters.
2. **Step does not shrink.** `factorial(n)` calling `factorial(n)` instead of `factorial(n - 1)` re-pushes the identical frame forever. The argument must be strictly smaller by whatever measure the base case tests.

A defensive habit: during development, add a depth guard to see the blowup coming:

```c
static void countdown_debug(unsigned int n, unsigned int depth) {
    if (depth > 1000000) {
        fprintf(stderr, "depth guard tripped at n=%u\n", n);
        return;
    }
    if (n == 0) {
        printf("0\n");
        return;
    }
    printf("%u\n", n);
    countdown_debug(n - 1, depth + 1);
}
```

The guard is scaffolding, not the fix. The fix is a correct base case plus a shrinking step. Remove the guard once both are proven.

---

## Recursion vs Iteration: When to Use Which

Recursion is a tool with a per-call cost. Use it where the problem is self-similar and the depth stays small. Use a loop where the problem is a flat repetition.

| | Recursion | Iteration |
| --- | --- | --- |
| Stack cost | O(depth) frames | O(1) |
| Natural fit | tree walks, divide and conquer, linked-list structure | counting, scanning arrays, countdowns |
| Failure mode | stack overflow on deep input | infinite loop on wrong condition |
| Readability | mirrors the definition (`n!`, `sum_digits`) | mirrors the machine (accumulator, index) |

Rules of thumb for this course:

1. **Prefer iteration for linear counting.** `factorial`, `countdown`, array scans: a loop is shorter, faster, and cannot overflow the stack. The recursive versions in this part are teaching tools, not production recommendations.
2. **Prefer recursion when the data is recursive.** A linked list node holds a pointer to a smaller list. A binary search halves an interval into a smaller interval. The code follows the shape of the data (part 03 leans on this).
3. **Check the depth before choosing recursion.** Depth O(log n) — binary search, balanced divide and conquer — is always safe. Depth O(n) on user-controlled `n` — countdown to a billion, factorial of a million — is a stack overflow waiting to happen. If depth can be linear in untrusted input, write the loop.
4. **Every recursion must pass the three-question test:**

```text
  [ ] What is the base case, and does it need no further calls?
  [ ] Is the recursive argument strictly smaller by the base case's measure?
  [ ] Is the maximum depth bounded (small constant, or O(log n))?
```

If any answer is no, rewrite as a loop or redesign the decomposition.

---

## The Complete Program

One file exercising all three correct recursions, compiled with the course flags:

```c
#include <stdio.h>
#include <stddef.h>

static void countdown(unsigned int n) {
    if (n == 0) {
        printf("0\n");
        return;
    }
    printf("%u\n", n);
    countdown(n - 1);
}

static unsigned long long factorial(unsigned int n) {
    if (n == 0) {
        return 1ULL;
    }
    return (unsigned long long)n * factorial(n - 1);
}

static unsigned int sum_digits(unsigned int n) {
    if (n < 10) {
        return n;
    }
    return (n % 10) + sum_digits(n / 10);
}

int main(void) {
    countdown(3);
    printf("5! = %llu\n", factorial(5));
    printf("sum_digits(1234) = %u\n", sum_digits(1234));
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer recur.c -o recur
./recur
```

```text
3
2
1
0
5! = 120
sum_digits(1234) = 10
```

Silent under the sanitizer. Silence is the passing state: no stack overflow, no undefined behavior, every base case reached.

---

## Common Bugs

| Bug | Symptom | Fix |
| --- | --- | --- |
| No base case | `stack-overflow`, hundreds of identical frames | add the trivial-input branch first |
| Base case after the call | same overflow; check never reached | test for base case before recursing |
| Step does not shrink (`f(n)` calls `f(n)`) | same overflow on every input | pass a strictly smaller argument |
| Step grows (`boom(n+1)`) | overflow, values increase in output | move the argument toward the base case |
| Deep linear recursion on huge `n` | overflow on large inputs only | rewrite as a loop |
| Signed overflow in `factorial` (`int`) | UBSan `signed integer overflow` | use `unsigned long long`, keep `n <= 20` |

The AddressSanitizer report for the last row looks like this when `factorial` uses plain `int` and `n` reaches 13:

```text
=================================================================
==412902==ERROR: UndefinedBehaviorSanitizer: signed integer overflow: 479001600 * 13 cannot be represented in type 'int'
    #0 0x5612aa043240 in factorial /tmp/recur.c:9
    #1 0x5612aa0432a1 in main /tmp/recur.c:18

SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior /tmp/recur.c:9 in factorial
```

The recursion is fine; the arithmetic type is wrong. That distinction — structural bug vs type bug — is exactly what the sanitizer pair (`address` for the stack, `undefined` for the arithmetic) is for.

---

## Checklist

```text
[ ] every recursive function has a base case tested FIRST
[ ] the recursive argument is strictly smaller by the base case's measure
[ ] locals are treated as per-frame private (no cross-frame assumptions)
[ ] maximum depth is bounded: constant, digits, or O(log n) — never O(user input)
[ ] linear counting problems use a loop unless the exercise asks for recursion
[ ] factorial uses unsigned long long and test inputs stay at n <= 20
[ ] helpers are static; read-only helpers take const where applicable
[ ] builds with -fsanitize=address,undefined, runs with no diagnostics
```

---

#### [Previous](../../2-data-structures/challenge.md) | [Next](./02-part.md)
