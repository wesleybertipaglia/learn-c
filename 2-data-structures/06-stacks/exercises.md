# Exercise List 6 - Stacks

Six exercises, from the mechanics of part 1 to the applications of part 2. Keep one stack implementation per program, compile with:

```text
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer
```

Ownership rule for all six: the stack owns its buffer (array version) or its nodes (linked version). `main` calls exactly one `destroy`. Underflow from input returns `-1` and prints `error` — it never aborts.

## How to Work These Exercises

- Do them in order: 1 gives you the container, 2–4 use it, 5 rebuilds it linked, 6 combines two stacks.
- Keep each exercise in its own `.c` file (for example `ex1.c` … `ex6.c`). Do not share one stack implementation across files by copy-pasting bugs — retype it and compare.
- After every exercise run both the happy path and the error path shown (empty pop, bad brackets, `5 +`), then run under ASan/LSan and confirm zero reports.
- If `gcc` warns, fix the warning before testing: `-Wall -Wextra -pedantic` catches the `size_t` vs `int` and missing-`const` mistakes these exercises are designed to surface.

---

## Exercise 1: Array Stack Push / Pop / Peek

Implement the part-1 array stack and drive it with a small script of commands.

```c
#include <stddef.h>
#include <stdbool.h>

typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} Stack;

static void stack_init(Stack *s);
static void stack_destroy(Stack *s);
static bool stack_is_empty(const Stack *s);
static int stack_push(Stack *s, int value);   /* 0 ok, -1 out of memory */
static int stack_pop(Stack *s, int *out);     /* 0 ok, -1 underflow */
static int stack_peek(const Stack *s, int *out);

int main(void) {
    /* TODO: init, run the pushes/pops below, print after each step, destroy. */
    return 0;
}
```

```text
# script
push 10, push 20, push 30, peek, pop, pop, pop, pop (one too many)

# output
pushed 10 (size=1, capacity=4)
pushed 20 (size=2, capacity=4)
pushed 30 (size=3, capacity=4)
peek: 30
popped 30 (size=2)
popped 20 (size=1)
popped 10 (size=0)
error: pop on empty stack
```

Requirements:

- Start with `data == NULL, size == 0, capacity == 0`; first `push` allocates 4, then double (`4, 8, 16, ...`).
- `realloc` goes into a `tmp` pointer; on `NULL` return `-1` and keep the old buffer valid.
- Readers (`is_empty`, `peek`) take `const Stack *`.
- The final over-pop prints `error: pop on empty stack` and does not abort.
- `main` ends with `stack_destroy`; ASan/LSan output is clean.

---

## Exercise 2: Balanced Brackets

Read one line (up to 255 characters) and report whether `()`, `[]`, and `{}` are balanced. Ignore all other characters.

```c
#include <stdbool.h>

static bool balanced(const char *s);   /* TODO: char-stack implementation */

int main(void) {
    char line[256];
    /* TODO: read with fgets, strip newline, print balanced / NOT balanced. */
    return 0;
}
```

```text
# input
a+(b*[c-{d}])

# output
balanced
```

```text
# input
([)]

# output
NOT balanced
```

```text
# input
((

# output
NOT balanced
```

Requirements:

- Push every opener; on a closer, `pop` and check the pair matches — a counter is not enough (`([)]` defeats it).
- Popping an empty stack means `NOT balanced`, not a crash.
- Leftover openers after the scan mean `NOT balanced`.
- The internal `char` stack uses doubling growth with a `tmp` `realloc` and is destroyed on every path, including out-of-memory.
- Extra credit: report the 0-based index of the first offending bracket.

---

## Exercise 3: Postfix Calculator

Read one line like `3 4 + 2 *` and print the integer result, or `error` for bad input.

```c
static int eval_postfix(const char *expr, int *result);  /* 0 ok, -1 error */

int main(void) {
    char line[256];
    /* TODO: fgets, eval_postfix, print result or "error". */
    return 0;
}
```

```text
# input
3 4 + 2 *

# output
14
```

```text
# input
10 2 / 3 -

# output
2
```

```text
# input
5 +

# output
error
```

Requirements:

- Operands are integers (handle multi-digit and negatives via `strtol`); operators are `+ - * /`.
- Each operator pops `b` then `a` and pushes `a op b` — swapping the order breaks `-` and `/`.
- These are all `error`: stack underflow, unknown token, divide by zero, zero or 2+ values left at the end.
- `strtok` runs on a `malloc`+`strcpy` copy, never on the caller's `const char *` or a literal; both the copy and the stack are freed on every path.
- Ownership: the evaluator owns its stack and its copy; `main` owns the input line.

---

## Exercise 4: Reverse a String with a Stack

Read one line and print it reversed, using a stack rather than swapping in place.

```c
static int reverse_print(const char *s);  /* 0 ok, -1 on allocation failure */

int main(void) {
    char line[256];
    /* TODO: fgets, strip newline, reverse_print. */
    return 0;
}
```

```text
# input
hello

# output
olleh
```

```text
# input
racecar

# output
racecar
```

Requirements:

- Push every character, then pop them all — the LIFO order is the reversal. Do not use the `strlen - 1 - i` idiom; the point is the stack.
- The `char` stack grows by doubling via `tmp`; empty input prints an empty line and still succeeds.
- Underflow after the pushes would mean a logic bug (prove it cannot happen), but still guard the `pop` return.
- Strip the `fgets` newline before pushing, or the output starts with a blank line.
- Destroy the stack before returning; no leaks on any path.

---

## Exercise 5: Linked Stack

Rebuild exercise 1 on the part-2 linked stack: same script, same output shape, no array, no `realloc`.

```c
typedef struct Node {
    int value;
    struct Node *next;
} Node;

typedef struct {
    Node *top;
    size_t size;
} LinkedStack;

static void lstack_init(LinkedStack *s);
static void lstack_destroy(LinkedStack *s);
static int lstack_push(LinkedStack *s, int value);  /* 0 ok, -1 out of memory */
static int lstack_pop(LinkedStack *s, int *out);    /* 0 ok, -1 underflow */
static int lstack_peek(const LinkedStack *s, int *out);

int main(void) {
    /* TODO: same script as exercise 1, then destroy. */
    return 0;
}
```

```text
# script
push 10, push 20, push 30, peek, pop, pop, pop, pop (one too many)

# output
pushed 10 (size=1)
pushed 20 (size=2)
pushed 30 (size=3)
peek: 30
popped 30 (size=2)
popped 20 (size=1)
popped 10 (size=0)
error: pop on empty stack
```

Requirements:

- `push` is one `malloc` + pointer splice; `pop` is one `free`. No capacity field, no copying.
- `NULL` from `malloc` returns `-1` with the stack unchanged.
- `destroy` saves `next` before `free` and resets `top == NULL, size == 0`.
- Underflow returns `-1`; the trailing over-pop prints the `error` line.
- LSan clean: one `free` per pushed node, exactly once.

---

## Exercise 6: Min-Stack (`get_min` in O(1)) or Undo Buffer

Pick one. Both force a second stack to work alongside the first.

### Option A: Min-Stack

Support `push`, `pop`, and `get_min` — all in O(1).

```c
typedef struct {
    int *data;       /* main stack */
    int *mins;       /* mins[i] = minimum of data[0..i] */
    size_t size;
    size_t capacity;
} MinStack;

static int min_push(MinStack *s, int value);
static int min_pop(MinStack *s, int *out);
static int min_get_min(const MinStack *s, int *out);  /* 0 ok, -1 empty */
static void min_destroy(MinStack *s);

int main(void) {
    /* TODO: drive the script below. */
    return 0;
}
```

```text
# script
push 5, push 3, push 7, min, pop, min, pop, min, pop, min

# output
pushed 5, min=5
pushed 3, min=3
pushed 7, min=3
popped 7, min=3
popped 3, min=5
popped 5, empty
error: min on empty stack
```

Requirements:

- On `push`, store `value < mins[size-1] ? value : mins[size-1]` (first element: `value`). `get_min` then reads `mins[size-1]` in O(1) — no scan.
- Both buffers double together via `tmp` `realloc`; if the second `realloc` fails, roll back without losing data and return `-1`.
- `pop`/`get_min` on empty return `-1`; the final `min` prints the `error` line.
- `min_destroy` frees both buffers.

### Option B: Undo Buffer

Implement a tiny integer accumulator with undo: `+n` adds, `-n` subtracts, `undo` reverts the last change, `print` shows the value.

```c
/* TODO: action stack holding inverse deltas; undo pops and applies. */
```

```text
# script
+5, +3, print, undo, print, undo, print, undo

# output
value=8
value=5
value=0
error: nothing to undo
```

Requirements:

- Each `+n`/`-n` pushes its inverse (`-n`/`+n`); `undo` pops the inverse and applies it.
- Undo on an empty stack prints `error: nothing to undo` and changes nothing.
- The action stack is a regular array stack (doubling, `tmp`, status returns) or a linked stack — your choice, stated in a comment.
- Destroy the stack before exit; ASan/LSan clean.

---

#### [Previous](./02-part.md) | [Next](../07-queues/01-part.md)
