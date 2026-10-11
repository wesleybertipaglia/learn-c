# Exercise List 5 - Linked Lists

Six exercises on chained nodes, plus one instruction that applies to all of them. Every program must compile with `gcc -std=c11 -Wall -Wextra -pedantic -g file.c -o file` with **zero** warnings, then also compile and run clean under:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer file.c -o file_san
```

Build the diagrams on paper before the code in the editor. Every list bug is a wrong arrow; the sanitizer only tells you which line discovered it.

---

## Exercise 1: Build, Print, Free

Allocate three nodes by hand, link them `10 -> 20 -> 30 -> NULL`, print the list, print its length, free everything.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

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

/* OWNERSHIP: takes ownership of `head`; frees every node.
   After this call the list no longer exists. */
static void destroy_list(Node *head) {
    while (head != NULL) {
        Node *next = head->next;   /* save BEFORE free */
        free(head);
        head = next;
    }
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

    /* TODO: link them so head -> second -> third -> NULL. */

    /* TODO: call print_list and list_length, then destroy_list
       and set all three pointers to NULL. */

    return 0;
}
```

Expected output (after the TODOs are done):

```text
10 -> 20 -> 30 -> NULL
length = 3
```

Requirements:

- Fill in the two TODOs. The linking is exactly two assignments; write them, then draw the three-box diagram of the result before running anything.
- Use `%zu` for the `size_t` length. Explain in a comment what goes wrong with `%d` when the count exceeds `INT_MAX` (and why the compiler warns about it under `-Wall`).
- After `destroy_list`, set `head = NULL;` (and `second = NULL; third = NULL;` if you keep them). Explain in a comment why `destroy_list` alone cannot clear the caller's pointers.
- Delete the `free` calls in the failure path one at a time and record the LeakSanitizer report for each. `16 byte(s)` per missing `free` is one `Node`.
- Extra credit: make `print_list(NULL)` and `list_length(NULL)` meaningful. What should each print/return, and why is that the only sane answer?

---

## Exercise 2: `push_front` and `pop_front`

Build a stack-like interface on the front of the list: push values, pop them back, prove LIFO order.

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node Node;

struct Node {
    int value;
    Node *next;
};

/* OWNERSHIP: takes ownership of `head` (may be NULL).
   OWNERSHIP: on success returns the NEW head (caller owns it).
   On failure returns NULL; the old list is UNCHANGED and the
   caller still owns it. */
static Node *push_front(Node *head, int value) {
    /* TODO: malloc one Node, check NULL, set value/next, return it. */
    (void)head;
    (void)value;
    return NULL;
}

/* OWNERSHIP: MUTATES *head. On success stores the removed value in
   *out (if out != NULL), frees the old head, returns 0.
   On empty list returns -1 and touches nothing. */
static int pop_front(Node **head, int *out) {
    /* TODO: handle head == NULL and *head == NULL, save victim,
       advance *head, free victim. */
    (void)head;
    (void)out;
    return -1;
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
    int v = 0;

    /* TODO: push 10, then 20, then 30 (check each return for NULL).
       TODO: pop until empty, printing each value.
       TODO: confirm pop on empty returns -1. */

    destroy_list(head);
    head = NULL;
    (void)v;

    return 0;
}
```

Expected output:

```text
pushed 10, pushed 20, pushed 30
popped 30
popped 20
popped 10
pop on empty: -1
```

Requirements:

- Implement both TODO functions. `push_front` must leave the old list intact on `malloc` failure; `pop_front` must leave an empty list untouched.
- In `main`, check every `push_front` return against `NULL`. On failure, print to `stderr`, destroy whatever was built so far, and return 1. Explain in a comment why the partial list still needs destroying.
- Call `pop_front(&head, NULL)` once (a `NULL` out-parameter). Explain in a comment why that call must still free the node and advance the head.
- Deliberately write `push_front(head, 99);` without storing the return (one line, then delete it). Run under the sanitizer and paste the leak report. Explain in a comment where the leaked node was born vs where it was lost.
- Extra credit: write `static size_t list_length(const Node *head)` and assert the length is 3 after the pushes and 0 after the pops.

---

## Exercise 3: `insert_at` and `remove_at` with `Node **`

Insert and remove by index. Index `0` is the front; anything past the end fails without touching the list.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

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

/* OWNERSHIP: takes ownership of `head`. Inserts `value` AT `index`.
   OWNERSHIP: on success returns the new head (caller owns it).
   On failure returns NULL; the old list is UNCHANGED. */
static Node *insert_at(Node *head, size_t index, int value) {
    /* TODO: index == 0 is push_front logic. Otherwise walk to
       index - 1, reject NULL early, splice node->next BEFORE
       prev->next. */
    (void)index;
    (void)value;
    return head;
}

/* OWNERSHIP: MUTATES *head. Removes the node AT `index`, frees it,
   stores its value in *out if out != NULL. Returns 0 / -1. */
static int remove_at(Node **head, size_t index, int *out) {
    /* TODO: delegate index 0 to the pop_front pattern
       (victim = *head; *head = victim->next; free).
       Otherwise walk to index - 1 and unlink. */
    (void)head;
    (void)index;
    (void)out;
    return -1;
}

/* OWNERSHIP: borrows `head`. */
static void print_list(const Node *head) {
    for (const Node *p = head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}

/* OWNERSHIP: takes ownership of `head`. */
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

    /* TODO: insert 25 at index 2, insert 5 at index 0,
       remove index 1, remove index 3, try remove index 9
       (must fail with -1 and change nothing). Print after each. */

    destroy_list(head);
    head = NULL;

    return 0;
}
```

Expected output:

```text
start:          10 -> 20 -> 30 -> NULL
insert 25 at 2: 10 -> 20 -> 25 -> 30 -> NULL
insert 5 at 0:  5 -> 10 -> 20 -> 25 -> 30 -> NULL
remove at 1:    5 -> 20 -> 25 -> 30 -> NULL
remove at 3:    5 -> 20 -> 25 -> NULL
remove at 9:    -1 (unchanged) 5 -> 20 -> 25 -> NULL
```

Requirements:

- Explain in a comment above `remove_at` why the parameter is `Node **head` and not `Node *head`. Name the one index that forces it.
- The splice order (`node->next = prev->next` before `prev->next = node`) is load-bearing. Swap the two lines, run the program, and paste whatever the sanitizer reports (expect a self-loop or a lost tail). Then swap them back.
- `remove_at` with `index == 0` must reuse the pop-front logic, not duplicate it. Say in a comment what breaks when the logic lives in two places.
- Every failure path (bad index, `malloc` failure) must leave the list unchanged. Write a test that captures the printed list before and after a failing call and compares them.
- Extra credit: implement `delete_value(Node **head, int target)` with the `Node **link` idiom from part 02 and use it to delete `20` by value.

---

## Exercise 4: Reverse a List

Reverse a singly linked list in place: no new nodes, no array, three pointers and one loop.

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

/* OWNERSHIP: takes ownership of `head`; returns the NEW head of the
   reversed list (caller owns it). Reuses the existing nodes:
   no malloc, no free. */
static Node *reverse_list(Node *head) {
    /* TODO: walk with prev / cur / next. Save next BEFORE rewiring,
       point cur->next at prev, advance both. Return prev. */
    (void)head;
    return NULL;   /* replace */
}

/* OWNERSHIP: borrows `head`. */
static void print_list(const Node *head) {
    for (const Node *p = head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}

/* OWNERSHIP: takes ownership of `head`. */
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

    printf("before: ");
    print_list(head);

    /* TODO: head = reverse_list(head); print again; reverse back
       and confirm the original order returns. */

    destroy_list(head);
    head = NULL;

    return 0;
}
```

Expected output:

```text
before: 10 -> 20 -> 30 -> NULL
after:  30 -> 20 -> 10 -> NULL
back:   10 -> 20 -> 30 -> NULL
```

Requirements:

- Draw the three-pointer diagram (`prev`, `cur`, `next`) for the first two iterations before coding. The function is five lines; the diagram is the actual work.
- Handle `reverse_list(NULL)` (returns `NULL`) and a one-node list (returns it unchanged). Test both in `main` and explain in comments why no special-case branch is needed if the loop is written correctly.
- Count the nodes before and after with a `size_t` helper. A reversal that loses or duplicates a node changes the count; assert it does not.
- Write the buggy version that forgets `next = cur->next` before rewiring (one line removed). Run it, paste the sanitizer or crash output, and explain in a comment which node's address was lost and why the rest of the list is unreachable afterward.
- Extra credit: write an iterative `reverse_print(const Node *head)` that prints in reverse **without** reversing (recursion or a counted two-pass is fine) and say in comments why it cannot run in O(1) extra memory without mutating the list.

---

## Exercise 5: Doubly Linked List Append and Remove

Maintain `head`, `tail`, and `size` across appends at both ends and removals from the middle.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

typedef struct DNode DNode;

struct DNode {
    int value;
    DNode *prev;
    DNode *next;
};

typedef struct {
    DNode *head;
    DNode *tail;
    size_t size;
} DList;

static void dlist_init(DList *list) {
    list->head = NULL;
    list->tail = NULL;
    list->size = 0;
}

/* OWNERSHIP: returns a NEW heap node; the CALLER owns it.
   Returns NULL on failure. */
static DNode *dnode_new(int value) {
    DNode *node = malloc(sizeof(DNode));
    if (node == NULL) {
        return NULL;
    }
    node->value = value;
    node->prev = NULL;
    node->next = NULL;
    return node;
}

/* OWNERSHIP: MUTATES *list. Returns 0 / -1 (list unchanged). */
static int dlist_push_back(DList *list, int value) {
    /* TODO: node->prev = tail; link tail->next or set head+tail
       when empty; size++. */
    (void)list;
    (void)value;
    return -1;
}

/* OWNERSHIP: MUTATES *list. Returns 0 / -1 (list unchanged). */
static int dlist_push_front(DList *list, int value) {
    /* TODO: mirror image of push_back. */
    (void)list;
    (void)value;
    return -1;
}

/* OWNERSHIP: MUTATES *list; frees `node` (must belong to *list). */
static void dlist_remove(DList *list, DNode *node) {
    /* TODO: stitch prev->next and next->prev, or move head/tail
       at the ends. free(node); size--. */
    (void)list;
    (void)node;
}

/* OWNERSHIP: borrows `list`. */
static void dlist_print_forward(const DList *list) {
    const DNode *p;
    if (list == NULL) {
        return;
    }
    for (p = list->head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}

/* OWNERSHIP: borrows `list`. */
static void dlist_print_backward(const DList *list) {
    const DNode *p;
    if (list == NULL) {
        return;
    }
    for (p = list->tail; p != NULL; p = p->prev) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}

/* OWNERSHIP: takes ownership of *list's nodes; leaves it empty. */
static void dlist_destroy(DList *list) {
    DNode *cur;
    if (list == NULL) {
        return;
    }
    cur = list->head;
    while (cur != NULL) {
        DNode *next = cur->next;
        free(cur);
        cur = next;
    }
    list->head = NULL;
    list->tail = NULL;
    list->size = 0;
}

int main(void) {
    DList list;

    dlist_init(&list);

    /* TODO: push_back 10, 20, 30; push_front 5. Print forward AND
       backward after each step; they must be exact reverses.
       TODO: remove the node holding 20 via dlist_remove.
       TODO: destroy and confirm head/tail NULL, size 0. */

    dlist_destroy(&list);

    return 0;
}
```

Expected output:

```text
forward:  5 -> 10 -> 20 -> 30 -> NULL
backward: 30 -> 20 -> 10 -> 5 -> NULL
size = 4
after remove 20:
forward:  5 -> 10 -> 30 -> NULL
backward: 30 -> 10 -> 5 -> NULL
size = 3
after destroy: head=NULL tail=NULL size=0
```

Requirements:

- After **every** mutation, print forward and backward and check they are reverses. Write a `static int dlist_check(const DList *list)` helper that walks both directions, verifies `n->next->prev == n` everywhere, counts nodes, and compares the count to `size`. Call it after each step.
- The empty-list branch in each push function is mandatory. Delete it once, run `push_back` on an empty list, and paste the sanitizer's `SEGV` report (a `NULL->next` write). Then restore it.
- In `dlist_remove`, update **both** sides for middle nodes and move `head`/`tail` at the ends. Write a comment per branch naming which of the four cases (middle / first / last / only) it handles.
- Verify `size` is exact after each operation with `printf("size = %zu\n", list.size)`. Explain in a comment why `size` cannot be recomputed from `head` in O(1).
- Extra credit: find the node holding `20` with a `dlist_find`, remove it, then remove the head and the tail. Print forward/backward after each of the three removals.

---

## Exercise 6: Cycle Detection (Floyd) — plus Merge Bonus

A list that ends in `NULL` terminates. A list with a loop never does — `print_list` on one hangs forever. Floyd's tortoise-and-hare finds the loop without extra memory.

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

/* OWNERSHIP: borrows `head`. Returns 1 if some node's `next` chain
   loops back, 0 otherwise. Never modifies the list. */
static int has_cycle(const Node *head) {
    /* TODO: slow = head, fast = head. While fast and fast->next
       are non-NULL: slow = slow->next; fast = fast->next->next;
       if (slow == fast) return 1. Return 0 at the end. */
    (void)head;
    return 0;   /* replace */
}

/* OWNERSHIP: borrows `head`. Bounded print: at most `max` nodes. */
static void print_bounded(const Node *head, size_t max) {
    const Node *p = head;
    size_t i = 0;
    while (p != NULL && i < max) {
        printf("%d -> ", p->value);
        p = p->next;
        i++;
    }
    if (p != NULL) {
        printf("... (stopped after %zu)\n", max);
    } else {
        printf("NULL\n");
    }
}

/* OWNERSHIP: takes ownership of `head` (must be ACYCLIC). */
static void destroy_list(Node *head) {
    while (head != NULL) {
        Node *next = head->next;
        free(head);
        head = next;
    }
}

int main(void) {
    Node *head = NULL;
    Node *tail = NULL;

    head = push_front(head, 30);
    head = push_front(head, 20);
    head = push_front(head, 10);
    if (head == NULL) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    printf("acyclic: %d\n", has_cycle(head));

    /* TODO: find the tail (node with next == NULL), point
       tail->next at head (creates a cycle), confirm has_cycle
       returns 1 with print_bounded (NOT print_list: it would hang),
       then BREAK the cycle (tail->next = NULL) BEFORE destroying. */

    destroy_list(head);
    head = NULL;
    (void)tail;

    return 0;
}
```

Expected output:

```text
acyclic: 0
cyclic: 1
10 -> 20 -> 30 -> 10 -> 20 -> 30 -> ... (stopped after 6)
after break: 0
```

Requirements:

- Never call an unbounded traversal on a cyclic list — not `print_list`, not `list_length`, not `destroy_list`. Explain in a comment above each call site whether the list is known-acyclic at that point and why the call is safe.
- Breaking the cycle before `destroy_list` is not optional. Explain in a comment what `destroy_list` would do on a cycle (loop forever, or free the same node twice — pick one and justify it by tracing three iterations).
- Test `has_cycle(NULL)` (expect 0) and a one-node self-loop (`node->next = node`, expect 1). The self-loop is the case that breaks implementations starting `fast = head->next` without a `NULL` check — confirm yours survives it.
- State the complexity in a comment: time O(n), extra space O(1). Say in one sentence why a "remember every visited pointer in an array" version is O(n) space and when that trade would be acceptable.
- Extra credit: implement `Node *merge_sorted(Node *a, Node *b)` that reuses the existing nodes (no `malloc`) by rewiring `next` pointers, returning the new head. Ownership: takes ownership of both inputs, returns one combined list the caller owns. Test on `1 -> 4 -> 7` and `2 -> 3 -> 8`, expect `1 -> 2 -> 3 -> 4 -> 7 -> 8 -> NULL`, destroy once.

---

## Sanitize Everything

Do this for **every** exercise above, and for every exercise from now on:

1. Compile with warnings as errors and with both sanitizers:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -Werror -g \
       -fsanitize=address,undefined -fno-omit-frame-pointer main.c -o main_san
   ```

2. Run it several times, including the empty list and the one-node list. Boundary bugs hide at length 0 and 1.
3. Fix everything the sanitizer reports. It is not a suggestion.
4. Re-run until the output contains no sanitizer diagnostics at all. Silence is the passing state.
5. Then compile the plain version and confirm identical behavior:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
   ```

Report, for each program, in a comment at the top of the file:

```text
[ ] compiles with -Werror, zero warnings
[ ] no sanitizer diagnostics
[ ] every malloc has a matching free
[ ] every free argument came from malloc (no stack frees, no double frees)
[ ] no pointer used after free; destroy leaves head NULL
[ ] every index/pointer checked before use (no walk past NULL)
[ ] ownership documented on every pointer-returning function
```

That checklist is the habit the whole module is trying to build. A list program that passes it may still have the wrong values in the wrong order — but it will not corrupt the heap finding out.

---

#### [Previous](./03-part.md) | [Next](../06-stacks/01-part.md)
