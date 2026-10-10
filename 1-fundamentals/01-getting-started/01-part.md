# Getting Started in C - Quick Tutorial

Let's write our first C program: a small console application that prints something to the terminal.

## Requirements

1. **Install a C compiler**: A compiler turns the source code you write into a program your computer can run. C has no official compiler, so we use one of these:
   - **Linux (Debian/Ubuntu)**: install `build-essential` with `sudo apt install build-essential`.
   - **Linux (Fedora)**: install `gcc` with `sudo dnf install gcc`.
   - **macOS**: run `xcode-select --install` (installs Clang, which is a perfectly good C compiler).
   - **Windows**: install [MSYS2](https://www.msys2.org/), then `pacman -S mingw-w64-ucrt-x86_64-gcc`, or use the [WSL](https://learn.microsoft.com/windows/wsl/install).
2. **Check the installation**:

```bash
gcc --version
```

If you see a version number, you are ready to go. Any C compiler works for this course — the examples were tested with GCC and Clang, but nothing here is compiler-specific.

3. **Choose an editor**:
   - **Visual Studio Code**: light, free, and excellent C support through the [C/C++ extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools). Recommended.
   - **Vim/Neovim**: fast once configured, but you pay the setup cost up front.
   - **Emacs**: powerful, same deal.
   - **CLion / Rider / Visual Studio**: full IDEs, heavier, but everything is preconfigured.

You will also want a **terminal**. Every exercise in this course is compiled and run from the command line.

---

## Our First Program

Create a file called `hello.c`:

```c
#include <stdio.h>

int main(void) {
    printf("Hello, World!\n");
    return 0;
}
```

Now compile it and run it:

```bash
gcc hello.c -o hello
./hello
```

Output:

```text
Hello, World!
```

## Breaking That Command Down

```bash
gcc hello.c -o hello
```

| Piece | Meaning |
| --- | --- |
| `gcc` | the compiler driver (could be `clang`, `cc`, ...) |
| `hello.c` | the source file being compiled |
| `-o hello` | the name of the output binary |

The compiler turned `hello.c` into an executable file named `hello` that your operating system can load and run directly.

> Under the hood, `gcc` runs four steps: **preprocess** (expand `#include` and macros), **compile** (C source to assembly), **assemble** (assembly to machine code), **link** (join your code with the C library). You will see each step explicitly in the next part.

---

#### [Previous](../../README.md) | [Next](./02-part.md)