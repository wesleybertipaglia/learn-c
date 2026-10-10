# Exercise List 1 - Arrays

## Exercise 1: Reverse

Read 10 integers into an array and print them in reverse order, without modifying the array.

```text
# input
3 1 4 1 5 9 2 6 5 3

# output
original: 3 1 4 1 5 9 2 6 5 3
reversed: 3 5 6 2 9 5 1 4 1 3
```

The last valid index is `9`, so the countdown starts there and the condition is `i >= 0`. Watch out for the classic mistake of using `i > 0` and losing the first element of the output.

## Exercise 2: Statistics

Given an array of numbers, print the largest, the smallest, the average, and how many values are **strictly above** the average.

```text
# input
23 87 12 45 91 8 60 74

# output
largest: 91
smallest: 8
average: 50.00
above the average: 4
```

Requirements:

- Seed the max and the min with `values[0]`, not with `0`.
- Compute the average as `(double)total / COUNT`, or you get integer division.
- The count needs a second pass, because the average is only known after the first one.

## Exercise 3: Linear Search

Write this function:

```c
int linear_search(const int values[], int n, int target);
```

It walks the array and returns the **index** of the first match, or `-1` if the value is not there. Then read a value from the user and use it.

```text
# input
values:  42 17 93 8 55 17 60 3 93 29
search:  17

# output
found 17 at index 1
```

```text
# input
values:  42 17 93 8 55 17 60 3 93 29
search:  99

# output
99 not found
```

Requirements:

- The length is a parameter, not `sizeof(values)`.
- `-1` and not `0` for failure, because index `0` is a valid answer.
- Extra credit: report **every** index where the value appears, not just the first.

## Exercise 4: Digit Counter

Read an integer and count how many times each digit appears, using an array of 10 counters. Print the whole table so digits with a count of zero are visible too.

```text
# input
90210

# output
digit : count
  0  :    2
  1  :    1
  2  :    1
  3  :    0
  4  :    0
  5  :    0
  6  :    0
  7  :    0
  8  :    0
  9  :    1
```

Extract each digit with `rest % 10` and drop it with `rest /= 10`. Handle three details explicitly:

- A negative input: work on the absolute value so the `-` is not treated as a digit.
- The input `0`: it contains one digit, but a `while (rest > 0)` loop never runs for it. Handle it separately.
- The counters array must be initialized (`int counters[10] = {0};`), otherwise the "unused" digits print garbage.

## Exercise 5: Remove at an Index

Write this function from part 3:

```c
int remove_at(int arr[], int n, int index);
```

It removes the element at `index`, shifting the following elements left so the order is preserved, and returns the **new** size. An out-of-range index must leave the array untouched.

```text
# start
[ 10, 20, 30, 40, 50 ]  (n = 5)

# output
remove_at(values, n, 2) -> removed 30
now: [ 10, 20, 40, 50 ]  (n = 4)
remove_at(values, n, 0) -> removed 10
now: [ 20, 40, 50 ]  (n = 3)
remove_at(values, n, 5) -> index out of range
now: [ 20, 40, 50 ]  (n = 3)
remove_at(values, n, 2) -> removed the last one
now: [ 20, 40 ]  (n = 2)
```

Requirements:

- The shift loop runs left to right. Try running it backwards and explain in a comment why the array gets destroyed.
- Reject an invalid index with `index < 0 || index >= n`.
- Always assign the result: `n = remove_at(values, n, 2);` — the array's `sizeof` never changes, only `n` does.
- Removing the last element must work: the shift loop simply does not execute.

## Exercise 6: Matrix Transpose

Transpose a 3x3 integer matrix into a second matrix and print both. The transpose swaps rows and columns.

```text
# output
original:
     1   2   3
     4   5   6
     7   8   9

transposed:
     1   4   7
     2   5   8
     3   6   9
```

Requirements:

- Print with a width specifier so the columns line up.
- The printing function must take the row size as a parameter: `void print_matrix(const int m[][3], const char *label);`. Try `const int m[]` and read the compiler error.
- If `-pedantic` complains about passing a mutable matrix to a `const` parameter (it does, before C23), drop the `const` or make the caller's matrix `const`.
- Extra credit: transpose **in place**, without a second matrix, using a temporary variable. Ask yourself what that reveals about how a 2D array is laid out in memory.

---

#### [Previous](./03-part.md) | [Next](../02-strings/01-part.md)