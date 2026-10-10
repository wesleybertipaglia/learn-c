# Exercise List 2 - Syntax, I/O, `const` and `static`

## Exercise 1: Receipt

Read a product name (single word), a quantity and a unit price from the user and print a receipt line like:

```text
# input
Product: Mouse
Quantity: 3
Unit price: 49.90

# output
Product: Mouse
Total: 149.70
```

Use `%.2f` for the total. Remember the `&` in `scanf`.

## Exercise 2: A Formatted Profile Card

Read a name, an age and a height, then print them aligned using width specifiers, like a real CLI interface:

```text
# input
Name: Wesley
Age: 26
Height: 1.75

# output
Name    : Wesley
Age     : 26
Height  : 1.75 m
```

Use `%s`, `%d` and `%.2f`, plus width/alignment to make the output neat.

## Exercise 3: Safe Input Reader

Write a function with this signature:

```c
int read_int(const char *prompt, int *out);
```

It prints the prompt, reads an integer with `scanf`, checks the return value, and returns `0` on success or `-1` on invalid input (without writing to `*out`). Use it three times in `main`.

## Exercise 4: What Does `static` Do?

Write a program with a function `void counter(void)` that increments a **static local** and prints how many times it has been called. Call it three times from `main` and then move the counter to a **file-scope** variable. Explain in comments why the output is the same.

## Exercise 5: `const` Correctness

Write these functions and check which lines the compiler rejects:

```c
void f1(const int *p);       /* can we change *p?  can we change p?  */
void f2(int * const p);       /* can we change *p?  can we change p?  */
void f3(const int *p);       /* does this compile? */
void f4(const char *const s); /* is s itself protected? */
```

Answer each question in a comment next to the declaration.

## Exercise 6: Escapes and Multi-line Output

Print an ASCII "box" using `+`, `-` and `|`, then print a tab-separated table of three columns (`Item`, `Qty`, `Price`), using `\t` and width specifiers so the columns line up.

---

#### [Previous](./03-part.md) | [Next](../03-data-types/01-part.md)