# Singly Linked Lists

An array keeps every element side by side in one block. That is fast, but it forces one decision up front: how many elements will there ever be. A singly linked list makes the opposite trade. Each element lives in its own block, and each block points at the next one. Nothing is contiguous, nothing has a fixed size, and the list can grow one `malloc` at a time.

---

## The Node

Everything in this chapter is built from one struct:

```c
struct Node {
    int value;
    struct Node *next;
};

typedef struct Node Node;
```

Two things about this definition deserve attention.

First, the struct refers to itself. Inside `struct Node`, the declaration `struct Node *next` is legal because a pointer to an incomplete type is allowed. The compiler does not need to know how big `struct Node` is in order to store its address. It only needs the size when you write `sizeof(struct Node)` or access `->value`, and by then the definition is complete.

Second, there are two kinds of data in every node. `value` is the payload — the thing you care about. `next` is the plumbing — the address of the following node, or `NULL` when there is none. Keeping those two roles separate in your head prevents most linked-list bugs: payload code touches `value`, plumbing code touches `next`, and the two rarely mix.

| Field | Type | Role |
| --- | --- | --- |
| `value` | `int` | the data stored in this position |
| `next` | `struct Node *` | address of the next node, or `NULL` at the end |

A list is nothing more than a pointer to the first node. That pointer is called the **head**. An empty list is a `NULL` head — not a special object, not a zero-length block, just `NULL`.

```c
Node *head = NULL;   /* an empty list */
```

---

## What a List Looks Like

Three nodes holding `10`, `20`, `30`:

```text
  head
   |
   v
  +-------+------+    +-------+------+    +-------+------+
  | value | next |--->| value | next |--->| value | next |---> NULL
  |  10   | 0x.. |    |  20   | 0x.. |    |  30   | NULL |
  +-------+------+    +-------+------+    +-------+------+

  each box is ONE malloc'd block, somewhere on the heap
  the arrows are the `next` pointers, tying them in order
  the last arrow points at NULL, which is how you know it ends
```

Compare that picture to an array:

An array is one contiguous block found by index arithmetic; a list is one block per node (`[10|*]--->[20|*]--->[30|NULL]`), scattered across the heap and found only by following `next`.

| | **Array** | **Singly linked list** |
| --- | --- | --- |
| Blocks | one contiguous block | one block per node |
| Finding element `i` | `a[i]`, O(1) | walk `i` links, O(n) |
| Growing by one | may need `realloc` + copy | one `malloc`, O(1) at front |
| Memory overhead | none per element | one pointer per node |
| Empty case | length 0, block may exist | head is `NULL` |

The list wins when the size is unknown and insertions happen at the front. It loses whenever you need random access.

---

## One Node, on the Heap

Every node is born the same way: one `malloc`, one `NULL` check, two initializations.

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node Node;

struct Node {
    int value;
    Node *next;
};

/* OWNERSHIP: returns a NEW heap node; the CALLER must free() it
   (directly or via destroy_list). Returns NULL on failure. */
static Node *node_new(int value) {
    Node *node = malloc(sizeof(Node));

    if (node == NULL) {
        return NULL;
    }

    node->value = value;
    node->next = NULL;
    return node;
}
```

Three rules are packed into those ten lines, and all three are load-bearing:

1. **`malloc(sizeof(Node))`, not `malloc(sizeof(Node *))`.** You want room for the struct, not for a pointer to it. The short size compiles, runs, and corrupts the heap on every assignment.
2. **Check against `NULL`.** `malloc` returns `NULL` when it fails. Dereferencing it is undefined behavior.
3. **Initialize both fields.** `malloc` leaves garbage. A node whose `next` is garbage is a traversal that walks into unmapped memory. Setting `next = NULL` makes the new node a valid one-element list by itself.

The compile command for the rest of this chapter is the same one the previous module established:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer list.c -o list
```

---

## NULL-Termination

Every singly linked list ends with `NULL`. That is the only length information the list carries, and every traversal depends on it.

```text
  head
   |
   v
  [10|*]--->[20|*]--->[30|NULL]

  the NULL in the last node is the terminator
  without it, the loop below has no stopping condition
```

The traversal idiom, which you will write dozens of times before this chapter is over:

```c
for (const Node *p = head; p != NULL; p = p->next) {
    /* visit p->value */
}
```

Read it out loud: start at the head; while the pointer is not `NULL`; follow it to the next node. The `const` says this loop only reads — it never modifies `value` and never rewires `next`. Any function that only reads the list should take `const Node *head`.

What happens when `next` is garbage instead of `NULL`? The loop keeps walking past the end:

```c
/* BUG: node_new forgot `node->next = NULL` */
Node *a = node_new(10);
Node *b = node_new(20);
a->next = b;
/* b->next is garbage: 0x3f2a... instead of NULL */
```

```text
=================================================================
==391102==ERROR: AddressSanitizer: SEGV on unknown address 0x3f2a19c0b558 (pc 0x5612aa0431d8 bp 0x7ffc2e1aa4a0 sp 0x7ffc2e1aa490 T0)
==391102==The signal is caused by a READ memory access.
==391102==Hint: address points to the zero page.
    #0 0x5612aa0431d8 in print_list /tmp/list.c:31
    #1 0x5612aa0432aa in main /tmp/list.c:52

SUMMARY: AddressSanitizer: SEGV /tmp/list.c:31 in print_list
```

The report points at the loop body — `p->value` on a garbage `p`. The fix is never in the loop. The fix is at construction: every node starts with `next = NULL`, and every function that unlinks a node resets the pointers it touched.

---

## Building a Chain by Hand

Before helpers, build three nodes manually so the plumbing is visible:

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node Node;

struct Node {
    int value;
    Node *next;
};

static Node *node_new(int value) {
    Node *node = malloc(sizeof(Node));
    if (node == NULL) {
        return NULL;
    }
    node->value = value;
    node->next = NULL;
    return node;
}

int main(void) {
    Node *head;
    Node *second;
    Node *third;

    head = node_new(10);
    second = node_new(20);
    third = node_new(30);
    if (head == NULL || second == NULL || third == NULL) {
        fprintf(stderr, "out of memory\n");
        free(head);
        free(second);
        free(third);
        return 1;
    }

    /* link them: each `next` points at the following node */
    head->next = second;
    second->next = third;
    /* third->next is already NULL: the terminator */

    for (const Node *p = head; p != NULL; p = p->next) {
        printf("%d ", p->value);
    }
    printf("\n");

    /* manual teardown, in link order; destroy_list does this generally */
    free(head);
    head = NULL;
    free(second);
    second = NULL;
    free(third);
    third = NULL;

    return 0;
}
```

```text
10 20 30
```

The linking step is two assignments and nothing else. Drawing the state after each one is the fastest way to debug:

```text
  before linking: head--->[10|NULL]   second--->[20|NULL]   third--->[30|NULL]
  after both links: head--->[10|*]--->[20|*]--->[30|NULL]
```

If you link in the wrong order — `second->next = third` before checking for `NULL`, or `head->next = third` by accident — the diagram shows it immediately. Code review for linked lists is diagram review.

---

## Reading: Print and Length

Two read-only helpers. Both take `const Node *`, both use `size_t` for counts, neither frees anything:

```c
/* OWNERSHIP: borrows `head`; does not free it and does not keep it. */
static void print_list(const Node *head) {
    for (const Node *p = head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}

/* OWNERSHIP: borrows `head`; returns how many nodes it chains. */
static size_t list_length(const Node *head) {
    size_t count = 0;

    for (const Node *p = head; p != NULL; p = p->next) {
        count++;
    }
    return count;
}
```

```text
10 -> 20 -> 30 -> NULL
```

`print_list(NULL)` prints `NULL` and nothing else. That is correct: an empty list prints as its own terminator. `list_length(NULL)` returns `0`. Every list function must answer the empty-list question, and the answer is almost always "do nothing, return the identity value."

Note the `const` in both signatures. It is a promise to the caller: calling `print_list` cannot modify the list. Drop the `const` and the promise disappears, even if the body happens not to write. Keep it and the compiler enforces the read-only intent.

---

## Writing: `push_front`

The cheapest mutation a singly linked list supports is inserting at the front. No traversal, no search: point the new node at the old head, return the new node as the head.

```text
  before push_front(head, 5), head = [10|*]-->[20|*]-->[30|NULL]

  1. allocate [5|NULL]

  2. new_node->next = head:

     [5|*]--->[10|*]-->[20|*]-->[30|NULL]

  3. head = new_node (return it; the caller stores it):

     head--->[5|*]--->[10|*]-->[20|*]-->[30|NULL]
```

```c
/* OWNERSHIP: takes ownership of `head` (may be NULL for empty list).
   OWNERSHIP: on success returns the NEW head, which the caller owns.
   On failure returns NULL and the old list is UNCHANGED (still owned
   by the caller and still must be freed). */
static Node *push_front(Node *head, int value) {
    Node *node = malloc(sizeof(Node));

    if (node == NULL) {
        return NULL;
    }

    node->value = value;
    node->next = head;
    return node;
}
```

The caller stores the return value. Forgetting to store it is the classic leak:

```c
Node *head = NULL;

head = push_front(head, 30);
head = push_front(head, 20);
head = push_front(head, 10);
if (head == NULL) {
    fprintf(stderr, "out of memory\n");
    return 1;
}
print_list(head);   /* 10 -> 20 -> 30 -> NULL */
```

Why must the caller write `head = push_front(head, ...)`? Because C passes the pointer **by value**. `push_front` receives a copy of `head`; assigning to its parameter cannot change the caller's variable. The new head travels back through the return value. (The next part shows the `Node **` alternative, which lets the function update the caller's pointer directly.)

On failure, the contract matters: the old list is untouched. The caller still owns it and must still free it. A `push_front` that freed the list on failure would force every caller to reason about two ownership states. Returning `NULL` with the list intact keeps exactly one rule: the caller always owns whatever it owned before the call, plus the new head on success.

---

## Ownership: Who Frees What

The rule for the whole chapter:

> **The caller owns the list. The list functions borrow it, except the ones whose documented job is to create or destroy nodes.**

Concretely:

| Function | Takes | Returns | Who frees |
| --- | --- | --- | --- |
| `node_new` | a value | a new node | the caller |
| `push_front` | the old head | the new head | the caller (the new head) |
| `print_list`, `list_length` | `const Node *` | nothing / a count | nobody — they borrow |
| `destroy_list` | the head | nothing | `destroy_list` frees every node |

`destroy_list` walks the list and frees each node exactly once. It must save the next pointer **before** freeing, because after `free(p)` the expression `p->next` is a use-after-free:

```c
/* OWNERSHIP: takes ownership of `head`; frees every node.
   After this call the list no longer exists. */
static void destroy_list(Node *head) {
    while (head != NULL) {
        Node *next = head->next;   /* save BEFORE free */
        free(head);
        head = next;
    }
}
```

The wrong order, and what the sanitizer says about it:

```c
/* BUG: reads head->next AFTER head is dead */
static void destroy_broken(Node *head) {
    while (head != NULL) {
        free(head);
        head = head->next;   /* use-after-free: head is dangling */
    }
}
```

```text
=================================================================
==391220==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000018 at pc 0x5612aa0432c1 bp 0x7ffc2e1aa4a0 sp 0x7ffc2e1aa490
READ of size 8 at 0x502000000018 thread T0
    #0 0x5612aa0432c1 in destroy_broken /tmp/list.c:18
    #1 0x5612aa04334a in main /tmp/list.c:40
    #2 0x7a2b1d62a600 in __libc_start_call_main ../sysdeps/nptl/libc-start_call_main.h:59

0x502000000018 is located 8 bytes inside of 16-byte region [0x502000000010,0x502000000020)
freed by thread T0 here:
    #0 0x7a2b1d82b60f in free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:52
    #1 0x5612aa0432a9 in destroy_broken /tmp/list.c:17

SUMMARY: AddressSanitizer: heap-use-after-free /tmp/list.c:18 in destroy_broken
```

`READ of size 8` is the `next` pointer. `freed by ... destroy_broken /tmp/list.c:17` is the `free` one line above: the next address was read from dead memory. Save it first.

After destroying, clear the caller's pointer, because `destroy_list` — like `free` — receives only a copy:

```c
destroy_list(head);
head = NULL;
```

`head` is now a `NULL` empty list again, not a dangling pointer. A second `destroy_list(head)` is then safe: `destroy_list(NULL)` does nothing, and `free(NULL)` does nothing. That mirrors the `free` + `= NULL` discipline from the previous module.

---

## The Complete Program

One `malloc` per node, one traversal idiom, one destroy function:

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node Node;

struct Node {
    int value;
    Node *next;
};

static Node *push_front(Node *head, int value) {
    Node *node = malloc(sizeof(Node));
    if (node == NULL) {
        return NULL;
    }
    node->value = value;
    node->next = head;
    return node;
}

static void print_list(const Node *head) {
    for (const Node *p = head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}

static size_t list_length(const Node *head) {
    size_t count = 0;
    for (const Node *p = head; p != NULL; p = p->next) {
        count++;
    }
    return count;
}

static void destroy_list(Node *head) {
    while (head != NULL) {
        Node *next = head->next;
        free(head);
        head = next;
    }
}

int main(void) {
    Node *head = NULL;

    head = push_front(head, 30);
    head = push_front(head, 20);
    head = push_front(head, 10);
    if (head == NULL) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    print_list(head);
    printf("length = %zu\n", list_length(head));

    destroy_list(head);
    head = NULL;

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer list.c -o list
./list
```

```text
10 -> 20 -> 30 -> NULL
length = 3
```

Silent under the sanitizer. Silence is the passing state: no leak report, no use-after-free, every `malloc` matched by exactly one `free`.

---

## Checklist

```text
[ ] struct Node { int value; struct Node *next; } with a typedef
[ ] every malloc is sizeof(Node), checked against NULL
[ ] every new node gets next = NULL at construction
[ ] empty list is NULL; every function handles NULL head
[ ] traversal is for (p = head; p != NULL; p = p->next)
[ ] read-only functions take const Node *head
[ ] counts and lengths are size_t, printed with %zu
[ ] push_front returns the new head; caller stores it
[ ] destroy_list saves next BEFORE free, frees each node once
[ ] destroy_list(head); head = NULL; at every teardown
[ ] builds with -fsanitize=address,undefined, runs with no diagnostics
```

---

#### [Previous](../04-dynamic-memory/exercises.md) | [Next](./02-part.md)
