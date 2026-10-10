# Exercise List 6 - Loops

Six exercises to consolidate what you learned about `while`, `do-while` and `for`. Every one of them compiles with zero warnings under:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

Write the loop, run it, compare your output with the blocks shown here, then break it on purpose and fix it. Reading the loop out loud — "start at, until when, step by" — catches most mistakes before the compiler ever sees them.

---

## Exercise 1 — Numbers 1 to N

Read an integer `N` and print the numbers from 1 to `N`, separated by spaces.

**Prefer `for`.** The loop counts items from a known start to a known end, which is exactly what the header is for.

```text
How many numbers? 1 2 3 4 5
```

Also check the edge cases: `N = 1` prints one number, and `N = 0` prints none at all — which is why the condition is `i <= n`, not `i < n`. A loop that runs zero times is a valid loop, not a bug.

Extra: a second pass that prints the same range backwards, from `N` down to 1. That one starts at `i = n` and steps with `i--`.

---

## Exercise 2 — Factorial

Read a non-negative integer `N` and print `N!`. Define `0!` as `1`.

```text
n = 10! = 3628800
```

**Prefer `for`,** because the answer is built by multiplying every value in a range, and the header makes that range obvious.

Use `long long` for the accumulator. `20!` is `2432902008176640000`, which just fits. `21!` does not, and signed integer overflow is undefined behaviour — on this machine you get:

```text
n = 21! = -4249290049419214848
```

No warning, no error, and a plausible-looking negative answer. That is the whole reason to know where the limits are before you start multiplying.

---

## Exercise 3 — Fibonacci, First 10 Terms

Print the first 10 terms of the Fibonacci sequence, where each term is the sum of the two before it, starting `0, 1`.

```text
0 1 1 2 3 5 8 13 21 34
```

**Prefer `for` with a fixed count,** since the number of iterations is known before you start.

The trap: you need the last two terms at the same time, so you cannot overwrite one before saving it. Compute the next value into a temporary, then shift. A `while` loop with a counter would also work, but a fixed `for (int i = 0; i < 10; i++)` states the requirement better.

---

## Exercise 4 — Sum of the Digits

Read an integer and print how many digits it has and the sum of those digits. Negative input must work (count and sum the digits of its absolute value), and `0` has one digit.

```text
integer = digits = 5, sum = 25
```

for `90817`, and

```text
integer = digits = 5, sum = 15
```

for `-12345`.

**Prefer `do-while`,** because a number must be processed at least once — `0` has a digit, and a `while` loop would skip it and leave the digit count at zero.

The tools are `% 10` to take the last digit and `/ 10` to remove it (review the Operators chapter). Remember the sign: `n < 0 ? -n : n` gives the magnitude without changing the original.

---

## Exercise 5 — Prime Checker

Read an integer and decide whether it is prime. A prime has exactly two divisors, 1 and itself, so 0 and 1 are not prime.

```text
n = 7 is prime
n = 91 is not prime
n = 7919 is prime
```

`91 = 7 * 13`, so `d = 7` finds the divisor. `7919` is prime, so the loop runs all 87 iterations from `d = 2` to `d = 88` without finding anything, and the function returns `1`.

**Prefer `for`,** and stop as soon as you find a divisor with `break` — there is no point testing the rest.

**Why `sqrt(n)` is enough:** if `n = a * b` and both `a` and `b` are greater than `sqrt(n)`, then `a * b > sqrt(n) * sqrt(n) = n`, which is a contradiction. So every composite number has at least one divisor at or below `sqrt(n)`. Testing to `n` does the same work and is thousands of times slower:

```c
for (int d = 2; d < n; d++)           { /* correct, but checks everything */ }
for (int d = 2; (long)d * d <= n; d++) { /* stops at the square root */ }
```

On the machine this course was written on, testing `n = 1000000007`:

```text
naive (d < n): 1 in 1.457 s
sqrt  (d*d<=n): 1 in 0.000 s
```

The naive version did 1,000,000,005 divisions; the square root version did 31,621 — about 31,600 times fewer.

Write the condition as `d * d <= n` with `d` promoted to `long` (`(long)d * d`). Plain `d * d` overflows `int` once `d` passes 46340, and a negative or wrapped product fails the `<= n` test, so the loop ends early and the program calls a prime composite. With `n = 2147483647` (`INT_MAX`, which is prime) the broken version answers:

```text
int  d*d: 0
long d*d: 1
```

Neither line is right — `INT_MAX` is prime, so only the second version is correct. And this bug is invisible for every `n` below about 2 billion, so it ships.

Also guard `n < 2` before the loop. Without that check, `0` and `1` reach the loop body and the program answers "prime".

---

## Exercise 6 — Number Guessing Game

The program picks a secret number between 1 and 100 and gives the player 5 attempts. Each wrong guess reports "too high" or "too low". Reading input stops after the attempts are used, and the program says whether the player won.

Winning run:

```text
I am thinking of a number between 1 and 100.
You have 5 attempts.
attempt 1:   too high
attempt 2:   too low
attempt 3:   too low
attempt 4:   too high
attempt 5:   correct, in 5 attempts
```

Losing run:

```text
I am thinking of a number between 1 and 100.
You have 5 attempts.
attempt 1:   too low
attempt 2:   too low
attempt 3:   too low
attempt 4:   too low
attempt 5:   too low
out of attempts, the number was 42
```

**Prefer `while` with an attempts counter,** because the loop's exit depends on something computed inside the body (the guess) and on a limit that is not the loop variable. Keep the counter outside the loop and increment it explicitly — the "forgot the update" infinite loop from part 1 is one character away.

Two details worth getting right:

- Non-numeric input must not consume an attempt. Check `scanf("%d", &guess) != 1`, discard the rest of the line, and `continue` without incrementing.
- Use `break` when the player wins so you do not print "out of attempts" afterwards, and a flag so you can tell the two endings apart.

---

## Checklist Before You Run

| Check | Why |
| --- | --- |
| The counter is initialized to the right starting value | Off-by-one at the first iteration |
| The condition is `< n` when counting items, `<= n` when the last value is included | Off-by-one at the end |
| The update moves toward the condition | Infinite loop |
| The loop terminates for every input, including 0, 1 and negative values | Hangs on the edge cases |
| `scanf` return values are checked | Garbage input becomes garbage output |
| No unused variables and no sign-compare warnings | The compiler is telling you about a real defect |

---

#### [Previous](./03-part.md) | [Next](../07-functions/01-part.md)