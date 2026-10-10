# Reading and Building Strings Safely

The previous part gave you the tools in `<string.h>`. This part is about the two ends of a string's life: getting text **in** without overflowing a buffer, and building text **out** without losing data. These are where real programs read and write user input.

---

## `scanf("%s")` Is the Wrong Tool

From part 1, `%s` has two problems: it stops at the first whitespace, so it cannot read `"Ana Maria"`, and you must compute its bound by hand, so it can overflow if you get the arithmetic wrong.

```c
char name[32];
scanf("%31s", name);      /* reads "Ana" out of "Ana Maria" */
```

What you want for a line of text is `fgets`.

---

## `fgets` Reads a Whole Line

```c
char *fgets(char *s, int size, FILE *stream);
```

It reads from `stream` until either a newline or `size - 1` characters, whichever comes first, then appends `'\0'`. It returns `s` on success and `NULL` at end of file or on error.

```c
char line[100];
if (fgets(line, sizeof(line), stdin) != NULL) {
    printf("read: %s", line);
}
```

Three things make it better than `scanf`:

| | `scanf("%99s", buf)` | `fgets(buf, 100, stdin)` |
| --- | --- | --- |
| spaces | stops at the first one | reads them all |
| the newline | leaves it in the input stream | includes it in the result |
| the bound | you compute it in the format string | it is the same number as the buffer |
| overflow risk | if you mistype the number | impossible |

That last row is the real win. With `scanf`, `char buf[16]` needs `%15s` and getting it wrong is silent. With `fgets`, the buffer size appears exactly once and is used for both.

---

## The Leftover Newline

`fgets` stops **after** the newline and stores it. So `line` usually ends with `'\n'`, and that byte is inside your string, counted by `strlen`:

```text
  what the user typed          what is in the buffer
  h e l l o <Enter>  ---->    h e l l o \n \0
                              ^^^^^^    ^
                              5 chars   strlen says 6
```

```c
char line[100];
fgets(line, sizeof(line), stdin);

printf("[%s]\n", line);        /* [hello\n]  -> the ] lands on the next line */
printf("%zu\n", strlen(line)); /* 6, not 5 */
```

You must remove it. The standard way uses `strcspn`:

```c
line[strcspn(line, "\n")] = '\0';
```

### What `strcspn` Does

```c
size_t strcspn(const char *s, const char *reject);
```

It returns the **length of the initial run of characters in `s` that does not appear in `reject`** — that is, the offset of the first character that *is* in the reject set. With `"\n"` as the reject set, it finds the first newline:

```text
  s      =  h  e  l  l  o  \n  \0
  index    0  1  2  3  4   5   6
           |<--->|          ^
           strcspn returns 5: the first 5 characters are not '\n'
```

So `line[strcspn(line, "\n")] = '\0';` writes a terminator exactly at the newline's position, cutting it off. When there is no newline in the buffer, `strcspn` returns `strlen` and the assignment writes `'\0'` one past the last character — which is where the terminator already was, so it changes nothing. Safe in both cases.

`strcspn` is non-destructive: it only reports, it does not modify. That makes it the right tool for finding delimiters without `strtok`'s side effects.

If the line can also end with `\r` (Windows-style input redirected in), strip that too:

```c
line[strcspn(line, "\r\n")] = '\0';
```

### Observed

```c
#include <stdio.h>
#include <string.h>

#define SIZE 40

static void show(const char *label, const char *s) {
    printf("%s", label);
    printf("strlen = %zu, text = \"%s\"\n", strlen(s), s);
    printf("  bytes:");
    for (const char *p = s; *p != '\0'; p++) {
        printf(" %d", (unsigned char)*p);
    }
    printf("\n");
}

int main(void) {
    char line[SIZE];

    printf("type something, then Enter:\n> ");
    if (fgets(line, sizeof(line), stdin) == NULL) {
        fprintf(stderr, "no input\n");
        return 1;
    }
    printf("\n");
    show("fgets read:  ", line);

    size_t pos = strcspn(line, "\n");
    line[pos] = '\0';
    show("after strip: ", line);

    printf("\nnow a line longer than the %d-byte buffer:\n", SIZE);
    printf("> ");
    if (fgets(line, sizeof(line), stdin) == NULL) {
        fprintf(stderr, "no input\n");
        return 1;
    }
    printf("\n");
    show("fgets read:  ", line);

    return 0;
}
```

```text
type something, then Enter:
> 
fgets read:  strlen = 12, text = "hello world
"
  bytes: 104 101 108 108 111 32 119 111 114 108 100 10
after strip: strlen = 11, text = "hello world"
  bytes: 104 101 108 108 111 32 119 111 114 108 100

now a line longer than the 40-byte buffer:
> 
fgets read:  strlen = 39, text = "abcdefghijklmnopqrstuvwxyz0123456789ABC"
  bytes: 97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 112 113 114 115 116 117 118 119 120 121 122 48 49 50 51 52 53 54 55 56 57 65 66 67
```

The last byte listed, `10`, is `'\n'` in decimal. After stripping, the length drops from 12 to 11.

---

## Truncation: the Buffer Size Argument

The second read shows the other half of `fgets`'s contract. The input was 44 characters and the buffer holds 40, so `fgets` stored **39 characters plus `'\0'`** and **stopped**. The rest of the line is still waiting in the input stream, waiting for your next `fgets` call.

```text
  the user typed:      a b c d e f g h ... A B C D E F G H \n
                       ^                         ^      ^
                       first byte            39th byte   still in the buffer

  fgets stored:        a b c d e f g h ... A B C \0
                       ^                         ^
                       first byte            39th byte, then the terminator

  left over in stdin:                              D E F G H \n
                                                     ^
                                                     the NEXT fgets reads this
```

This is **truncation**, not corruption. The good news:

- The buffer is never overflowed. `fgets` guarantees it.
- The string is always valid: there is always a `'\0'`.
- The data is not lost, it is just still queued.

The bad news: if you do not know truncation happened, you silently process a truncated line. And if you call `scanf` afterwards to get the number, that number comes from the **remainder of the previous line**, which is its own trap.

How to detect it: `strlen(line) == SIZE - 1` means the line may have been cut. More robustly:

```c
size_t len = strlen(line);
if (len == SIZE - 1 && line[len - 1] != '\n') {
    fprintf(stderr, "line too long, it was truncated\n");
}
```

If the newline is there, the line ended before the limit. If it is missing and the buffer is full, the line was longer than the buffer.

### Draining the Rest of a Too-Long Line

If you want to discard the remainder instead of processing it, keep calling `fgets` with a throwaway buffer until you see the newline or EOF:

```c
/* after detecting truncation */
char drain[256];
while (fgets(drain, sizeof(drain), stdin) != NULL && strchr(drain, '\n') == NULL) {
    /* keep throwing away */
}
```

---

## Building Strings with `snprintf`

The other end of a string's life is construction. As part 2 showed, `strcpy` and `strcat` have no bounds, so `snprintf` is the function to reach for:

```c
int snprintf(char *dest, size_t size, const char *fmt, ...);
```

It formats into `dest`, writes **at most `size` bytes** including the terminator, and returns the number of characters the full result **would** have needed, excluding the terminator.

### Reading the Return Value

That return value is the useful part. It tells you whether anything was lost:

```text
  needed < size   ->  the whole thing fit
  needed >= size  ->  it was TRUNCATED, needed bytes but had size - 1
  needed < 0      ->  an encoding error occurred, nothing was written
```

```c
#include <stdio.h>
#include <stdlib.h>

/* the size arrives as a parameter, so the compiler cannot check it statically */
static int build(char *buf, size_t size, const char *label, int number) {
    return snprintf(buf, size, "%s-%d", label, number);
}

int main(int argc, char *argv[]) {
    const char *label = (argc > 1) ? argv[1] : "a very long label";
    int number = (argc > 2) ? atoi(argv[2]) : 42;

    char buf[16];
    int needed = build(buf, sizeof(buf), label, number);

    printf("buf     = \"%s\"\n", buf);
    printf("needed  = %d  (what the full result would need)\n", needed);
    printf("size    = %zu\n", sizeof(buf));
    printf("truncated? %s\n", (needed >= (int)sizeof(buf)) ? "yes" : "no");

    char small[32];
    int fits = build(small, sizeof(small), "short", number);

    printf("\nsmall   = \"%s\"\n", small);
    printf("needed  = %d\n", fits);
    printf("truncated? %s\n", (fits >= (int)sizeof(small)) ? "yes" : "no");

    return 0;
}
```

```text
buf     = "a very long lab"
needed  = 20  (what the full result would need)
size    = 16
truncated? yes

small   = "short-42"
needed  = 8
truncated? no
```

The first buffer holds 16 bytes, so it can store 15 characters. `"a very long label-42"` would need 21 characters, so it was cut to `"a very long lab"` and `needed` (20) is larger than the buffer. The second call had room, so nothing was lost.

The habit worth building: **always check the return value of `snprintf` when the input can be close to the limit.** It is the only way to know that data was discarded, and silently discarding data is how a program starts producing wrong answers that nobody can reproduce.

### Appending Without `strcat`

Because `snprintf` appends as easily as it copies, you never need `strcat`:

```c
char path[256] = "/home/user";

snprintf(path + strlen(path), sizeof(path) - strlen(path), "/documents");
```

The second argument is the space **left**, not the total size. Subtract `strlen(path)` so the two halves of the arithmetic cannot disagree.

### A Reusable Helper

One function that answers "did it fit?" takes the check out of every call site:

```c
#include <stdarg.h>

/* returns 0 on success, -1 if the result was truncated */
static int format(char *dst, size_t size, const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    int needed = vsnprintf(dst, size, fmt, args);
    va_end(args);

    if (needed < 0 || (size_t)needed >= size) {
        return -1;
    }
    return 0;
}
```

`vsnprintf` is `snprintf`'s variadic twin: it takes a `va_list` instead of the arguments directly, and requires `<stdarg.h>`. `va_start` and `va_end` are mandatory and easy to forget, which is why plain `snprintf` is usually the better choice at an individual call site.

In use, together with the appending idiom:

```c
#include <stdarg.h>
#include <stdio.h>
#include <string.h>

/* returns 0 on success, -1 if the result was truncated */
static int format(char *dst, size_t size, const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    int needed = vsnprintf(dst, size, fmt, args);
    va_end(args);

    if (needed < 0 || (size_t)needed >= size) {
        return -1;
    }
    return 0;
}

int main(void) {
    char path[32] = "/home/user";

    snprintf(path + strlen(path), sizeof(path) - strlen(path), "/documents");
    printf("path = \"%s\"\n", path);

    char big[16];
    if (format(big, sizeof(big), "%s-%d", "a very long label", 42) != 0) {
        printf("the result was truncated\n");
    }
    printf("big  = \"%s\"\n", big);

    return 0;
}
```

```text
path = "/home/user/documents"
the result was truncated
big  = "a very long lab"
```

Note that `format` is never used to write into `path` while `path` is also an argument: passing the same buffer as destination and as a `%s` argument is undefined behaviour, because `snprintf` reads the source while writing the destination. Copy first, or build into a separate buffer.

---

## A Program: Read Lines Until EOF, Reverse Each One

The payoff example. It reads every line from standard input until end of file, reverses each one, and prints them in aligned columns.

```c
#include <stdio.h>
#include <string.h>

#define MAX_LINES 32
#define LINE_SIZE 128

static void reverse(char *s) {
    size_t n = strlen(s);
    for (size_t i = 0; i < n / 2; i++) {
        char tmp = s[i];
        s[i] = s[n - 1 - i];
        s[n - 1 - i] = tmp;
    }
}

int main(void) {
    char storage[MAX_LINES][LINE_SIZE];
    char reversed[MAX_LINES][LINE_SIZE];
    int count = 0;

    printf("type some lines, one after another (Ctrl-D to stop):\n");

    size_t width = 0;
    while (count < MAX_LINES &&
           fgets(storage[count], LINE_SIZE, stdin) != NULL) {
        storage[count][strcspn(storage[count], "\n")] = '\0';

        size_t n = strlen(storage[count]);
        if (n > width) {
            width = n;
        }
        count++;
    }

    printf("\n%zu lines read\n\n", (size_t)count);

    for (int i = 0; i < count; i++) {
        snprintf(reversed[i], LINE_SIZE, "%s", storage[i]);
        reverse(reversed[i]);
        printf("%-*s | %s\n", (int)width, storage[i], reversed[i]);
    }

    return 0;
}
```

```text
type some lines, one after another (Ctrl-D to stop):

4 lines read

hello       | olleh
world       | dlrow
programming | gnimmargorp
abc         | cba
```

Every decision in it, one by one:

- **`fgets` in the `while` condition.** It returns `NULL` at EOF, so `fgets(...) != NULL` *is* the loop. `Ctrl-D` ends the input; you do not detect that yourself.
- **`strcspn` to strip the newline**, so a stored line has no trailing `'\n'`.
- **`snprintf` to copy** into the second array before reversing, so both the original and the reversal survive to the printing loop. Copying with `snprintf` instead of `strcpy` keeps the bounded habit.
- **`%-*s`** to align: `*` takes the width from the argument list, `-` left-aligns, `s` prints the string. The first column is padded to the width of the longest line.
- **Two passes.** The width is only known once every line has been read, which is why the reading loop and the printing loop are separate.
- **`count < MAX_LINES`** in the loop condition. This is the guard that prevents overflowing `storage`; more than 32 lines would otherwise write past the end. Same class of bug as a missing `scanf` bound, in a place where it is easy to forget.

---

## `getline`: the POSIX Alternative

POSIX added a function that removes the awkwardness of `fgets` entirely:

```c
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
```

You pass a `char *` that is initially `NULL` and a size of `0`, and `getline` allocates or reallocates as needed — including for a line longer than any buffer you would have guessed:

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void) {
    char *line = NULL;
    size_t capacity = 0;

    printf("lines (POSIX getline, allocates for you):\n");
    while (getline(&line, &capacity, stdin) != -1) {
        line[strcspn(line, "\n")] = '\0';
        printf("  [%zu] \"%s\"\n", strlen(line), line);
    }

    free(line);
    return 0;
}
```

```text
lines (POSIX getline, allocates for you):
  [1] "a"
  [2] "bb"
  [3] "ccc"
```

No size argument, no truncation, no `MAX_LINES` guard. The price:

1. **It is not standard C.** MSVC does not have it. You must define `_POSIX_C_SOURCE 200809L` before the includes, or the declaration is hidden and you get an implicit-declaration error.
2. **It allocates with `malloc`**, so you must `free` it. Forgetting is a memory leak, which the Dynamic Memory chapter explains properly.
3. **You cannot pass a fixed array.** `line` has to be a `char *` pointing at `NULL`.

For a program that reads arbitrarily long lines on Linux or macOS, `getline` is the right tool. For portable C, or when you can reason about the maximum length, `fgets` with a generous buffer is. Know both.

---

## Checklist

- Use `fgets(buf, sizeof(buf), stdin)`, not `scanf("%s")`, for anything that can contain a space.
- Strip the newline with `s[strcspn(s, "\n")] = '\0';`.
- `fgets` truncates instead of overflowing. Detect it when `strlen` reaches the buffer limit with no newline present, and drain the rest if you must.
- Check the return values of `scanf` and `snprintf`. They are the only signals that input was bad or output was cut.
- Build strings with `snprintf`, not `strcpy` and `strcat`.
- `snprintf` returns the length the full result **would** need, so `>= size` means truncated.
- To append: `snprintf(dst + strlen(dst), sizeof(dst) - strlen(dst), ...)`.
- On POSIX, `getline` allocates for you, but it is not standard C and it must be `free`d.

---

#### [Previous](./02-part.md) | [Next](./exercises.md)