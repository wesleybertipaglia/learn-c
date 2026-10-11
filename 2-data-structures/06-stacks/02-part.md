# Linked Stacks and Applications

Part 1 built a stack on a dynamic array. It works, and its `push` is amortized O(1) — but once in a while a single `push` pays for a full `realloc` copy, and the buffer always holds up to twice the memory it needs.

A linked stack removes both quirks: every `push` and `pop` allocates or frees exactly one node, each in true O(1) time, with no spare capacity and no copying. The price is one pointer per element and no indexing. Then, with both implementations in hand, this part puts stacks to work: brackets, postfix, undo, and the call stack itself.

```text
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer
```

---

## A Stack of Nodes

The linked stack reuses the singly linked node, but only ever touches the head:

```c
#include <stddef.h>

typedef struct Node {
    int value;
    struct Node *next;
} Node;

typedef struct {
    Node *top;          /* head of the list == top of the stack */
    size_t size;        /* element count, so size is still O(1) */
} LinkedStack;
```

The mapping is direct:

```text
  push(30) onto  [20, 10]:

  BEFORE                        AFTER
  top                           top
   |                             |
   v                             v
  +----+----+   +----+----+      +----+----+   +----+----+   +----+----+
  | 20 |  o--->| 10 | NULL|      | 30 |  o--->| 20 |  o--->| 10 | NULL|
  +----+----+   +----+----+      +----+----+   +----+----+   +----+----+
   size = 2                       size = 3
```

`push` links a new node in front. `pop` unlinks the front node. Nothing else is ever visited, so position in the middle does not exist as an operation — exactly the LIFO discipline.

### `push` and `pop` in True O(1)

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

typedef struct Node {
    int value;
    struct Node *next;
} Node;

typedef struct {
    Node *top;
    size_t size;
} LinkedStack;

static void lstack_init(LinkedStack *s) {
    s->top = NULL;
    s->size = 0;
}

static bool lstack_is_empty(const LinkedStack *s) {
    return s->size == 0;
}

/* Returns 0 on success, -1 on allocation failure. */
static int lstack_push(LinkedStack *s, int value) {
    Node *n = malloc(sizeof(*n));
    if (n == NULL) {
        return -1;
    }
    n->value = value;
    n->next = s->top;
    s->top = n;
    s->size++;
    return 0;
}

/* Returns 0 on success, -1 on underflow. */
static int lstack_pop(LinkedStack *s, int *out) {
    if (s->top == NULL) {
        return -1;
    }
    Node *n = s->top;
    *out = n->value;
    s->top = n->next;
    free(n);
    s->size--;
    return 0;
}

/* Returns 0 on success, -1 on underflow. */
static int lstack_peek(const LinkedStack *s, int *out) {
    if (s->top == NULL) {
        return -1;
    }
    *out = s->top->value;
    return 0;
}
```

Every operation touches at most two pointers. No `realloc`, no copying, no capacity field.

### Ownership and `destroy`

Each node is owned by the stack from the moment `malloc` succeeds until `pop` or `destroy` calls `free`. The destroy loop pops logically until nothing remains:

```c
static void lstack_destroy(LinkedStack *s) {
    Node *cur = s->top;
    while (cur != NULL) {
        Node *next = cur->next;   /* save before free: cur->next dies with cur */
        free(cur);
        cur = next;
    }
    s->top = NULL;
    s->size = 0;
}
```

Forgetting `lstack_destroy` leaks one allocation *per element*, not one per stack — LeakSanitizer lists every node:

```text
=================================================================
==12345==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 16 byte(s) in 1 object(s) allocated from:
    #0 0x... in malloc ...
    #1 0x... in lstack_push lstack.c:...

Direct leak of 16 byte(s) in 1 object(s) allocated from:
    #0 0x... in malloc ...
    #1 0x... in lstack_push lstack.c:...

SUMMARY: AddressSanitizer: 32 byte(s) leaked in 2 allocation(s).
```

The classic use-after-free here is reading `cur->next` after `free(cur)`. Save `next` first — the pattern above — and the loop is safe.

A small complete run (push 10, 20, 30, then pop to empty):

```c
    LinkedStack s;
    lstack_init(&s);
    lstack_push(&s, 10);
    lstack_push(&s, 20);
    lstack_push(&s, 30);
    int v = 0;
    while (!lstack_is_empty(&s)) {   /* pop prints 30, 20, 10 */
        lstack_pop(&s, &v);
        printf("popped %d (size=%zu)\n", v, s.size);
    }
    if (lstack_pop(&s, &v) != 0) {
        printf("pop on empty linked stack -> -1\n");
    }
    lstack_destroy(&s);
```

```text
popped 30 (size=2)
popped 20 (size=1)
popped 10 (size=0)
pop on empty linked stack -> -1
```

---

## Array Stack Versus Linked Stack

| Aspect | Array stack (`int *data`) | Linked stack (`Node *top`) |
| --- | --- | --- |
| `push` time | O(1) amortized, O(n) on doubling copy | O(1) always (one `malloc`) |
| `pop` time | O(1), no `free` per element | O(1), one `free` per element |
| `peek` | O(1) via `data[size-1]` | O(1) via `top->value` |
| Memory overhead | up to 2x slack (`capacity < 2*size`) | one `next` pointer per element |
| Locality | contiguous, cache-friendly | scattered nodes, cache-hostile |
| Indexing `s.data[i]` | possible (but breaks LIFO discipline) | impossible without walking |
| Allocation count | O(log n) `realloc`s total | n `malloc`s + n `free`s |
| Failure mode | `realloc` fails, stack unchanged | `malloc` fails, stack unchanged |
| Best for | numeric stacks, evaluators, tight loops | large/rarely-moved items, no-copy guarantee, teaching pointers |

Rule of thumb: default to the array stack for `int` and small structs (cache wins). Use the linked stack for big elements or when you need a hard per-operation bound.

Both share the same contract from part 1: `const` on readers, status returns (`0`/`-1`), `NULL` checks on every allocation, and no abort on underflow from input.

---

## Application 1: Balanced Parentheses

"Are the brackets balanced?" is the canonical stack problem. Scan left to right: push every opener, and on every closer pop and check it matches.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <string.h>

#define CAP_START 4

typedef struct {
    char *data;
    size_t size;
    size_t capacity;
} CharStack;

static void cs_init(CharStack *s) {
    s->data = NULL;
    s->size = 0;
    s->capacity = 0;
}

static void cs_destroy(CharStack *s) {
    free(s->data);
    s->data = NULL;
    s->size = 0;
    s->capacity = 0;
}

static int cs_push(CharStack *s, char c) {
    if (s->size == s->capacity) {
        size_t new_cap = (s->capacity == 0) ? CAP_START : s->capacity * 2;
        char *tmp = realloc(s->data, new_cap * sizeof(*s->data));
        if (tmp == NULL) {
            return -1;
        }
        s->data = tmp;
        s->capacity = new_cap;
    }
    s->data[s->size++] = c;
    return 0;
}

static int cs_pop(CharStack *s, char *out) {
    if (s->size == 0) {
        return -1;
    }
    s->size--;
    *out = s->data[s->size];
    return 0;
}

static bool matches(char open, char close) {
    return (open == '(' && close == ')')
        || (open == '[' && close == ']')
        || (open == '{' && close == '}');
}

/* Returns true when every bracket closes in the right order. */
static bool balanced(const char *s) {
    CharStack st;
    cs_init(&st);
    bool ok = true;

    for (size_t i = 0; s[i] != '\0'; i++) {
        char c = s[i];
        if (c == '(' || c == '[' || c == '{') {
            if (cs_push(&st, c) != 0) {
                ok = false;       /* out of memory: report unbalanced */
                break;
            }
        } else if (c == ')' || c == ']' || c == '}') {
            char open = 0;
            if (cs_pop(&st, &open) != 0 || !matches(open, c)) {
                ok = false;       /* closer with no opener, or wrong kind */
                break;
            }
        }
    }
    if (st.size != 0) {
        ok = false;               /* openers left over */
    }
    cs_destroy(&st);
    return ok;
}

int main(void) {
    const char *cases[] = {
        "([]{})", "([)]", "(((", "())", "", "a+(b*[c-{d}])"
    };
    for (size_t i = 0; i < sizeof(cases) / sizeof(cases[0]); i++) {
        printf("%-14s -> %s\n", cases[i], balanced(cases[i]) ? "balanced" : "NOT balanced");
    }
    return 0;
}
```

```text
([]{})         -> balanced
([)]           -> NOT balanced
(((            -> NOT balanced
())            -> NOT balanced
               -> balanced
a+(b*[c-{d}])  -> balanced
```

Why a stack and not a counter? A counter accepts `([)]`, which has equal counts but wrong order — only the stack remembers nesting. Underflow is input, not a bug: `")"` pops an empty stack and returns `false`.

---

## Application 2: Postfix Evaluation

In postfix (`3 4 +`), the operator follows its operands, so no precedence rules are needed. The algorithm is a stack in its purest form: push numbers, and on an operator pop two operands, apply, push the result.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

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
    s->data[s->size++] = value;
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

/* Evaluates "3 4 + 2 *" style input. Returns 0 and sets *result, or -1. */
static int eval_postfix(const char *expr, int *result) {
    Stack s;
    stack_init(&s);
    int status = -1;

    /* strtok needs a mutable copy; the caller's string is const. */
    char *copy = malloc(strlen(expr) + 1);
    if (copy == NULL) {
        stack_destroy(&s);
        return -1;
    }
    strcpy(copy, expr);

    char *tok = strtok(copy, " ");
    while (tok != NULL) {
        char *end = NULL;
        long v = strtol(tok, &end, 10);
        if (end != tok && *end == '\0') {
            if (stack_push(&s, (int)v) != 0) {
                goto done;              /* out of memory */
            }
        } else if (strlen(tok) == 1 && strchr("+-*/", tok[0]) != NULL) {
            int b = 0, a = 0;
            if (stack_pop(&s, &b) != 0 || stack_pop(&s, &a) != 0) {
                goto done;              /* too few operands: bad input */
            }
            int r = 0;
            switch (tok[0]) {
            case '+': r = a + b; break;
            case '-': r = a - b; break;
            case '*': r = a * b; break;
            default: /* '/' */
                if (b == 0) {
                    goto done;          /* divide by zero: bad input */
                }
                r = a / b;
                break;
            }
            if (stack_push(&s, r) != 0) {
                goto done;
            }
        } else {
            goto done;                  /* unknown token: bad input */
        }
        tok = strtok(NULL, " ");
    }

    /* Exactly one value must remain: the answer. */
    if (s.size == 1 && stack_pop(&s, result) == 0 && s.size == 0) {
        status = 0;
    }

done:
    free(copy);
    stack_destroy(&s);
    return status;
}

int main(void) {
    const char *cases[] = {
        "3 4 +", "3 4 + 2 *", "10 2 / 3 -", "5 +", "3 4 + 2"
    };
    for (size_t i = 0; i < sizeof(cases) / sizeof(cases[0]); i++) {
        int r = 0;
        if (eval_postfix(cases[i], &r) == 0) {
            printf("%-12s -> %d\n", cases[i], r);
        } else {
            printf("%-12s -> error\n", cases[i]);
        }
    }
    return 0;
}
```

```text
3 4 +        -> 7
3 4 + 2 *    -> 14
10 2 / 3 -   -> 2
5 +          -> error
3 4 + 2      -> error
```

Trace `3 4 + 2 *`: `3` push, `4` push, `+` pops `4,3` pushes `7`, `2` push, `*` pops `2,7` pushes `14` — one value left, the answer.

The two error cases show the contract: `"5 +"` underflows the second pop, and `"3 4 + 2"` leaves two values. Both return `-1`; neither aborts. Division by zero is checked the same way — input validation, not container logic.

---

## Application 3: Undo (and the Call Stack)

An undo buffer is a stack of *actions*. Each user operation pushes its inverse; Undo pops the inverse and applies it:

```text
  type "hi"    push <delete 2>     stack: [ delete-2 ]
  type "!"     push <delete 1>     stack: [ delete-2, delete-1 ]
  undo         pop <delete 1>      apply: remove "!" 
  undo         pop <delete 2>      apply: remove "hi"
  undo         stack empty -> menu option disabled, never a crash
```

Real editors store diffs or command objects instead of full snapshots, but the discipline is identical: LIFO, bounded optionally by dropping the *bottom* (the only stack that ever touches the bottom), and underflow mapped to "nothing to undo".

Your program already uses a stack you never see: the **call stack**. Each call pushes a frame (return address, locals, saved registers); each `return` pops it:

```text
  main() calls eval_postfix() calls stack_push():

  high address
  +---------------+   <- stack_push frame (top)
  | push locals   |
  +---------------+   <- eval_postfix frame
  | expr, s, copy |
  +---------------+   <- main frame (bottom)
  | cases, s      |
  +---------------+
  low address
```

That is why infinite recursion dies with `stack overflow`: each call pushes a frame and never pops, until the fixed-size call stack (typically 8 MB) runs out. The data-structure stack is the same idea with the heap as backing store. A debugger's `backtrace` is literally a peek down the call stack.

---

## Bugs and ASan Notes

| Bug | What happens | Fix |
| --- | --- | --- |
| `free(cur); cur = cur->next;` | use-after-free, heap-use-after-free under ASan | save `next` before `free` |
| forgetting `destroy` on linked stack | one leak per node | destroy loop above |
| `strtok` on a string literal | segfault: literals are read-only | `malloc` + `strcpy` copy first |
| popping operands in wrong order (`a` then `b`) | `10 2 -` gives `-8` | pop `b` first, then `a`; `a - b` |
| `assert(size > 0)` inside pop | debug builds abort on bad input | return `-1`, let caller print `error` |
| mixing the two stacks' helpers | type error or silent truncation (`char` vs `int`) | one stack type per value type, or generics (module 3) |

The use-after-free above reports as `heap-use-after-free` in `lstack_destroy` under ASan — read the "freed by" frame to find the premature `free`.

---

## Checklist

- [ ] I can implement linked `push`/`pop`/`peek` in O(1) with status returns and explain why no `realloc` is needed.
- [ ] My `destroy` saves `next` before `free` and leaves `top == NULL, size == 0`.
- [ ] I can state the array-vs-linked trade-off (cache + slack vs pointer-per-node + hard O(1)).
- [ ] I can trace balanced-brackets on `([)]` and explain why a counter is insufficient.
- [ ] I can trace `3 4 + 2 *` and list the three postfix error cases (underflow, leftover values, divide by zero).
- [ ] I can describe undo as a stack of inverse actions and the call stack as push-on-call / pop-on-return.
- [ ] Readers take `const *`, allocations are `NULL`-checked, underflow returns `-1`, ASan/LSan are clean.

---

#### [Previous](./01-part.md) | [Next](./exercises.md)
