# The String Library

`<string.h>` is the toolbox for the conventions from the previous part. Most of its functions are powerful and dangerous in equal measure, and almost every one of them has a bounded alternative. This part goes function by function: what it does, how it fails, and what to use instead.

Every declaration comes from `<string.h>`. Forgetting the include is a compile error, not a subtle problem — which is one of the good things about it.

---

## The Function Table

| Function | Purpose | Danger | Safer choice |
| --- | --- | --- | --- |
| `size_t strlen(const char *s)` | count characters | no bound, needs a real terminator | keep the length yourself |
| `char *strcpy(char *d, const char *s)` | copy | no bound on the destination | `snprintf` |
| `char *strncpy(char *d, const char *s, size_t n)` | bounded copy | **may not terminate** | `strncpy` + manual `'\0'` |
| `char *strcat(char *d, const char *s)` | append | no bound | `snprintf` |
| `char *strncat(char *d, const char *s, size_t n)` | bounded append | terminates, but easy to get `n` wrong | `snprintf` |
| `int strcmp(const char *a, const char *b)` | compare | returns a value, **not** a boolean | test `== 0`, `< 0`, `> 0` |
| `int strncmp(const char *a, const char *b, size_t n)` | bounded compare | same rule about the result | test `== 0` |
| `char *strchr(const char *s, int c)` | find a character | reads to the terminator | fine if `s` is valid |
| `char *strrchr(const char *s, int c)` | find the last occurrence | same | fine |
| `char *strstr(const char *h, const char *n)` | find a substring | same | fine |
| `char *strtok(char *s, const char *delim)` | split into tokens | **destroys `s`**, keeps hidden state | write your own splitter |
| `int snprintf(char *d, size_t n, const char *fmt, ...)` | format into a buffer | truncates, returns what *would* fit | prefer this |
| `size_t strcspn(const char *s, const char *reject)` | length of the initial run *not* in `reject` | none | fine |
| `int memset(void *p, int c, size_t n)` | fill bytes | none | fine |

The pattern to notice: **`strcpy`, `strcat` and `strtok` should be rare in your code.** Everything they do has a bounded form.

---

## `strlen`

```c
size_t strlen(const char *s);
```

Walks from `s` until it finds `'\0'` and returns the number of characters before it. The terminator is not counted.

```text
  "programming"
   ^
   |  strlen counts from here
   v
  p r o g r a m m i n g \0
  0 1 2 3 4 5 6 7 8 9 10 11   <- strlen returns 11
                                   (\0 is at index 11, not counted)
```

It returns `size_t`, so print it with `%zu`.

The danger is not the function, it is the assumption behind it: **`strlen` has no length parameter and no way to stop safely.** If the array you pass has no `'\0'` inside it, `strlen` reads past the end. This is not a rare accident — it is what happens whenever you build a string with `strncpy` (below), when you read a fixed number of bytes from a file, or when you take a substring without adding the terminator.

The habit worth building: **when you know the length, keep it in a variable instead of asking for it later.**

---

## `strcpy` and the Overflow It Guarantees

```c
char *strcpy(char *dest, const char *src);
```

Copies `src`, terminator included, into `dest`. It copies as many bytes as `src` needs and stops caring how big `dest` is. The only thing that keeps it from always being a bug is that the compiler can sometimes see both sizes:

```c
char small[8] = "hi";
char *literal = "a string that is definitely longer than eight bytes";

strcpy(small, literal);   /* 44 bytes into an 8-byte buffer: overflow */
```

### The Bounded Replacement: `snprintf`

`snprintf` writes into a buffer of a size you give it, and never writes more than that — including the terminator:

```c
int snprintf(char *dest, size_t size, const char *fmt, ...);
```

Because it is also a formatter, it replaces the awkward `strcpy` + `strcat` pairs as well:

```c
char buf[64] = "hello";

strcat(buf, ", ");        /* risky: no bound */
strcat(buf, "world");     /* risky: no bound */

/* instead */
snprintf(buf, sizeof(buf), "%s, %s!", "hello", "world");
```

Three benefits in one call:

1. **Bounded.** It cannot write past `size`. If the result does not fit, it truncates and still terminates.
2. **Formatted.** `"%s, %s!"` does in one line what two `strcat` calls did, and cannot get out of sync.
3. **It reports.** The return value is the number of characters the full result *would* have needed, excluding the terminator. Compare it against the buffer size to detect truncation — the trick developed in part 3.

```text
== strcpy and strcat ==
buf = "hello, world" (buffer is 64 bytes)

== snprintf instead of strcpy and strcat ==
safe = "hello, world!", snprintf returned 13
```

`13` is the length of `"hello, world!"`, which fits comfortably in 64 bytes, so nothing was lost.

A useful helper wrapping this up:

```c
/* returns false if the text did not fit */
static bool set_text(char *dst, size_t size, const char *src) {
    int needed = snprintf(dst, size, "%s", src);
    return needed >= 0 && (size_t)needed < size;
}
```

`<stdbool.h>` gives you `bool`, `true` and `false`. Most C code uses `int` for this, which works fine too.

---

## `strncpy` and Its Trap

```c
char *strncpy(char *dest, const char *src, size_t n);
```

`strncpy` copies **at most `n` bytes**, and that is where the problem lives: if `src` is `n` bytes or longer, there is no room left for the terminator, so **`strncpy` does not add one.** It fills the remaining bytes with `'\0'` and leaves the destination unterminated.

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char src[] = "0123456789";      /* 10 characters, 11 bytes */

    char dst[8];
    strncpy(dst, src, sizeof(dst));

    printf("src = \"%s\" (%zu bytes)\n", src, sizeof(src));
    printf("dst after strncpy(dst, src, sizeof(dst)):\n");
    for (size_t i = 0; i < sizeof(dst); i++) {
        printf("  dst[%zu] = %d%s\n", i, (int)(unsigned char)dst[i],
               dst[i] == '\0' ? "   <- terminator" : "");
    }

    printf("\nprinting dst with %%s is undefined: there is no terminator\n");

    return 0;
}
```

```text
src = "0123456789" (11 bytes)
dst after strncpy(dst, src, sizeof(dst)):
  dst[0] = 48
  dst[1] = 49
  dst[2] = 50
  dst[3] = 51
  dst[4] = 52
  dst[5] = 53
  dst[6] = 54
  dst[7] = 55

printing dst with %s is undefined: there is no terminator
```

Eight digits, **zero terminators**. `%s` on that buffer keeps reading past the end of the array. AddressSanitizer catches it:

```text
=================================================================
==375734==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x720212ff0028 at pc 0x7602154d0cae bp 0x7ffdeb8fbd40 sp 0x7ffdeb8fbd0 T0
READ of size 9 at 0x720212ff0028 thread T0
    #0 0x7602154f18d3 in printf s2_strncpy2.c:9
    #1 0x594933e1538e in main s2_strncpy2.c:9

Address 0x720212ff0028 is located in stack of thread T0 at offset 40 in frame
    #0 0x594933e15258 in main s2_strncpy2.c:4

  This frame has 2 object(s):
    [32, 40) 'dst' (line 6) <== Memory access at offset 40 overflows this variable
    [64, 75) 'src' (line 5)
SUMMARY: AddressSanitizer: stack-buffer-overflow s2_strncpy2.c:9 in main
```

A read of 9 bytes from an 8-byte buffer: `printf` was looking for a terminator, did not find one, and kept going.

### Using `strncpy` Correctly

Reserve one byte yourself. Always:

```c
#include <stdio.h>
#include <string.h>

/* the correct way to use strncpy: reserve one byte for the terminator */
static void safe_copy(char *dst, size_t dst_size, const char *src) {
    strncpy(dst, src, dst_size - 1);
    dst[dst_size - 1] = '\0';
}

int main(void) {
    char src[] = "0123456789";
    char a[8];
    char b[16];

    strncpy(a, src, sizeof(a));
    a[sizeof(a) - 1] = '\0';
    printf("sizeof 8, source truncated to 7 chars: \"%s\"\n", a);

    safe_copy(b, sizeof(b), src);
    printf("sizeof 16, source copied whole:        \"%s\"\n", b);

    return 0;
}
```

```text
sizeof 8, source truncated to 7 chars: "0123456"
sizeof 16, source copied whole:        "0123456789"
```

The rule is one line long: **`strncpy(dst, src, sizeof dst - 1); dst[sizeof dst - 1] = '\0';`** Two lines, always both, never one without the other.

Even so, most of the time `snprintf` is the better answer because it does the same job in one line and cannot be half-forgotten.

---

## `strcat` and `strncat`

```c
char *strcat(char *dest, const char *src);              /* unbounded */
char *strncat(char *dest, const char *src, size_t n);  /* at most n from src */
```

`strcat` appends `src` after the terminator of `dest`. Same problem as `strcpy`: it does not know how big `dest` is, and one forgotten bound check is a stack smash.

```c
char buf[16] = "hello";
strcat(buf, " and welcome");    /* 18 bytes into 16: overflow */
```

`strncat` is better behaved: it copies at most `n` bytes from `src` **and always terminates**. Note that `n` limits the source only — it does not check the destination size, so it is bounded against `src` but not against `dest`.

```c
char buf[16] = "hello";
strncat(buf, " and welcome", sizeof(buf) - strlen(buf) - 1);   /* correct */
```

Still, `snprintf` expresses the same intent more clearly:

```c
snprintf(buf + strlen(buf), sizeof(buf) - strlen(buf), "%s", " and welcome");
```

and `strncat` is deprecated in POSIX 2008 in favour of exactly this.

---

## `strcmp` Does Not Return a Boolean

```c
int strcmp(const char *a, const char *b);
int strncmp(const char *a, const char *b, size_t n);
```

`strcmp` compares byte by byte and returns:

| Result | Meaning |
| --- | --- |
| `0` | the strings are equal |
| negative | `a` sorts before `b` |
| positive | `a` sorts after `b` |

**The magnitude is unspecified.** The standard only requires the sign. Some implementations return `-1`/`1`; glibc returns the difference between the first differing bytes, so comparing `"apple"` with `"apples"` gives `-115` — the `'s'` at index 5 against the `'\0'` at index 5.

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    const char *apple = "apple";
    const char *apples = "apples";

    printf("strcmp returns the byte difference, not necessarily -1 or 1:\n");
    printf("  strcmp(\"apple\", \"apples\") = %d\n", strcmp(apple, apples));
    printf("  strcmp(\"apples\", \"apple\") = %d\n", strcmp(apples, apple));
    printf("  strcmp(\"apple\",  \"apple\") = %d\n", strcmp(apple, apple));

    printf("\nso these are the only valid tests:\n");
    printf("  strcmp(a, b) == 0   -> %s  (equal)\n",
           (strcmp(apple, apple) == 0) ? "true" : "false");
    printf("  strcmp(a, b) <  0   -> %s  (a sorts first)\n",
           (strcmp(apple, apples) < 0) ? "true" : "false");
    printf("  strcmp(a, b) >  0   -> %s  (b sorts first)\n",
           (strcmp(apples, apple) > 0) ? "true" : "false");

    printf("\nand these are bugs, because they assume the value is exactly 1:\n");
    printf("  strcmp(\"apples\", \"apple\") == 1 -> %s\n",
           (strcmp(apples, apple) == 1) ? "true" : "false");
    printf("  strcmp(\"b\", \"a\") == 1            -> %s  (works by luck)\n",
           (strcmp("b", "a") == 1) ? "true" : "false");

    return 0;
}
```

```text
strcmp returns the byte difference, not necessarily -1 or 1:
  strcmp("apple", "apples") = -115
  strcmp("apples", "apple") = 115
  strcmp("apple",  "apple") = 0

so these are the only valid tests:
  strcmp(a, b) == 0   -> true  (equal)
  strcmp(a, b) <  0   -> true  (a sorts first)
  strcmp(a, b) >  0   -> true  (b sorts first)

and these are bugs, because they assume the value is exactly 1:
  strcmp("apples", "apple") == 1 -> false
  strcmp("b", "a") == 1            -> true  (works by luck)
```

`strcmp("b", "a") == 1` happens to be true on this machine because `'b' - 'a'` is 1. Change the strings and the test silently starts failing. Three valid forms:

```c
if (strcmp(a, b) == 0) { }   /* equal: this is the one you want 95% of the time */
if (strcmp(a, b) <  0) { }   /* a sorts first: use this for sorting */
if (strcmp(a, b) >  0) { }   /* a sorts after */

/* never this */
if (strcmp(a, b) == 1) { }   /* BUG */
if (strcmp(a, b) != 0) { }   /* fine: "not equal" is exactly the negation of zero */
```

`strcmp` is case-sensitive and compares raw bytes, so `"apple"` sorts **before** `"Apple"` because `'A'` (65) is less than `'a'` (97). For case-insensitive comparison you need `tolower` on each byte, or `strcasecmp` from `<strings.h>` (POSIX, not standard C).

---

## `strchr` and `strstr`

```c
char *strchr(const char *s, int c);      /* first occurrence of a character */
char *strrchr(const char *s, int c);     /* last occurrence  */
char *strstr(const char *h, const char *n); /* first occurrence of a substring */
```

All three return a **pointer** to the match, or `NULL` if there is none. They return `char *` so that `const char *` arguments can be passed — assigning to `char *` when the source is `const` is a discarded-qualifier warning.

```text
== strchr ==
found 'q' at offset 4: "quick brown fox"
looking for 'z': not found

== strstr ==
searching for "brown": brown fox
searching for "green": not found
```

The idiomatic use, and the reason these return pointers:

```c
const char *sentence = "the quick brown fox";

const char *q = strchr(sentence, 'q');
if (q != NULL) {
    printf("at offset %ld: %s\n", (long)(q - sentence), q);
}

if (strstr(sentence, "brown") == NULL) {
    printf("not found\n");
}
```

`q - sentence` is the offset, in characters, of the match. That is pointer subtraction inside one string, which is well defined.

Because they return a pointer and not an index, both work on `const` data without a cast. And because they stop at the terminator, they are safe as long as the string they scan really is terminated.

---

## `strtok` Mutates Its Argument

```c
char *strtok(char *s, const char *delimiters);
```

`strtok` splits a string in place. It **replaces each delimiter it finds with `'\0'`**, so the original string is destroyed. This is not documented as a side effect; it is simply how it works.

```text
== strtok ==
  field 1: "Ana"
  field 2: "31"
  field 3: "Lisbon"
  field 4: "engineer"
csv after strtok: "Ana"  (the commas were replaced)
```

```text
  before:  A n a , 3 1 , L i s b o n , e n g i n e e r \0
  after:   A n a \0 3 1 \0 L i s b o n \0 e n g i n e e r \0
                  ^         ^           ^
           strtok wrote these terminators over the commas
```

One call is needed to start (with the string), and then `NULL` to continue — the function **keeps the current position in a static variable inside itself**.

```c
char *token = strtok(csv, ",");          /* first call: pass the string */
while (token != NULL) {
    printf("%s\n", token);
    token = strtok(NULL, ",");           /* later calls: pass NULL */
}
```

The consequences you must know before using it:

1. **It destroys the input.** Keep a copy if you need the original.
2. **It keeps hidden static state.** The position lives inside the library, not in a variable you control. That means it is **not reentrant**: two loops using `strtok` at the same time (nested loops, a function called from inside the loop, two threads) will clobber each other.
3. **It cannot be called on a string literal**, because it writes to it:

```c
char *token = strtok("a,b,c", ",");   /* UNDEFINED: the literal is read-only */
```

4. Empty fields are skipped, which silently loses data. `"a,,b"` yields two tokens, not three.

For those reasons, in modern C you usually write your own splitter or use `strsep` (BSD) / `strtok_r` (POSIX), both of which keep the state in a variable you pass in and both of which report empty fields. The standard C alternative is `strcspn` (next part), which is non-destructive.

---

## A Complete String Toolkit

Everything from this part, in one program:

```c
#include <stdio.h>
#include <string.h>

static void banner(const char *title) {
    printf("== %s ==\n", title);
}

static int count_char(const char *s, char target) {
    int count = 0;
    for (const char *p = s; *p != '\0'; p++) {
        if (*p == target) {
            count++;
        }
    }
    return count;
}

static void to_upper(char *s) {
    for (; *s != '\0'; s++) {
        if (*s >= 'a' && *s <= 'z') {
            *s = (char)(*s - 'a' + 'A');
        }
    }
}

static int starts_with(const char *s, const char *prefix) {
    return strncmp(s, prefix, strlen(prefix)) == 0;
}

int main(void) {
    banner("strlen");
    const char *word = "programming";
    printf("\"%s\" has %zu characters, needs %zu bytes with the terminator\n",
           word, strlen(word), strlen(word) + 1);

    banner("strcpy and strcat");
    char buf[64] = "hello";
    strcat(buf, ", ");
    strcat(buf, "world");
    printf("buf = \"%s\" (buffer is %zu bytes)\n", buf, sizeof(buf));

    banner("snprintf instead of strcpy and strcat");
    char safe[64] = "hello";
    int written = snprintf(safe, sizeof(safe), "%s, %s!", "hello", "world");
    printf("safe = \"%s\", snprintf returned %d\n", safe, written);

    banner("strcmp");
    printf("\"apple\" vs \"apples\" = %d  (negative: apple sorts first)\n",
           strcmp("apple", "apples"));
    printf("\"apples\" vs \"apple\" = %d  (positive: apples sorts first)\n",
           strcmp("apples", "apple"));
    printf("\"apple\"  vs \"apple\"  = %d  (zero: equal)\n", strcmp("apple", "apple"));

    banner("strchr");
    const char *sentence = "the quick brown fox";
    const char *found = strchr(sentence, 'q');
    if (found != NULL) {
        printf("found 'q' at offset %ld: \"%s\"\n", (long)(found - sentence), found);
    }
    printf("looking for 'z': %s\n",
           (strchr(sentence, 'z') == NULL) ? "not found" : "found");

    banner("strstr");
    const char *sub = strstr(sentence, "brown");
    printf("searching for \"brown\": %s\n", (sub == NULL) ? "not found" : sub);
    printf("searching for \"green\": %s\n",
           (strstr(sentence, "green") == NULL) ? "not found" : "found");

    banner("helpers written by hand");
    printf("count of 'o' in \"%s\" = %d\n", sentence, count_char(sentence, 'o'));
    printf("starts_with(\"foobar\", \"foo\") = %s\n",
           starts_with("foobar", "foo") ? "true" : "false");
    printf("starts_with(\"foobar\", \"bar\") = %s\n",
           starts_with("foobar", "bar") ? "true" : "false");

    char upper[32];
    snprintf(upper, sizeof(upper), "%s", word);
    to_upper(upper);
    printf("to_upper(\"%s\") = \"%s\"\n", word, upper);

    banner("strtok");
    char csv[] = "Ana,31,Lisbon,engineer";
    int field = 1;
    for (char *token = strtok(csv, ","); token != NULL;
         token = strtok(NULL, ",")) {
        printf("  field %d: \"%s\"\n", field++, token);
    }
    printf("csv after strtok: \"%s\"  (the commas were replaced)\n", csv);

    return 0;
}
```

```text
== strlen ==
"programming" has 11 characters, needs 12 bytes with the terminator
== strcpy and strcat ==
buf = "hello, world" (buffer is 64 bytes)
== snprintf instead of strcpy and strcat ==
safe = "hello, world!", snprintf returned 13
== strcmp ==
"apple" vs "apples" = -1  (negative: apple sorts first)
"apples" vs "apple" = 1  (positive: apples sorts first)
"apple"  vs "apple"  = 0  (zero: equal)
== strchr ==
found 'q' at offset 4: "quick brown fox"
looking for 'z': not found
== strstr ==
searching for "brown": brown fox
searching for "green": not found
== helpers written by hand ==
count of 'o' in "the quick brown fox" = 2
starts_with("foobar", "foo") = true
starts_with("foobar", "bar") = false
to_upper("programming") = "PROGRAMMING"
== strtok ==
  field 1: "Ana"
  field 2: "31"
  field 3: "Lisbon"
  field 4: "engineer"
csv after strtok: "Ana"  (the commas were replaced)
```

Note `starts_with`: it uses `strncmp` with `strlen(prefix)` and tests `== 0`, which is exactly the pattern from the `strcmp` section applied to a bounded comparison.

---

## Summary

- `strlen` counts bytes before the terminator and has no bound; keep lengths in variables when you have them.
- `strcpy` and `strcat` have no bound at all. Use `snprintf`, which formats, bounds, and reports.
- `strncpy` does not terminate the destination when the source fills it. Always follow it with `dst[sizeof dst - 1] = '\0';`, or use `snprintf`.
- `strncat` always terminates, but bounds only the source.
- `strcmp` returns the **sign** of a comparison, not `-1`/`0`/`1`. Test `== 0`, `< 0`, `> 0`. `== 1` is a bug.
- `strchr` and `strstr` return a pointer to the match or `NULL`, which lets you compute the offset.
- `strtok` destroys its argument, keeps state in a static variable, cannot take a string literal, and skips empty fields.
- Prefer `snprintf` for anything that builds a string. It is the one function in this table with no trap.

Next part: reading text from the keyboard correctly and building strings without overflow.

---

#### [Previous](./01-part.md) | [Next](./03-part.md)