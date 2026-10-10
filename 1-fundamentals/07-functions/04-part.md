# Recursion

A recursive function is a function that calls itself. That is the entire definition. What makes it correct is not the self-call — it is that every self-call moves one step closer to a case that does not recurse.

---

## The Two Cases

Every recursive function needs both:

| Case | What it does |
| --- | --- |
| **Base case** | the input where the answer is known; returns without recursing |
| **Recursive case** | calls itself on a smaller input, then combines the results |

A function with a recursive case but no base case recurses until the stack runs out. A function with a base case but no progress toward it never terminates.

---

## Example: Factorial

`n!` is defined as `n * (n - 1) * ... * 1`, with `0! = 1`. That definition is itself recursive: `n!` is `n` times `(n - 1)!`.

```c
#include <stdio.h>

int factorial(int n);

int main(void) {
    printf("5! = %d\n", factorial(5));

    return 0;
}

int factorial(int n) {
    if (n <= 1) {
        return 1;         /* base case */
    }

    return n * factorial(n - 1);   /* recursive case */
}
```

```text
5! = 120
```

### Tracing It

`factorial(5)` does not compute anything itself. It calls `factorial(4)`, which calls `factorial(3)`, which calls `factorial(2)`, which calls `factorial(1)`, which hits the base case and returns `1`. The returns then unwind:

```text
factorial(1) = 1
factorial(2) = 2 * factorial(1)             = 2
factorial(3) = 3 * factorial(2)             = 6
factorial(4) = 4 * factorial(3)             = 24
factorial(5) = 5 * factorial(4)             = 120
```

Two details to notice in the code. The base case is `n <= 1`, not `n == 0`, so that a negative input returns `1` instead of recursing downward forever. And the multiplication happens **after** the recursive call returns, so `n` must be a value the function still has when it comes back — which is why the parameter cannot be clobbered before the call.

---

## The Call Stack

Every active call gets its own **stack frame** on the stack: space for its parameters, its locals, and the return address telling the CPU where to resume. Recursion means several frames of the same function exist at once.

```text
call stack                          stack grows toward the bottom
(low addresses)
   +-------------------------+
   | main()                  |
   |   n = 5                 |
   +-------------------------+
   | factorial(5)  n = 5    |   <- entered, waiting on the call below
   |   return addr, locals   |
   +-------------------------+
   | factorial(4)  n = 4    |   <- entered, waiting on the call below
   |   return addr, locals   |
   +-------------------------+
   | factorial(3)  n = 3    |   <- entered, waiting on the call below
   +-------------------------+
   | factorial(2)  n = 2    |   <- entered, waiting on the call below
   +-------------------------+
   | factorial(1)  n = 1    |   <- base case, computes 1 and returns
   +-------------------------+
   | factorial(2)  returns   |   <- frame popped, 2 * 1
   +-------------------------+
   | factorial(3)  returns   |   <- frame popped, 3 * 2
   +-------------------------+
   | factorial(4)  returns   |   <- frame popped, 4 * 6
   +-------------------------+
   | factorial(5)  returns   |   <- frame popped, 5 * 24
   +-------------------------+
```

Each frame is alive exactly until the call it is waiting on returns. The stack is the mechanism that lets `factorial(5)` and `factorial(4)` both be "the current value of `n`" at the same time.

You can watch this directly:

```c
#include <stdio.h>

int sum(int n);

int main(void) {
    printf("sum(1..4) = %d\n", sum(4));
    return 0;
}

int sum(int n) {
    printf("  enter sum(%d)\n", n);

    if (n == 0) {
        printf("  base case, returning 0\n");
        return 0;
    }

    int partial = sum(n - 1);

    printf("  leaving sum(%d), partial = %d\n", n, partial);

    return partial + n;
}
```

```text
  enter sum(4)
  enter sum(3)
  enter sum(2)
  enter sum(1)
  enter sum(0)
  base case, returning 0
  leaving sum(1), partial = 0
  leaving sum(2), partial = 1
  leaving sum(3), partial = 3
  leaving sum(4), partial = 6
sum(1..4) = 10
```

All the "enter" lines come first, then all the "leaving" lines. That is the whole shape of recursion: go down to the base case, come back up combining results.

---

## Example: Recursive Countdown

A countdown does not need a partial result, so it prints and recurses before returning:

```c
#include <stdio.h>

void print_countdown(int n);

int main(void) {
    print_countdown(5);
    printf("liftoff!\n");

    return 0;
}

void print_countdown(int n) {
    if (n <= 0) {
        return;           /* base case */
    }

    printf("%d\n", n);
    print_countdown(n - 1);
}
```

```text
5
4
3
2
1
liftoff!
```

`void` is fine here: `print_countdown` produces no value, so `return;` is the whole base case. Writing `return;` instead of `return 0;` matters — a `return 0;` inside a `void` function is a constraint violation and gcc rejects it.

---

## The Cost: Stack Frames

Each call is not free. It costs:

- a stack frame: space for parameters and locals, taken from the stack
- a call and a return: push the return address, jump, jump back
- for the compiler: potentially a lot of work, since it must verify the stack stays balanced

So for a job that a loop can do, a loop is always faster. Compare these two for summing 1 to `n`:

| Approach | Cost |
| --- | --- |
| `for` loop | one frame, `n` additions |
| `sum(n - 1) + n` recursion | `n` frames, `n` calls, `n` returns |

For `n = 100000` the difference is invisible to you. For `n` in the millions, the recursion **runs out of stack**.

```c
#include <stdio.h>
#include <stdlib.h>

int countdown(int n);

int main(int argc, char *argv[]) {
    int depth = (argc > 1) ? atoi(argv[1]) : 500000;

    printf("%d\n", countdown(depth));

    return 0;
}

int countdown(int n) {
    if (n <= 0) {
        return 0;
    }
    return countdown(n - 1);
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
./main 100000    # fine
./main 500000    # dies
```

```text
$ ./main 100000
0
$ ./main 500000
Segmentation fault (core dumped)
```

No message, no exit code — the program walked off the end of the stack. The default stack size on Linux is 8 MB (`ulimit -s`), and each `-O0` frame of `countdown` takes 32 bytes, so it dies somewhere between 100,000 and 500,000 calls. The exact number depends on the frame size and the limit, which is why you should never write code that depends on surviving "close to the edge".

The rule that follows: **use a loop unless recursion genuinely models the problem.** Recursion earns its keep on data that is defined recursively — lists and trees, divide and conquer, directory walking, parsers — where the loop version would need an explicit stack to imitate what the call stack gives you for free.

---

## Naive Recursion and Exponential Blowup

The clearest example of recursion going wrong is naive Fibonacci:

```c
long long fib_naive(int n) {
    if (n < 2) {
        return n;
    }
    return fib_naive(n - 1) + fib_naive(n - 2);
}
```

It computes `fib(n-1)` and `fib(n-2)` — and each of those computes its own `fib(n-3)`. The same values get recomputed over and over, and the total number of calls grows roughly like the **golden ratio to the n**, about 1.618ⁿ.

Counting the calls:

```c
#include <stdio.h>
#include <time.h>

static long long calls;

long long fib_counted(int n) {
    calls++;
    if (n < 2) {
        return n;
    }
    return fib_counted(n - 1) + fib_counted(n - 2);
}

int main(void) {
    for (int n = 10; n <= 35; n += 5) {
        calls = 0;
        clock_t t0 = clock();
        fib_counted(n);
        double s = (double)(clock() - t0) / CLOCKS_PER_SEC;
        printf("n=%2d  calls=%lld  %.4f s\n", n, calls, s);
    }

    return 0;
}
```

```text
n=10  calls=177  0.0000 s
n=15  calls=1973  0.0000 s
n=20  calls=21891  0.0000 s
n=25  calls=242785  0.0005 s
n=30  calls=2692537  0.0060 s
n=35  calls=29860703  0.0603 s
```

Read the call counts: +5 to `n` multiplies the work by roughly **11**. Going from 30 to 35 took ten times longer. That is the signature of exponential growth.

And `n = 40`, on the machine this course was written on:

```text
fib(40) = 102334155
calls = 331160281
time = 0.663 s
```

**331 million function calls** to compute a ten-digit number. On a slower machine, or at `n = 50`, that is minutes instead of a second. Note that this is measured wall-clock time on one modern CPU; your number will differ, but the ratio between the rows above will not.

### The Fix: Remember What You Computed

Memoization stores each result the first time and reuses it, which turns the exponential into a linear loop:

```c
#include <stdio.h>

long long fib_memo(int n, long long memo[]) {
    if (n < 2) {
        return n;
    }
    if (memo[n] != 0) {
        return memo[n];
    }
    return (memo[n] = fib_memo(n - 1, memo) + fib_memo(n - 2, memo));
}

int main(void) {
    long long memo[41] = {0};

    printf("fib(40) memoized = %lld\n", fib_memo(40, memo));

    return 0;
}
```

```text
fib(40) memoized = 102334155
```

Same answer, and it finishes instantly — 79 calls instead of 331 million, because each value is computed once and every later request for it is a lookup. The "0 means not computed yet" test works because no Fibonacci number is 0 except the first two, which the `n < 2` case already returned.

The Advanced module builds proper hash-table memoization and iterative versions; the point here is that **recursive does not have to mean exponential**, and that when you see one, the fix is almost always to avoid recomputing.

---

## Tail Recursion

A call is in **tail position** when it is the very last thing the function does — its result is returned directly, with nothing left to do afterwards.

```c
int countdown(int n) {
    if (n <= 0) {
        return 0;
    }
    return countdown(n - 1);   /* nothing left to do */
}
```

Compare with `sum`, where `+ n` happens after the call, so that call is *not* in tail position and cannot be optimized away.

A tail-recursive call needs no frame of its own: the current frame can be reused, because after the call returns there is nothing left to restore. A compiler that applies **tail-call optimization** turns the recursion into a loop with no stack growth.

GCC does this at `-O2` and above. Here is a function that walks an array backwards looking for a zero, written in tail-recursive style:

```c
#include <stdio.h>

int find_zero(const int *values, int count);

int main(void) {
    int values[8];

    for (int i = 0; i < 8; i++) {
        values[i] = i * 3 + 1;
    }
    values[5] = 0;

    printf("find_zero = %d\n", find_zero(values, 8));

    return 0;
}

int find_zero(const int *values, int count) {
    if (count == 0) {
        return -1;
    }
    if (values[count - 1] == 0) {
        return count - 1;
    }
    return find_zero(values, count - 1);   /* tail position */
}
```

```text
find_zero = 5
```

Build it both ways and look at what the compiler produced:

```bash
gcc -std=c11 -Wall -Wextra -pedantic    -g main.c -o main_O0
gcc -std=c11 -Wall -Wextra -pedantic -O2 -g main.c -o main_O2
objdump -d main_O0 --no-show-raw-insn | sed -n '/<find_zero>:/,/^$/p'
objdump -d main_O2 --no-show-raw-insn | sed -n '/<find_zero>:/,/^$/p'
```

At `-O0` the function genuinely recurses, with a frame set up and torn down on every call, and the `call` at the end:

```text
    1239:	mov    -0xc(%rbp),%eax
    123c:	lea    -0x1(%rax),%edx
    1245:	mov    %rax,%rdi
    1248:	call   11f7 <find_zero>
    124d:	leave
```

At `-O2` there is no `call`, no `push %rbp`, no `leave`. The self-call became a backward `jmp`, and the function is now a loop that reuses the same frame:

```text
    1220:	sub    $0x1,%rax
    1224:	cmp    %rdx,%rax
    1227:	je     1238 <find_zero+0x38>
    1229:	mov    -0x4(%rdi,%rax,4),%ecx
    122d:	test   %ecx,%ecx
    122f:	jne    1220 <find_zero+0x20>
```

The `jne` at the end jumps back to `1220`, which is the same `sub $0x1,%rax` the recursive call would have done. The recursion is gone, and the stack can no longer overflow no matter how long the array is.

> Do not rely on it. The C standard does not require tail-call optimization, so it may not happen on your compiler, with your flags, or for your function. Write a loop if you need a guarantee.

---

## Command Line Arguments

`main` is the one function that the operating system calls, and it is the only one that can receive arguments. Its full form is:

```c
int main(int argc, char *argv[])
```

| Parameter | What it is |
| --- | --- |
| `argc` | **arg**ument **c**ount: how many strings were passed, including the program name |
| `argv` | **arg**ument **v**ector: an array of pointers to those strings |

`argv[0]` is the program name as it was invoked. `argv[1]` through `argv[argc - 1]` are what you typed. `argv[argc]` is guaranteed to be a null pointer, so `argv[argc] == NULL` is another way to test whether more arguments follow.

Note that `argv` is an array of **pointers**, not an array of characters. Each `argv[i]` points at a string the operating system built for you; that string can be any length, which is why this is not a fixed-size buffer and why you must not `strcpy` one argument into a short local array.

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("argc = %d\n", argc);

    for (int i = 0; i < argc; i++) {
        printf("argv[%d] = \"%s\"\n", i, argv[i]);
    }

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g args.c -o args
./args one two "three four"
```

```text
argc = 4
argv[0] = "./args"
argv[1] = "one"
argv[2] = "two"
argv[3] = "three four"
```

Three things to notice:

- **`argc` is 4, not 3.** The program name counts.
- **`"three four"` arrives as one argument.** The shell is what decides where arguments split; quotes keep a group together. This is why `argv[i]` is a `char *` and never a fixed-size buffer — the program does not know how long an argument will be.
- **`argv[0]` is whatever you typed.** Running `/usr/bin/args` gives the full path; running `./args` gives `./args`. Never assume it is a bare name.

### Checking argc Before Using argv

`argv[1]` does not exist unless the user supplied it. Reading it anyway is undefined behaviour, and it is the single most common crash in command line C programs. Always check first, and report the problem on `stderr` with a non-zero exit code:

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "usage: %s <number> <exponent>\n", argv[0]);
        return 1;
    }

    long long base = strtoll(argv[1], NULL, 10);
    int power = atoi(argv[2]);
    long long result = 1;

    for (int i = 0; i < power; i++) {
        result *= base;
    }

    printf("%s^%s = %lld\n", argv[1], argv[2], result);

    return 0;
}
```

```bash
./power 2 10
```

```text
2^10 = 1024
```

With no arguments:

```text
usage: ./power <number> <exponent>
```

and the exit code is `1`, which is what tells the shell or a script that it failed.

`strtoll` and `atoi` come from `<stdlib.h>` and convert the strings to numbers. Neither checks for errors — a stricter program would use `strtoll`'s second argument to confirm conversion succeeded. `fprintf(stderr, ...)` rather than `printf`, so error messages do not pollute the output when it is piped somewhere.

---

## When to Use Recursion

| Situation | Approach |
| --- | --- |
| Counting, summing, searching a flat sequence | Loop |
| Fixed count known in advance, simple math | Loop |
| Trees, linked lists, nested directories | Recursion |
| Divide and conquer (sort, binary search) | Recursion |
| Parsers and grammars | Recursion |
| Problem split into two similar halves | Recursion, then check the complexity |

And when you do write one, check four things:

1. Is there a base case that returns without recursing?
2. Does every recursive call move strictly toward it?
3. Does it return on every path?
4. How many calls does it make for the largest realistic input? If the answer is exponential, add memoization or write the loop.

---

#### [Previous](./03-part.md) | [Next](./exercises.md)