# Doubly Linked Lists and Patterns

A singly linked list can only move forward. Given a node, you can reach its successor, but reaching its predecessor means walking from the head all over again. A doubly linked list pays one extra pointer per node to remove that asymmetry: each node points both ways. The reward is O(1) removal when you already hold the node, and O(1) append when you keep a tail pointer.

---

## The Doubly Node

```c
typedef struct DNode DNode;

struct DNode {
    int value;
    DNode *prev;
    DNode *next;
};
```

| Field | Meaning |
| --- | --- |
| `value` | the payload, as before |
| `prev` | address of the previous node, or `NULL` at the front |
| `next` | address of the next node, or `NULL` at the back |

Three nodes holding `10`, `20`, `30`, drawn with both directions:

```text
  NULL <--+-------+------+<-->+-------+------+<-->+-------+------+--> NULL
          | value | 10   |    | value | 20   |    | value | 30   |
          | prev  | NULL |    | prev  | 0x.. |    | prev  | 0x.. |
          | next  | 0x.. |--->| next  | 0x.. |--->| next  | NULL |
          +-------+------+    +-------+------+    +-------+------+

  every arrow has a partner pointing back
  the ends terminate in NULL on BOTH sides
```

The invariant that must hold after every operation:

> **For every node `n` with a successor: `n->next->prev == n`. For every node `n` with a predecessor: `n->prev->next == n`.**

If either half breaks, forward and backward traversals disagree about the list's contents, and exactly one of them walks into garbage. Every doubly-linked bug in this part is a violation of that invariant.

Construction mirrors the singly case — one `malloc`, one `NULL` check, three initializations:

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

/* OWNERSHIP: returns a NEW heap node; the CALLER must free() it
   (directly or via dlist_destroy). Returns NULL on failure. */
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
```

---

## The List Object: Head, Tail, Size

Singly lists got away with a bare `Node *head` because every operation starts at the front. Doubly lists want O(1) append at the back, which needs the tail without a traversal. The standard answer is a small owning struct:

```c
typedef struct {
    DNode *head;    /* first node, or NULL when empty */
    DNode *tail;    /* last node, or NULL when empty  */
    size_t size;    /* how many nodes, kept exact      */
} DList;

static void dlist_init(DList *list) {
    list->head = NULL;
    list->tail = NULL;
    list->size = 0;
}
```

```text
  empty:   head = NULL, tail = NULL, size = 0

  one node [10]:

  head --+              +-- tail
         v              v
        +-------+------+
        | value | 10   |
        | prev  | NULL |
        | next  | NULL |
        +-------+------+

  three nodes:

  head                                           tail
   |                                              |
   v                                              v
  [10|<-->][20|<-->][30]   size = 3
   ^ NULL-terminated        ^ NULL-terminated
   prev at front            next at back
```

`size` is maintained, not computed: every successful insert does `size++`, every removal does `size--`. That makes `size` O(1) instead of the O(n) walk `list_length` needed. The price is discipline — any mutation that forgets the counter corrupts it silently. Under the sanitizers there is no report for a wrong `size`; only tests catch it.

| | **Bare head pointer** | **Head + tail + size** |
| --- | --- | --- |
| Append | O(n) walk to the end | O(1) via `tail` |
| Remove last | O(n) walk to `size - 1` | O(1) via `tail` |
| Length | O(n) traversal | O(1) field read |
| Empty test | `head == NULL` | `size == 0` (and both pointers `NULL`) |
| Cost | nothing | one struct, counter discipline |

---

## O(1) Append and O(1) Removal

Append links the new node after the tail:

```c
/* OWNERSHIP: MUTATES *list; takes ownership of nothing, allocates one node.
   Returns 0 on success, -1 on malloc failure (list unchanged) or if
   list is NULL. */
static int dlist_push_back(DList *list, int value) {
    DNode *node;

    if (list == NULL) {
        return -1;
    }

    node = dnode_new(value);
    if (node == NULL) {
        return -1;
    }

    node->prev = list->tail;
    node->next = NULL;

    if (list->tail == NULL) {
        /* empty list: new node is both first and last */
        list->head = node;
        list->tail = node;
    } else {
        list->tail->next = node;
        list->tail = node;
    }
    list->size++;

    return 0;
}
```

Four pointer writes in the general case (`node->prev`, `node->next`, `tail->next`, `tail`), two plus the head assignment in the empty case. The empty case is the one people forget: with `tail == NULL`, the line `list->tail->next = node` would dereference `NULL`. The branch exists because the first node has no predecessor to link from.

Removal given the node itself — no search, no index:

```c
/* OWNERSHIP: MUTATES *list; frees `node`, which must belong to *list.
   The caller must not touch `node` after this call. */
static void dlist_remove(DList *list, DNode *node) {
    if (list == NULL || node == NULL) {
        return;
    }

    /* stitch the predecessor to the successor */
    if (node->prev != NULL) {
        node->prev->next = node->next;
    } else {
        list->head = node->next;   /* victim was first */
    }

    if (node->next != NULL) {
        node->next->prev = node->prev;
    } else {
        list->tail = node->prev;   /* victim was last */
    }

    free(node);
    list->size--;
}
```

Four cases, one function, no traversal:

```text
  middle:  [A]<-->[V]<-->[B]   =>  [A]<-->[B]
  first:   head->[V]<-->[B]     =>  head->[B], B->prev = NULL
  last:    [A]<-->[V]<-tail     =>  [A]<-tail, A->next = NULL
  only:    head->[V]<-tail      =>  head = tail = NULL, size = 0
```

Each branch preserves the invariant from both sides: the predecessor's `next` and the successor's `prev` are updated together, or the `head`/`tail` is moved when there is no predecessor/successor. Updating only one side compiles and passes a forward-only test — then fails every backward traversal. Test both directions after every mutation.

Front operations are symmetric and included for completeness:

```c
static int dlist_push_front(DList *list, int value) {
    DNode *node;

    if (list == NULL) {
        return -1;
    }
    node = dnode_new(value);
    if (node == NULL) {
        return -1;
    }

    node->prev = NULL;
    node->next = list->head;

    if (list->head == NULL) {
        list->head = node;
        list->tail = node;
    } else {
        list->head->prev = node;
        list->head = node;
    }
    list->size++;

    return 0;
}
```

---

## Manual Iterators: Walking Without an Index

Arrays iterate by index. Lists iterate by pointer. The idiom generalizes to both directions:

```c
/* OWNERSHIP: borrows `list`; prints front to back. */
static void dlist_print_forward(const DList *list) {
    if (list == NULL) {
        return;
    }
    for (const DNode *p = list->head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}

/* OWNERSHIP: borrows `list`; prints back to front. */
static void dlist_print_backward(const DList *list) {
    if (list == NULL) {
        return;
    }
    for (const DNode *p = list->tail; p != NULL; p = p->prev) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}
```

```text
forward:  10 -> 20 -> 30 -> NULL
backward: 30 -> 20 -> 10 -> NULL
```

If the two lines are not exact reverses, the `prev`/`next` invariant is broken — usually a mutation that updated one direction and forgot the other. Printing both ways is the cheapest invariant check available, and it belongs in every test:

```c
static const DNode *dlist_find(const DList *list, int target) {
    const DNode *p;

    if (list == NULL) {
        return NULL;
    }
    for (p = list->head; p != NULL; p = p->next) {
        if (p->value == target) {
            return p;
        }
    }
    return NULL;
}
```

Note the return type: `const DNode *`. The finder borrows; it does not transfer ownership. The caller may pass the result to `dlist_remove` (which needs a non-`const` pointer — a cast at that point documents the ownership handoff) or read through it. Returning a non-`const` pointer from a finder invites the caller to scribble on `next` directly, bypassing the invariant. Keep mutation inside `dlist_remove`.

An in-place update that keeps the plumbing intact touches only `value`:

```c
static int dlist_update(DList *list, int target, int replacement) {
    DNode *p;

    if (list == NULL) {
        return -1;
    }
    for (p = list->head; p != NULL; p = p->next) {
        if (p->value == target) {
            p->value = replacement;   /* payload only: links untouched */
            return 0;
        }
    }
    return -1;
}
```

---

## The Sentinel Node (Optional, but Worth Knowing)

The `dlist_remove` above has four cases because the ends are special: the first node has no `prev`, the last has no `next`. A **sentinel** removes the special cases by guaranteeing the list is never truly empty — a permanent dummy node sits in the ring, and real nodes come and go around it.

```text
  empty circular list with sentinel S (on the heap or embedded):

       +-----+
       |  S  |
       +-----+
      ^ next |
      | prev v
       +-----+

  S->next == S and S->prev == S: "empty" means "points to itself"

  with [10], [20]:

       +-----+<-->[10]<-->[20]<-->+-----+
       |  S  |                    | (back to S)
       +-----+                    +-----+
```

```c
/* Minimal sketch: circular sentinel. S->next is the first real node
   (or S when empty); S->prev is the last real node (or S when empty).
   No NULL checks inside the loop: the loop ends at S, not at NULL. */
static void sentinel_insert_before(DNode *pos, DNode *node) {
    node->prev = pos->prev;
    node->next = pos;
    pos->prev->next = node;
    pos->prev = node;
}
```

No branches: `pos` is never `NULL` (worst case it is the sentinel), so `pos->prev` always exists. Removal is equally branchless, and emptiness is `S.next == &S`.

| | **NULL-terminated** | **Sentinel ring** |
| --- | --- | --- |
| Empty representation | `head == NULL` | `sentinel.next == &sentinel` |
| End test in loop | `p != NULL` | `p != &sentinel` |
| Insert/remove branches | head/tail special cases | none (uniform) |
| Cost | nothing extra | one permanent node, trickier teardown |

The course does not require sentinels. Know they exist, recognize `p != &sentinel` when you see it in other people's code, and reach for one when a function accumulates its third `if (at the end)` branch.

---

## List vs Array vs Vector: When to Use What

| Need | Array (`int a[N]`) | Vector (`IntVec`) | Singly list | Doubly list |
| --- | --- | --- | --- | --- |
| Random access `a[i]` | O(1) | O(1) | O(n) | O(n) |
| Push at front | O(n) shift | O(n) shift | O(1) | O(1) |
| Push at back | impossible when full | O(1) amortized | O(n) without tail | O(1) with tail |
| Remove given the node | n/a | O(n) shift | O(n) find prev | O(1) |
| Memory per element | `sizeof(int)` | `sizeof(int)` | + 1 pointer | + 2 pointers |
| Cache behavior | best (contiguous) | best (contiguous) | worst (scattered) | worst (scattered) |
| Size known up front | required | not required | not required | not required |

The default choice is the vector. It has O(1) indexing, contiguous cache-friendly memory, and amortized O(1) push at the back — which is where most programs push. Reach for a list only when one of these holds:

- Insertions and removals happen mostly at the **front**, or at a held node, and happen often enough that shifting a vector shows up in measurements.
- Elements are large and moves are expensive, so pointer rewiring beats copying.
- The size oscillates wildly and per-node allocation matches the program's lifetime pattern (each node freed when its task completes).

Otherwise the vector wins, usually by a factor nobody has to measure to believe: walking a scattered list pointer-chases through the heap, while walking an array streams through the cache.

---

## Destroy in Both Directions (and Prove It)

Teardown walks forward — backward links need no special handling because the whole node is freed at once:

```c
/* OWNERSHIP: takes ownership of *list; frees every node,
   leaves *list empty (head = tail = NULL, size = 0).
   Safe to call on an empty or already-destroyed list. */
static void dlist_destroy(DList *list) {
    DNode *cur;

    if (list == NULL) {
        return;
    }

    cur = list->head;
    while (cur != NULL) {
        DNode *next = cur->next;   /* save BEFORE free */
        free(cur);
        cur = next;
    }

    list->head = NULL;
    list->tail = NULL;
    list->size = 0;
}
```

Proving the destroy worked means checking **both** ends and the counter, not just the head:

```c
int main(void) {
    DList list;

    dlist_init(&list);

    if (dlist_push_back(&list, 10) != 0 ||
        dlist_push_back(&list, 20) != 0 ||
        dlist_push_back(&list, 30) != 0) {
        fprintf(stderr, "out of memory\n");
        dlist_destroy(&list);
        return 1;
    }

    dlist_print_forward(&list);
    dlist_print_backward(&list);
    printf("size = %zu\n", list.size);

    dlist_destroy(&list);

    printf("after destroy: head=%s tail=%s size=%zu\n",
           list.head == NULL ? "NULL" : "DANGLING",
           list.tail == NULL ? "NULL" : "DANGLING",
           list.size);

    return 0;
}
```

```text
10 -> 20 -> 30 -> NULL
30 -> 20 -> 10 -> NULL
size = 3
after destroy: head=NULL tail=NULL size=0
```

The last line is the assertion that matters. A destroy that frees the nodes but leaves `tail` pointing at dead memory passes a forward-only check and fails the moment anything touches the back. Setting both to `NULL` and zeroing `size` makes the destroyed list a valid empty list — `dlist_destroy` twice is safe, and reuse after destroy works.

One broken variant, for the collection — freeing through the backward links while also walking forward double-frees nothing but reads dead memory:

```c
/* BUG: cur->prev is read from a node that may already be freed.
   Forward teardown needs ONLY cur->next. */
```

The rule from the singly chapter carries over unchanged: save what you need (`next`), free, move. The backward pointers come along for free because the allocator reclaims the entire `DNode` block, `prev` field included.

---

## Checklist

```text
[ ] DNode has value + prev + next; both links NULL at construction
[ ] invariant: n->next->prev == n and n->prev->next == n, always
[ ] DList keeps head + tail + size; size maintained, not recomputed
[ ] empty list: head = tail = NULL, size = 0; every function handles it
[ ] push_back / push_front branch on empty (tail/head == NULL)
[ ] dlist_remove updates BOTH sides, or moves head/tail at the ends
[ ] finders return const DNode *; mutation goes through remove/update
[ ] forward and backward prints are exact reverses in every test
[ ] dlist_destroy saves next BEFORE free, clears head/tail/size after
[ ] list vs array vs vector chosen deliberately, default is vector
[ ] builds with -fsanitize=address,undefined, runs with no diagnostics
```

---

#### [Previous](./02-part.md) | [Next](./exercises.md)
