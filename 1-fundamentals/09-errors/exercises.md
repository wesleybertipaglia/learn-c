# Exercise List 9 - Error Handling

Everything below uses only variables, functions, pointers and the error model from this module. There are no structs and no dynamic arrays yet, so you will use fixed-size arrays where a program needs storage.

For every program:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g file.c -o file
```

Zero warnings is the bar. If you need to demonstrate a bug, quote the warning in your notes and fix it in the same file with a comment.

---

## Exercise 1: `safe_open`

Write a function that opens a file and never lets a failure escape as a raw crash:

```c
FILE *safe_open(const char *path);
```

Requirements:

1. It returns the `FILE *` on success.
2. It returns `NULL` on failure, and calls `perror` with a label that names the file it was trying to open, not just the bare operation.
3. `main` checks the result before using it, prints a friendly message on stderr, and returns `1`.
4. On success it writes one line to the file, closes it, and returns `0`.

Test both branches. Running the program with no arguments should print the `perror` message you saw in part 01.

---

## Exercise 2: Rewrite a Fallible Function

Take a function that either works or silently produces garbage. Start from this one:

```c
int half(int n) {
    return n / 2;
}
```

Rewrite it so that it returns a status:

```c
int half_checked(int n, int *out);
```

Requirements:

1. Returns `0` on success and writes the result to `*out`.
2. Returns `-1` when the input is invalid (negative `n`, or `out == NULL`), after printing an explanation with `fprintf(stderr, ...)`.
3. `main` calls it twice: once with a valid value and once with an invalid one, and handles each outcome differently — one prints a result, the other prints a message and does not use `*out`.
4. Comment the original `half` and write one sentence explaining why the rewrite is better.

Bonus: use an `enum` instead of `-1`, with named values, and a `status_text` function that turns them into English.

---

## Exercise 3: Add Assertions, Then Defend Them

Take a small program — an average function, a list index lookup, or anything from the loops chapter. Add `assert` statements for the invariants you believe must always hold.

Requirements:

1. At least three assertions, each on a fact your own code guarantees.
2. A short comment above each one saying why it cannot be false.
3. Run it and paste the normal output.
4. Then temporarily break one assumption, compile, and paste the real abort message.
5. Build again with `-DNDEBUG` and paste the output. Note what changed.
6. Then, in prose: list which of your assertions would be **wrong** if the value being checked came from the user instead of from your code, and rewrite those as `if` statements with status returns.

The last item is the point of the exercise. An assertion that guards user input is a bug.

---

## Exercise 4: The Cleanup Label

Write a function that allocates two buffers and can fail at four points:

```c
int build_two_buffers(const char *left_text, const char *right_text,
                      int *left_out, int *right_out);
```

Requirements:

1. `malloc` two buffers of 10 `int`s each. Check both for `NULL`.
2. Parse `left_text` and `right_text` with `strtol`, and fail cleanly if either is not a number.
3. Use a single `cleanup:` label. No repeated `free` blocks.
4. On success, copy the results to `*left_out` / `*right_out` (plain `int[10]` arrays in `main`) and return `0`.
5. On any failure, free whatever was acquired and return `-1`.
6. `main` must not free anything on the failure path, and must not touch the out-arrays.
7. Verify with a run that succeeds and a run that fails, and paste both.

Bonus: explain in a comment why `free(NULL)` makes the cleanup block correct even when the first `malloc` failed.

---

## Exercise 5: Division Without Aborting

Write:

```c
int divide_checked(int a, int b, int *out);
```

Requirements:

1. Returns `STATUS_OK` and writes `a / b` to `*out` when `b != 0`.
2. Returns `STATUS_INVALID` without touching `*out` when `b == 0` or `out == NULL`.
3. `main` loops over a small table of input pairs, including `(1, 0)`, `(0, 5)` and `(-7, 2)`. For each one it prints either the quotient or the error, and keeps going — **it does not stop at the first error.**
4. Count successes and failures, print the totals at the end, and return `0` even though some divisions failed (those were handled, not fatal).

Bonus: what happens if you return `a / b` directly instead of using the out-parameter when `b == 0`? Run it and describe the result — do not ship it.

---

## Exercise 6: Move the Messages to `stderr`

Take one of your earlier programs that reports problems with `printf`. Requirements:

1. Every diagnostic goes to `fprintf(stderr, ...)`.
2. Every actual result stays on `stdout`.
3. Prove it works:

```bash
./program > results.txt 2> problems.txt
```

Paste the contents of both files. `results.txt` must contain only results, in order, with no error text mixed in.

4. Pick two `printf` error calls and give each a `perror` version instead, with a label naming the operation. Paste the real output.

---

## What The Exercises Are Testing

| Exercise | Habit it builds |
| --- | --- |
| 1 | wrap a failing library call, check before use |
| 2 | every fallible function returns a status that gets checked |
| 3 | `assert` for invariants only; `if` for anything a user controls |
| 4 | one cleanup label, no duplicated `free`, no leaks |
| 5 | handle the error and continue, instead of bailing out |
| 6 | results on `stdout`, diagnostics on `stderr` |

A submission that compiles clean, handles every failure branch, and prints a readable message for each one is done. A submission that only works on the happy path is not.

---

#### [Previous](./03-part.md) | [Next](../10-debugging/01-part.md)