# Exercise List 4 - File I/O

Six programs that cover the whole chapter: text reading with `fgets`, parsing with `strtok`, appending with timestamps, binary save/load of structs with count verification, a binary-safe copy with `ferror` checks, and a line-numbered substring search.

General rules for all six (graded):

- Compile with `gcc -std=c11 -Wall -Wextra -pedantic -g` with zero warnings.
- Every `fopen` checked (`NULL` + `perror`); program never continues with `NULL`.
- `fgets`, never `gets`; all buffers bounded (`sizeof` passed, `%Ns` widths).
- Binary files use `"rb"` / `"wb"` modes.
- Every opened stream closed on every path (`goto cleanup`, NULL-safe).
- Critical writes check both `fwrite`/`fputs`/`fprintf` results and `fclose`.

Skeletons below contain `(void)` casts so they compile warning-free before you start; delete each cast as you implement the real code.

---

## Exercise 1 — `wc`: Count Lines, Words, Bytes

Write `wc.c`: given one text file path in `argv[1]`, print `lines words bytes path`, like the simplified `wc`.

Skeleton:

```c
#include <ctype.h>
#include <stdio.h>

#define LINE_MAX 1024

static int count_file(const char *path, long *lines, long *words, long *bytes);

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

static int count_file(const char *path, long *lines, long *words, long *bytes) {
    FILE *fp = NULL;
    int rc = -1;
    (void)path; (void)lines; (void)words; (void)bytes;   /* drop as you implement */
    /* TODO: fopen(path, "r"), perror + goto cleanup on NULL */
    /* TODO: fgets loop with LINE_MAX buffer; isspace() word split */
    /* TODO: ferror check, checked fclose, fill outputs, rc = 0 */
    goto cleanup;   /* skeleton fails until implemented; your code replaces this */
cleanup:
    /* TODO: NULL-safe close */
    (void)fp;   /* drop once cleanup really closes fp */
    return rc;
}
```

Expected output:

```text
$ cat sample.txt
hello world
second line here
$ ./wc sample.txt
2 5 29 sample.txt
$ ./wc missing.txt
fopen: No such file or directory
$ echo $?
1
```

### Requirements

1. Read with `fgets(buf, sizeof buf, fp)`; loop on `!= NULL`.
2. Words split on `isspace()` (cast each byte to `unsigned char` first).
3. Bytes count every byte returned by `fgets` (including `'\n'`).
4. After the loop, `ferror` decides error vs clean EOF.
5. `fclose` checked; error paths close via `cleanup`.
6. Usage error (`argc != 2`) prints to `stderr`, returns nonzero.

---

## Exercise 2 — CSV Parse with `fgets` + `strtok`

Write `csv.c`: read a simple CSV file (no quoted commas) and print each field numbered. Delimiter is `','`; strip the trailing newline before tokenizing.

Skeleton:

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <string.h>

#define LINE_MAX 1024

static int dump_csv(const char *path);

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "usage: %s <file.csv>\n", argv[0]);
        return 1;
    }
    return dump_csv(argv[1]) == 0 ? 0 : 1;
}

static int dump_csv(const char *path) {
    FILE *fp = NULL;
    int rc = -1;
    (void)path;   /* drop as you implement */
    /* TODO: fopen(path, "r") + NULL check */
    /* TODO: fgets loop; line[strcspn(line, "\n")] = '\0' */
    /* TODO: strtok(line, ",") loop, printf("  field[%d] = [%s]\n", ...) */
    /* TODO: ferror + checked fclose */
    goto cleanup;   /* skeleton fails until implemented; your code replaces this */
cleanup:
    /* TODO: NULL-safe close */
    (void)fp;   /* drop once cleanup really closes fp */
    return rc;
}
```

Expected output:

```text
$ cat people.csv
id,name,score
1,ada,9.5
2,linus,8.0
$ ./csv people.csv
line 1: [id,name,score]
  field[0] = [id]
  field[1] = [name]
  field[2] = [score]
line 2: [1,ada,9.5]
  field[0] = [1]
  field[1] = [ada]
  field[2] = [9.5]
line 3: [2,linus,8.0]
  field[0] = [2]
  field[1] = [linus]
  field[2] = [8.0]
```

### Requirements

1. One `fgets` per row; buffer `LINE_MAX` passed via `sizeof`.
2. Newline stripped with `strcspn` before `strtok` (last field must not contain `'\n'`).
3. `strtok` delimiter `","`; handle empty input file (zero lines, success, no crash).
4. Warn on truncated long lines (no `'\n'` seen and not at EOF).
5. `ferror` after loop; checked `fclose`; single `cleanup` exit.

---

## Exercise 3 — Journal Append with Timestamp

Write `journal.c`: append `argv[2]` as a timestamped line to the file `argv[1]`. Create the file if missing; never truncate. Format: `[YYYY-MM-DD HH:MM:SS] message\n`.

Skeleton:

```c
#include <stdio.h>
#include <time.h>

static int journal_append(const char *path, const char *message);

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "usage: %s <journal.txt> <message>\n", argv[0]);
        return 1;
    }
    if (journal_append(argv[1], argv[2]) != 0) {
        return 1;
    }
    printf("appended to %s\n", argv[1]);
    return 0;
}

static int journal_append(const char *path, const char *message) {
    FILE *fp = NULL;
    int rc = -1;
    (void)path; (void)message;   /* drop as you implement */
    /* TODO: fopen(path, "a") + NULL check (creates, never truncates) */
    /* TODO: time(NULL) + localtime + strftime(buf, sizeof buf, "%F %T") */
    /* TODO: fprintf(fp, "[%s] %s\n", stamp, message), check < 0 */
    /* TODO: checked fclose is the durability gate */
    goto cleanup;   /* skeleton fails until implemented; your code replaces this */
cleanup:
    /* TODO: NULL-safe close */
    (void)fp;   /* drop once cleanup really closes fp */
    return rc;
}
```

Expected output:

```text
$ ./journal log.txt "started chapter 4"
appended to log.txt
$ ./journal log.txt "finished part 1"
appended to log.txt
$ cat log.txt
[2026-10-11 09:14:02] started chapter 4
[2026-10-11 09:14:09] finished part 1
```

### Requirements

1. Mode `"a"` exactly (append, create-if-missing, preserve existing content).
2. Timestamp via `time` + `localtime` + `strftime` into a bounded buffer; check `strftime` nonzero.
3. `fprintf` return checked (`< 0` = error → `perror`, `goto cleanup`).
4. `fclose` checked: success is only reported when the flush succeeded.
5. Both `argv` entries required; usage to `stderr` otherwise.

---

## Exercise 4 — Binary Save/Load of Structs with Verification

Write `records.c`: define `Record { uint32_t id; char name[32]; float score; }`, save a small table to `records.bin`, reload it, and print both counts. The file starts with a `uint32_t` count header.

Skeleton:

```c
#include <stdint.h>
#include <stdio.h>
#include <string.h>

#define NAME_MAX 32
#define CAP 128

typedef struct {
    uint32_t id;
    char name[NAME_MAX];
    float score;
} Record;

static int save_records(const char *path, const Record *recs, size_t n);
static int load_records(const char *path, Record *recs, size_t cap, size_t *out_n);

int main(void) {
    Record table[2] = {0};
    Record loaded[CAP] = {0};
    size_t n = 0;
    /* TODO: fill table[0] = {1, "ada", 9.5f}, table[1] = {2, "linus", 8.0f} */
    if (save_records("records.bin", table, 2) != 0) {
        return 1;
    }
    if (load_records("records.bin", loaded, CAP, &n) != 0) {
        return 1;
    }
    /* TODO: print each of the n loaded records */
    printf("loaded %zu records\n", n);
    return 0;
}

static int save_records(const char *path, const Record *recs, size_t n) {
    FILE *fp = NULL;
    int rc = -1;
    (void)path; (void)recs; (void)n;   /* drop as you implement */
    /* TODO: fopen(path, "wb") + check */
    /* TODO: fwrite(&count32, sizeof count32, 1, fp) == 1 else perror */
    /* TODO: fwrite(recs, sizeof recs[0], n, fp) == n else perror */
    /* TODO: checked fclose, fp = NULL, rc = 0 */
    goto cleanup;   /* skeleton fails until implemented; your code replaces this */
cleanup:
    /* TODO: NULL-safe close */
    (void)fp;   /* drop once cleanup really closes fp */
    return rc;
}

static int load_records(const char *path, Record *recs, size_t cap, size_t *out_n) {
    FILE *fp = NULL;
    int rc = -1;
    (void)path; (void)recs; (void)cap; (void)out_n;   /* drop as you implement */
    /* TODO: fopen(path, "rb") + check */
    /* TODO: fread header == 1 else ferror/feof branch */
    /* TODO: reject count > cap */
    /* TODO: fread body == count else truncated-file error */
    /* TODO: force name[NAME_MAX-1] = '\0', checked fclose */
    goto cleanup;   /* skeleton fails until implemented; your code replaces this */
cleanup:
    /* TODO: NULL-safe close (fclose failure forces rc = -1) */
    (void)fp;   /* drop once cleanup really closes fp */
    return rc;
}
```

Expected output:

```text
$ ./records
saved 2 records
loaded 2 records:
  [0] id=1 name=ada score=9.5
  [1] id=2 name=linus score=8.0
$ ./records   # truncated-file case after: head -c 10 records.bin > short.bin
error: truncated file: header=2 body=0
```

### Requirements

1. Modes `"wb"` / `"rb"`; fixed-width types only (`uint32_t`, `float` + inline `char[32]`).
2. Records zero-initialized (`= {0}`); `strncpy` + explicit terminator.
3. Every `fwrite`/`fread` count compared; short reads branch on `ferror` vs `feof`.
4. Header validated: `count > cap` is an error, never a big allocation.
5. Reloaded names force-terminated; `fclose` checked on both paths.

---

## Exercise 5 — `cp`: Binary-Safe Copy with `ferror` Checks

Write `cp.c`: copy `argv[1]` to `argv[2]` byte-exact, working for text and binary inputs. Use `fread`/`fwrite` with a fixed buffer and verify every chunk.

Skeleton:

```c
#include <stdio.h>

#define CHUNK 8192

static int copy_file(const char *src, const char *dst);

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "usage: %s <src> <dst>\n", argv[0]);
        return 1;
    }
    return copy_file(argv[1], argv[2]) == 0 ? 0 : 1;
}

static int copy_file(const char *src, const char *dst) {
    FILE *in = NULL;
    FILE *out = NULL;
    int rc = -1;
    static unsigned char buf[CHUNK];
    (void)src; (void)dst; (void)buf;   /* drop as you implement */
    /* TODO: fopen(src, "rb") and fopen(dst, "wb"), each checked */
    /* TODO: loop: n = fread(buf, 1, sizeof buf, in); */
    /*         if n > 0: fwrite(buf, 1, n, out) == n else perror */
    /* TODO: after loop: ferror(in) check */
    /* TODO: checked fclose(out) BEFORE fclose(in) ordering, NULL-safe */
    goto cleanup;   /* skeleton fails until implemented; your code replaces this */
cleanup:
    /* TODO: close both, guard NULL, preserve first error */
    (void)in; (void)out;   /* drop once cleanup really closes them */
    return rc;
}
```

Expected output:

```text
$ echo hello > a.txt
$ ./cp a.txt b.txt && cat b.txt
hello
$ ./cp missing.bin out.bin
fopen src: No such file or directory
$ cmp a.txt b.txt && echo identical
identical
```

### Requirements

1. Modes `"rb"` + `"wb"` (binary-safe: no newline translation anywhere).
2. Chunk loop uses `fread(buf, 1, sizeof buf, in)`; each `fwrite` count verified.
3. Post-loop `ferror(in)` check distinguishes clean EOF from read error.
4. `fclose(out)` checked (flush failures count as copy failure); both streams closed on all paths.
5. `cmp`/`diff` of source and copy must report identical for text and binary inputs.

---

## Exercise 6 — Mini `grep`: Substring per Line with Number

Write `mygrep.c`: print every line of `argv[2]` containing the substring `argv[1]`, prefixed with the 1-based line number: `lineno:line`.

Skeleton:

```c
#include <stdio.h>
#include <string.h>

#define LINE_MAX 2048

static int grep_file(const char *needle, const char *path);

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "usage: %s <needle> <file>\n", argv[0]);
        return 1;
    }
    if (argv[1][0] == '\0') {
        fprintf(stderr, "error: empty needle\n");
        return 1;
    }
    return grep_file(argv[1], argv[2]) == 0 ? 0 : 1;
}

static int grep_file(const char *needle, const char *path) {
    FILE *fp = NULL;
    int rc = -1;
    (void)needle; (void)path;   /* drop as you implement */
    /* TODO: fopen(path, "r") + NULL check */
    /* TODO: fgets loop with counter; strstr(line, needle) != NULL -> printf */
    /* TODO: ferror + checked fclose */
    goto cleanup;   /* skeleton fails until implemented; your code replaces this */
cleanup:
    /* TODO: NULL-safe close */
    (void)fp;   /* drop once cleanup really closes fp */
    return rc;
}
```

Expected output:

```text
$ cat notes.txt
roses are red
violets are blue
red car, blue car
plain line
$ ./mygrep red notes.txt
1:roses are red
3:red car, blue car
$ ./mygrep zzz notes.txt
$ echo $?
0
```

### Requirements

1. `fgets` with `sizeof` buffer; `strstr` for the substring test (case-sensitive).
2. Line numbers start at 1 and count physical lines read.
3. Match lines printed verbatim (keep the original `'\n'` from `fgets`); use `printf("%ld:%s", ...)`.
4. No match is still success (exit 0); only I/O errors fail.
5. Empty needle rejected with usage error; `ferror` + checked `fclose` as usual.

---

## How to Test All Six

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g wc.c -o wc
gcc -std=c11 -Wall -Wextra -pedantic -g csv.c -o csv
gcc -std=c11 -Wall -Wextra -pedantic -g journal.c -o journal
gcc -std=c11 -Wall -Wextra -pedantic -g records.c -o records
gcc -std=c11 -Wall -Wextra -pedantic -g cp.c -o cp
gcc -std=c11 -Wall -Wextra -pedantic -g mygrep.c -o mygrep

./wc sample.txt
./csv people.csv
./journal log.txt "smoke test"
./records
./cp a.txt b.txt && cmp a.txt b.txt && echo identical
./mygrep red notes.txt
```

All six must exit 0 on valid input, nonzero with a `perror` message on missing files, and produce zero compiler warnings.

---

#### [Previous](./03-part.md) | [Next](../05-modules-and-build/01-part.md)
