# Final Challenge of Module 1

You have finished the fundamentals: types, operators, conditionals, loops, functions, pointers, error handling and debugging. This project uses all of it, and nothing else. It is the last thing you build before module 2 introduces the features you are deliberately not allowed to use here.

---

## Objective

Write a **command-line bank account simulator** in C that manages a fixed number of accounts through a text menu, validates every piece of input the user types, and handles every failure with a status return instead of a crash.

The program must compile with zero warnings:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g bank.c -o bank
```

---

## Features

The menu, printed once at startup and again after every operation:

```text
  1) Create account
  2) Deposit
  3) Withdraw
  4) Show statement
  5) Apply monthly interest to all accounts
  6) Exit
```

| Option | Behaviour |
| --- | --- |
| 1 | Ask for an account holder name and an opening balance. Reject an empty name and a negative opening balance |
| 2 | Ask for an account number and an amount. Credit the account. Reject a negative amount and an unknown account |
| 3 | Ask for an account number and an amount. Debit the account **only if the balance is sufficient**. Otherwise print a message with the shortfall and change nothing |
| 4 | Print one account's number, holder and balance, or print all accounts |
| 5 | Apply a fixed monthly rate, for example 2% , to every account and print each new balance |
| 6 | Print a summary of how many accounts exist and the total balance across all of them, then exit |

---

## Requirements

### Storage

Use a fixed-size array of **parallel arrays**, since `struct` is not available yet:

```c
#define MAX_ACCOUNTS 10

static int    numbers[MAX_ACCOUNTS];
static char   names[MAX_ACCOUNTS][64];
static double balances[MAX_ACCOUNTS];
static int    account_count;
```

That is the shape you are limited to. It is awkward compared to what module 2 will give you, and that awkwardness is the point.

### Functions

Every menu option is a function. No option may be implemented inline in `main`.

```text
create_account   deposit   withdraw   show_statement   apply_interest   print_menu
```

Helper functions you will probably want:

```text
find_account     read_int   read_double   read_name   read_menu_choice
```

### Error handling

1. **Every function that can fail returns a status** and the caller checks it. No exceptions, no silent failure.
2. **Errors go to `stderr`** with `fprintf(stderr, ...)`. Results go to `stdout` with `printf`.
3. Use a named `enum` for statuses, not bare `-1`:

   ```c
   typedef enum {
       STATUS_OK = 0,
       STATUS_INVALID,
       STATUS_FULL,
       STATUS_NOT_FOUND,
       STATUS_INSUFFICIENT
   } status_t;
   ```

   and a `status_text` function that turns them into English for the user.
4. **Never ignore a return value**, including from `fgets` and `strtod`.

### Input validation

This is the part that separates a working program from a good one.

- `scanf` cannot be used to re-read a stream after a failed conversion: the bad characters stay in the buffer and every later read fails too. Use **`fgets`** to read a whole line into a buffer, then `strtol` or `strtod` on it.
- Validate with the C standard functions, not by hand: `strtol` returns `0` and sets `errno` to `EINVAL` when the text is not a number. Check `endptr` to detect trailing garbage.
- Reject out-of-range values with an `if`, never with `assert`. Your program must survive `-DNDEBUG`, and it must never abort because a user typed a letter.
- Loops that ask for input must **keep asking** until they get something valid. Do not exit the program on bad input.
- Bound every string copy. `strncpy` does not add a terminator for you, so set `dest[n] = '\0'` yourself, or check the length first and reject anything too long.
- Guard the array: `account_count < MAX_ACCOUNTS` before creating, and `0 <= index && index < account_count` before indexing. Remember `numbers[account_count]` is one past the end.

### Style

1. **`const` on every pointer parameter you do not modify.** `void print_menu(void)` and `static int find_account(const int *numbers, int count, int target)`.
2. **`static` on every helper function** that is not called from another file.
3. One declaration per line.
4. No global mutable state other than the account arrays above.
5. A comment above any function that returns a status, saying what each status means.

---

## Example of Operation

```text
$ gcc -std=c11 -Wall -Wextra -pedantic -g bank.c -o bank
$ ./bank
========================================
   Command-line Bank Account Simulator
========================================

  1) Create account
  2) Deposit
  3) Withdraw
  4) Show statement
  5) Apply monthly interest to all accounts
  6) Exit
========================================
Choice: 1
Holder name: Wesley
Opening balance: 1000
Account 1 created for Wesley.

Choice: 2
Account number: 1
Amount: 250.50
Deposited 250.50. New balance: 1250.50

Choice: 1
Holder name: Ana
Opening balance: 300
Account 2 created for Ana.

Choice: 3
Account number: 2
Amount: 500
error: insufficient funds. Balance is 300.00, shortfall is 200.00
Withdrawal refused. Balance unchanged.

Choice: 3
Account number: 2
Amount: 125.75
Withdrew 125.75. New balance: 174.25

Choice: 2
Account number: 99
Amount: 10
error: no account with number 99

Choice: 4
Account number: 1
----------------------------------------
Account 1
Holder:  Wesley
Balance: 1250.50
----------------------------------------

Choice: 4
All accounts:
  1  Wesley           1250.50
  2  Ana               174.25

Choice: 5
Monthly interest of 2.00% applied:
  1  Wesley           1275.51
  2  Ana               177.74

Choice: 6
----------------------------------------
Summary
Accounts: 2
Total balance: 1453.24
Thank you.
```

Two things to notice in that transcript:

- The failed withdrawal and the unknown account both print to `stderr` and **do not stop the program**. The menu comes back.
- The menu is reprinted after every operation, because the loop does not care whether the operation worked.

---

## Tips

### What you may not use

**You are not allowed to use `struct`, `malloc`, `calloc`, `realloc`, `free`, dynamic arrays, or any library that requires them.** Those all arrive in module 2 (Structs and Dynamic Memory). For this project:

```text
NOT ALLOWED                        USE INSTEAD
---------------------------------  --------------------------------
struct                             parallel arrays
malloc / free                      fixed-size arrays
variable-length arrays             MAX_ACCOUNTS constant
snprintf into a dynamic buffer     a fixed char array with a bound
```

If you find yourself wanting `malloc`, that is the correct instinct — write it down, then use a fixed array instead. Noticing what you would have needed is how you know what module 2 is for.

### `double balance` accumulates floating point error

Money in `double` is not exact. `0.1 + 0.2` is not `0.3`. Apply interest to a few accounts a few times and the totals stop matching what you expect:

```c
balance = balance * 1.02;    /* 100.00 -> 102.00, but 0.01 -> 0.0102 ... */
```

This is a deliberate teaching point, not a mistake in your code. Keep the balance as `double` (integers are worse for money), print with `%.2f`, and note in a comment that a real banking system would use integers of cents or a decimal type. Module 3 is where you would care about this properly.

### Read a line, then parse it

```c
static int read_int(const char *prompt, int *out) {
    char line[64];

    printf("%s", prompt);
    if (fgets(line, sizeof(line), stdin) == NULL) {
        return -1;                 /* end of input */
    }

    errno = 0;
    char *end = NULL;
    long value = strtol(line, &end, 10);

    if (end == line || errno == ERANGE || value < INT_MIN || value > INT_MAX) {
        return -1;
    }
    *out = (int)value;
    return 0;
}
```

For `double`, use `strtod` and check that `end != line`. If you use `scanf`, remember that `%d` for `int` and `%lf` for `double`.

### Guard the array every time

```c
static int find_account(const int *numbers, int count, int target) {
    int i;

    for (i = 0; i < count; i++) {
        if (numbers[i] == target) {
            return i;             /* valid only while i < count */
        }
    }
    return -1;
}
```

The caller must check the return value before using it as an index. `-1` is never a valid index.

### The money logic is the interesting part

`withdraw` has to do two things in the right order: check the balance, then change it. If you subtract first and check after, you have already corrupted the balance on the failure path.

```text
if (amount <= 0)                 -> STATUS_INVALID, nothing changed
if (index < 0)                  -> STATUS_NOT_FOUND, nothing changed
if (amount > balances[index])   -> STATUS_INSUFFICIENT, nothing changed
otherwise                       -> subtract, return STATUS_OK
```

Print the shortfall with `%.2f` so the user can see how much is missing.

---

## Extra challenge

If the required set is done and working, pick one of these:

1. **Interest on a minimum balance.** Accounts with a balance of at least 1000 earn 2%; the rest earn 0.5%. Show both rates in the output.
2. **Per-account transaction count.** Track how many deposits and withdrawals each account has had, in another parallel array, and print it in the statement. No `struct`, so three more arrays instead of one.
3. **Withdrawal limit.** Reject any single withdrawal above 500 with a distinct status and message. Add a sixth status to the enum.
4. **A search option.** Find every account whose holder name starts with a given letter, scanning the `char` array with `strncmp` so you never compare past the end.
5. **Reorder on removal.** Add option 7, "close account", and remove an entry by shifting the later elements down. Then prove the arrays stay consistent by showing a statement before and after. This is a real bug farm and worth every minute.

---

## Grading Checklist

### Compilation

```text
[ ] gcc -std=c11 -Wall -Wextra -pedantic -g bank.c -o bank, zero warnings
[ ] Also builds clean with -O2 -fanalyzer
[ ] Runs clean under -fsanitize=address,undefined
```

### Structure

```text
[ ] Every menu option is a function; nothing is written inline in main
[ ] Helpers are static
[ ] Pointer parameters that are not modified are declared const
[ ] One declaration per line
[ ] No struct, no malloc, no dynamic arrays anywhere
```

### Error handling

```text
[ ] Every fallible function returns a status, and every caller checks it
[ ] Statuses come from a named enum, with a status_text function
[ ] All error messages go to stderr; all results go to stdout
[ ] No return value is ignored anywhere, including fgets and strtod
[ ] No assert is used to validate anything the user typed
```

### Input handling

```text
[ ] fgets or an equivalent line read is used, so a bad line cannot wedge the loop
[ ] Numbers are parsed with strtol / strtod and validated
[ ] Out-of-range values are rejected with an if, and the program keeps going
[ ] Every array access is bounds checked first
[ ] Strings are copied with an explicit bound and a forced terminator
```

### Behaviour

```text
[ ] Option 1 rejects an empty name, a negative balance, and a full account list
[ ] Option 3 refuses an overdraft and leaves the balance untouched
[ ] Option 4 handles one account and all accounts
[ ] Option 5 applies interest to every account and prints the new balances
[ ] Option 6 prints a summary and exits with status 0
[ ] main returns 0 on normal exit, nonzero only on a real failure
```

### Discipline

```text
[ ] The sample transcript above is reproduced, or a better one is pasted
[ ] Every failure path in the code was tested by actually triggering it
[ ] Comments explain why a decision was made, not what the line does
```

A submission that runs, handles every menu option, rejects bad input without crashing, and compiles with zero warnings has met the bar. One that also survives a sanitizer run and refuses overdrafts correctly is well above it.

---

#### [Previous](./10-debugging/exercises.md) | [Next](../2-data-structures/01-arrays/01-part.md)