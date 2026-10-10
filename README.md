# C

C is a general-purpose programming language created by Dennis Ritchie at Bell Labs in 1972. It is one of the most widely used languages in the world: it powers operating systems (Linux, Windows, macOS kernels), embedded firmware, databases, game engines, and most of the software you already have installed.

C is small. The whole language fits in a single book, and it has very few rules — which is exactly what makes it powerful and exactly what makes it dangerous.

Main characteristics of C:
- Compiled and statically typed
- Close to the hardware: you manage memory yourself
- Fast and portable
- Procedural by default: you write functions that operate on data
- No garbage collector, no exceptions, no runtime hidden from you

## Why learn C?

- You finally understand what a **pointer**, a **struct**, and **memory** really are, instead of pretending they do not exist.
- Every other language you learn afterwards (C++, Rust, Go, JavaScript engines, Python's C layer) becomes easier to reason about.
- It is still the language of choice for constrained environments: microcontrollers, kernels, drivers, real-time systems.

## What you need to know before starting

Nothing. No prior programming experience is required. You only need a computer and a compiler (the first chapter shows how to install one).

The goal of this course is not to make you an algorithms specialist. The goal is to make you a **programmer who knows C** — someone who can read C code, write working programs, reason about memory, and use the language as a tool to solve problems.

The single most important warning before you continue:

> In C, the compiler will not stop you from writing code that reads memory you do not own, frees memory twice, or writes past the end of an array. These mistakes are silent until the day they are not. This course spends a lot of time on how to avoid them.

## How to use this course

Each topic is split into numbered parts (`01-part.md`, `02-part.md`, ...) and ends with `exercises.md`. Every module ends with a `challenge.md` that puts everything together.

The examples assume **C11** and are written for a Linux/macOS-like environment. On Windows, prefer [MSYS2](https://www.msys2.org/) (GCC) or [WSL](https://learn.microsoft.com/windows/wsl/install).

## Topics

### 1. Fundamentals

1. [Getting Started](./1-fundamentals/01-getting-started/01-part.md)
2. [Syntax](./1-fundamentals/02-syntax/01-part.md)
3. [Data Types](./1-fundamentals/03-data-types/01-part.md)
4. [Operators](./1-fundamentals/04-operators/01-part.md)
5. [Conditionals](./1-fundamentals/05-conditionals/01-part.md)
6. [Loops](./1-fundamentals/06-loops/01-part.md)
7. [Functions](./1-fundamentals/07-functions/01-part.md)
8. [Pointers](./1-fundamentals/08-pointers/01-part.md)
9. [Error Handling](./1-fundamentals/09-errors/01-part.md)
10. [Debugging](./1-fundamentals/10-debugging/01-part.md)

> [Final challenge of the module](./1-fundamentals/challenge.md)

### 2. Data Structures

1. [Arrays](./2-data-structures/01-arrays/01-part.md)
2. [Strings](./2-data-structures/02-strings/01-part.md)
3. [Structs](./2-data-structures/03-structs/01-part.md)
4. [Dynamic Memory](./2-data-structures/04-dynamic-memory/01-part.md)
5. [Linked Lists](./2-data-structures/05-linked-lists/01-part.md)
6. [Stacks](./2-data-structures/06-stacks/01-part.md)
7. [Queues](./2-data-structures/07-queues/01-part.md)
8. [Hash Tables](./2-data-structures/08-hashtables/01-part.md)

> [Final challenge of the module](./2-data-structures/challenge.md)

### 3. Advanced C

The practical skills that separate someone who can write C from someone who can work in C: recursion, sorting and searching by hand, bit manipulation, function pointers, files, and how to organize a real project with headers, libraries and a build system.

1. [Recursion and Algorithms](./3-advanced/01-recursion-and-algorithms/01-part.md)
2. [Bit Manipulation](./3-advanced/02-bit-manipulation/01-part.md)
3. [Function Pointers](./3-advanced/03-function-pointers/01-part.md)
4. [File I/O](./3-advanced/04-file-io/01-part.md)
5. [Modules and Build](./3-advanced/05-modules-and-build/01-part.md)

> [Final challenge of the module](./3-advanced/challenge.md)

### 4. Testing

1. [Introduction](./4-testing/01-introduction/01-part.md)
2. [Unit Tests](./4-testing/02-unit-tests/01-part.md)
3. [Test Frameworks](./4-testing/03-test-frameworks/01-part.md)

> [Final challenge of the module](./4-testing/challenge.md)

## Compile warnings are not noise

From the very first exercise onward, compile like this:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

Every warning you get is a real defect waiting for a specific input. Treat `-Wall -Wextra` as if they were errors: add `-Werror` if you need the discipline.

## Contributing

Found a typo, a broken example, or a confusing explanation? Open an issue or a pull request. Keep examples minimal, compilable, and free of undefined behavior.