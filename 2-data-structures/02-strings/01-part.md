# Strings

Everything you have read from the user so far has been a number. The moment you read a name, a word, a line of text, you are in a different world: C has no `string` type, only `char` arrays and a set of conventions that turn an array of bytes into text. Those conventions are simple, and breaking them produces undefined behavior immediately.

---

## A String Is an Array of `char` Ending in `'\0'`

A C string is not a type. It is a convention: **a sequence of `char` values followed by a zero byte.**

```text
  "Wesley"

  index     0      1      2      3      4      5      6
  +--------+-----+-----+-----+-----+-----+-----+---------+
  |   'W'  | 'e' | 's' | 'l' | 'e' | 'y' |  '\0'  |
  +--------+-----+-----+-----+-----+-----+-----+---------+
    0x57    0x65  0x73  0x6c  0x65  0x79    0x00
```

Six letters, but **seven bytes**. The seventh is the terminator, and it is not optional. Every function that takes a string, every `%s` in `printf`, finds the end of the text by scanning for that zero byte. Without it, the function keeps reading whatever bytes follow — into unrelated variables, other strings, eventually unmapped memory.

Here is that layout observed, not imagined:

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char greeting[] = "Wesley";       /* 7 bytes: 6 letters + '\0' */

    printf("sizeof(greeting) = %zu bytes\n", sizeof(greeting));
    printf("strlen(greeting) = %zu characters\n", strlen(greeting));
    printf("greeting         = \"%s\"\n", greeting);

    printf("\nbyte by byte:\n");
    for (int i = 0; i < (int)sizeof(greeting); i++) {
        if (greeting[i] == '\0') {
            printf("  [%d] = '\\0'  (terminator, not printed)\n", i);
        } else {
            printf("  [%d] = '%c'   (0x%02x)\n", i, greeting[i],
                   (unsigned char)greeting[i]);
        }
    }

    char short_greeting[20] = "hi";
    printf("\nchar short_greeting[20] = \"hi\"\n");
    printf("  sizeof = %zu bytes\n", sizeof(short_greeting));
    printf("  strlen = %zu\n", strlen(short_greeting));
    printf("  bytes:");
    for (int i = 0; i < 20; i++) {
        printf(" %d", (unsigned char)short_greeting[i]);
    }
    printf("\n");

    char exact[] = "hi";
    printf("\nchar exact[] = \"hi\"\n");
    printf("  sizeof = %zu bytes (the compiler counts the terminator)\n",
           sizeof(exact));

    return 0;
}
```

```text
sizeof(greeting) = 7 bytes
strlen(greeting) = 6 characters
greeting         = "Wesley"

byte by byte:
  [0] = 'W'   (0x57)
  [1] = 'e'   (0x65)
  [2] = 's'   (0x73)
  [3] = 'l'   (0x6c)
  [4] = 'e'   (0x65)
  [5] = 'y'   (0x79)
  [6] = '\0'  (terminator, not printed)

char short_greeting[20] = "hi"
  sizeof = 20 bytes
  strlen = 2
  bytes: 104 105 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0

char exact[] = "hi"
  sizeof = 3 bytes (the compiler counts the terminator)
```

Three rules to take from that output:

1. `sizeof` on a string array includes the terminator: 7 for `"Wesley"`.
2. `strlen` does **not** count it: 6.
3. `char s[20] = "hi";` gives you 20 bytes with 18 of them zeroed. That is your emergency room — you can store up to 19 characters and still have a valid string.

---

## String Literals Live in Read-Only Memory

`"Wesley"` written in the source is not an array you own. It is a **string literal**, and the standard requires that modifying one is undefined behavior. Compilers put literals in a read-only section of the program, so a write triggers a hardware fault.

The type of a literal is `char[7]`, but everywhere you use it, it decays to a pointer to its first character. That is why this compiles:

```c
char *s = "hi";   /* legal C, and a trap */
```

`char *` promises writable memory; the literal is not writable. The compiler only complains under `-Wwrite-strings`, which is not in `-Wall`.

Here is the trap, caught by AddressSanitizer:

```c
#include <stdio.h>

int main(void) {
    char *literal = "Wesley";   /* legal C: the literal decays to char * */

    printf("literal        = \"%s\"\n", literal);
    printf("sizeof literal = %zu bytes\n", sizeof("Wesley"));

    printf("\nnow writing to it:\n");
    literal[0] = 'X';   /* UNDEFINED BEHAVIOR: the literal is read-only memory */

    printf("after literal[0] = 'X': \"%s\"\n", literal);

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address s1_literal.c -o s1_literal
./s1_literal
```

```text
AddressSanitizer:DEADLYSIGNAL
=================================================================
==374895==ERROR: AddressSanitizer: SEGV on unknown address 0x5ad7b3d22020 (pc 0x5ad7b3d2129b bp 0x7ffdb1d139c0 sp 0x7ffdb1d1390 T0)
The signal is caused by a WRITE memory access.
    #0 0x5ad7b3d2129b in main s1_literal.c:10
    #1 0x783f6f62a600 in __libc_start_call_main ../sysdeps/nptl/libc-start.c:59
    #2 0x783f6f62a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #3 0x5ad7b3d21144 in _start (s1_literal+0x1144)

==374895==Register values:
rax = 0x00005ad7b3d22020  rbx = 0x0000000000000000  rcx = 0x00007f9d6414701  rdx = 0x0000000000000000
  ... (about 20 more lines of register dumps)
AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: SEGV s1_literal.c:10 in main
==374895==ABORTING
```

The write itself faults, so the program never reaches the `printf` that would have shown the result. Without AddressSanitizer, older programs would silently corrupt the literal and carry on — which is why this bug survived in the wild for decades.

**Rule: `char *p = "..."` for reading only. Use `const char *p = "..."` to say so.** Since C11 does not allow modifying a string literal, `const char *` is the honest type, and it compiles everywhere.

---

## Three Ways to Declare a String

```c
char a[20] = "hi";   /* a writable buffer of 20 bytes */
char b[] = "hi";     /* a writable array, size inferred: 3 bytes */
char *c = "hi";      /* a pointer to read-only memory */
```

All three print `"hi"`. They differ in what `sizeof` says and in whether you may write to them:

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char a[20] = "hi";
    char b[] = "hi";
    const char *c = "hi";

    printf("sizeof(a) = %zu   (declared 20, only 2 used)\n", sizeof(a));
    printf("sizeof(b) = %zu   (inferred from the literal)\n", sizeof(b));
    printf("sizeof(c) = %zu   (a pointer, not an array)\n", sizeof(c));

    printf("\nall three print the same: \"%s\" \"%s\" \"%s\"\n", a, b, c);

    strcpy(a, "changed");
    printf("\na is a writable buffer, so strcpy works: a = \"%s\"\n", a);
    printf("c is a pointer to read-only memory, so c = \"%s\" and c[0] = '%c'\n", c, c[0]);

    return 0;
}
```

```text
sizeof(a) = 20   (declared 20, only 2 used)
sizeof(b) = 3   (inferred from the literal)
sizeof(c) = 8   (a pointer, not an array)

all three print the same: "hi" "hi" "hi"

a is a writable buffer, so strcpy works: a = "changed"
c is a pointer to read-only memory, so c = "hi" and c[0] = 'h'
```

| Declaration | `sizeof` | Writable? | Lifetime |
| --- | --- | --- | --- |
| `char a[20] = "hi";` | 20 | yes | until the block ends |
| `char b[] = "hi";` | 3 | yes | until the block ends |
| `char *c = "hi";` | 8 (pointer) | **no** | the whole program |
| `const char *d = "hi";` | 8 (pointer) | no, and the compiler knows | the whole program |

Choose based on what you intend:

- You need room and you will write into it: `char buf[100];` and fill it yourself.
- The size is short and fixed: `char name[] = "Wesley";`
- You are only referencing constant text: `const char *label = "Wesley";`

The capacity of `char a[20]` is a hard limit. `strcpy(a, "a string longer than twenty characters")` writes 30-odd bytes into 20 bytes of stack. That is a buffer overflow, and it is the single most exploited bug in the history of C. The next part covers the safe alternatives.

---

## `strlen` Counts Characters, Not the Terminator

`strlen` walks the array from index 0 until it finds `'\0'`, and returns how many bytes it passed. It is in `<string.h>`, not `<stdio.h>` — a missing include is a compile error, not a silent problem.

```text
  char greeting[] = "Wesley";
                     ^
                     |  strlen starts here
                     v
  'W' 'e' 's' 'l' 'e' 'y' '\0'
   0   1   2   3   4   5    6
   |<---------- 6 ---------->|   returned value
                              ^   not counted: this is the terminator
```

Two consequences:

- **`strlen` counts bytes, not characters.** With `char` being 1 byte and ASCII, that is the same for English text. It is *not* the same for `á`, `ç`, emoji, or any UTF-8 text outside ASCII, where one visible character can be two or more bytes.
- **`strlen` only works on a real, null-terminated string.** It has no length parameter and no way to stop safely. Given a `char` array with no terminator inside it, `strlen` walks past the end of the array and reads whatever follows — undefined behavior. There is no version of `strlen` that is safe on arbitrary memory. That is why `strnlen` exists (POSIX, not standard C) and why functions with a size limit, like `strncpy` and `snprintf`, are the ones you reach for when the terminator might be missing.

---

## Printing with `%s`

`printf` with `%s` does exactly what `strlen` does: scan for the terminator and print everything before it.

```c
printf("%s\n", greeting);        /* Wesley      */
printf("%10s|\n", greeting);     /*       Wesley| */
printf("%-10s|\n", greeting);    /* Wesley      | */
printf("%.3s\n", greeting);      /* Wes  : precision on a string = max chars */
```

Note that `%s` needs a `char *`, not a `char` array variable in a function parameter context — after decay it is a `char *`, which is what `%s` expects. `%c` prints one character; `%s` prints the whole string.

---

## Reading with `scanf`

`scanf` reads formatted input. For a string you must give it a **buffer and a bound**:

```c
#include <stdio.h>

int main(void) {
    char word[16];

    printf("Type one word, then a space, then another word:\n> ");

    /* the 15 leaves room for the terminator */
    if (scanf("%15s", word) != 1) {
        fprintf(stderr, "invalid input\n");
        return 1;
    }
    printf("first word read: \"%s\"\n", word);

    return 0;
}
```

```text
Type one word, then a space, then another word:
> first word read: "Ana"
```

Feed it `Ana Maria Silva` and only `Ana` arrives. `%s` stops at the first whitespace character and skips leading whitespace, so it reads exactly one token. Names, addresses and sentences all contain spaces, which makes `%s` the wrong tool for them.

### The Bound Is Not Optional

The number in the format string must be **one less than the buffer size**, because `scanf` writes a terminating `'\0'` as well:

```c
char word[16];
scanf("%15s", word);   /* correct: 15 characters + '\0' = 16 bytes */
scanf("%16s", word);   /* WRONG: 16 characters + '\0' = 17 bytes, overflow */
scanf("%s", word);     /* WRONG: no bound at all, overflow on long input */
```

Every one of those three is a buffer overflow. Write `%15s` and when you change the buffer size, change the number too. A macro keeps them in sync:

```c
#define BUF_SIZE 64

char word[BUF_SIZE];
scanf("%63s", word);   /* still manual, but the two numbers are adjacent */
```

Better: use `fgets`, which takes the size as an argument and cannot be miscounted. That is the subject of part 3.

---

## A First String Program

Putting it together — declare a buffer, read a word, measure it, copy it into a second buffer, and print both:

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char word[32];
    char copy[32];

    printf("Enter a word: ");
    if (scanf("%31s", word) != 1) {
        fprintf(stderr, "invalid input\n");
        return 1;
    }

    size_t length = strlen(word);
    printf("word   = \"%s\"\n", word);
    printf("length = %zu\n", length);
    printf("fits in char[32]? %s\n", (length + 1 <= sizeof(word)) ? "yes" : "no");

    strcpy(copy, word);
    printf("copy   = \"%s\"\n", copy);

    return 0;
}
```

```text
Enter a word: word   = "programming"
length = 11
fits in char[32]? yes
copy   = "programming"
```

(`Enter a word: ` has no `\n`, so the user's typing and the first line of output end up on the same line. Interactively the cursor simply sits after the colon.)

The `length + 1` in the capacity check is the same off-by-one as everywhere else in this chapter: the terminator needs a byte too.

---

## Summary

- A C string is a `char` array with a `'\0'` at the end. `"Wesley"` is 7 bytes.
- `sizeof` counts the terminator, `strlen` does not.
- String literals are `char[N]` in read-only memory. Writing to one is undefined behavior and usually an immediate crash. Declare them `const char *`.
- `char buf[20] = "hi";` gives a writable 20-byte buffer; `char s[] = "hi";` gives exactly 3; `char *p = "hi";` gives a read-only pointer.
- `strlen` only works on a properly terminated string, and it counts bytes.
- `%s` prints up to the terminator. `%10s`, `%-10s` and `%.3s` control it.
- `scanf("%Ns", buf)` needs `N` to be the buffer size minus one, and it cannot read spaces. Use `fgets` for lines.

Next part: `<string.h>`, the library that built the conventions above, plus the safe way to do each of its jobs.

---

#### [Previous](../01-arrays/exercises.md) | [Next](./02-part.md)