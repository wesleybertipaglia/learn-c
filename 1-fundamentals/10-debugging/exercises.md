# Exercise List 10 - Debugging

These exercises are different from the earlier lists: **most of them hand you broken code on purpose.** Your job is not to write something that works from scratch, but to find the defect, explain it in writing, fix it, and prove the fix with a tool.

Two rules apply to everything here:

1. Paste the **real** output. If a tool reports something you did not run, the submission is wrong.
2. After fixing a bug, add a test case for it. A fix without a test is a coincidence.

---

## Exercise 1: The Off-By-One Loop

```c
#include <stdio.h>

int main(void) {
    int values[5] = {10, 20, 30, 40, 50};
    int total = 0;
    int i;

    for (i = 1; i <= 5; i++) {
        total += values[i];
    }

    printf("total = %d\n", total);

    return 0;
}
```

The intent is obvious: add up the five numbers, which is `10+20+30+40+50 = 150`.

Tasks:

1. Compile it with the full flags and run it. Paste the output.
2. Find the bug. Then answer with `gdb`: set a breakpoint on the `total +=` line, step through the loop, and paste the transcript showing the values of `i` and `total` at each stop.
3. Explain in two sentences what the loop does versus what it should do. Note that there are two off-by-one errors here, one on each side of the condition.
4. Fix it and paste the corrected output.
5. Bonus: with the original condition `i <= 5` and a *correct* start of `i = 0`, what does the loop read on its last pass? Say what that is called, and what it would print on a machine with different stack layout.

---

## Exercise 2: Reading Past the End of an Array

```c
#include <stdio.h>

int main(void) {
    int values[5] = {2, 4, 6, 8, 10};
    int i;
    int last_multiple = -1;

    for (i = 0; i <= 5; i++) {
        if (values[i] % 3 == 0) {
            last_multiple = values[i];
        }
    }

    printf("last multiple of 3 = %d\n", last_multiple);
    return 0;
}
```

The intent is to scan the five values and keep the last one divisible by 3, which is `6`.

Tasks:

1. Run it plain. It prints `last multiple of 3 = 0` — wrong, and a value that is not even in the array.
2. Compile with the sanitizers and run it again:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer over.c -o over
   ./over
   ```

   Paste the **entire** report. UBSan fires first with `index 5 out of bounds for type 'int [5]'`, then AddressSanitizer stops the program with `stack-buffer-overflow`. Identify the `ERROR:` line, the offending source line, and the memory map line showing `[32, 52) 'values'`.
3. Explain why the plain build did not crash. What is `values[5]` in that binary, and why did the `0` printed happen to be divisible by 3?
4. Fix it by giving the loop a correct bound. Paste the fixed output.
5. Bonus: rewrite the scan as a function taking `(int *values, int count)`, the way the earlier parts did, and show both versions agree. Then explain in one sentence why a function that receives only a pointer cannot know the array's length by itself.

---

## Exercise 3: The Leak

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *data = malloc(10 * sizeof(int));
    int *copy = malloc(10 * sizeof(int));
    int i;

    for (i = 0; i < 10; i++) {
        data[i] = i * i;
    }
    for (i = 0; i < 10; i++) {
        copy[i] = data[i] + 1;
    }

    printf("copy[9] = %d\n", copy[9]);

    free(data);
    return 0;
}
```

Tasks:

1. Run it plain. Paste the output. Note that everything looks correct.
2. Check for leaks with valgrind if it is installed:

   ```bash
   valgrind --leak-check=full --show-leak-kinds=all ./leak
   ```

   Paste the full report. If valgrind is not installed on your machine, say so plainly in your notes, then get the same diagnosis from LeakSanitizer:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address -fno-omit-frame-pointer leak.c -o leak_asan
   ./leak_asan
   ```

3. Label each block of the report: which category is `copy` in, and what is `data` in?
4. Fix the leak and re-run the tool. Paste the clean report, ending in zero bytes lost.
5. Bonus: change the fix so `copy` points at `data` instead of being a separate allocation, and explain what you gave up.

---

## Exercise 4: `=` Instead of `==`

```c
#include <stdio.h>

int main(void) {
    int n = 5;

    if (n = 5) {
        printf("true\n");
    } else {
        printf("false\n");
    }

    printf("n = %d\n", n);
    return 0;
}
```

Tasks:

1. Compile with `-Wall` and paste the **exact** warning gcc gives, with the option name in brackets.
2. Run it and paste the output. Then change `n` to `7` and run it again.
3. Explain, in terms of operator precedence, what the condition actually evaluates to. Why does gcc warn?
4. Fix it with `==`. Then, in a comment in the source, show how to write an assignment inside a condition **on purpose** with `if ((n = 5) > 0)`, and explain why the extra parentheses make it legal.
5. Bonus: find a case where the fix changes behaviour. What does `if (x = 0)` mean?

---

## Exercise 5: It Crashes Only for One Input

```c
#include <stdio.h>

static int slots[10];

static int store_at(long position, int value) {
    int index = (int)position;

    printf("storing %d at index %d of 10\n", value, index);
    slots[index] = value;
    printf("stored\n");
    return 0;
}

int main(void) {
    long position;
    int value;

    printf("position: ");
    if (scanf("%ld", &position) != 1) {
        fprintf(stderr, "not a number\n");
        return 1;
    }
    printf("value: ");
    if (scanf("%d", &value) != 1) {
        fprintf(stderr, "not a number\n");
        return 1;
    }

    return store_at(position, value);
}
```

Run it twice:

```bash
printf '3\n42\n' | ./store
printf '9999999999\n42\n' | ./store
```

Tasks:

1. Run it with `3`. Paste the output: it works and reports `stored`.
2. Run it with `9999999999`. Paste the exact message from the shell. Describe what changed.
3. Diagnose it with `gdb`, not by guessing:

   ```bash
   gdb ./store
   ```

   Paste a transcript with `run`, then `bt`, then `frame 0`, then `info locals` and `print index`. The interesting part is that `position` is still `9999999999` while `index` is `1410065407` — a garbage address far outside `slots`. Identify the exact line that faulted.
4. Explain, using the data types chapter, what `9999999999` is, what `long` can hold, what `int` can hold, and why the cast on line 7 produced a different number. Then explain why the `printf` on line 8 did not already crash.
5. Fix it: check the value against a limit before the cast, and print a proper error message with `fprintf(stderr, ...`. Paste both runs after the fix, one succeeding and one rejected.
6. Bonus: the check belongs in `main`, not in `store_at`. Argue both sides in three lines, then state which one you chose and why.

---

## Exercise 6: Zero Warnings, One Past Program

Take any one of your earlier exercises — from the loops, functions, pointers or error-handling chapters — and put it through the whole gauntlet:

```bash
# 1. the standard build
gcc -std=c11 -Wall -Wextra -pedantic -g program.c -o program

# 2. optimization, which exposes uninitialized reads
gcc -std=c11 -Wall -Wextra -pedantic -g -O2 program.c -o program_o2

# 3. the static analyzer
gcc -std=c11 -Wall -Wextra -pedantic -g -fanalyzer program.c -o program_fan

# 4. the sanitizers
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer program.c -o program_san
```

Tasks:

1. Paste the output of each of the four builds. List every warning you got, with the exact text.
2. Fix each warning. Do not silence one with a cast or a `(void)` unless the cast is genuinely correct — if you do, write one sentence saying why.
3. Run the sanitized build and paste its output. Fix anything it finds.
4. Paste a table of the before and after:

   ```text
   | build        | before            | after             |
   |--------------|-------------------|-------------------|
   | -Wall        | 2 warnings        | 0                 |
   | -O2          | 1 warning         | 0                 |
   | -fanalyzer   | 1 finding         | 0                 |
   | sanitizers   | heap-buffer-overflow | clean          |
   ```

5. Bonus: add `-Werror` to all four commands and confirm they all build clean.

---

## What These Exercises Are Testing

| Exercise | Skill |
| --- | --- |
| 1 | read a wrong value with `gdb` and locate the missing iteration |
| 2 | use a sanitizer to prove an out-of-bounds access that plain builds hide |
| 3 | read a leak report and classify each block |
| 4 | understand precedence, and why gcc warns about `if (x = 5)` |
| 5 | diagnose a crash from a backtrace instead of guessing |
| 6 | run every tool at your disposal and get to zero findings |

A note on the habit being built: exercises 1 and 2 are the same shape of bug — a loop bound that is off by one — and 4 and 5 are both "the compiler or the debugger already told you, if you read it". That is the point. C gives you at least three tools that will tell you exactly where the bug is. Learning to read them is the whole skill.

---

#### [Previous](./03-part.md) | [Next](./challenge.md)