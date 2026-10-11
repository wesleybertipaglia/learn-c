# Stacks with Arrays

A stack is the simplest container with a rule: **last in, first out** (LIFO). The last element you put in is the first one you take out. Think of a stack of plates in a cafeteria: you add a plate on top, you remove a plate from the top. You never reach into the middle.

C has no built-in stack type. You build one from an array, a size, and a capacity — and that small struct teaches you almost everything about dynamic containers: ownership, growth, and how to report failure without crashing.

Compile everything in this chapter with:

```text
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer
```

---

## The LIFO Rule

A stack supports exactly three essential operations:

| Operation | What it does | Rule |
| --- | --- | --- |
| `push(value)` | puts `value` on top | grows the stack by one |
| `pop(&out)` | removes the top and stores it in `out` | fails if the stack is empty |
| `peek(&out)` | reads the top without removing it | fails if the stack is empty |

There is no "insert at index 2" and no "remove the bottom". That restriction is the point: by allowing access only at the top, every operation is O(1), and the logic of callers (parsers, evaluators, undo buffers) becomes trivial to reason about.

```text
  push(10)   push(20)   push(30)   pop -> 30   peek -> 20
  +------+   +------+   +------+   +------+   +------+
  |      |   |      |   |  30  |   |      |   |      |
  +------+   +------+   +------+   +------+   +------+
  |      |   |  20  |   |  20  |   |  20  |   |  20  |  <- top
  +------+   +------+   +------+   +------+   +------+
  |  10  |   |  10  |   |  10  |   |  10  |   |  10  |
  +------+   +------+   +------+   +------+   +------+
```

---

## The Struct: `size` Versus `capacity`

```c
#include <stddef.h>

typedef struct {
    int *data;          /* heap buffer holding the elements, bottom at index 0 */
    size_t size;        /* number of elements currently stored */
    size_t capacity;    /* number of elements the buffer can hold without growing */
} Stack;
```

`size` and `capacity` answer different questions:

- `size` is how many elements are logically in the stack right now. The top element, when `size > 0`, lives at `data[size - 1]`.
- `capacity` is how many elements fit before the next `push` must grow the buffer. Invariant: `size <= capacity` always holds.

A freshly created stack has `size == 0` and `capacity == 0` with `data == NULL`. That is a valid empty stack, not a broken one: the first `push` allocates.

### `stack_init` and `stack_destroy`: Who Owns What

The stack **owns** its `data` buffer. Whoever calls `stack_init` must later call `stack_destroy`, exactly like `malloc` and `free`.

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} Stack;

static void stack_init(Stack *s) {
    s->data = NULL;
    s->size = 0;
    s->capacity = 0;
}

static void stack_destroy(Stack *s) {
    free(s->data);
    s->data = NULL;
    s->size = 0;
    s->capacity = 0;
}
```

`stack_destroy` is safe to call on an empty or already-destroyed stack because `free(NULL)` is a no-op. Setting the fields back to zero turns a use-after-destroy into a clean empty stack instead of a dangling pointer.

> Ownership rule for this whole chapter: the `Stack` struct owns `data`. No other code may `free` it, and `pop` never frees — it only hands you a *copy* of the top `int`.

---

## Reading: `is_empty` and `peek`

Readers take a `const Stack *`, because they never modify the container. Writers take a `Stack *`.

```c
#include <stdbool.h>

static bool stack_is_empty(const Stack *s) {
    return s->size == 0;
}

static size_t stack_size(const Stack *s) {
    return s->size;
}

/* Returns 0 on success, -1 when the stack is empty. */
static int stack_peek(const Stack *s, int *out) {
    if (s->size == 0) {
        return -1;
    }
    *out = s->data[s->size - 1];
    return 0;
}
```

Note the status-return convention used everywhere here:

- `0` means success.
- `-1` means a *recoverable* failure: empty stack, bad argument, out of memory.

The caller decides what to do. A calculator prints `error: stack underflow`. A test asserts. Nobody calls `abort()`, `exit()`, or `assert` inside the container just because input was bad — input is the caller's problem, and a library that kills the process on bad input cannot be used in a long-running program.

Underflow from user input must never abort. This is the rule:

```c
int top;
if (stack_peek(&s, &top) != 0) {
    fprintf(stderr, "error: peek on empty stack\n");
    /* recover: ask again, skip line, return error — but do not abort */
}
```

---

## Writing: `push` with Doubling Growth

`push` is where arrays earn their keep. If there is room (`size < capacity`), store and increment. If not, grow the buffer — by doubling — then store.

```c
/* Returns 0 on success, -1 on allocation failure. */
static int stack_push(Stack *s, int value) {
    if (s->size == s->capacity) {
        size_t new_cap = (s->capacity == 0) ? 4 : s->capacity * 2;
        int *tmp = realloc(s->data, new_cap * sizeof(*s->data));
        if (tmp == NULL) {
            return -1;              /* s->data is still valid: nothing leaked */
        }
        s->data = tmp;
        s->capacity = new_cap;
    }
    s->data[s->size] = value;
    s->size++;
    return 0;
}
```

Three details matter:

1. **`realloc` goes into a temporary.** Writing `s->data = realloc(s->data, ...)` loses the old pointer when `realloc` returns `NULL`, leaking the whole stack. The `tmp` pattern keeps the old buffer intact on failure.
2. **Check for `NULL`.** `malloc` and `realloc` can fail. On failure `push` returns `-1` and the stack is unchanged: the element was not added, but nothing that was there is lost.
3. **Start at 4, then double.** Growing by one element each time (`capacity + 1`) makes `n` pushes cost O(n^2), because every push copies. Doubling gives amortized O(1) per push: the copies are rare enough that the average cost is constant.

The growth sequence looks like this:

| Pushes so far | `capacity` before | Action |
| --- | --- | --- |
| 0 | 0 | allocate 4 |
| 1–4 | 4 | store directly |
| 5 | 4 | `realloc` to 8, then store |
| 6–8 | 8 | store directly |
| 9 | 8 | `realloc` to 16, then store |

### Diagram: A Stack Growing

Push `10, 20, 30, 40, 50` into an empty stack with initial capacity 4. The fifth push triggers the first `realloc`:

```text
  after push(10), push(20), push(30), push(40):   size=4  capacity=4

  index:    [0]   [1]   [2]   [3]
          +-----+-----+-----+-----+
          | 10  | 20  | 30  | 40  |   <- top is index 3 (value 40)
          +-----+-----+-----+-----+
              data (4 x int = 16 bytes)

  push(50): size == capacity, so realloc to 8 first:

  index:    [0]   [1]   [2]   [3]   [4]   [5]   [6]   [7]
          +-----+-----+-----+-----+-----+-----+-----+-----+
          | 10  | 20  | 30  | 40  | 50  |  ?  |  ?  |  ?  |   <- top is index 4
          +-----+-----+-----+-----+-----+-----+-----+-----+
              new data (8 x int = 32 bytes, old contents copied)

          size = 5, capacity = 8
          slots [5..7] are allocated but indeterminate: never read them.
```

The bottom of the stack is always `data[0]`. The top is always `data[size - 1]`. `capacity` counts slots; `size` counts live values.

---

## Writing: `pop`

`pop` is the mirror of `push`. It reports underflow with `-1` instead of crashing:

```c
/* Removes the top element, stores it in *out. Returns 0 / -1. */
static int stack_pop(Stack *s, int *out) {
    if (s->size == 0) {
        return -1;              /* underflow: caller handles it */
    }
    s->size--;
    *out = s->data[s->size];
    return 0;
}
```

`pop` intentionally does **not** shrink the buffer. Halving on every pop causes thrashing when the size hovers around a boundary (push grows, pop shrinks, push grows again — every operation copies). If you ever add shrinking, do it lazily, for example only when `size < capacity / 4`. This chapter never shrinks: simple and correct.

`pop` also does not zero the vacated slot. The value at `data[size]` after a pop is stale but harmless, because nothing reads past `size`. Zeroing would cost time on every pop for no safety gain — the invariant "only indices below `size` are valid" already protects you.

---

## A Complete Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <stddef.h>

typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} Stack;

static void stack_init(Stack *s) {
    s->data = NULL;
    s->size = 0;
    s->capacity = 0;
}

static void stack_destroy(Stack *s) {
    free(s->data);
    s->data = NULL;
    s->size = 0;
    s->capacity = 0;
}

static bool stack_is_empty(const Stack *s) {
    return s->size == 0;
}

static int stack_peek(const Stack *s, int *out) {
    if (s->size == 0) {
        return -1;
    }
    *out = s->data[s->size - 1];
    return 0;
}

static int stack_push(Stack *s, int value) {
    if (s->size == s->capacity) {
        size_t new_cap = (s->capacity == 0) ? 4 : s->capacity * 2;
        int *tmp = realloc(s->data, new_cap * sizeof(*s->data));
        if (tmp == NULL) {
            return -1;
        }
        s->data = tmp;
        s->capacity = new_cap;
    }
    s->data[s->size] = value;
    s->size++;
    return 0;
}

static int stack_pop(Stack *s, int *out) {
    if (s->size == 0) {
        return -1;
    }
    s->size--;
    *out = s->data[s->size];
    return 0;
}

static void stack_dump(const Stack *s) {
    printf("stack (size=%zu, capacity=%zu): bottom [ ", s->size, s->capacity);
    for (size_t i = 0; i < s->size; i++) {
        printf("%d ", s->data[i]);
    }
    printf("] top\n");
}

int main(void) {
    Stack s;
    stack_init(&s);

    int values[] = {10, 20, 30, 40, 50};
    for (size_t i = 0; i < sizeof(values) / sizeof(values[0]); i++) {
        if (stack_push(&s, values[i]) != 0) {
            fprintf(stderr, "out of memory\n");
            stack_destroy(&s);
            return 1;
        }
        printf("pushed %d  ", values[i]);
        stack_dump(&s);
    }

    int top = 0;
    if (stack_peek(&s, &top) == 0) {
        printf("peek: %d (size still %zu)\n", top, s.size);
    }

    int v = 0;
    while (!stack_is_empty(&s)) {
        stack_pop(&s, &v);
        printf("popped %d  ", v);
        stack_dump(&s);
    }

    if (stack_pop(&s, &v) != 0) {
        printf("pop on empty stack -> -1 (underflow handled, no crash)\n");
    }
    if (stack_peek(&s, &v) != 0) {
        printf("peek on empty stack -> -1 (underflow handled, no crash)\n");
    }

    stack_destroy(&s);
    return 0;
}
```

```text
pushed 10  stack (size=1, capacity=4): bottom [ 10 ] top
pushed 20  stack (size=2, capacity=4): bottom [ 10 20 ] top
pushed 30  stack (size=3, capacity=4): bottom [ 10 20 30 ] top
pushed 40  stack (size=4, capacity=4): bottom [ 10 20 30 40 ] top
pushed 50  stack (size=5, capacity=8): bottom [ 10 20 30 40 50 ] top
peek: 50 (size still 5)
popped 50  stack (size=4, capacity=8): bottom [ 10 20 30 40 ] top
popped 40  stack (size=3, capacity=8): bottom [ 10 20 30 ] top
popped 30  stack (size=2, capacity=8): bottom [ 10 20 ] top
popped 20  stack (size=1, capacity=8): bottom [ 10 ] top
popped 10  stack (size=0, capacity=8): bottom [ ] top
pop on empty stack -> -1 (underflow handled, no crash)
peek on empty stack -> -1 (underflow handled, no crash)
```

Watch the `capacity` column: it jumps `0 -> 4 -> 8` exactly at the doublings, while `size` climbs one by one. That is amortized O(1) push in action.

---

## Bugs You Will Write (and How ASan Catches Some)

### Bug 1: Aborting on Underflow

```c
int stack_pop_bad(Stack *s) {
    return s->data[--s->size];   /* BUG: size_t wraps to huge value when empty */
}
```

Popping an empty stack with `size == 0` decrements a `size_t` to `SIZE_MAX` and reads far out of bounds. With ASan:

```text
=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x...
READ of size 4 at 0x... thread T0
    #0 0x... in stack_pop_bad stack_bad.c:3
    #1 0x... in main stack_bad.c:12

SUMMARY: AddressSanitizer: heap-buffer-overflow stack_bad.c:3 in stack_pop_bad
```

The fix is the status return: check `size == 0` first and return `-1`. Never let input drive an unchecked index.

### Bug 2: `realloc` Directly Into the Live Pointer

```c
s->data = realloc(s->data, new_cap * sizeof(*s->data));  /* BUG: leak on failure */
if (s->data == NULL) {
    return -1;
}
```

When `realloc` fails it returns `NULL` **and leaves the old buffer allocated**. Assigning directly overwrites the only pointer to that buffer: the old memory leaks and the stack is left with `data == NULL` but `size > 0`. Always use the `tmp` pattern from `stack_push` above.

### Bug 3: Forgetting `stack_destroy`

```c
int main(void) {
    Stack s;
    stack_init(&s);
    stack_push(&s, 1);
    return 0;   /* BUG: s.data never freed */
}
```

LeakSanitizer (part of ASan) reports it:

```text
=================================================================
==12345==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 16 byte(s) in 1 object(s) allocated from:
    #0 0x... in realloc ...
    #1 0x... in stack_push stack.c:...

SUMMARY: AddressSanitizer: 16 byte(s) leaked in 1 allocation(s).
```

Every successful `stack_init` — more precisely, every first successful `push` — must be paired with exactly one `stack_destroy`.

### Bug 4: `int` Indices and `data[size]` After Pop

Using `int i` for indices invites a negative-index bug; using `size_t` makes the empty-check `s->size == 0` natural. And reading `s->data[s->size]` right after a pop reads the stale slot — remember the top is at `size - 1` *before* decrementing, or at `size` *after* decrementing but only via the saved copy pattern in `stack_pop`.

| Bug | Symptom | Fix |
| --- | --- | --- |
| pop/peek without empty check | heap-buffer-overflow, `size_t` wrap | return `-1` on `size == 0` |
| `ptr = realloc(ptr, ...)` | leak + corrupted stack on OOM | `tmp` + `NULL` check |
| missing `destroy` | LeakSanitizer report | one `destroy` per stack |
| shrinking on every pop | O(n^2) thrash | never shrink, or shrink at `size < cap/4` |
| aborting on bad input | dead server / failed test | status return, caller decides |

---

## Complexity Summary

| Operation | Time | Notes |
| --- | --- | --- |
| `push` | O(1) amortized, O(n) worst case | worst case is the doubling copy |
| `pop` | O(1) | just `size--` |
| `peek` | O(1) | just `data[size-1]` |
| `is_empty` / `size` | O(1) | field read |
| `destroy` | O(1) | one `free` |

Space is O(capacity), which is always less than `2 * size` once the stack has grown (except the tiny initial 4). That factor-of-two slack is the price of amortized O(1) push.

---

## Checklist

- [ ] I can state the LIFO rule and name the top element as `data[size - 1]`.
- [ ] I keep `size` (live elements) and `capacity` (allocated slots) separate, with `size <= capacity`.
- [ ] `push` doubles (`4, 8, 16, ...`) through a `tmp` pointer and checks for `NULL` without losing the old buffer.
- [ ] `pop` and `peek` return `-1` on an empty stack and never abort on input-driven underflow.
- [ ] Readers take `const Stack *`; `malloc`/`realloc` results are always checked.
- [ ] Every stack has exactly one `stack_destroy`, and ASan/LSan output is clean.
- [ ] I can draw the growth diagram and explain why doubling is amortized O(1).

---

#### [Previous](../05-linked-lists/exercises.md) | [Next](./02-part.md)
