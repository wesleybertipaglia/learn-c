# Final Challenge of Module 3

You have finished advanced C: recursion and hand-written algorithms, bit manipulation, function pointers, file I/O, and multi-file builds. This project uses all of it. It is the last thing you build before the testing module teaches you how to prove it works.

---

## Objective

Write a **command-line student registry** in C, built as a real multi-file project with a `Makefile`, that stores student records, sorts and searches them with function-pointer-driven algorithms, packs per-student status into bit flags, and persists everything to both a text (CSV) file and a binary snapshot.

The project must compile with zero warnings under strict flags and run sanitizer-clean:

```bash
make
make sanitize
./registry
```

---

## Features

The menu, printed once at startup and again after every operation:

```text
  1) Add student
  2) List students
  3) Sort by field
  4) Find by id (binary search)
  5) Toggle status flag
  6) Save (CSV + binary)
  7) Load (CSV)
  8) Exit
```

| Option | Behaviour |
| --- | --- |
| 1 | Ask for id (`int`), name and grade (`double`). Reject duplicates, empty names, and out-of-range grades. Append to the dynamic array |
| 2 | Print every record as `id  name  grade  flags`, in current order |
| 3 | Ask which field (`id`, `name`, `grade`). Sort with `qsort` using the matching comparator from a dispatch table — no `switch` on the field inside the sort call |
| 4 | Ask for an id. Sort a **copy** of the id order if needed, then `bsearch` it. Report the index and the record, or "not found". Never binary-search an unsorted array |
| 5 | Ask for an id and a flag (`active`, `scholar`, `graduated`). Flip that single bit with `^`, leaving the others untouched, and print the flags before and after in binary |
| 6 | Write `registry.csv` (one `id;name;grade;flags` line per student) **and** `registry.bin` (a `fwrite` snapshot with a count header). Check every `fwrite`/`fclose` return |
| 7 | Load `registry.csv` back with `fgets` + `strtok`/`strtol`/`strtod`, skipping malformed lines with a message to `stderr`. The load **replaces** the in-memory array |
| 8 | Print a summary (count, average grade computed **recursively**, save reminder if dirty) and exit 0 with everything freed |

---

## Requirements

### Project layout

No single-file solutions. The registry is three translation units plus headers:

```text
registry/
  Makefile
  include/
    student.h      /* Record, flags enum, public API with guards */
    storage.h      /* CSV + binary API */
  src/
    main.c         /* menu loop only; no logic inline */
    student.c      /* array mgmt, comparators, sort/search, flags, recursion */
    storage.c      /* text + binary persistence */
```

`student.h` declares and `student.c` defines:

```c
#define NAME_MAX 48

typedef enum {
    FLAG_ACTIVE   = 1u << 0,
    FLAG_SCHOLAR  = 1u << 1,
    FLAG_GRADUATED = 1u << 2
} status_flag_t;

typedef struct {
    int      id;
    char     name[NAME_MAX];
    double   grade;
    uint32_t flags;    /* bit pack of status_flag_t */
} Student;

typedef int (*student_cmp_t)(const void *, const void *);
```

Rules:

- Include guards in every header (`#ifndef STUDENT_H`). No variable or non-`static` function **defined** in a header.
- `static` on every helper not in the public API. `extern` only if you truly share a global — and then justify it in a comment (the right answer is almost always "I removed it").
- The `Makefile` has at least `all`, `clean` and `sanitize` targets, builds with `-Iinclude`, and treats warnings as errors:
  ```make
  CC = gcc
  CFLAGS = -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude
  ```

### Algorithms (by hand where it counts)

1. **Recursion.** The average grade is computed by a recursive sum over the array (`sum_rec(arr, n)` with base case `n == 0`). Document why recursion is fine here (depth equals the student count — and what you would do if it had a million rows).
2. **Sort via function pointers.** Three comparators (`cmp_id`, `cmp_name`, `cmp_grade`) behind one dispatch table. The comparators return `-1`/`0`/`1` with comparisons — never `return a - b` on `int`s and never `<` on `double`s subtracted. `qsort` does the sorting; choosing *how* is the table's job.
3. **Binary search.** `bsearch` with `cmp_id` on an array sorted by id. If the array is not currently id-sorted, sort a temporary copy (or re-sort in place and say so). Searching unsorted memory is a wrong answer, not an optimization.
4. **Bits.** Flags live in one `uint32_t`. Set with `|`, clear with `& ~`, toggle with `^`, test with `&`. Print them with a `print_bits` helper (`%u`/`0x%x` plus an 8/32-bit diagram). Shifts only on unsigned types, never `>= 32`.

### File I/O

- Text format is CSV with `;` separators (names may contain spaces, never `;` — reject `;` on input):
  ```text
  101;Wesley;8.50;1
  102;Ana;9.25;3
  ```
- Load path: `fopen` checked with `perror`, lines with `fgets`, parse with `strtol`/`strtod` + `endptr` checks, malformed lines skipped with `line N: <reason>` on `stderr`. `fclose` checked.
- Binary path: `fwrite(&count, sizeof count, 1, fp)` then `fwrite(vec, sizeof(Student), count, fp)`, verifying each call wrote exactly what was asked. Document the padding caveat in a comment: this snapshot is only valid on the same platform/compiler, which is why the CSV is the portable format.
- Every failure path uses the `goto cleanup` pattern: one label, `fclose(NULL)`-safe (`FILE *fp = NULL; ... cleanup: if (fp) fclose(fp);`).

### Error handling and style

1. A `status_t` enum (`OK`, `INVALID`, `DUPLICATE`, `NOT_FOUND`, `NOMEM`, `IO_ERROR`) with `status_text`. Every fallible function returns one; `main` checks all of them.
2. Errors to `stderr`, results to `stdout`. Prove it: `./registry < demo.txt > out.txt 2> err.txt`.
3. `const` on read-only pointer parameters, `size_t` for counts/indices, `static` helpers, one declaration per line.
4. `fgets` for all input; `strtol`/`strtod` with range checks for numbers. No `scanf("%s")`, no `gets`, no unbounded `strcpy` of user input.

---

## Example of Operation

```text
$ make
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude -c src/main.c -o src/main.o
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude -c src/student.c -o src/student.o
gcc -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude -c src/storage.c -o src/storage.o
gcc src/main.o src/student.o src/storage.o -o registry

$ ./registry
========================================
   Student Registry (module 3)
========================================

  1) Add student
  2) List students
  3) Sort by field
  4) Find by id (binary search)
  5) Toggle status flag
  6) Save (CSV + binary)
  7) Load (CSV)
  8) Exit
========================================
Choice: 1
Id: 102
Name: Ana
Grade: 9.25
Added Ana (id 102).

Choice: 1
Id: 101
Name: Wesley
Grade: 8.50
Added Wesley (id 101).

Choice: 2
Students (2):
  102  Ana      9.25  flags=0x00000000 (00000000)
  101  Wesley   8.50  flags=0x00000000 (00000000)

Choice: 3
Field (id/name/grade): grade
Sorted by grade.

Choice: 2
Students (2):
  101  Wesley   8.50  flags=0x00000000 (00000000)
  102  Ana      9.25  flags=0x00000000 (00000000)

Choice: 5
Id: 102
Flag (active/scholar/graduated): scholar
Ana flags: 00000000 -> 00000010

Choice: 4
Id: 101
Found at index 0: 101 Wesley 8.50

Choice: 6
Saved 2 student(s) to registry.csv and registry.bin.

Choice: 8
----------------------------------------
Summary
Students: 2
Average grade (recursive): 8.88
Goodbye.
```

And `registry.csv` afterwards:

```text
101;Wesley;8.50;0
102;Ana;9.25;2
```

Notes:

- Sort by grade reordered the listing; find-by-id still worked because the search sorts (a copy) by id first.
- Toggling `scholar` changed only bit 1: `00000000 -> 00000010`. The other bits are untouched.
- The average `8.88` came from the recursive sum, printed with `%.2f`.

---

## Tips

### The comparator is where the bug lives

```c
/* WRONG: overflows when a is INT_MAX and b is negative */
static int cmp_id_wrong(const void *a, const void *b) {
    return ((const Student *)a)->id - ((const Student *)b)->id;
}

/* RIGHT: compare, do not subtract */
static int cmp_id(const void *a, const void *b) {
    int x = ((const Student *)a)->id;
    int y = ((const Student *)b)->id;

    if (x < y) {
        return -1;
    }
    if (x > y) {
        return 1;
    }
    return 0;
}
```

The wrong version passes every test with small ids and fails the day ids get large. Same for `double`: compare with `<`/`>`, never subtract and cast.

### Binary search needs sorted input — enforce it

```c
static status_t find_by_id(Student *vec, size_t n, int id, const Student **out) {
    /* bsearch requires id order; ensure it on a copy or re-sort in place */
    ...
}
```

If option 3 last sorted by grade, the array is *not* id-ordered. Either keep a note of the current order or sort a heap copy by id inside the find. An unsorted `bsearch` returns nonsense with no error — the most silent bug in this project.

### One label cleans everything

```c
FILE *fp = NULL;
status_t st = STATUS_IO_ERROR;

fp = fopen(path, "w");
if (fp == NULL) {
    perror("fopen");
    goto cleanup;
}
...
st = STATUS_OK;
cleanup:
if (fp != NULL) {
    fclose(fp);
}
return st;
```

`fp` starts `NULL` so every path through the function is safe, including the one where `fopen` itself failed.

---

## Extra challenge

If the required set builds with `-Werror`, passes the sanitizers, and round-trips through both file formats, pick one:

1. **Index file.** Keep `registry.idx` (sorted ids + file offsets) so find-by-id seeks instead of scanning. Rebuild it on every save.
2. **Stable multi-key sort.** Sort by grade, then by name within equal grades, with a single composite comparator. Prove stability with three tied grades.
3. **Recursive listing.** Print the registry with a recursive walk instead of a loop, and add a depth guard that refuses absurd sizes with a clean status.
4. **Flag queries.** Add "list all scholars" using only bit tests (`flags & FLAG_SCHOLAR`), without touching names or grades.
5. **Shared library.** Build `libregistry.a` with `ar rcs` from `student.o` + `storage.o`, link `main.o` against `-L. -lregistry`, and document the link order in the `Makefile`.

---

## Grading Checklist

### Build

```text
[ ] make builds with -std=c11 -Wall -Wextra -Werror -pedantic -g -Iinclude, zero warnings
[ ] make clean && make rebuilds from scratch
[ ] make sanitize (or the documented ASan/UBSan flags) runs with no diagnostics
[ ] include guards in every header; no definitions in headers; helpers static
```

### Structure

```text
[ ] src/main.c is the menu loop only; logic lives in student.c / storage.c
[ ] Sort goes through qsort + dispatch table of comparators (no field switch at call site)
[ ] Search goes through bsearch on id order (copy re-sorted or order tracked)
[ ] Flags are one uint32_t with | & ~ ^ and a print_bits helper; shifts unsigned-only
[ ] Average uses a genuinely recursive sum with a base case
```

### Persistence

```text
[ ] Save writes registry.csv and registry.bin; every fwrite/fclose return checked
[ ] Load parses with fgets + strtol/strtod + endptr, skips bad lines with messages
[ ] Binary format documented as platform-specific; CSV documented as portable
[ ] All file paths go through goto cleanup with NULL-initialized FILE *
```

### Error handling

```text
[ ] status_t enum + status_text; every fallible function returns one, all checked
[ ] Errors on stderr, results on stdout (redirection proof)
[ ] Duplicate id, bad grade, unknown flag, missing file all handled, program continues
[ ] No assert on user input or file contents
```

### Behaviour

```text
[ ] Option 3 sorts by all three fields correctly (verified by listing after each)
[ ] Option 4 finds existing ids and reports missing ones without crashing
[ ] Option 5 flips exactly one bit (before/after binary print proves it)
[ ] Options 6+7 round-trip: save, exit, load in a fresh run, identical listing
[ ] Option 8 prints count + recursive average and exits 0 leak-free
```

A submission that builds multi-file, sorts and searches through function pointers, toggles single bits, and round-trips both file formats sanitizer-clean has met the bar. One that also documents its padding, guards its recursion depth, or ships a static library is well above it.

---

#### [Previous](./05-modules-and-build/exercises.md) | [Next](../4-testing/01-introduction/01-part.md)
