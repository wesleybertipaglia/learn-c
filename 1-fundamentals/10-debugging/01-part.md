# Debugging Mindset and the Compiler

Debugging is not a talent. It is a procedure, and the first step of the procedure is not reading your code carefully — it is **making the compiler do the work for you.** A modern `gcc` with the right flags catches a large fraction of real bugs before the program ever runs.

---

## Read the Message Format

Every diagnostic from gcc has the same shape:

```text
file.c:12:20: error: the message
   ^
```

| Part | Meaning |
| --- | --- |
| `file.c` | the file |
| `12` | the line |
| `20` | the column |
| `error:` / `warning:` | how bad it is |
| the rest | what is wrong |

Then gcc echoes the offending line with a caret under the exact column:

```text
p5_const.c:5:13: error: assignment of read-only location '*name'
    5 |     name[0] = 'J';
      |             ^
```

Read it the same way every time: **file, line, column, then the message.** The caret points at the precise token, which means you usually do not need to look at the source at all to know the fix.

---

## Three Kinds of Bad

| | What happens | Do you get a binary? | Example |
| --- | --- | --- | --- |
| **Error** | compilation stops | no | missing `#include`, type mismatch, syntax error |
| **Warning** | compilation continues | yes, but the compiler suspects a bug | unused variable, wrong `%d` for a `double` |
| **Undefined behavior** | compiles silently, does anything at runtime | yes | reading past an array, using an uninitialized pointer, dividing by zero |

Undefined behaviour is the dangerous one, because nothing tells you. The compiler is not required to diagnose it, so a program with UB can work for years and then fail after an unrelated change. **Every UB bug started as a warning someone ignored, or as code the compiler could not see.**

The golden rule from the README applies:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g file.c -o file
```

`-g` adds debug symbols for `gdb` (covered in the next part). `-std=c11` says which C you are writing. `-Wall -Wextra -pedantic` turn on the warnings.

---

## The Warning Table

Treat this table as a diagnostic reference. Every line of it is a bug the compiler already found for you.

| Warning | Flag | What it really means | Typical fix |
| --- | --- | --- | --- |
| `unused variable ‘x’` | `-Wunused-variable` (in `-Wall`) | you declared something and never used it — often a leftover from a refactor, or a variable you *meant* to print | use it, or delete it. If it is a `static` at file scope, it is dead code |
| `unused parameter ‘verbose’` | `-Wunused-parameter` (in `-Wextra`) | the function takes an argument it does not use — usually an interface requirement, or a forgotten `if` | use it, or mark it clearly: `(void)verbose;` |
| `implicit declaration of function ‘strcpy’` | `-Wimplicit-function-declaration` | **in C11 this is an error.** You called a function without declaring it, which almost always means a missing `#include` | add the header (`<string.h>` for `strcpy`, `<math.h>` for `sqrt`) |
| `assignment to ‘int *’ from ‘int’ makes pointer from integer without a cast` | `-Wint-conversion` (in `-Wall`, an error) | you assigned a value where an address was expected — the missing-`&` bug from the pointers chapter | `p = &x;` not `p = x;` |
| `format ‘%d’ expects argument of type ‘int’, but argument 2 has type ‘double’` | `-Wformat` (in `-Wall`) | your format string and your argument disagree. This reads garbage from the stack | `%d` for `int`, `%f`/`%.2f` for `double`, `%s` for `char *`, `%zu` for `size_t` |
| `control reaches end of non-void function` | `-Wreturn-type` (in `-Wall`) | a function declared to return something can fall off the end, so the caller gets whatever was in the return register | add a `return` on every path |
| `‘data’ is used uninitialized` | `-Wuninitialized` (in `-Wall`, often only at `-O2`) | you read a variable that was never assigned. The value is whatever was on the stack | initialize every variable at declaration |
| `suggest parentheses around assignment used as truth value` | `-Wparentheses` (in `-Wall`) | you wrote `if (x = 5)` instead of `if (x == 5)`. Assignment happened; the comparison did not | `==` to compare. Wrap assignments on purpose: `if ((x = 5) > 0)` |
| `stack-based buffer overflow` | `-Wanalyzer-out-of-bounds` (`-fanalyzer`) | gcc's static analyzer proved a write goes past the end of a local array | fix the size, or use `snprintf` |
| `pointer ‘p’ used after ‘free’` | `-Wuse-after-free` | you freed a pointer and then used it again | set the pointer to `NULL` after `free` |

### What `-Wall` and `-Wextra` Actually Turn On

| | `-Wall` | `-Wextra` |
| --- | --- | --- |
| unused variables | yes | yes |
| implicit declarations | yes | yes |
| int/pointer confusion | yes | yes |
| format mismatches | yes | yes |
| missing return paths | yes | yes |
| uninitialized reads | yes (needs `-O2` sometimes) | yes |
| unused parameters | no | **yes** |
| sign-compare (`<` between `int` and `size_t`) | no | **yes** |
| shadowed variables | no | **yes** |
| missing field initializers in structs | no | **yes** |

`-Wall` is the bare minimum. `-Wextra` catches the second tier — mostly the ones caused by an interface you do not fully implement yet. Use both, always.

---

## Three Real Bugs, Three Warnings

### Bug 1: The missing header

```c
#include <stdio.h>

int main(void) {
    char buffer[32];

    strcpy(buffer, "hello");
    printf("%s\n", buffer);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g bug1.c -o bug1
```

```text
bug1.c: In function ‘main’:
bug1.c:6:5: error: implicit declaration of function ‘strcpy’ [-Wimplicit-function-declaration]
    6 |     strcpy(buffer, "hello");
      |     ^~~~~~
bug1.c:2:1: note: include ‘<string.h>’ or provide a declaration of ‘strcpy’
    1 | #include <stdio.h>
  +++ |+#include <string.h>
    2 | 
bug1.c:6:5: warning: incompatible implicit declaration of built-in function ‘strcpy’ [-Wbuiltin-declaration-mismatch]
    6 |     strcpy(buffer, "hello");
      |     ^~~~~~
bug1.c:6:5: note: include ‘<string.h>’ or provide a declaration of ‘strcpy’
```

Note what happened: in C11 this is an **error**, so no binary is produced. Older compilers only warned, and the program ran with a guessed function signature — which is how tiny buffer overflows from `strcpy` used to happen in real software.

The `note:` line gives you the exact fix, including the suggested `#include` diff. Add `#include <string.h>` and it builds.

### Bug 2: The wrong format specifier

```c
#include <stdio.h>

int main(void) {
    double rate = 0.15;

    printf("count = %d\n", rate);
    printf("rate = %d\n", rate);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g bug2.c -o bug2
```

```text
bug2.c: In function ‘main’:
bug2.c:6:22: warning: format ‘%d’ expects argument of type ‘int’, but argument 2 has type ‘double’ [-Wformat=]
    6 |     printf("count = %d\n", rate);
      |                     ~^     ~~~~
      |                      |     |
      |                      int   double
      |                     %f
bug2.c:7:21: warning: format ‘%d’ expects argument of type ‘int’, but argument 2 has type ‘double’ [-Wformat=]
    7 |     printf("rate = %d\n", rate);
      |                    ~^     ~~~~
      |                     |     |
      |                     int   double
      |                    %f
```

Both warnings carry the whole story: gcc tells you the expected type, the actual type, and the specifier you should have used. `%f` is the answer.

This class of bug is worth understanding rather than just fixing. `printf` is variadic: it takes an unspecified number of arguments, so the compiler cannot fully check the call. When you write `%d` and pass a `double`, `printf` reads 8 bytes where it expected 4 and prints an integer that was never there. The program does not crash. It prints a wrong number.

### Bug 3: An integer where an address belongs

```c
#include <stdio.h>

static int find_index(char **names, int count, const char *target) {
    int i;

    for (i = 0; i < count; i++) {
        if (names[i] == target) {
            return i;
        }
    }
    return -1;
}

int main(void) {
    int *value;

    value = find_index(NULL, 3, "x");
    printf("%d\n", value[0]);
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g bug3.c -o bug3
```

```text
bug3.c: In function ‘main’:
bug3.c:17:11: error: assignment to ‘int *’ from ‘int’ makes pointer from integer without a cast [-Wint-conversion]
   17 |     value = find_index(NULL, 3, "x");
      |           ^
```

`find_index` returns an `int`, and `value` is an `int *`. The compiler refuses, because accepting it would mean treating the integer `-1` as an address — address `0xffffffffffffffff`, which is not mapped. This is the missing-`&` family from the pointers chapter, caught before the program exists.

---

## And One the Compiler Misses

These three were free. Here is what you still have to find yourself. The program reads an array that was never written:

```c
#include <stdio.h>

static int find_first(int *values, int n, int target) {
    int i;
    int found = -1;

    for (i = 0; i < n; i++) {
        if (values[i] == target) {
            found = values[i];
        }
    }
    return found;
}

int main(void) {
    int data[3];
    int idx = 0;

    printf("%d\n", find_first(data, 3, idx));
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -O2 bug4.c -o bug4
```

```text
In function ‘find_first’,
    inlined from ‘main’ at bug4.c:19:5:
bug4.c:8:19: warning: ‘data’ is used uninitialized [-Wuninitialized]
    8 |         if (values[i] == target) {
      |             ~~~~~~^~~
bug4.c: In function ‘main’:
bug4.c:16:9: note: ‘data’ declared here
   16 |     int data[3];
      |         ~~~~
```

`-O2` was needed here. Uninitialized-read analysis depends on inlining, so the same source can be silent at `-O0` and loud at `-O2`. Add `-O2` to your debug build once in a while and see what shows up.

The remaining UB — off-by-one loop bounds, use after `free`, overflow from `strcpy` into a short buffer — is mostly invisible to `-Wall`. That is what sanitizers and valgrind are for, and they are the subject of the third part of this chapter.

---

## `-pedantic`

`-pedantic` rejects **extensions that are not in the C standard.** With `-std=c11 -pedantic`, gcc accepts only portable C11:

```c
#include <stdio.h>

int main(void) {
    int x = 5;

    typeof(x) y = x * 2;   /* GNU extension, not C11 */
    printf("%d\n", y);
    return 0;
}
```

```text
bug5.c: In function ‘main’:
bug5.c:6:5: error: implicit declaration of function ‘typeof’ [-Wimplicit-function-declaration]
    6 |     typeof(x) y = x * 2;
      |     ^~~~~~
bug5.c:6:14: error: expected ‘;’ before ‘y’
    6 |     typeof(x) y = x * 2;
      |              ^~
      |              ;
bug5.c:7:20: error: ‘y’ undeclared (first use in this function)
    7 |     printf("%d\n", y);
      |                    ^
bug5.c:7:20: note: each undeclared identifier is reported only once for each function it appears in
```

`typeof` is a GNU extension. Compiling the same file with `-std=gnu11` accepts it and prints `10`. `-pedantic` is how you find out that the code you are proud of will not compile on MSVC, on a different compiler, or on your colleague's machine.

It is a portability check, and it costs nothing on Linux where gcc's default is `-std=gnu17`.

---

## `-Werror`

`-Werror` turns every warning into an error, so a warning stops the build:

```c
#include <stdio.h>

int main(void) {
    int unused;

    printf("hi\n");
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -Werror bug6.c -o bug6
```

```text
bug6.c: In function ‘main’:
bug6.c:4:9: error: unused variable ‘unused’ [-Werror=unused-variable]
    4 |     int unused;
      |         ^~~~~~
cc1: all warnings being treated as errors
```

Note `[-Werror=unused-variable]`: the original warning name is preserved inside the flag, so you still know what happened.

Use it for anything with a CI pipeline. A build that only goes green when there are zero warnings is the cheapest bug-prevention system in existence.

---

## The Mental Model

```text
1. Errors      stop the build. Fix them before anything else.
2. Warnings    the compiler suspects a bug. Assume it is right.
3. UB          nothing will tell you. Use -O2, sanitizers, valgrind.
```

A working habit:

```bash
# while writing
gcc -std=c11 -Wall -Wextra -pedantic -g file.c -o file

# before you call it done
gcc -std=c11 -Wall -Wextra -pedantic -g -O2 -Werror file.c -o file

# for memory bugs (next part)
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined file.c -o file
```

If you find yourself reading code line by line to find a bug, stop and recompile with one more flag. The compiler has read every line already.

---

#### [Previous](../09-errors/exercises.md) | [Next](./02-part.md)