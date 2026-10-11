# Binary Files and Structs

Text files store human-readable characters. Binary files store raw memory bytes: integers in their native encoding, structs exactly as the compiler laid them out, arrays with no separators and no newlines. They are compact, fast, and exact — and they are not portable without extra work.

This part covers `fwrite`/`fread` on structs, the `sizeof` + count contract, the padding and portability warning, random access with `fseek`/`ftell`/`rewind`, and a complete save/load example for an array of records with verification of the returned count.

Compile every example with:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

---

## 1. Why Binary

| Property | Text file | Binary file |
| --- | --- | --- |
| Content | characters + `'\n'` | raw bytes (`fwrite` output) |
| Numbers | `"12345"` = 5 bytes | `int32_t 12345` = 4 bytes |
| Readable in editor | yes | no (shows garbage) |
| Round-trip of `double` | may lose precision | exact |
| Struct save | field by field with `fprintf` | one `fwrite` call |
| Portable across machines | mostly yes | only with care (padding, endianness) |

Use text when humans or other tools must read the file (configs, CSV, logs). Use binary when the file is a private snapshot: save files, caches, sensor dumps, fixed record tables.

Open binary files with a `"b"` in the mode:

```c
FILE *fp = fopen("data.bin", "wb");
if (fp == NULL) {
    perror("fopen");
    return 1;
}
```

| Mode | Meaning |
| --- | --- |
| `"rb"` | read binary |
| `"wb"` | write binary (truncate or create) |
| `"ab"` | append binary |

Append `+` for read/write (`"rb+"`, `"wb+"`, `"ab+"`).

On Linux `"w"` and `"wb"` behave identically today. The `"b"` still matters: it documents intent, and on Windows it disables newline translation (`'\n'` <-> `"\r\n"`), which would corrupt binary data.

---

## 2. `fwrite` and `fread`: The `sizeof` + Count Contract

```c
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
```

Both move `nmemb` items of `size` bytes each. Both return the number of **complete items** transferred — not bytes. That return value is the entire error protocol:

```c
#include <stdint.h>
#include <stdio.h>

int main(void) {
    int32_t values[4] = {10, 20, 30, 40};

    FILE *fp = fopen("nums.bin", "wb");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    size_t wrote = fwrite(values, sizeof values[0], 4, fp);
    if (wrote != 4) {
        perror("fwrite");
        fclose(fp);
        return 1;
    }

    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }

    /* Read them back: */
    int32_t back[4] = {0};
    fp = fopen("nums.bin", "rb");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    size_t got = fread(back, sizeof back[0], 4, fp);
    if (got != 4) {
        fprintf(stderr, "short read: wanted 4, got %zu (%s)\n",
                got, ferror(fp) ? "I/O error" : "truncated file");
        fclose(fp);
        return 1;
    }

    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }

    for (size_t i = 0; i < 4; i++) {
        printf("%d ", back[i]);
    }
    printf("\n");
    return 0;
}
```

```text
10 20 30 40
```

Rules:

1. Pass `sizeof element` as `size` and the element count as `nmemb`: `fwrite(p, sizeof *p, n, fp)` is the idiom. Never the reverse, never the total byte count as `size` with count 1.
2. Always compare the return value against `nmemb`. A short count means EOF (on read) or failure (on write, usually disk full). On a short read, call `ferror` to distinguish truncation (`feof`) from a real I/O error.

---

## 3. Structs: One Call per Record

A struct can be written as a single item:

```c
#include <errno.h>
#include <stdint.h>
#include <stdio.h>
#include <string.h>

#define NAME_MAX 32

typedef struct {
    uint32_t id;
    char name[NAME_MAX];
    float score;
} Record;

int main(void) {
    Record r = {0};
    r.id = 7;
    strncpy(r.name, "ada", sizeof r.name);
    r.name[sizeof r.name - 1] = '\0';
    r.score = 9.5f;

    FILE *fp = fopen("one.bin", "wb");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }
    if (fwrite(&r, sizeof r, 1, fp) != 1) {
        perror("fwrite");
        fclose(fp);
        return 1;
    }
    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }

    Record back = {0};
    fp = fopen("one.bin", "rb");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }
    if (fread(&back, sizeof back, 1, fp) != 1) {
        fprintf(stderr, "error: cannot read record (%s)\n",
                ferror(fp) ? strerror(errno) : "truncated file");
        fclose(fp);
        return 1;
    }
    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }

    printf("id=%u name=%s score=%.1f\n", back.id, back.name, (double)back.score);
    return 0;
}
```

```text
id=7 name=ada score=9.5
```

Requirements for this to be safe: the struct must contain no pointers (`char name[32]` is stored inline and survives; `char *name` would store a meaningless address), the whole struct must be zero-initialized first (`= {0}`) so padding bytes are deterministic, and fixed char arrays must be explicitly null-terminated (`strncpy` + `'\0'`).

---

## 4. Padding and Portability Warning

The compiler may insert invisible **padding** bytes between fields so each field sits at its natural alignment:

```text
  struct { char tag; uint32_t id; }  (padding visible)

  offset 0:  tag (1 byte)
  offset 1:  *** 3 padding bytes ***
  offset 4:  id (4 bytes)
  sizeof == 8, not 5
```

Consequences:

| Issue | Effect | Mitigation in this course |
| --- | --- | --- |
| Padding bytes | file contains uninitialized garbage unless zeroed | always `= {0}` before filling |
| Different compilers / flags | different padding, different `sizeof` | only read files written by the same build |
| Endianness | `0x01020304` stored byte-reversed on other CPUs | document "native-endian, same-machine only" |
| `long` / pointer sizes | 4 vs 8 bytes across platforms | use `<stdint.h>` fixed types (`uint32_t`, `int32_t`) |
| `float` encoding | almost always IEEE-754, but not guaranteed | same-build assumption |

Check your layout once with `sizeof` and `offsetof`:

```c
#include <stddef.h>
#include <stdio.h>

typedef struct {
    char tag;
    unsigned int id;
    char name[32];
    float score;
} Record;

int main(void) {
    printf("sizeof(Record) = %zu, offsetof(id) = %zu\n",
           sizeof(Record), offsetof(Record, id));
    return 0;
}
```

```text
sizeof(Record) = 44, offsetof(id) = 4
```

The 3 bytes after `tag` are padding. If you need a portable file format (different machines, long-term storage), do not dump structs: write each field with explicit sizes and byte order. That is beyond this part — here the contract is **same program, same machine, same build**.

---

## 5. Random Access: `fseek`, `ftell`, `rewind`

Sequential access reads from start to end. Random access jumps:

```c
int fseek(FILE *stream, long offset, int whence);
long ftell(FILE *stream);
void rewind(FILE *stream);
```

| `whence` | Base |
| --- | --- |
| `SEEK_SET` | start of file (`fseek(fp, 40, SEEK_SET)` = second 40-byte record) |
| `SEEK_CUR` | current position |
| `SEEK_END` | end of file (`fseek(fp, 0, SEEK_END)` + `ftell` = file size) |

```text
  file positions (record file, 40 bytes per record)

  byte:  0        40        80        120
         | rec 0   | rec 1   | rec 2   |
                       ^
         fseek(fp, 1 * 40, SEEK_SET) lands here
```

Example: jump to the second record and read it:

```c
#include <stdio.h>

#define NAME_MAX 32

typedef struct {
    unsigned int id;
    char name[NAME_MAX];
    float score;
} Record;

int main(void) {
    FILE *fp = fopen("records.bin", "rb");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    long size = 0;
    if (fseek(fp, 0, SEEK_END) != 0 || (size = ftell(fp)) < 0) {
        perror("fseek/ftell");
        fclose(fp);
        return 1;
    }
    printf("file size: %ld bytes (%ld records)\n",
           size, size / (long)sizeof(Record));

    rewind(fp);
    if (fseek(fp, (long)sizeof(Record), SEEK_SET) != 0) {
        perror("fseek");
        fclose(fp);
        return 1;
    }

    Record r = {0};
    if (fread(&r, sizeof r, 1, fp) != 1) {
        fprintf(stderr, "error: no second record (%s)\n",
                ferror(fp) ? "I/O error" : "file too short");
        fclose(fp);
        return 1;
    }
    printf("record[1]: id=%u name=%s score=%.1f\n",
           r.id, r.name, (double)r.score);

    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }
    return 0;
}
```

Notes: always check `fseek` (nonzero = failure) and `ftell` (`-1L` = failure); `rewind` returns nothing and clears the error indicator, so prefer `fseek(fp, 0, SEEK_SET)` when failure matters. `fseek` with arbitrary offsets is only fully defined on binary streams, and `ftell` returns `long`, so files over 2 GB need POSIX `ftello`/`fseeko` — this course stays under that limit.

---

## 6. Complete Example: Save and Load an Array of Records

Save writes the count first (as a fixed-width header), then the records. Load reads the header, validates it, then reads exactly that many records and verifies the count.

```c
#include <stdint.h>
#include <stdio.h>
#include <string.h>

#define NAME_MAX 32
#define MAX_RECORDS 1024

typedef struct {
    uint32_t id;
    char name[NAME_MAX];
    float score;
} Record;

static int save_records(const char *path, const Record *recs, size_t n) {
    FILE *fp = NULL;
    int rc = -1;

    fp = fopen(path, "wb");
    if (fp == NULL) {
        perror("fopen");
        goto cleanup;
    }

    uint32_t count = (uint32_t)n;
    if (fwrite(&count, sizeof count, 1, fp) != 1) {
        perror("fwrite header");
        goto cleanup;
    }
    if (n > 0 && fwrite(recs, sizeof recs[0], n, fp) != n) {
        perror("fwrite records");
        goto cleanup;
    }

    if (fclose(fp) != 0) {
        perror("fclose");
        fp = NULL;
        goto cleanup;
    }
    fp = NULL;
    rc = 0;

cleanup:
    if (fp != NULL) {
        fclose(fp);
    }
    return rc;
}

static int load_records(const char *path, Record *recs, size_t cap,
                        size_t *out_n) {
    FILE *fp = NULL;
    int rc = -1;

    fp = fopen(path, "rb");
    if (fp == NULL) {
        perror("fopen");
        goto cleanup;
    }

    uint32_t count = 0;
    if (fread(&count, sizeof count, 1, fp) != 1) {
        fprintf(stderr, "error: cannot read header (%s)\n",
                ferror(fp) ? "I/O error" : "empty file");
        goto cleanup;
    }
    if (count > cap) {
        fprintf(stderr, "error: file claims %u records, buffer holds %zu\n",
                count, cap);
        goto cleanup;
    }

    size_t got = fread(recs, sizeof recs[0], count, fp);
    if (got != count) {
        fprintf(stderr, "error: truncated file: header=%u body=%zu (%s)\n",
                count, got, ferror(fp) ? "I/O error" : "EOF");
        goto cleanup;
    }
    for (size_t i = 0; i < got; i++) {
        recs[i].name[NAME_MAX - 1] = '\0';
    }

    *out_n = got;
    rc = 0;

cleanup:
    if (fp != NULL) {
        if (fclose(fp) != 0) {
            perror("fclose");
            rc = -1;
        }
    }
    return rc;
}

int main(void) {
    /* String-literal init zero-fills the rest incl. padding. */
    Record table[3] = {
        { 1, "ada", 9.5f },
        { 2, "linus", 8.0f },
        { 3, "grace", 9.0f },
    };

    if (save_records("records.bin", table, 3) != 0) {
        return 1;
    }
    printf("saved 3 records (%zu bytes each)\n", sizeof table[0]);

    Record loaded[MAX_RECORDS] = {0};
    size_t n = 0;
    if (load_records("records.bin", loaded, MAX_RECORDS, &n) != 0) {
        return 1;
    }

    printf("loaded %zu records:\n", n);
    for (size_t i = 0; i < n; i++) {
        printf("  [%zu] id=%u name=%s score=%.1f\n",
               i, loaded[i].id, loaded[i].name, (double)loaded[i].score);
    }
    return 0;
}
```

```text
saved 3 records (40 bytes each)
loaded 3 records:
  [0] id=1 name=ada score=9.5
  [1] id=2 name=linus score=8.0
  [2] id=3 name=grace score=9.0
```

File layout (`records.bin`, 4 + 3*40 = 124 bytes): a `count` header at byte 0, then `Record[0]` at byte 4, `Record[1]` at byte 44, `Record[2]` at byte 84. Without the stored count the loader could only guess from the file size, and a half-written file divides evenly often enough to look valid — the header plus the `got != count` check turns truncation into a loud error.

---

## Common Bugs

| Bug | Symptom | Fix |
| --- | --- | --- |
| missing `"b"` in mode | corrupted binary on Windows (`\r\n` translation) | `"wb"`, `"rb"`, `"ab"` always |
| `fwrite(p, n, 1, fp)` reversed args | return value means "1 big blob", short counts hidden | `fwrite(p, sizeof *p, n, fp)` |
| return of `fread`/`fwrite` ignored | truncated file loads silently | compare against expected count, branch on `ferror` |
| struct with `char *` dumped raw | reload gives dangling pointers | only fixed arrays (`char name[32]`) in file structs |
| padding not zeroed | nondeterministic bytes, flaky diffs | `= {0}` every record before filling |
| `long`/`int` in file struct | different size on other platform | fixed-width `<stdint.h>` types |
| `fseek`/`ftell` unchecked | silent wrong position on pipes | check `fseek != 0`, `ftell < 0` |
| assuming binary is portable | works here, garbage there | document "same build, same machine" or design a real format |

---

## Checklist

- [ ] Binary files opened with `"rb"` / `"wb"` / `"ab"`.
- [ ] `fwrite`/`fread` called as `(ptr, sizeof *ptr, count, fp)`.
- [ ] Returned item count compared against the requested count.
- [ ] Short reads branch on `ferror` (I/O error) vs `feof` (truncation).
- [ ] File structs use fixed-width types and inline char arrays, never pointers.
- [ ] Every record zero-initialized so padding is deterministic.
- [ ] `fseek` return and `ftell == -1L` handled; `rewind` only where failure is impossible.
- [ ] Array save writes an explicit count header; load validates `count <= capacity`.
- [ ] Every `fopen`/`fclose`/`fwrite` checked; program builds with `gcc -std=c11 -Wall -Wextra -pedantic -g` clean.

---

#### [Previous](./01-part.md) | [Next](./03-part.md)
