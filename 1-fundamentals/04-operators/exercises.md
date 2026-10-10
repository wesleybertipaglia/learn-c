# Exercise List 4 - Operators

## Exercise 1: Swap Without a Temporary

Read two integers and swap them three different ways.

```text
# input
a = 7
b = 13

# output
start:    a = 7, b = 13
arithmetic: a = 13, b = 7
xor:      a = 7, b = 13
restored: a = 13, b = 7
```

Requirements:

- **Arithmetic swap**: `a = a + b; b = a - b; a = a - b;`. Explain in comments why the order matters, and what breaks if the two values are large enough that `a + b` overflows.
- **XOR swap**: `a ^= b; b ^= a; a ^= b;`. Explain in comments why this works when `a` and `b` are distinct variables, and why `a ^= a; b ^= a; a ^= b;` (the classic broken version) fails when both variables hold the same value.
- **Temporary variable swap** with `const char *tmp` style indirection is not possible for ints; instead do it with a third `int` and explain why that version is the one to use in real code.

## Exercise 2: Average That Works

Read two integers and print the average correctly, in three different ways.

```text
# input
a = 1
b = 2

# output
integer division: 1
double operand:   1.50
cast:            1.50
```

Then explain in comments why `(a + b) / 2` and `(a + b) / 2.0` differ, and what happens if `a` and `b` are large enough that `a + b` overflows an `int`. Add a safe version that adds as `long long`.

## Exercise 3: Leap Year

Read a year and decide whether it is a leap year.

The rule:

1. A year is a leap year if it is divisible by 4,
2. **except** if it is also divisible by 100,
3. **except** if it is also divisible by 400.

So 2024 is a leap year, 1900 is not, 2000 is.

```text
# input
Year: 1900

# output
1900 is NOT a leap year
```

Write the solution twice: once with `if`/`else if`, and once with a single boolean expression using `&&`, `||` and `%`. Explain in comments why the conditions must be ordered the way they are, and why removing the `% 400` clause is the most common bug in this exercise. Add a loop that lists all leap years from 2000 to 2100.

## Exercise 4: Decimal to Binary Without `/` or `%`

Convert a number to binary using only bit operations.

```text
# input
Number: 173

# output
173 in binary is 10101101
```

Requirements:

- First version: loop with a mask. Find the highest set bit with `1 << (bits - 1)`, then walk down printing `value & mask`. Explain in comments why you must start from the highest set bit and not from bit 0.
- Second version: the division-by-2 approach, using `>>` for the quotient and `& 1` for the remainder, collecting digits into an array in reverse order and then printing them backwards.
- Third version: recursive. Explain why the recursive version prints from the most significant digit naturally.
- Print `%d` for each bit separated by no spaces, and print the value in hex next to it.

## Exercise 5: User Permissions with Flags

Simulate a permission system with four flags packed into one byte.

```text
#define PERM_READ  1u   /* 0001 */
#define PERM_WRITE 2u   /* 0010 */
#define PERM_EXEC  4u   /* 0100 */
#define PERM_ADMIN 8u   /* 1000 */
```

```text
# input
Give the user permissions (1=read, 2=write, 4=exec, 8=admin), or 0 to start:
5

# output
Permissions: read=yes write=yes exec=yes admin=no (raw 5)
```

Requirements:

- Start with `unsigned char perms = 0;` and loop forever asking for a command.
- Commands: `r` (set read), `w` (set write), `x` (set exec), `a` (set admin), `c` (clear all), `t` (toggle read), `q` (quit).
- Use `|=` to set, `&= ~` to clear and `^=` to toggle. Never assign a literal mask value.
- Print the raw value in hex and the four flags in a readable way.
- Explain in comments why `unsigned char` is a better choice than `int` here, and why the max value with 4 flags is 15.

## Exercise 6: What Does This Evaluate To?

Write a program with **no `printf`** that forces the compiler to evaluate each expression, then print your predictions and the real results side by side.

Predict the value of each expression, then check it:

| Expression | Your prediction |
| --- | --- |
| `5 / 2` | |
| `-5 / 2` | |
| `-5 % 2` | |
| `5 / 2.0` | |
| `(char)300` | |
| `1 << 31 >> 31` | |

Requirements:

- Use a `volatile int` or a `volatile double` for the operands so the compiler cannot fold the expressions away, and print the results with the correct specifier for each type.
- Explain each one in comments:
  - Why `5 / 2` is `2` (integer division, truncation).
  - Why `-5 / 2` is `-2`, not `-2.5` and not `-3`.
  - Why `-5 % 2` is `-1`, and how the sign follows the dividend.
  - Why `5 / 2.0` is `2.5` and what type the expression has.
  - Why `(char)300` is `44`: the value does not fit in a `char`, so it is reduced modulo 256, and `44` is `,`.
  - Why `1 << 31` is the most negative `int` (0x80000000) and why shifting that right again is **implementation-defined** for a signed value — then show the unsigned version, `(unsigned)1 << 31 >> 31`, which gives `1` on every platform.

```text
5 / 2                 = 2
-5 / 2                = -2
-5 % 2                = -1
5 / 2.0               = 2.500000
(char)300             = 44
1 << 31 >> 31         = -1   (implementation-defined: -1 or 0)
(unsigned)1 << 31 >> 31 = 1  (defined everywhere)
```

---

## Extra Challenge: Operator Precedence Quiz

Write a program that prints the result of these expressions, then explain each one. Add parentheses until the result is obvious to a human reader.

```c
1 + 2 * 3 == 7
1 & 2 == 0
!0 + 1
1 << 1 + 1
x = y = z = 0
5 & 3 | 8 ^ 2
1 ? 2 : 3 ? 4 : 5
'a' + 1
(char)('a' + 1)
sizeof(int) > -1
```

The last one is interesting: `sizeof` produces an **unsigned** value, so comparing it with `-1` converts the `-1` to a huge unsigned number and the comparison is always true.

---

#### [Previous](./04-part.md) | [Next](../05-conditionals/01-part.md)