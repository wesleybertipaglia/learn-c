# Exercise List 1 - Introduction to C

> **Compile every exercise with:**
> `gcc -std=c11 -Wall -Wextra -pedantic -g exercicio1.c -o exercicio1`
>
> Remember: every warning is a bug waiting to happen.

## Exercise 1: Personalized Greeting

Ask the user for their name and print a personalized greeting.

**Example:**

```text
# input
Enter your name: Wesley

# output
Hello, Wesley!
```

## Exercise 2: Simple Questions

Ask three simple questions (name, age, city) and print all the answers in a single complete sentence.

**Example:**

```text
# input
Enter your name: Wesley
Enter your age: 26
Enter your city: Sao Paulo

# output
Wesley is 26 years old and lives in Sao Paulo
```

## Exercise 3: Favorite Quote

Ask the user for a favorite phrase and an author, then print the phrase in quotes followed by the author.

**Example:**

```text
# input
Enter a phrase: The present is a gift that you can enjoy twice
Enter the author: Mr. Oogway

# output
"The present is a gift that you can enjoy twice" ~ Mr. Oogway
```

## Exercise 4: A Program with Two Files

Split Exercise 2 into three files to practice multi-file compilation:

- `main.c` — contains `main`
- `pessoa.h` — declares the `pessoa` struct and the `exibir_pessoa` function
- `pessoa.c` — implements `exibir_pessoa`

Compile all of them with a single `gcc` command.

## Exercise 5: A Makefile

Write a `Makefile` for Exercise 4 that builds `pessoa` and has a `clean` target. Verify that `make clean` removes the binary.

## Exercise 6: Multi-line Program

Write a program that:
1. Prints a menu with three options using a single `printf` with `\n` characters.
2. Asks for the user's name.
3. Prints the name three times, one per line, using **one** `printf` call.

---

#### [Previous](./03-part.md) | [Next](../02-syntax/01-part.md)