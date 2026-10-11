# Text Files

A text file is a sequence of bytes that follows a convention: lines of printable characters separated by newline characters (`'\n'`). C does not have a special "text file" type. It has a `FILE *` stream, and a small set of functions that read and write characters, lines, and formatted values through it.

This part covers the text side only: opening with `fopen` modes `"r"`, `"w"`, `"a"`, checking for `NULL` with `perror`, line I/O with `fgets`/`fputs`, formatted I/O with `fprintf`/`fscanf` and its limits, the difference between `EOF`, `feof`, and `ferror`, and closing every stream with a checked `fclose`.

Compile every example in this chapter with:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

---

## 1. Opening a File: `fopen` and Modes

```c
#include <stdio.h>

FILE *fp = fopen("notes.txt", "r");
if (fp == NULL) {
    perror("fopen");
    return 1;
}
```

`fopen(path, mode)` returns a `FILE *` on success and `NULL` on failure. The mode string controls what you are allowed to do and what happens to existing content:

| Mode | Meaning | File must exist? | Existing content | Position |
| --- | --- | --- | --- | --- |
| `"r"` | read, text | yes, else `NULL` | preserved | start |
| `"w"` | write, text | no, created | **truncated to zero** | start |
| `"a"` | append, text | no, created | preserved | end (every write goes to end) |
| `"r+"` | read + write | yes | preserved | start |
| `"w+"` | read + write | no, created | **truncated** | start |
| `"a+"` | read + append | no, created | preserved | end for writes |

Three facts beginners get wrong: `"w"` destroys data (it empties the file immediately — use `"r"` or `"a"` to keep content); `"a"` always writes at the end even after `fseek`; and on Windows there is a text/binary distinction (`"r"` vs `"rb"`), covered in the next part.

```text
  fopen("notes.txt", "r")
  +----------------------------------+
  | path: "notes.txt"                |
  | mode: "r"  -> read, keep content |
  +----------------------------------+
       | success                | failure
       v                        v
  FILE *fp (usable)        NULL + errno set
                           -> call perror("fopen")
```

### Always Check for `NULL` and Call `perror`

`fopen` fails for ordinary reasons: the file does not exist, you have no permission, the directory does not exist, too many files are open. The failure signal is `NULL`, and the reason is stored in `errno`.

```c
#include <stdio.h>

int main(void) {
    FILE *fp = fopen("does-not-exist.txt", "r");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    /* ... use fp ... */

    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }
    return 0;
}
```

```text
$ ./main
fopen: No such file or directory
$ echo $?
1
```

`perror("fopen")` prints your prefix, then a colon, then the human-readable reason for the current `errno` value. Never ignore a `NULL` from `fopen`: every later call with that pointer is undefined behavior (usually a segfault).

Rule for this chapter: **every `fopen` is checked, every failure calls `perror`, and the program never continues with a `NULL` stream.**

---

## 2. Line I/O: `fgets` and `fputs`

For text files, the workhorse pair is `fgets` (read one line) and `fputs` (write one string):

```c
char *fgets(char *s, int size, FILE *stream);
int   fputs(const char *s, FILE *stream);
```

`fgets` reads at most `size - 1` characters, stops after a newline if one fits, and always null-terminates `s`. It returns `s` on success and `NULL` on end-of-file or error.

```c
#include <stdio.h>

#define LINE_MAX 256

int main(void) {
    FILE *fp = fopen("notes.txt", "r");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    char line[LINE_MAX];
    while (fgets(line, sizeof line, fp) != NULL) {
        fputs(line, stdout);
    }

    if (ferror(fp)) {
        perror("fgets");
        fclose(fp);
        return 1;
    }

    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }
    return 0;
}
```

Running it on a three-line `notes.txt` echoes the file unchanged.

### What `fgets` Guarantees

| Guarantee | Why it matters |
| --- | --- |
| Never writes more than `size - 1` chars + `'\0'` | no buffer overflow when `size` is correct |
| Keeps the `'\n'` if it fit | you can tell a full line from a truncated one |
| Returns `NULL` at EOF or on error | loop `while (fgets(...) != NULL)` terminates |
| Always null-terminates on success | the buffer is always a valid string |

### Never Use `gets`

```c
char buf[64];
gets(buf);   /* NEVER: removed from C11, always unsafe */
```

`gets` has no size parameter, so it cannot stop at the end of the buffer. It was removed from the C11 standard entirely. Use `fgets(buf, sizeof buf, stdin)` instead:

```c
char buf[64];
if (fgets(buf, sizeof buf, stdin) != NULL) {
    printf("you typed: %s", buf);
}
```

### The Newline Problem

`fgets` keeps the trailing `'\n'`. That is usually what you want for copying, but not for parsing. Strip it explicitly with `strcspn`:

```c
char line[256] = "hello\n";
line[strcspn(line, "\n")] = '\0';
printf("[%s]\n", line);   /* prints [hello] */
```

### Long Lines: Truncation

If a line is longer than the buffer, `fgets` returns what fit and leaves the rest for the next call. For this course a fixed buffer of 256 or 1024 bytes plus a documented limit is enough; production code must detect the missing newline (`line[strlen(line)-1] != '\n'` before EOF means truncation).

---

## 3. Formatted I/O: `fprintf` and `fscanf`

`fprintf` writes formatted text, exactly like `printf` but to a stream. It is safe and convenient for writing:

```c
#include <stdio.h>

int main(void) {
    FILE *fp = fopen("scores.txt", "w");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    fprintf(fp, "name=%s score=%d\n", "ada", 95);
    fprintf(fp, "name=%s score=%d\n", "linus", 88);

    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }
    return 0;
}
```

```text
$ cat scores.txt
name=ada score=95
name=linus score=88
```

`fscanf` is the mirror image, and it is far more dangerous. It parses formatted input, but it has sharp edges:

| `fscanf` limit | Consequence |
| --- | --- |
| `%s` without a width reads unbounded input | buffer overflow; always write `%255s` for a 256-byte buffer |
| Return value is the count of assigned items, not a boolean | must compare against the expected count |
| Leaves the newline in the stream | the next `%c` reads `'\n'`, not the next token |
| Stops at the first mismatch, leaving input unread | the stream is now out of sync |

Preferred pattern: read a line with `fgets`, then parse it with `sscanf`. The line boundary keeps failures contained:

```c
#include <stdio.h>

#define LINE_MAX 256

int main(void) {
    FILE *fp = fopen("scores.txt", "r");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    char line[LINE_MAX];
    while (fgets(line, sizeof line, fp) != NULL) {
        char name[64];
        int score = 0;
        /* width 63 protects name[64]; check for exactly 2 items */
        if (sscanf(line, "name=%63s score=%d", name, &score) == 2) {
            printf("%-8s -> %d\n", name, score);
        } else {
            fprintf(stderr, "skip malformed line: %s", line);
        }
    }

    if (ferror(fp)) {
        perror("fgets");
        fclose(fp);
        return 1;
    }

    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }
    return 0;
}
```

```text
ada      -> 95
linus    -> 88
```

If you must use `fscanf` directly, always bound `%s` and always check the count:

```c
char word[64];
int n = 0;

/* WRONG: fscanf(fp, "%s", word);         unbounded */
/* RIGHT: */
if (fscanf(fp, "%63s %d", word, &n) != 2) {
    fprintf(stderr, "parse error or EOF\n");
}
```

---

## 4. `EOF`, `feof`, and `ferror`: Three Different Things

| Symbol | What it is | Meaning |
| --- | --- | --- |
| `EOF` | negative `int` constant (usually `-1`) | "no more data or an error occurred" |
| `feof(fp)` | function, nonzero when | the EOF indicator on `fp` is set |
| `ferror(fp)` | function, nonzero when | the error indicator on `fp` is set |

The indicators are sticky flags on the stream. `fgets` returning `NULL` means *either* end-of-file *or* an error. You must ask which one:

```text
  fgets(...) returns NULL
         |
         v
  +-------------+     yes     +--------+
  | feof(fp)?   | ----------> |  EOF:  |
  +-------------+             | normal |
         | no                 |  end   |
         v                    +--------+
  +-------------+
  | ferror(fp)? | --- yes --> real I/O error, call perror
  +-------------+
```

### The Classic Bug: `while (!feof(fp))`

```c
/* WRONG: feof is only true AFTER a read fails */
while (!feof(fp)) {
    fgets(line, sizeof line, fp);
    printf("%s", line);   /* prints last line twice on some inputs */
}
```

`feof` predicts nothing. It reports that a *past* read hit the end. The loop above runs one extra iteration because `feof` is still false after the last successful read. The correct loop tests the read itself:

```c
/* RIGHT: test the read, then distinguish EOF from error */
while (fgets(line, sizeof line, fp) != NULL) {
    printf("%s", line);
}
if (ferror(fp)) {
    perror("read failed");
}
```

Same rule for `fgetc`: it returns `int`, not `char`, precisely so it can return every byte value *plus* the distinct `EOF` sentinel:

```c
int c;   /* int, not char */
while ((c = fgetc(fp)) != EOF) {
    fputc(c, stdout);
}
if (ferror(fp)) {
    perror("fgetc");
}
```

Storing the result in a `char` first destroys the distinction between byte `0xFF` and `EOF` on platforms with signed `char`.

---

## 5. Always Close: Checked `fclose`

```c
int fclose(FILE *stream);
```

`fclose` flushes buffered output, releases the descriptor, and returns `0` on success or `EOF` on failure. For files opened for writing, closing is part of writing: buffered data can still fail (disk full, broken pipe, network share gone) at flush time.

```c
FILE *fp = fopen("out.txt", "w");
if (fp == NULL) {
    perror("fopen");
    return 1;
}

fputs("important data\n", fp);

/* The write is NOT safe until this succeeds: */
if (fclose(fp) != 0) {
    perror("fclose");
    return 1;
}
```

For read-only streams a failed `fclose` is rarely fatal, but checking it anyway is harmless and consistent. For write streams, **never ignore the return value**.

---

## 6. Complete Example: Read Line by Line and Count

This program counts lines, words, and bytes in a text file — a small `wc`. It demonstrates every rule in this part: checked `fopen`, bounded `fgets`, `ferror` after the loop, checked `fclose`.

```c
#include <ctype.h>
#include <stdio.h>
#include <string.h>

#define LINE_MAX 1024

static int count_file(const char *path,
                      long *out_lines,
                      long *out_words,
                      long *out_bytes) {
    FILE *fp = fopen(path, "r");
    if (fp == NULL) {
        perror("fopen");
        return -1;
    }

    long lines = 0;
    long words = 0;
    long bytes = 0;
    int in_word = 0;
    char buf[LINE_MAX];

    while (fgets(buf, sizeof buf, fp) != NULL) {
        for (size_t i = 0; buf[i] != '\0'; i++) {
            unsigned char c = (unsigned char)buf[i];
            bytes++;
            if (c == '\n') {
                lines++;
            }
            if (isspace(c)) {
                in_word = 0;
            } else if (!in_word) {
                in_word = 1;
                words++;
            }
        }
        /* Detect a line longer than the buffer (no newline before EOF): */
        size_t len = strlen(buf);
        if (len > 0 && buf[len - 1] != '\n' && !feof(fp)) {
            fprintf(stderr, "warning: long line truncated at %d bytes\n",
                    LINE_MAX);
        }
    }

    if (ferror(fp)) {
        perror("fgets");
        fclose(fp);
        return -1;
    }

    if (fclose(fp) != 0) {
        perror("fclose");
        return -1;
    }

    *out_lines = lines;
    *out_words = words;
    *out_bytes = bytes;
    return 0;
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "usage: %s <file>\n", argv[0]);
        return 1;
    }

    long lines = 0;
    long words = 0;
    long bytes = 0;
    if (count_file(argv[1], &lines, &words, &bytes) != 0) {
        return 1;
    }

    printf("%ld %ld %ld %s\n", lines, words, bytes, argv[1]);
    return 0;
}
```

```text
$ cat poem.txt
roses are red
violets are blue
C reads line by line
and counts them too
$ ./main poem.txt
4 15 72 poem.txt
```

---

## Common Bugs

| Bug | Symptom | Fix |
| --- | --- | --- |
| `fopen` return not checked | segfault on missing file | `if (fp == NULL) { perror("fopen"); ... }` |
| `"w"` instead of `"r"`/`"a"` | file silently emptied | re-read the mode table before opening |
| `gets(buf)` | stack smash, removed from C11 | `fgets(buf, sizeof buf, stdin)` |
| `fscanf(fp, "%s", buf)` | overflow on long token | `"%63s"` for a 64-byte buffer |
| `while (!feof(fp))` | last line processed twice | `while (fgets(...) != NULL)` |
| `char c = fgetc(fp)` | `0xFF` confused with `EOF` | `int c; while ((c = fgetc(fp)) != EOF)` |
| ignoring `fclose` after write | silent data loss on full disk | `if (fclose(fp) != 0) perror(...)` |
| forgetting `fclose` on error path | descriptor leak | single `cleanup` label (next parts) |
| assuming `fgets` strips `'\n'` | double newlines on echo | strip with `strcspn` when parsing |

---

## Checklist

- [ ] Every `fopen` is followed by a `NULL` check with `perror`.
- [ ] Mode string is deliberate: `"r"` reads, `"w"` truncates, `"a"` appends.
- [ ] Input uses `fgets` with `sizeof` buffer; `gets` appears nowhere.
- [ ] `fscanf`/`sscanf` use bounded `%Ns` and the return count is compared.
- [ ] Read loops test the read (`!= NULL`, `!= EOF`), never `!feof`.
- [ ] After the loop, `ferror` distinguishes error from clean EOF.
- [ ] `fgetc` result is stored in an `int`.
- [ ] Every opened stream is closed on every path; write streams check `fclose`.
- [ ] Program compiles with `gcc -std=c11 -Wall -Wextra -pedantic -g` with zero warnings.

---

#### [Previous](../03-function-pointers/exercises.md) | [Next](./02-part.md)
