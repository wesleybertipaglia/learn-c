# Insertion and Deletion

`push_front` is the easy case: the head is right there, no search, no special cases. Everything else — inserting in the middle, removing the fifth element, deleting by value — has to solve two harder problems. First, find the right position without walking off the end. Second, rewire the pointers so no node is lost and no pointer dangles. This part builds those operations and names the three bugs they produce.

---

## The Shape of Every Mutation

Insert and delete are mirror images. Insert splices a new node between two existing ones. Delete cuts one out. Both touch exactly two pointers.

```text
  insert 25 between [20] and [30]:

  before:   ...-->[20|*]--->[30|*]--->...
  after:    ...-->[20|*]--->[25|*]--->[30|*]--->...
                    ^^^^^^^^  ^^^^^^^^
                    rewired   new node points at successor

  delete [20] from head-->[10|*]-->[20|*]-->[30|NULL]:

  before:   head-->[10|*]-->[20|*]-->[30|NULL]
  after:    head-->[10|*]------------>[30|NULL]   [20|?] freed
                    ^^^^^^^^
                    predecessor skips over the victim
```

The predecessor's `next` is the only pointer that changes in a deletion. The victim's `next` is irrelevant the moment the predecessor skips it — except that you must have saved whatever you still need before calling `free`.

| Operation | Pointers touched | Cost |
| --- | --- | --- |
| `push_front` | new node's `next`, head | O(1), no traversal |
| `insert_at` index `i` | new node's `next`, predecessor's `next` | O(n), walk to `i - 1` |
| `pop_front` | head | O(1), no traversal |
| `remove_at` index `i` | predecessor's `next` (or head) | O(n), walk to `i - 1` |
| `delete_value` | predecessor's `next` (or head) | O(n), walk to match |

Every O(n) in that table is the same loop: walk `i` links from the head, stopping if you hit `NULL` early.

---

## `pop_front`: the Simplest Removal

Remove the head, hand its value back, free the node:

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

static void destroy_list(Node *head) {
    while (head != NULL) {
        Node *next = head->next;
        free(head);
        head = next;
    }
}

/* OWNERSHIP: borrows nothing; MUTATES *head (caller's list).
   On success stores the removed value in *out (if out != NULL),
   frees the old head, advances *head, returns 0.
   On empty list returns -1 and touches nothing. */
static int pop_front(Node **head, int *out) {
    Node *victim;

    if (head == NULL || *head == NULL) {
        return -1;
    }

    victim = *head;
    if (out != NULL) {
        *out = victim->value;
    }
    *head = victim->next;
    free(victim);

    return 0;
}
```

Why `Node **head` — a pointer to the caller's pointer? Because removing the first node changes **which node is first**. The function must update the caller's `head` variable, and C passes arguments by value. A `Node *` parameter would be a copy; assigning to it would move nobody's head. A `Node **` parameter is the address of the caller's variable, so `*head = victim->next` rewrites the caller's head.

```text
  caller:   head = 0xA0 ([10|*]-->[20|NULL])
  call:     pop_front(&head, &v)

  inside:   head (param) = 0xFF00  --->  caller's head (0xA0)
            *head        = 0xA0      victim = 0xA0
            *head = victim->next     --->  caller's head = 0xB0

  caller:   head = 0xB0 ([20|NULL]), v = 10, [10|?] freed
```

Callers pass the address, and check the return:

```c
int main(void) {
    Node *head = NULL;
    int v = 0;

    head = push_front(head, 20);
    head = push_front(head, 10);
    if (head == NULL) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    while (head != NULL) {
        if (pop_front(&head, &v) != 0) {
            break;
        }
        printf("popped %d\n", v);
    }
    /* head is NULL here; nothing left to free */

    return 0;
}
```

```text
popped 10
popped 20
```

`pop_front` on an empty list returns `-1` and does nothing — `*head` stays `NULL`, `*out` untouched. The `head == NULL` check (a `NULL` double pointer, meaning the caller passed no address at all) is defensive: dereferencing it would be undefined behavior, so reject it.

---

## `insert_at` and `remove_at`

Position-based operations share one helper: walk to the node **before** the target. To insert or remove at index `i` (with `i == 0` meaning the front), you need the node at index `i - 1`.

```c
/* OWNERSHIP: takes ownership of `head` (may be NULL).
   Inserts `value` so it ends up AT `index` (0 = front).
   OWNERSHIP: on success returns the new head (caller owns it).
   On failure (bad index or malloc) returns NULL and the old list
   is UNCHANGED — caller still owns it and must still free it. */
static Node *insert_at(Node *head, size_t index, int value) {
    Node *node;
    Node *prev;
    size_t i;

    if (index == 0) {
        node = malloc(sizeof(Node));
        if (node == NULL) {
            return NULL;
        }
        node->value = value;
        node->next = head;
        return node;
    }

    /* walk to index - 1; head is non-NULL here only if index > 0 */
    prev = head;
    for (i = 0; i < index - 1; i++) {
        if (prev == NULL) {
            return NULL;   /* index past the end; list unchanged */
        }
        prev = prev->next;
    }
    if (prev == NULL) {
        return NULL;       /* e.g. insert_at(NULL, 3, x) */
    }

    node = malloc(sizeof(Node));
    if (node == NULL) {
        return NULL;       /* list unchanged */
    }
    node->value = value;
    node->next = prev->next;
    prev->next = node;
    return head;
}
```

The two-assignment splice order matters. `node->next = prev->next` first (the new node points where the predecessor pointed), then `prev->next = node` (the predecessor points at the new node). Reversed, the second assignment overwrites the address the first one needed:

```text
  correct order:

  prev-->[20|*]--->[30|...]      node = [25|NULL]

  node->next = prev->next:   [25|*]--->[30|...]
  prev->next = node:         prev-->[20|*]--->[25|*]--->[30|...]

  reversed (BUG):  prev->next = node first:

  prev-->[20|*]--->[25|NULL]     (address of [30] is lost)
  node->next = prev->next:   [25|*]--->[25|...]  (self-loop!)
```

`remove_at` needs the double pointer again, because index `0` removes the head:

```c
/* OWNERSHIP: MUTATES *head. Removes the node AT `index`,
   frees it, stores its value in *out if out != NULL.
   Returns 0 on success, -1 on bad index or empty list. */
static int remove_at(Node **head, size_t index, int *out) {
    Node *prev;
    Node *victim;
    size_t i;

    if (head == NULL || *head == NULL) {
        return -1;
    }

    if (index == 0) {
        return pop_front(head, out);
    }

    prev = *head;
    for (i = 0; i < index - 1; i++) {
        if (prev->next == NULL) {
            return -1;
        }
        prev = prev->next;
    }

    victim = prev->next;
    if (victim == NULL) {
        return -1;   /* index == length: nothing there */
    }
    if (out != NULL) {
        *out = victim->value;
    }
    prev->next = victim->next;
    free(victim);

    return 0;
}
```

`remove_at(&head, 0, &v)` delegates to `pop_front` so the head-removal logic lives in exactly one place. Duplicating it would create two functions that must stay in sync and eventually will not.

A short session exercising both:

```c
static void print_list(const Node *head) {
    for (const Node *p = head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}
```

```text
after pushes:      10 -> 20 -> 30 -> NULL
insert 25 at 2:    10 -> 20 -> 25 -> 30 -> NULL
insert 5 at 0:     5 -> 10 -> 20 -> 25 -> 30 -> NULL
remove at 1:       5 -> 20 -> 25 -> 30 -> NULL
remove at 3:       5 -> 20 -> 25 -> NULL
remove at 9 fails: 5 -> 20 -> 25 -> NULL (unchanged, returns -1)
```

Every out-of-range index returns the failure value and leaves the list untouched. Functions that fail without side effects are easy to test and easy to trust; functions that half-rewire before failing are neither.

---

## `delete_value`: Removal by Content

Sometimes the caller knows the value, not the position. Delete the **first** node whose `value` matches:

```c
/* OWNERSHIP: MUTATES *head. Frees the first node with
   value == target. Returns 0 if one was removed, -1 if
   no node matched (list unchanged). */
static int delete_value(Node **head, int target) {
    Node **link;

    if (head == NULL || *head == NULL) {
        return -1;
    }

    /* `link` always points AT the pointer that leads to the
       node under inspection: first &head's target, then each
       node's `next` field in turn. */
    link = head;
    while (*link != NULL) {
        if ((*link)->value == target) {
            Node *victim = *link;
            *link = victim->next;
            free(victim);
            return 0;
        }
        link = &(*link)->next;
    }
    return -1;
}
```

This is the double-pointer idiom in full. Instead of tracking `prev` and special-casing the head, `link` points at the pointer that **leads to** the current node — which is `*head` for the first node and `prev->next` for every other. Updating `*link` therefore handles the head and the middle with the same line: `*link = victim->next`.

```text
  list: head-->[10|*]-->[20|*]-->[30|NULL], delete 10 (the head):

  link = &head, *link = [10]. Match:
    victim = [10], *link = victim->next  =>  head = [20|*]-->[30|NULL]

  list: head-->[10|*]-->[20|*]-->[30|NULL], delete 20 (the middle):

  link = &head, *link = [10]. No match, link = &[10].next.
  *link = [20]. Match:
    victim = [20], *link = victim->next  =>  [10|*]-->[30|NULL]
```

One local, no `prev`, no `if (head...)` branch. The cost is conceptual: you are thinking in terms of "the pointer that points here" rather than "the node before here." Once that clicks, every head-special-case in linked-list code collapses into the same three lines.

---

## The Optional Dummy-Head Technique

An alternative that also removes the head special case: keep a **dummy** node on the stack whose `next` is the real head. Every real node then has a predecessor (the dummy for the first one), so insertion and deletion never touch the caller's head directly — until you write it back at the end.

```c
static int delete_value_dummy(Node **head, int target) {
    Node dummy;   /* lives on the STACK; never malloc'd, never free'd */
    Node *prev;
    Node *victim;

    if (head == NULL || *head == NULL) {
        return -1;
    }

    dummy.next = *head;
    prev = &dummy;
    while (prev->next != NULL) {
        if (prev->next->value == target) {
            victim = prev->next;
            prev->next = victim->next;
            free(victim);
            *head = dummy.next;   /* write the (maybe new) head back */
            return 0;
        }
        prev = prev->next;
    }
    return -1;
}
```

| | **Double pointer (`Node **`**)** | **Dummy head** |
| --- | --- | --- |
| Extra memory | none | one stack struct |
| Head update | automatic (`*link` may be `*head`) | manual (`*head = dummy.next`) |
| Lines of code | fewer | more, but each line is simpler |
| Risk | misunderstanding `&(*link)->next` | forgetting the write-back |

Both are correct. The course uses the double pointer as the default and the dummy as the backup plan when the double pointer feels too clever for the function at hand. Pick one per function and never mix them in the same loop.

---

## Three Bugs, Three Reports

Every deletion bug is one of these three. Learn their reports and you will diagnose them in seconds.

### Bug 1: Use-after-free (reading `next` from dead memory)

Covered in the previous part, repeated here because deletion is where it bites: saving the successor **after** freeing instead of before.

```c
/* BUG: victim->next is read after victim is freed */
prev->next = victim->next;   /* correct line ... */
free(victim);                /* ... in the wrong order: */
```

becomes, when swapped:

```c
free(victim);
prev->next = victim->next;   /* heap-use-after-free */
```

```text
=================================================================
==392010==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000020 at pc 0x5612bb0432e1 bp 0x7ffc9e1bb4a0 sp 0x7ffc9e1bb490
READ of size 8 at 0x502000000020 thread T0
    #0 0x5612bb0432e1 in remove_at /tmp/del.c:44
    #1 0x5612bb04341a in main /tmp/del.c:71

0x502000000020 is located 8 bytes inside of 16-byte region [0x502000000018,0x502000000028)
freed by thread T0 here:
    #0 0x7a3b2d82b60f in free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:52
    #1 0x5612bb0432cd in remove_at /tmp/del.c:43

SUMMARY: AddressSanitizer: heap-use-after-free /tmp/del.c:44 in remove_at
```

`READ of size 8` at the `next` offset, freed one line earlier in the same function. Fix: unlink first, free last.

### Bug 2: Lost head (leak by overwriting the only pointer)

```c
/* BUG: head is overwritten WITHOUT freeing the old first node */
head = head->next;   /* the old head's block is now unreachable */
```

No crash, no sanitizer error on the line itself — the program runs perfectly and leaks one node per call. The report arrives at exit, from LeakSanitizer:

```text
=================================================================
==392044==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 16 byte(s) in 1 object(s) allocated from:
    #0 0x7a3b2d82b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x5612bb04328a in push_front /tmp/del.c:22
    #2 0x5612bb0434c0 in main /tmp/del.c:80

SUMMARY: AddressSanitizer: 16 byte(s) leaked in 1 allocation(s).
```

`16 byte(s)` is one `Node`. `#1 push_front` is where the leaked node was born; the leak itself is the missing `free` in the removal. Fix: `free` the victim — `pop_front` does this — before moving the head.

### Bug 3: Dangling `next` (free without unlinking)

```c
/* BUG: frees the node but leaves prev->next pointing at it */
free(victim);
/* missing: prev->next = victim->next; (saved beforehand) */
```

The list still chains into freed memory. The next traversal reads it:

```text
=================================================================
==392078==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000038 at pc 0x5612bb0431a8 bp 0x7ffc9e1bb4a0 sp 0x7ffc9e1bb490
READ of size 4 at 0x502000000038 thread T0
    #0 0x5612bb0431a8 in print_list /tmp/del.c:30
    #1 0x5612bb0434e2 in main /tmp/del.c:84

0x502000000038 is located 0 bytes inside of 16-byte region [0x502000000038,0x502000000048)
freed by thread T0 here:
    #0 0x7a3b2d82b60f in free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:52
    #1 0x5612bb0432f1 in remove_at /tmp/del.c:47

SUMMARY: AddressSanitizer: heap-use-after-free /tmp/del.c:30 in print_list
```

Note the misdirection: the crash is in `print_list`, but the bug is in `remove_at` — the `freed by` frame names the real culprit. `READ of size 4` (an `int`, the `value` field) instead of size 8 distinguishes it from Bug 1: here the traversal reached a fully freed node through a stale link. Fix: rewire `prev->next` **before** freeing, so no pointer into the victim survives it.

| Bug | What survives the `free` | Report | Fix order |
| --- | --- | --- | --- |
| Use-after-free | the `free`d pointer is read directly | `heap-use-after-free`, same function | save `next`, then `free` |
| Lost head | nothing points at the block | `Direct leak`, at exit | `free` before moving head |
| Dangling `next` | a live link points at dead memory | `heap-use-after-free`, later traversal | rewire `prev->next` before `free` |

---

## Checklist

```text
[ ] pop_front takes Node **head and int *out, returns 0 / -1
[ ] insert_at returns the (maybe new) head; failure leaves list unchanged
[ ] splice order: node->next = prev->next BEFORE prev->next = node
[ ] remove_at delegates index 0 to pop_front
[ ] delete_value uses Node **link so head needs no special case
[ ] every out-of-range index returns -1 with no side effects
[ ] unlink BEFORE free; save victim->next first
[ ] free exactly once per removed node; no link points at freed memory
[ ] dummy-head version writes *head = dummy.next back on success
[ ] all three bug reports recognized on sight
[ ] builds with -fsanitize=address,undefined, runs with no diagnostics
```

---

#### [Previous](./01-part.md) | [Next](./03-part.md)
