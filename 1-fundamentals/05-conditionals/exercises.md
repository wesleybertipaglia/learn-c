# Exercise List 5 - Conditionals

## Exercise 1: Positive, Negative or Zero

Read an integer and classify it.

```text
# input
Number: -7

# output
-7 is negative
```

Requirements:

- Solve it twice: once with `if`/`else if`/`else`, and once with a single `?:` expression printing all three messages.
- Then try with a `double` input and explain in comments why `-0.0` prints as `-0.000000` and why `x == 0.0` is true for it.
- Bonus: read a `char` and classify it as digit, uppercase, lowercase or other using `if` chains with `'0'`..`'9'` ranges.

## Exercise 2: Largest of Three Numbers

Read three numbers and print the largest, then the smallest, then the middle one.

```text
# input
a = 12
b = 47
c = 23

# output
largest: 47
smallest: 12
middle: 23
```

Requirements:

- Solve with nested `if`/`else` first.
- Then solve with `?:` only, without a single `if`.
- Explain in comments why comparing three numbers pairwise needs at least **two** comparisons to find the largest, and how many comparisons the naive version does.
- Bonus: accept a variable number of integers (read until the user enters 0) and find the largest in a `while` loop. See the Loops chapter, but try it now if you can.

## Exercise 3: Calculator with `switch`

Read two numbers and an operator character, then compute the result.

```text
# input
Number A: 12
Operator: +
Number B: 8

# output
12 + 8 = 20
```

Requirements:

- Support `+`, `-`, `*`, `/` and `%`. Use `switch` on the operator.
- `default` must reject anything else.
- Handle division by zero: for `/` check `b == 0` and print `error: division by zero`; for `%` check `b == 0` and print `error: modulo by zero`.
- Explain in comments why the read of the operator needs a leading space: `scanf(" %c", &op)`.
- Handle the "leftover newline" trap deliberately: enter the operator first with `scanf(" %c")`, then the second number with `scanf("%lf")`, and show what happens if you swap the order without the space.

Bonus: add an exponent case `^` implemented with a loop (there is no `**` operator in C), and reject negative exponents.

## Exercise 4: FizzBuzz

Read an integer `n` and print the FizzBuzz sequence from 1 to `n`.

For each number:

- divisible by both 3 and 5: print `FizzBuzz`
- divisible by 3 only: print `Fizz`
- divisible by 5 only: print `Buzz`
- otherwise: print the number itself

```text
# input
n = 15

# output
1
2
Fizz
4
Buzz
Fizz
7
8
Fizz
Buzz
11
Fizz
13
14
FizzBuzz
```

Requirements:

- Write it with `if`/`else if`, checking the "both" case **first**. Explain in comments what happens if you check 3 before checking both, and paste the broken output.
- Then rewrite it using a single `?:` chain or a `switch`, which makes the "both first" ordering visually obvious.
- Reject `n <= 0` and any input above 10000 (a `while` loop would hang).
- Explain in comments why `i % 15 == 0` is equivalent to `i % 3 == 0 && i % 5 == 0`, and when that shortcut stops being true.

## Exercise 5: Triangle Validator

Read three side lengths and decide whether they can form a triangle.

The rule: the sum of any two sides must be **greater than** the third. Equality produces a degenerate (flat) figure, not a triangle.

```text
# input
a = 3
b = 4
c = 5

# output
3, 4, 5 form a right triangle
perimeter: 12
```

Requirements:

- Print `X, Y, Z form a valid triangle`, or `X, Y, Z do not form a valid triangle` when a sum is less than or equal to the third side.
- For a valid triangle, also print the perimeter, the area (using Heron's formula with `sqrt` from `<math.h>`, compile with `-lm`) and, if `a*a + b*b == c*c` for any permutation, say which kind of triangle it is: equilateral, isosceles, right or scalene.
- Reject negative sides.
- Explain in comments why `<=` and not `<` in the triangle rule, and what a degenerate triangle looks like: `(1, 2, 3)` has a perimeter of 6 and an area of 0.

```text
# input
a = 1
b = 2
c = 3

# output
1, 2, 3 do not form a valid triangle
```

## Exercise 6: Rock, Paper, Scissors

Play against the computer until the player quits.

```text
# input
Your choice (r/p/s, q to quit): r
Computer chose: s

You win!

Your choice (r/p/s, q to quit): p
Computer chose: r

You lose!

Your choice (r/p/s, q to quit): q
Final score: 1 wins, 1 losses
```

Requirements:

- Use `rand()` and `srand()` from `<stdlib.h>`, seeding once at the start with `srand((unsigned)time(NULL))` from `<time.h>`. Include both headers.
- Explain in comments why `srand` is called **once**, before the loop, and what happens if you call it inside the loop: you get the same "random" sequence every time.
- Use `rand() % 3` to pick the computer's move, and explain in comments why `% 3` is slightly biased (RAND_MAX is not a multiple of 3) and why that does not matter for a game.
- Detect the winner with `if`/`else if`: paper beats rock, scissors beats paper, rock beats scissors, and equal choices are a draw.
- Keep a score of wins, losses and draws across rounds, and print it when the player quits.
- Reject invalid input with `default`, do not consume it, and do not count the round.
- Bonus: refuse to draw while the player keeps losing (use `rand() % 2` after a loss), and comment on why that is not fair.

---

## Extra Challenge: A Small Decision Table

Read an HTTP-like status code and print a human message using `switch`.

```text
# input
Status code: 404

# output
404 Not Found
```

Requirements:

- Handle 200, 201, 204, 301, 304, 400, 401, 403, 404, 500 and 503 with `switch`.
- Add a fallback that classifies ranges: 2xx is success, 3xx is redirection, 4xx is a client error, 5xx is a server error.
- Explain in comments why you cannot write `case 200 ... 299:` (there is no range syntax in C) and what you would use instead.

---

#### [Previous](./03-part.md) | [Next](../06-loops/01-part.md)