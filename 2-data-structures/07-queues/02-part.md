# Linked Queues and Deques

The previous part built a queue on a circular array: fast, cache-friendly, but fixed in size unless you resize. This part builds the same FIFO contract on **linked nodes**, where growing means allocating one node instead of copying the world — and then generalizes to a **deque**, a queue open at both ends.

Compile everything in this part with:

```text
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer file.c -o file
```

---

## A Queue of Nodes: Head and Tail

An array queue needs wraparound arithmetic because the front moves through fixed storage. A linked queue sidesteps the problem: the front is a pointer, the back is a pointer, and removing the front is just `head = head->next`.

The struct keeps **both** ends:

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
    int value;
    struct Node *next;
} Node;

typedef struct {
    Node *head;     /* front: where dequeues happen */
    Node *tail;     /* back: where enqueues happen */
    size_t count;
} LQueue;

/* OWNERSHIP: the LQueue owns every node reachable from `head`.
   OWNERSHIP: lqueue_destroy frees them all; enqueue transfers the new
   node to the queue; dequeue hands one value (not a node) to the caller. */
static void lqueue_init(LQueue *q) {
    if (q == NULL) {
        return;
    }
    q->head = NULL;
    q->tail = NULL;
    q->count = 0;
}

static int lqueue_empty(const LQueue *q) {
    return q == NULL || q->head == NULL;
}

static size_t lqueue_size(const LQueue *q) {
    return (q == NULL) ? 0 : q->count;
}
```

Why two pointers? With only `head`, enqueue must walk the whole list to find the end — `O(n)` per insertion. With `tail` cached, both operations are `O(1)`:

```text
  enqueue 30 onto [10 -> 20]:

   before: head -> [10] -> [20] -> NULL (tail at [20])
   after:  head -> [10] -> [20] -> [30] -> NULL (tail at [30])
```

---

## Enqueue at the Tail, Dequeue from the Head

```c
static int lqueue_enqueue(LQueue *q, int value) {
    Node *n;

    if (q == NULL) {
        return -1;
    }
    n = malloc(sizeof(Node));
    if (n == NULL) {
        return -1;
    }
    n->value = value;
    n->next = NULL;

    if (q->tail == NULL) {          /* empty queue: first node */
        q->head = n;
        q->tail = n;
    } else {
        q->tail->next = n;
        q->tail = n;
    }
    q->count++;
    return 0;
}

static int lqueue_dequeue(LQueue *q, int *out) {
    Node *old;

    if (q == NULL || q->head == NULL) {
        return -1; /* empty: report, do not abort */
    }
    old = q->head;
    if (out != NULL) {
        *out = old->value;
    }
    q->head = old->next;
    if (q->head == NULL) {          /* last node removed */
        q->tail = NULL;
    }
    free(old);
    q->count--;
    return 0;
}

static int lqueue_peek(const LQueue *q, int *out) {
    if (q == NULL || q->head == NULL) {
        return -1;
    }
    if (out != NULL) {
        *out = q->head->value;
    }
    return 0;
}

static void lqueue_destroy(LQueue *q) {
    Node *cur;
    Node *next;

    if (q == NULL) {
        return;
    }
    cur = q->head;
    while (cur != NULL) {
        next = cur->next;
        free(cur);
        cur = next;
    }
    q->head = NULL;
    q->tail = NULL;
    q->count = 0;
}
```

Three details deserve attention: the empty-enqueue branch sets both pointers (no `tail->next` exists yet); the last-dequeue branch clears `tail` when `head` becomes `NULL` (else use-after-free); every `malloc` is checked and each node is freed exactly once.

A driver:

```c
int main(void) {
    LQueue q;
    int v;

    lqueue_init(&q);

    lqueue_enqueue(&q, 10);
    lqueue_enqueue(&q, 20);
    lqueue_enqueue(&q, 30);
    printf("size=%zu front=", lqueue_size(&q));
    lqueue_peek(&q, &v);
    printf("%d\n", v);

    while (!lqueue_empty(&q)) {
        lqueue_dequeue(&q, &v);
        printf("dequeued %d (size=%zu)\n", v, lqueue_size(&q));
    }

    if (lqueue_dequeue(&q, &v) != 0) {
        printf("dequeue on empty queue -> status -1, no crash\n");
    }

    lqueue_destroy(&q);
    return 0;
}
```

```text
size=3 front=10
dequeued 10 (size=2)
dequeued 20 (size=1)
dequeued 30 (size=0)
dequeue on empty queue -> status -1, no crash
```

---

## Deques: Open at Both Ends

A **deque** (pronounced "deck", double-ended queue) allows insertion and removal at both ends. Four operations instead of two:

| Operation | Meaning |
| --- | --- |
| `push_back` / `pop_front` | the plain queue behavior |
| `push_front` | insert at the front (like a stack push, but on a deque) |
| `pop_back` | remove from the back |
| `push_front` + `pop_front` together | the deque used as a stack |

A singly linked list implements three of the four in `O(1)`. `pop_back` is the problem: without a `prev` pointer, removing the tail requires walking from `head` to the second-to-last node — `O(n)`. So a real deque uses **doubly linked** nodes:

```text
  doubly linked deque [10 <-> 20 <-> 30]:

   NULL <-- +----+ <--> +----+ <--> +----+ --> NULL
            | 10 |      | 20 |      | 30 |
            +----+      +----+      +----+
            ^                           ^
            head                        tail
```

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct DNode {
    int value;
    struct DNode *prev;
    struct DNode *next;
} DNode;

typedef struct {
    DNode *head;
    DNode *tail;
    size_t count;
} Deque;

static void deque_init(Deque *d) {
    if (d == NULL) {
        return;
    }
    d->head = NULL;
    d->tail = NULL;
    d->count = 0;
}

static int deque_push_back(Deque *d, int value) {
    DNode *n;

    if (d == NULL) {
        return -1;
    }
    n = malloc(sizeof(DNode));
    if (n == NULL) {
        return -1;
    }
    n->value = value;
    n->prev = d->tail;
    n->next = NULL;
    if (d->tail == NULL) {
        d->head = n;
        d->tail = n;
    } else {
        d->tail->next = n;
        d->tail = n;
    }
    d->count++;
    return 0;
}

static int deque_push_front(Deque *d, int value) {
    DNode *n;

    if (d == NULL) {
        return -1;
    }
    n = malloc(sizeof(DNode));
    if (n == NULL) {
        return -1;
    }
    n->value = value;
    n->prev = NULL;
    n->next = d->head;
    if (d->head == NULL) {
        d->head = n;
        d->tail = n;
    } else {
        d->head->prev = n;
        d->head = n;
    }
    d->count++;
    return 0;
}

static int deque_pop_front(Deque *d, int *out) {
    DNode *old;

    if (d == NULL || d->head == NULL) {
        return -1;
    }
    old = d->head;
    if (out != NULL) {
        *out = old->value;
    }
    d->head = old->next;
    if (d->head == NULL) {
        d->tail = NULL;
    } else {
        d->head->prev = NULL;
    }
    free(old);
    d->count--;
    return 0;
}

static int deque_pop_back(Deque *d, int *out) {
    DNode *old;

    if (d == NULL || d->tail == NULL) {
        return -1;
    }
    old = d->tail;
    if (out != NULL) {
        *out = old->value;
    }
    d->tail = old->prev;
    if (d->tail == NULL) {
        d->head = NULL;
    } else {
        d->tail->next = NULL;
    }
    free(old);
    d->count--;
    return 0;
}

static void deque_destroy(Deque *d) {
    DNode *cur;
    DNode *next;

    if (d == NULL) {
        return;
    }
    cur = d->head;
    while (cur != NULL) {
        next = cur->next;
        free(cur);
        cur = next;
    }
    d->head = NULL;
    d->tail = NULL;
    d->count = 0;
}
```

Each pop updates **two** links: the deque end pointer and the surviving neighbor's back-pointer. Forgetting `d->head->prev = NULL` after a `pop_front` leaves a dangling `prev` that a later `pop_back` could follow into freed memory.

---

---

## Circular vs Linked: Which Queue?

| | Circular buffer (part 1) | Linked queue (this part) |
| --- | --- | --- |
| Storage | one contiguous block | one node per element |
| `enqueue` / `dequeue` | `O(1)`, no allocation in steady state | `O(1)`, one `malloc`/`free` per operation |
| Memory overhead | none per element; may waste unused capacity | one pointer (two for deque) per element |
| Cache behavior | excellent: neighbors are adjacent | poor: nodes scattered on the heap |
| Capacity | fixed unless resized (`O(n)` copy, logical order) | unbounded until `malloc` fails |
| `peek` / `size` | `O(1)` | `O(1)` |
| Failure mode | full (status `-1`) or grow | `malloc` returns `NULL` (status `-1`) |
| Best for | bounded buffers, embedded code, batch pipelines | unbounded workloads, frequent grow/shrink, deques |

Rules of thumb: known ceiling — circular; unknown length or `pop_back` — linked.

---

## Application 1: A Ticket-Line Simulation

Queues earn their keep in simulations: model arrivals, serve the front, collect statistics.

```c
#include <stdio.h>

int main(void) {
    LQueue line;
    int arrivals[] = { 3, 0, 2, 1, 0, 4 };
    int steps = 6;
    int served = 0;
    int max_len = 0;
    int t, v, i;

    lqueue_init(&line);

    for (t = 0; t < steps; t++) {
        for (i = 0; i < arrivals[t]; i++) {
            lqueue_enqueue(&line, t * 100 + i); /* ticket id */
        }
        if ((size_t)lqueue_size(&line) > (size_t)max_len) {
            max_len = (int)lqueue_size(&line);
        }
        if (!lqueue_empty(&line)) {
            lqueue_dequeue(&line, &v);
            served++;
            printf("step %d: serve ticket %d (line=%zu)\n",
                   t, v, lqueue_size(&line));
        } else {
            printf("step %d: idle (line=0)\n", t);
        }
    }

    printf("served=%d still waiting=%zu longest line=%d\n",
           served, lqueue_size(&line), max_len);
    lqueue_destroy(&line);
    return 0;
}
```

```text
step 0: serve ticket 0 (line=2)
step 1: serve ticket 1 (line=1)
step 2: serve ticket 2 (line=2)
step 3: serve ticket 200 (line=2)
step 4: serve ticket 201 (line=1)
step 5: serve ticket 202 (line=4)
served=6 still waiting=4 longest line=5
```

The three statistics — served, still waiting, longest line — are the whole point of the exercise version. A queue turns "who waits how long" from a vague worry into numbers.

---

## Application 2: BFS Preview

Breadth-first search visits a graph level by level, and the data structure that enforces "level by level" is a queue: push the start node, then repeatedly dequeue a node and enqueue its unvisited neighbors.

```text
      A                 queue         visited
      / \               -----         -------
     B   C              [A]           A
    /                 dequeue A, enqueue B C
   D                  [B C]           A B C
                      dequeue B, enqueue D
                      [C D]           A B C D
                      dequeue C, dequeue D
                      []              done: A B C D
```

Enqueue the root. Dequeue a node, print it, enqueue its children left-to-right. The queue guarantees parents print before children and left siblings before right ones — the FIFO order *is* the level order. Trees get a full chapter later; for now, remember the shape: **BFS = queue + visited set**.

---

## Destroy and Common Bugs

Every linked queue needs a destroy that walks from `head` and frees each node exactly once. The classic bugs:

**Bug 1: losing `tail` on the last dequeue.** `head` becomes `NULL` but `tail` still points at the freed node. The next enqueue does `q->tail->next = n` through a dangling pointer:

```text
==1234==ERROR: AddressSanitizer: heap-use-after-free on address 0x...
WRITE of size 8 at 0x... thread T0
    #0 in lqueue_enqueue lqueue.c:35
    #1 in main lqueue.c:120
freed by thread T0 here:
    #0 in free
    #1 in lqueue_dequeue lqueue.c:58
SUMMARY: AddressSanitizer: heap-use-after-free lqueue.c:35 in lqueue_enqueue
```

Fix: `if (q->head == NULL) { q->tail = NULL; }` on every removal path.

**Bug 2: freeing before reading `next`.** `free(cur); cur = cur->next;` reads through freed memory. Save `next` first, then free.

**Bug 3: deque pop that fixes one side only.** After `pop_front`, both `d->head` and `d->head->prev` must be updated (mirror for `pop_back`). Updating only the end pointer leaves a neighbor pointing into freed memory.

| Symptom | Likely cause |
| --- | --- |
| crash on the second enqueue after draining the queue | `tail` not reset to `NULL` when the last node left |
| ASan heap-use-after-free in destroy | node freed twice, or `next` read after `free` |
| deque works from one end, corrupts from the other | missing `prev`/`next` update on the surviving neighbor |
| leak report listing `Node` blocks | `destroy` never called, or early `return` before it |

---

## Summary

- A linked queue keeps `head` (dequeue end) and `tail` (enqueue end). Both operations are `O(1)`; with only `head`, enqueue degrades to `O(n)`.
- Enqueue on an empty queue sets both pointers; dequeue of the last node clears both pointers.
- A deque adds `push_front` and `pop_back`. `pop_back` in `O(1)` needs doubly linked nodes.
- Every pop updates the end pointer **and** the surviving neighbor's back-link.
- Circular buffers win on cache and allocation churn; linked queues win on unbounded growth. The table above is the decision procedure.
- Queues drive simulations (ticket lines, spoolers) and BFS: enqueue neighbors, dequeue the next to visit.
- `destroy` walks from `head` saving `next` before each `free`. Underflow returns `-1`; it never aborts.

---

## Checklist

- I keep both `head` and `tail` and can explain what each one is for.
- Enqueue links via `tail->next` (or sets both pointers when empty) and advances `tail`.
- Dequeue advances `head`, frees exactly the removed node, and clears `tail` when the queue drains.
- Read-only helpers take `const LQueue *` / `const Deque *`; mutating ones return a status.
- Deque nodes carry `prev` and `next`; every pop repairs both sides of the cut.
- `destroy` saves `next`, frees the node, and nulls both ends plus the count.
- `malloc` results are checked; `NULL` queues and empty queues return `-1`, never crash.
- The program compiles with `-std=c11 -Wall -Wextra -pedantic` plus ASan/UBSan with zero warnings.

---

#### [Previous](./01-part.md) | [Next](./exercises.md)
