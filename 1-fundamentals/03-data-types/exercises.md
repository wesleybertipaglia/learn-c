# Exercise List 3 - Data Types

## Exercise 1: Size Report

Write a program that prints the `sizeof` of every basic type, with the correct `printf` format for each one.

```text
# output
char:           1 bytes
short:          2 bytes
int:            4 bytes
long:           8 bytes
long long:      8 bytes
unsigned int:   4 bytes
float:          4 bytes
double:         8 bytes
long double:   16 bytes
```

Requirements:

- Print the size and the type name side by side, using width specifiers so the columns line up.
- The output must be produced by **code that works on any machine**. Do not hardcode the numbers.
- Write a comment next to each line explaining why that format specifier is the correct one.
- Then try compiling with `printf("%d\n", sizeof(int));` and read the warning. Add the right specifier (`%zu` for `size_t`, `%lu` with a cast if you insist, or `PRIu64` from `<inttypes.h>` for a fixed-width type).

## Exercise 2: One Integer, Three Views

Read an `int` from the user and print it three ways:

1. In hexadecimal, using `%x`.
2. In decimal.
3. Byte by byte, from the most significant byte to the least significant byte, showing each byte in hex and the character it represents if it is printable.

```text
# input
Number: 123456

# output
decimal : 123456
hex     : 1e240
bytes   : 00 00 01 E2
chars   : . . . non-printable(226)
```

Hints:

- `sizeof(int)` tells you how many bytes to loop over.
- Shift right by 8 bits for each step: `n >> (8 * i)` with `i` counting down from `sizeof(int) - 1` to 0.
- Mask each byte with `0xFF` so you print only that byte: `(n >> (8 * i)) & 0xFF`.
- Use `isprint` from `<ctype.h>` to decide whether to show a character.
- Extra credit: also show the bytes in reverse order (little endian first) and explain in comments why the order differs.

## Exercise 3: Where Does the Overflow Come From?

Read a `long long` from the user using `scanf` with `%lld`, then double it and print the result.

```text
# input
Value: 9223372036854775807

# output
read    : 9223372036854775807
doubled : -9223372036854775808
```

Requirements:

- Now run it again and type a value **bigger** than `LLONG_MAX`, for example `9223372036854775807` followed by more digits.
- Explain in comments what `scanf` did with that input. Compare the return value against 1.
- Explain in comments why doubling `LLONG_MAX` is undefined behavior, and why the value you see printed is not guaranteed to be the same with `-O2`.
- Then make the program safe: compare against `LLONG_MAX / 2` before doubling, and print `overflow` instead of computing.

```c
/* the safe version */
if (value > LLONG_MAX / 2) {
    printf("overflow\n");
} else {
    printf("doubled : %lld\n", value * 2);
}
```

## Exercise 4: Signed Bytes, Unsigned Bytes

Write a program that takes a number from 0 to 255 and stores it in **both** a `signed char` and an `unsigned char`, then explains the difference.

```text
# input
Value: 200

# output
unsigned char: 200
signed char  : -56
bits         : 11001000
```

Requirements:

- Explain in comments why `signed char` can hold 0..127 while `unsigned char` holds 0..255 with the same single byte.
- Print the binary representation of the byte. Extract each bit with a mask and a shift (`value & (1 << i)`).
- Ask the user for a value from -128 to 127 too, store it in a `signed char`, print it back, and explain why the value survives.
- Then print `CHAR_MIN` and `CHAR_MAX` from `<limits.h>` and explain in comments that plain `char` is signed on x86 but unsigned on ARM and PowerPC — which is why you should use `int8_t` or `signed char` when the signedness matters.

```c
#include <limits.h>

/* On x86-64 Linux: CHAR_MIN = -128, CHAR_MAX = 127  (char is signed)
   On ARM Linux:    CHAR_MIN = 0,    CHAR_MAX = 255 (char is unsigned) */
printf("char range: %d to %d\n", CHAR_MIN, CHAR_MAX);
```

## Exercise 5: Exact Widths with `<stdint.h>`

Write a program that reads a byte value from 0 to 255 and prints it in hex, in binary, and as a signed and an unsigned value.

```text
# input
Byte: 173

# output
hex     : AD
binary  : 10101101
unsigned: 173
signed  : -83
type    : int32_t=4 uint8_t=1
```

Requirements:

- Read the input with `scanf("%u", ...)` into an `unsigned int`, check the return value, and reject anything above 255.
- Store it in a `uint8_t`. Print it with `printf("%u\n", (unsigned)value)` — casting to `unsigned` because `uint8_t` promotes to `int` in the argument list.
- Print the binary form with a loop from bit 7 down to bit 0.
- Derive the signed value by casting the same byte to `int8_t` and printing it with `%d`.
- Print `sizeof` for `int32_t`, `uint8_t`, `int_least16_t` and `int_fast32_t`, using `%zu` every time.
- Explain in comments why this program produces identical output on a 32-bit and a 64-bit machine, while a program using `long` would not.

---

## Extra Challenge: Type Detective

Without using `sizeof`, determine the range of each integer type on the machine by testing the limits directly:

```c
int i = 0;
while (i + 1 > i) {
    i++;      /* stops when i is INT_MAX, because i + 1 overflows */
}
printf("INT_MAX found as %d\n", i);
```

Run it for `int`, `long`, `long long` and `unsigned char`. Then explain in comments:

- Why the loop with a **signed** `int` is safe (overflow is undefined, but the comparison `i + 1 > i` is never true under the standard's rules, so the loop condition is always false at the limit and the optimizer may remove the loop entirely).
- Write the same loop with `unsigned char` and explain why that one is well defined and wraps.
- Which version can be trusted to print the real limit? Only the unsigned one.

---

#### [Previous](./04-part.md) | [Next](../04-operators/01-part.md)