# Exercise List 2 - Strings

## Exercise 1: Full Name

Read a full name with `fgets` (it contains spaces, so `scanf("%s")` cannot do it), then print how many words and how many letters it has.

```text
# input
Ana Maria da Silva

# output
text   = "Ana Maria da Silva"
words  = 4
letters= 15
```

Requirements:

- Strip the newline: `line[strcspn(line, "\r\n")] = '\0';`. Without it, the last "letter" is a `'\n'` and the count is off by one.
- Count words without `strtok`: use a flag that says whether you are currently inside a word, so that repeated spaces do not create empty words.
- Extra credit: print the name in title case (`ana maria` becomes `Ana Maria`).

## Exercise 2: Your Own `<string.h>`

Write these three without calling the library version:

```c
int  my_strlen(const char *s);
void my_strcpy(char *dst, const char *src);
int  my_strcmp(const char *a, const char *b);
```

Then check them against the real ones.

```text
# output
my_strlen("container")   = 9   (strlen says 9)
my_strlen("")           = 0   (strlen says 0)
my_strcpy -> "copied by hand"   (strlen 14)
my_strcmp("container", "contents") = -4  (negative: first is smaller)
my_strcmp("contents", "container") = 4  (positive)
my_strcmp("same", "same")           = 0  (zero: equal)
```

Requirements:

- `my_strcmp` must return 0, a negative value, or a positive value — and **never** more than that. Do not return `a[i] - b[i]` on `char` values without the `unsigned char` cast, or the sign can be wrong for bytes above 127.
- `my_strlen` returns `size_t` in the standard library. Returning `int` is simpler for a beginner and fine for short strings; say so in a comment.
- `my_strcpy` has no destination size parameter, exactly like `strcpy`. Write a comment naming the one rule that makes calling it safe: the destination must be at least `my_strlen(src) + 1` bytes.
- Test `my_strcmp("same", "same")`: it must return exactly `0`, not `1`.

## Exercise 3: Reverse a String, Twice

Write two versions of an in-place reverse.

```text
# output
original:  "pointers"
index:     "sretniop"
pointers:  "sretniop"
same result? yes
```

Requirements:

- Version one uses subscripts: `s[i]` and `s[n - 1 - i]`.
- Version two uses only pointers: a `char *` at each end, moving with `left++` and `right--`, and `while (left < right)`.
- Both need a temporary variable. Write a comment explaining what `a = b; b = a;` would do instead.
- What happens with an empty string in version two? `s + strlen(s) - 1` then points *before* the buffer. Explain in a comment why that is not a dereference and why the loop body never runs. (Forming that pointer is still questionable practice; note it.)
- Extra credit: a third version using only `char *` arithmetic with `*p++` and `*(q--)`.

## Exercise 4: Palindrome

Write `int is_palindrome(const char *s)` that ignores case and spaces.

```text
# output
Never odd or even            -> palindrome
Ana                          -> palindrome
hello                        -> not a palindrome
Was it a car or a cat I saw  -> palindrome
```

Requirements:

- **`strlen` on the raw string is not enough.** Write a comment saying why: `"Never odd or even"` has 19 characters but only 13 non-space characters, so the indices of the mirrored pair do not line up if you use the raw length. Clean the string into a second buffer first (lowercase, spaces removed), then take the length of *that*.
- Empty string: is it a palindrome? Decide, and say so in a comment.
- Extra credit: do it without a second buffer, using two pointers that skip spaces and lowercase as they go.

## Exercise 5: Vowel Count

Read a sentence and count each vowel with an array of five counters.

```text
# input
The quick brown fox jumps over the lazy dog

# output
vowel  count
  a        1
  e        3
  i        1
  o        4
  u        2
  all     11
```

Requirements:

- Convert uppercase to lowercase with arithmetic (`c - 'A' + 'a'`) rather than `tolower`, so `<ctype.h>` is not needed.
- Look each character up in a `const char vowels[] = "aeiou";` string, or use `strchr`. Both are fine; pick one and be able to explain it.
- Print the table with a width specifier so it lines up, and print the total at the end.
- Careful: `'o'` appears twice in `"brown"` and twice in `"over"`. Re-count the expected output before you trust it.

## Exercise 6: Split a CSV Line

Split a comma-separated line into fields with `strtok` and print them numbered.

```text
# input
Ana,31,Lisbon,engineer

# output
  1: Ana
  2: 31
  3: Lisbon
  4: engineer
```

Requirements:

- **Explain in comments why `strtok` on a string literal is a bug.** `strtok` writes to its first argument: it replaces each delimiter with `'\0'`. A literal is in read-only memory, so `strtok("a,b", ",")` is undefined behaviour and usually crashes. That is why the program copies the literal into a writable array first.
- The first call passes the string; the following calls pass `NULL`. Write a comment explaining that `strtok` keeps the current position in a static variable inside the library.
- The original line is destroyed by the call. Write a comment about what that means for the rest of your program.
- Extra credit: split on `", "` instead and note the leading spaces in every field. Then explain in a comment why that is worse, and what `strtok(csv, " ,")` would do about it.

---

#### [Previous](./03-part.md) | [Next](../03-structs/01-part.md)