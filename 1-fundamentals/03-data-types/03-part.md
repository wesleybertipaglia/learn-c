# Characters

`char` is an integer type that happens to store a character code. That is the whole idea: a character *is* a small integer, and the encoding table is what turns it back into a letter on your screen.

## Character Literals

```c
char a = 'a';
char A = 'A';
char zero = '0';
char nl = '\n';
```

Single quotes mean "one character". The value of `'a'` is the number your computer assigns to that character:

```c
#include <stdio.h>

int main(void) {
    printf("%c\n", 'a');
    printf("%d\n", 'a');   /* ASCII code */
    printf("%d\n", 'A');
    printf("%d\n", '0');

    printf("%d -> %c\n", 97, 97);

    return 0;
}
```

```text
a
97
65
48
97 -> a
```

## ASCII Values

Letters, digits and punctuation are assigned consecutive numbers, so `'z' - 'a'` is always 25 and `'5' - '0'` is always 5. That property holds for every C implementation:

```c
#include <stdio.h>

int main(void) {
    printf("digits are consecutive: %d\n", '9' - '0');   /* 9 */
    printf("letters are consecutive: %d\n", 'c' - 'a');  /* 2 */

    char c = 'F';
    if (c >= 'A' && c <= 'Z') {
        printf("%c is an uppercase letter\n", c);
    }

    /* convert without <ctype.h> */
    int lower = c - 'A' + 'a';
    printf("lowercase: %c\n", (char)lower);

    return 0;
}
```

```text
digits are consecutive: 9
letters are consecutive: 2
F is an uppercase letter
lowercase: f
```

The `<ctype.h>` header has proper functions for this when you need locale-aware behavior: `isdigit`, `isalpha`, `toupper`, `tolower`. They are safer than arithmetic because they only ever return true for a character the locale considers valid.

| Range | Typical characters |
| --- | --- |
| 0 to 31 | control codes: `\n` is 10, `\t` is 9, `\0` is 0 |
| 32 to 47 | space and punctuation: `!` `"` `#` ... `/` |
| 48 to 57 | digits `'0'` to `'9'` |
| 65 to 90 | uppercase `'A'` to `'Z'` |
| 91 to 96 | punctuation: `[` `\` `]` `^` `_` `` ` `` |
| 97 to 122 | lowercase `'a'` to `'z'` |

---

## `'a'` vs `"a"`

This is the distinction to get right early:

| Literal | What it is | Size |
| --- | --- | --- |
| `'a'` | an `int` holding the code for `a` | 1 value, 4 bytes of `int` |
| `"a"` | an array of 2 `char`, terminated with `\0` | 2 bytes |

```c
#include <stdio.h>

int main(void) {
    char  letter = 'a';
    char  text[] = "a";

    printf("sizeof('a')     = %zu\n", sizeof('a'));
    printf("sizeof(\"a\")    = %zu\n", sizeof("a"));
    printf("sizeof(text)   = %zu\n", sizeof(text));

    printf("letter = %d, text[0] = %d, text[1] = %d\n",
           letter, text[0], text[1]);

    return 0;
}
```

```text
sizeof('a')     = 4
sizeof("a")    = 2
sizeof(text)   = 2
letter = 97, text[0] = 97, text[1] = 0
```

Why two bytes? Because a string in C is a run of characters followed by a **null terminator** (`'\0'`, code 0), which marks the end. Without it, code reading the string would have no idea where to stop. `sizeof` includes that terminator, so `sizeof("abc")` is 4.

The practical rule:

- `'a'` — a single character value, for `char` variables and `switch` cases.
- `"a"` — a string of one character, for `char *` parameters and `printf("%s", ...)`.

---

## Escape Sequences

Some characters cannot be typed literally, and others mean something special in a string. Both cases use a backslash.

| Escape | Meaning | Value |
| --- | --- | --- |
| `\n` | newline | 10 |
| `\t` | horizontal tab | 9 |
| `\r` | carriage return | 13 |
| `\0` | null terminator | 0 |
| `\\` | a literal backslash | 92 |
| `\'` | a literal single quote | 39 |
| `\"` | a literal double quote | 34 |
| `\?` | a literal question mark (trigraph escape) | 63 |
| `\a` | alert / bell | 7 |
| `\b` | backspace | 8 |
| `\f` | form feed | 12 |
| `\v` | vertical tab | 11 |
| `\x41` | character by hex value | `'A'` |
| `\101` | character by octal value | `'A'` |

```c
#include <stdio.h>

int main(void) {
    printf("Tab:\tbetween\twords\n");
    printf("Newline escaped: \\n\n");
    printf("Quote: \" and apostrophe: '\n");
    printf("Backslash: \\\\\n");
    printf("Hex \\x41 is %c, octal \\101 is %c\n", '\x41', '\101');
    printf("Bell:\a\n");

    return 0;
}
```

> Beware of `\x` escapes in strings: `"\x41BC"` is a single escape `\x41BC` (C consumes as many hex digits as it can), which overflows a `char` and may produce a warning. Write `"\x41" "BC"` when you want two separate things.

---

## Reading and Writing One Character at a Time

For menus and parsers, `getchar` and `putchar` are simpler than `scanf`:

```c
#include <stdio.h>

int main(void) {
    printf("Type two letters, then Enter:\n");

    int first = getchar();
    int second = getchar();
    int nl = getchar();          /* the Enter key */

    (void)nl;

    putchar(first);
    putchar('\n');
    putchar(second);
    putchar('\n');

    printf("codes: %d %d\n", first, second);

    return 0;
}
```

Two details worth noticing:

- `getchar` returns an `int`, not a `char`, so that it can also return `EOF` (a negative value) meaning "no more input". Storing the result in a `char` and comparing to `EOF` breaks on platforms where `char` is unsigned.
- When the user presses Enter, the newline is still in the input buffer. Mixing `scanf` and `getchar` without accounting for leftover newlines is the classic "my `getchar` reads the newline" bug. The fix is either to consume the leftover `'\n'` explicitly or to use one input function consistently.

```c
#include <stdio.h>

int main(void) {
    int number;

    printf("Number: ");
    if (scanf("%d", &number) != 1) {
        return 1;
    }

    int c = getchar();        /* discard the leftover '\n' */
    (void)c;

    printf("Now type a letter: ");
    int ch = getchar();
    if (ch != EOF) {
        printf("You typed '%c' (%d)\n", ch, ch);
    }

    return 0;
}
```

---

## `char *`: A Pointer to a String

A **string literal** in C is not a first-class value. It is an array of `char` living in read-only memory, and when you write its name in an expression, C converts the array to the address of its first character.

```c
#include <stdio.h>

int main(void) {
    char  c = 'H';
    char *s = "Hello";

    printf("c  = '%c'\n", c);
    printf("s  = \"%s\"\n", s);        /* %s walks from s to the \0 */
    printf("s[0] = '%c'\n", s[0]);     /* same as *s */
    printf("s[1] = '%c'\n", s[1]);
    printf("last byte = %d\n", (int)sizeof("Hello") - 1);   /* the \0 */

    printf("sizeof(s) = %zu (pointer size, not string length)\n", sizeof(s));

    return 0;
}
```

`char *s = "Hello";` reads as "s points at the character `'H'`". The `*` means "pointer to", and `char` says what it points at. `%s` prints by starting at `s` and reading until the null terminator.

Two warnings:

- **Never write through a string literal.** `s[0] = 'J';` overwrites read-only memory and typically crashes with a segmentation fault. When you need to modify a string, copy it into an array: `char s[] = "Hello";`.
- **`sizeof(s)` is the pointer size (8), not the string length (5).** For the length you need `strlen` from `<string.h>`, covered in the Strings chapter.

The full story — what an address is, how `*` works, what "null pointer" means — is in the Pointers chapter. For now, treat `char *` as the type you use for text arguments.

---

#### [Previous](./02-part.md) | [Next](./04-part.md)