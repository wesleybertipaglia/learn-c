# How the Compilation Works

C programs are not handed to the CPU directly. A series of transformations happens between the `.c` file you write and the binary that runs. Knowing each step makes compiler errors far less mysterious.

## The Compilation Pipeline

```text
hello.c
   │
   ├─▶ Preprocessor   ──▶  hello.i     (text with #include expanded)
   │      └─▶ Compiler      ──▶  hello.s     (assembly)
   │             └─▶ Assembler    ──▶  hello.o     (object code)
   │                    └─▶ Linker    ──▶  hello      (executable)
```

You can stop at any stage by passing flags to `gcc`:

```bash
gcc -E hello.c                   # preprocess only → prints the expanded source
gcc -S hello.c -o hello.s        # compile to assembly
gcc -c hello.c -o hello.o        # assemble to an object file
gcc hello.o -o hello             # link into an executable
```

---

## 1. Preprocessor

The preprocessor works on text before the compiler sees anything. Its job is to handle `#include`, `#define` and conditional compilation. It knows nothing about C syntax — it is literally a text editor with very good macros.

```c
#include <stdio.h>   /* paste the contents of the standard header here */

#define SIZE 10       /* every SIZE in the file becomes 10 */
```

After preprocessing, the `#include` line is gone and replaced by thousands of lines from `stdio.h`.

---

## 2. Compiler

The compiler parses the preprocessed source, checks the syntax and semantics, and generates assembly. This is where type errors are reported:

```text
hello.c: In function 'main':
hello.c:5:8: error: 'nmae' undeclared (first use in this function)
    5 |     nmae("Hello\n");
      |     ^~~~
```

The format is always `file:line:column: error: message`, and the `^~~~` marker points at the exact token. Learn to read it: the column number tells you exactly which character to look at.

---

## 3. Assembler

The assembler converts assembly into machine code and writes an **object file** (`.o`). An object file contains your compiled functions, but also *unresolved references* to functions you used but did not define, like `printf`.

---

## 4. Linker

The linker takes your object files plus the standard library (and anything else you ask for) and produces a single executable. It also reports **undefined references** when a function is used but never defined:

```text
/usr/bin/ld: undefined reference to `my_function'
collect2: error: ld returned 1 exit status
```

This error means: "I found a call to `my_function`, but nowhere in the files I was given is a definition of it." Either you forgot to write the function, or you forgot to pass its file to `gcc`.

---

## Compiling Multiple Files

Real programs are split into `.c` files and `.h` headers:

```text
project/
├── main.c        # calls the functions
├── calculator.c  # implements them
└── calculator.h  # declares them
```

`calculator.h`:

```c
#ifndef CALCULATOR_H
#define CALCULATOR_H

int add(int a, int b);
int sub(int a, int b);

#endif
```

`calculator.c`:

```c
#include "calculator.h"

int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }
```

`main.c`:

```c
#include <stdio.h>
#include "calculator.h"

int main(void) {
    printf("%d\n", add(2, 3));
    return 0;
}
```

Compile everything in one command — `gcc` figures out the order for you:

```bash
gcc -std=c11 -Wall -Wextra main.c calculator.c -o app
```

Or compile each file separately and link later:

```bash
gcc -std=c11 -Wall -Wextra -c main.c calculator.c
gcc main.o calculator.o -o app
```

The `.h` file is not compiled. It is text pasted into every `.c` file that includes it — which is why the `#ifndef` guard exists: it prevents the same declarations from appearing twice.

---

## The Build Pipeline You Should Always Use

Never compile with a bare `gcc main.c`. Use:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

| Flag | Why |
| --- | --- |
| `-std=c11` | compile against a known standard, not whatever default your distro picks |
| `-Wall -Wextra` | enable the warnings that catch real bugs |
| `-pedantic` | reject non-standard extensions |
| `-g` | include debug symbols, required by the debugger |

---

## Automating With Make

Typing compile commands by hand does not scale. A `Makefile` describes how to build the project:

```makefile
CC      = gcc
CFLAGS  = -std=c11 -Wall -Wextra -pedantic -g
TARGET  = app
SRCS    = main.c calculator.c

all: $(TARGET)

$(TARGET): $(SRCS)
	$(CC) $(CFLAGS) $(SRCS) -o $(TARGET)

clean:
	rm -f $(TARGET)
```

Save it as `Makefile`, then:

```bash
make        # builds the project
make clean  # removes the binary
```

> Note the tab character before `$(CC)` and `rm` — Make requires tabs for recipe lines. If you get "missing separator", you used spaces.

---

#### [Previous](./01-part.md) | [Next](./03-part.md)