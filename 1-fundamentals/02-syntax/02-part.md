# Input and Output

C is a compiled language: it does not have a `Console.WriteLine`. All standard input and output happens through functions declared in `<stdio.h>`, which talk to the operating system on your behalf.

---

## Output with `printf`

`printf` stands for "print formatted". The format string contains literal text plus **conversion specifiers** that mark where each argument should be placed.

```c
printf("Hello, World!\n");
```

The `\n` is a **newline**. When printing multi-line output, use it — it works on every operating system.

### Format Specifiers

| Specifier | Type | Example output |
| --- | --- | --- |
| `%d` or `%i` | `int` | `42` |
| `%u` | `unsigned int` | `42` |
| `%f` | `double` / `float` (promoted) | `3.140000` |
| `%.2f` | `double` with 2 decimals | `3.14` |
| `%c` | `char` | `A` |
| `%s` | C string (`char *`) | `hello` |
| `%p` | pointer | `0x7ffd3c` |
| `%x` / `%X` | hexadecimal | `ff` / `FF` |
| `%ld` / `%lld` | `long` / `long long` | `9000000000` |
| `%zu` | `size_t` | `1024` |
| `%%` | a literal `%` | `%` |

```c
int    age = 26;
double price = 19.99;
char   grade = 'A';
char   name[] = "Wesley";

printf("Name: %s\n", name);
printf("Age: %d\n", age);
printf("Price: %.2f\n", price);
printf("Grade: %c\n", grade);
```

### Width and Alignment

```c
printf("%5d\n",   42);    /* "   42"  — right aligned, width 5 */
printf("%-5d|\n", 42);    /* "42   |" — left aligned (the - is the flag) */
printf("%05d\n",  42);    /* "00042" — zero padded */
```

This is what you use to build aligned tables and menus.

### Escaped Characters

```text
\n   newline        \t   tab         \\   backslash
\r   carriage ret   \"   quote       \0   null terminator
```

---

## Input with `scanf`

`scanf` reads formatted input, but **it works on `char *`, not on strings**. When you read a string you must give it a buffer and its size.

### Reading Numbers

```c
int age;
double height;

printf("Age: ");
scanf("%d", &age);          /* note the & — see the Pointers chapter */

printf("Height: ");
scanf("%lf", &height);       /* %lf for double; %f alone is for floats */
```

### Reading Strings — the Safe Way

```c
char name[100];

printf("Name: ");
if (scanf("%99s", name) != 1) {
    printf("invalid input\n");
}
```

> **Critical rule:** the number in the format string (`%99s`) must be **one smaller** than the buffer size (`100`), because `scanf` also writes a terminating `\0`. Writing `%100s` into a 100-byte buffer overflows it by one byte. This is a buffer overflow — a real, exploitable bug.

`%s` also stops at the first whitespace, so it cannot read names like "Ana Maria". For that, read the whole line with `fgets` (covered in the Strings chapter).

---

## The Problem with `scanf`

`scanf` is powerful but has sharp edges that generate undefined behavior:

```c
char word[10];

scanf("%d", word);        /* WRONG: writes 4 bytes of int into a char array */
scanf("%s", word);        /* WRONG: no bound, can overflow the buffer */

int n;
scanf("%d", n);           /* WRONG: missing & — the compiler may warn,
                             and the value is never stored */
```

Two habits make input safe:

1. **Always check the return value.** `scanf` returns the number of items successfully converted. If it does not return what you expect, the input was invalid and your variables are untouched.
2. **Bound every `%s` with a width.** Prefer `fgets` for whole lines.

```c
int age;
if (scanf("%d", &age) != 1) {
    fprintf(stderr, "expected a number\n");
    return 1;
}
```

---

## Writing to `stderr`

`printf` writes to **stdout**. Diagnostics belong on **stderr**, so that piping the program's output to a file does not mix error messages with results:

```c
printf("result: %d\n", result);            /* the product of the program */
fprintf(stderr, "error: invalid input\n"); /* a problem with the user */
```

---

#### [Previous](./01-part.md) | [Next](./03-part.md)