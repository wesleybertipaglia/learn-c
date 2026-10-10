# Functions

So far every program has been one long `main`. That works for a dozen lines. The moment you copy a block for the third time, you need a function: a named, reusable piece of code that takes inputs and gives back an output.

---

## Why Functions

A function gives you four things:

| Benefit | What it means in practice |
| --- | --- |
| **Naming** | `compute_average(...)` explains itself; a block of arithmetic does not |
| **Reuse** | Written once, called from anywhere |
| **Decomposition** | Each function does one job, so each can be understood on its own |
| **Testing** | You can test a function without running the whole program |

The last point is why the Testing module exists. Functions are the unit of testing in C.

---

## Anatomy of a Definition

```c
return_type name(parameters) {
    body
}
```

A complete one:

```c
int add(int a, int b) {
    return a + b;
}
```

| Piece | Name | What it is |
| --- | --- | --- |
| `int` | return type | the type of the value handed back to the caller |
| `add` | function name | how the caller refers to it |
| `int a, int b` | parameters | typed names for the inputs |
| `{ ... }` | body | the code that runs |

Calling it looks like any other function call:

```c
int result = add(2, 3);   /* result is 5 */
```

A function with a `return_type` other than `void` **must** return a value on every path out. Returning nothing from an `int` function is undefined behaviour: the caller receives whatever happened to be in the return register.

---

## Declare Before You Use

C is compiled top to bottom. When the compiler reaches a function call, it must already know the function's signature — how many arguments it takes and what type they are. That knowledge comes from a declaration.

```c
int add(int a, int b);   /* prototype */
```

Read it as: "somewhere in this file there is a function called `add` taking two `int`s and returning an `int`." A prototype has **no body**, so nothing between the `;` and the next declaration is executed.

Without the prototype, the compiler has never heard of `add`:

```c
#include <stdio.h>

int main(void) {
    int result = add(2, 3);

    printf("result = %d\n", result);

    return 0;
}

int add(int a, int b) {
    return a + b;
}
```

```text
f2.c: In function ‘main’:
f2.c:4:18: error: implicit declaration of function ‘add’ [-Wimplicit-function-declaration]
    4 |     int result = add(2, 3);
      |                  ^~~
```

Note that gcc says **error** here: modern GCC has stopped assuming an implicit `int add()` and refuses to compile. Older compilers only warned, which is how buggy 1980s C code survived — the compiler guessed the types and the program crashed at runtime instead.

### The Standard Order

Put all prototypes at the top of the file, right after the `#include` lines, and all definitions below `main`:

```c
#include <stdio.h>

int add(int a, int b);

int main(void) {
    int result = add(2, 3);

    printf("result = %d\n", result);

    return 0;
}

int add(int a, int b) {
    return a + b;
}
```

```text
result = 5
```

Compile with zero warnings. Both orders — prototypes first, or `main` first — are fine as long as every call is preceded by a declaration. The compiler does not care where `main` sits in the file, so put it wherever you like; the convention is first.

### What `<stdio.h>` Is Doing

`printf` is not built into the language. It is a function declared in a header, and `#include <stdio.h>` pastes those declarations into your file. That is why `printf` works while your own `add` does not: nobody has told the compiler what `add` looks like. The same rule applies to your own code — a header file full of prototypes is exactly the mechanism (covered in part 4).

---

## Parameters and Arguments

- **Parameter**: the variable in the definition (`a`, `b` above).
- **Argument**: the value you pass at the call site (`2`, `3` above).

They have different names because they are different things living in different places. Each argument is matched to the parameter of the same position:

```c
add(2, 3);       /* a receives 2, b receives 3 */
add(100, 200);   /* a receives 100, b receives 200 */
```

Parameters are ordinary local variables. They exist only during the call and are gone when it returns. `a` and `b` in one call have nothing to do with `a` and `b` in another — the next part covers exactly why.

---

## `void`: No Return Value

A function that produces nothing is declared `void`:

```c
void print_banner(const char *title);
```

The `void` in the parameter list — `(void)` — is a separate idea: it means "this function takes **no** parameters", and it is the modern way to say it. A bare `()` in a definition means "unspecified parameters" in C, which is old and confusing; write `(void)`.

A `void` function should not `return` a value, but `return;` with no expression is legal and means "stop here, I am done".

---

## Returning Early

`return` does two things: it hands a value back, and it **ends the function immediately**. Everything after it is skipped, even if it is inside a loop or an `if`.

That makes it the cleanest way to bail out of a long function:

```c
const char *classify(int x) {
    if (x < 0) {
        return "negative";
    }
    if (x == 0) {
        return "zero";
    }
    return "positive";
}
```

Read it top to bottom: handle the special cases first, fall through to the general answer at the end. Compare with nesting everything inside an `else` chain:

```c
/* harder to read: one more level of braces per case */
const char *classify(int x) {
    if (x < 0) {
        return "negative";
    } else {
        if (x == 0) {
            return "zero";
        } else {
            return "positive";
        }
    }
}
```

Same behaviour, and the second version is a `return` buried at the bottom of four brace levels. This style is called a **guard clause**, and using it is one of the fastest ways to make C code readable.

The pattern scales up. This is the same function shape:

```c
int max(int a, int b) {
    if (a > b) {
        return a;
    }
    return b;
}
```

---

## Complete Example

```c
#include <stdio.h>

int max(int a, int b);
void print_banner(const char *title);
int is_positive(int x);

int main(void) {
    print_banner("Demo");

    printf("max(3, 7) = %d\n", max(3, 7));
    printf("max(-2, -9) = %d\n", max(-2, -9));

    if (is_positive(5)) {
        printf("5 is positive\n");
    } else {
        printf("5 is not positive\n");
    }

    if (!is_positive(0)) {
        printf("0 is not positive\n");
    }

    return 0;
}

int max(int a, int b) {
    if (a > b) {
        return a;
    }
    return b;
}

void print_banner(const char *title) {
    printf("== %s ==\n", title);
}

int is_positive(int x) {
    return x > 0;   /* any non-zero value counts as true */
}
```

```text
== Demo ==
max(3, 7) = 7
max(-2, -9) = -2
5 is positive
0 is not positive
```

Two details in `is_positive`. First, returning a comparison result (`x > 0`) is idiomatic C: comparisons produce `1` for true and `0` for false, which is exactly what an `int` return should hold. Second, note the missing `else` in `main` — C does not need one when each branch ends in `return` or `break`, but writing them anyway is harmless. What C *does* need from you is braces around every block, since there is no indentation-based syntax.

---

## Functions Cannot Be Nested

C has no nested functions. A function definition can only appear at **file scope**, never inside another function's body:

```c
int helper(void) {
    return 1;
}

int main(void) {

    int inner(void) {   /* ISO C forbids this */
        return 2;
    }

    return inner() + helper();
}
```

```text
f5.c: In function ‘main’:
f5.c:7:5: warning: ISO C forbids nested functions [-Wpedantic]
    7 |     int inner(void) {
      |     ^~~
```

gcc accepts this as a **GNU extension**, which is why it compiles at all — but it is not C11. Add `-Werror` and it becomes an error:

```text
f5.c: In function ‘main’:
f5.c:7:5: error: ISO C forbids nested functions [-Werror=pedantic]
    7 |     int inner(void) {
      |     ^~~
cc1: all warnings being treated as errors
```

The extension exists so GCC can optimize nested functions into real stack closures, but the behaviour is not portable and the code will not compile on another compiler. Write the inner function at file scope:

```c
int helper(void);
int inner(void);

int main(void) {
    return inner() + helper();
}

int helper(void) {
    return 1;
}

int inner(void) {
    return 2;
}
```

The same restriction applies to declaring variables: C has no block-level `struct` definitions before C23 and no function declarations inside blocks that would need to be nested scopes. Everything lives at file scope or inside a block, and functions live at file scope only.

---

## The Signature of `main`

`main` is not an ordinary function. The standard defines a small number of valid forms:

| Form | Meaning |
| --- | --- |
| `int main(void)` | **Correct.** No arguments, returns a status code |
| `int main(int argc, char *argv[])` | Correct, with command line arguments (covered in part 4) |
| `int main()` | Legal in C, but means "unspecified parameters" — write `(void)` |
| `void main(void)` | **Not standard.** Do not write this |

You will see `void main()` in a lot of old code, tutorials and embedded projects. It is a common mistake that keeps working because many compilers allow it:

```c
#include <stdio.h>

void main(void) {
    printf("hi\n");
}
```

```text
f6.c:3:6: warning: return type of ‘main’ is not ‘int’ [-Wmain]
    3 | void main(void) {
      |      ^~~~
```

A warning, not an error — so the code ships, and then behaves unpredictably: the value your `main` returns is unspecified, and code that checks the exit status gets garbage.

### The Return Value of `main`

`return 0` from `main` means "success". Any other value means failure, and the shell sees it:

```bash
./program; echo "exit code: $?"
```

| Value | Convention |
| --- | --- |
| `0` | success |
| `1` | generic failure (what you return on bad input) |
| other | whatever your program documents |

Because `main` implicitly returns `0` if you reach the closing brace, an explicit `return 0;` is optional but always clearer. Return a **non-zero** value when the user gave you bad input — that is how scripts and CI systems detect failure.

---

## Naming and Layout Conventions

There is one hard rule (a symbol beginning with `_` is reserved for the implementation) and a lot of convention:

| Thing | Convention | Example |
| --- | --- | --- |
| Functions | `snake_case` | `read_line`, `is_prime` |
| Variables | `snake_case` | `total_count` |
| Constants / macros | `UPPER_SNAKE_CASE` | `MAX_SIZE` |
| Pointers to structs | `struct_name *name` | `node_t *head` |
| A function that only reads its arguments | `is_`, `has_`, `can_` prefix | `is_valid` |

And the layout rules that keep code readable:

- Open `{` on the same line as the function signature or statement.
- Indent four spaces per level.
- Braces around every `if`, `for` and `while` body, even one-liners.
- Declare variables at the top of the block (C89 habit, still the safest style) or in the place you need them.

---

## Checklist for Writing a Function

1. Does the name say what it does, without needing the body to be read?
2. Does it do exactly one thing?
3. Does every path return a value (if the return type is not `void`)?
4. Is the prototype above every call site?
5. Does it modify anything the caller passed in? Not yet — that is the next part.

---

#### [Previous](../06-loops/exercises.md) | [Next](./02-part.md)