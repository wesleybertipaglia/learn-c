# Exercise List 7 - Queues

Six exercises on FIFO queues, plus one instruction that applies to all of them. Every program must compile with `gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer file.c -o file` and produce **zero** warnings.

Underflow and overflow never abort on user input: return a status (`0` for success, `-1` for failure) and let the caller print a message. Check every `malloc` and `realloc` (through a temporary pointer); `free` exactly once per allocation.

---

## Exercise 1: Circular Queue Enqueue / Dequeue

Implement the fixed-capacity circular buffer from part 1: `head`, `tail`, `count`, wraparound with `% capacity`.

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int *data;
    size_t head;
    size_t tail;
    size_t count;
    size_t capacity;
} Queue;

/* OWNERSHIP: init allocates; the Queue owns `data` afterwards.
   OWNERSHIP: destroy frees it; enqueue/dequeue/peek only borrow it. */
static int queue_init(Queue *q, size_t capacity);
static void queue_destroy(Queue *q);
static int queue_enqueue(Queue *q, int value);      /* 0 ok, -1 full */
static int queue_dequeue(Queue *q, int *out);       /* 0 ok, -1 empty */
static int queue_peek(const Queue *q, int *out);    /* 0 ok, -1 empty */

int main(void) {
    Queue q;

    if (queue_init(&q, 4) != 0) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    queue_enqueue(&q, 10);
    queue_enqueue(&q, 20);
    queue_enqueue(&q, 30);

    /* dequeue two, enqueue two more so tail wraps, then drain */
    /* ... your driver here ... */

    queue_destroy(&q);
    return 0;
}
```

```text
# output
enqueued 10 20 30 (size=3)
dequeued 10
dequeued 20
enqueued 40 50 (wrapped)
draining: 30 40 50
dequeue on empty -> rejected, no crash
```

Requirements:

- `head` and `tail` advance with `(i + 1) % capacity` on every operation.
- Empty is `count == 0`, full is `count == capacity`. Do not use `head == tail` as a state test; explain in a comment why it is ambiguous.
- `peek` takes `const Queue *` and does not move either index.
- Dequeue from an empty queue and enqueue into a full queue both return `-1` with a message; neither aborts.
- `static` on all helpers; `const` on every parameter that only reads.
- Extra credit: add `size_t queue_size(const Queue *q)` and print it after every step.

---

## Exercise 2: Resize Preserving Order

Extend exercise 1 with a grow operation that keeps logical order. Start with capacity 3, wrap the ring, then double it and prove the order survived.

```c
/* Grows to new_capacity, preserving logical (FIFO) order.
   OWNERSHIP: on success the old block is freed and replaced.
   OWNERSHIP: on failure the old block is untouched; returns -1. */
static int queue_resize(Queue *q, size_t new_capacity);
```

```text
# output
before grow (cap=3): 30 40 50   [head=2, wrapped]
after grow (cap=6):  30 40 50   [head=0]
enqueued 60 70 80 after grow
draining: 30 40 50 60 70 80
```

Requirements:

- Copy with `tmp[i] = q->data[(q->head + i) % q->capacity]` for `i` in `0 .. count - 1`. Explain in a comment why a plain `tmp[i] = q->data[i]` scrambles the queue after a wrap.
- Reset `head` to `0` and `tail` to `count % new_capacity` after the copy.
- Reject `new_capacity < q->count` and `new_capacity == 0` with `-1`.
- Allocate into a temporary pointer; only `free` the old block and install the new one after the copy. Show the ASan-clean run.
- Extra credit: write `queue_grow` that doubles the capacity and call it automatically from `enqueue` instead of reporting full.

---

## Exercise 3: Linked Queue

Rebuild the same FIFO contract on singly linked nodes with `head` + `tail`, so the queue is unbounded.

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
    int value;
    struct Node *next;
} Node;

typedef struct {
    Node *head;
    Node *tail;
    size_t count;
} LQueue;

/* OWNERSHIP: the LQueue owns all nodes reachable from `head`.
   OWNERSHIP: enqueue transfers the new node in; dequeue frees the
   removed node and copies only its value out; destroy frees the rest. */
static void lqueue_init(LQueue *q);
static void lqueue_destroy(LQueue *q);
static int lqueue_enqueue(LQueue *q, int value);
static int lqueue_dequeue(LQueue *q, int *out);
static int lqueue_peek(const LQueue *q, int *out);
```

```text
# output
enqueued 10 20 30 (size=3, front=10)
dequeued 10 (size=2)
dequeued 20 (size=1)
enqueued 40 (size=2)
draining: 30 40
dequeue on empty -> rejected, no crash
```

Requirements:

- Enqueue links after `tail` (`O(1)`). Explain in a comment why keeping only `head` would make enqueue `O(n)`.
- The first enqueue sets both `head` and `tail`; dequeue of the last node clears both. Missing either half is a use-after-free — say which one in a comment.
- Every `malloc` is checked; every removed node is freed exactly once; `destroy` saves `next` before each `free`.
- Run under ASan/LSan with zero reports after draining and destroying.
- Extra credit: implement `void lqueue_print(const LQueue *q)` that prints front-to-back without modifying the queue.

---

## Exercise 4: Deque

Implement a double-ended queue on doubly linked nodes: `push_front`, `push_back`, `pop_front`, `pop_back`, all `O(1)`.

```c
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

/* OWNERSHIP: same as LQueue — the Deque owns its nodes;
   pops copy the value out and free the node; destroy frees the rest. */
static void deque_init(Deque *d);
static void deque_destroy(Deque *d);
static int deque_push_front(Deque *d, int value);
static int deque_push_back(Deque *d, int value);
static int deque_pop_front(Deque *d, int *out);
static int deque_pop_back(Deque *d, int *out);
```

```text
# output
push_back 10 20 / push_front 5 -> front-to-back: 5 10 20
pop_front -> 5
pop_back -> 20
remaining: 10
pop_front -> 10 (empty now)
pop_back on empty -> rejected, no crash
```

Requirements:

- Each insertion sets both the neighbor link and the end pointer; each removal repairs both (`head` + `head->prev`, or `tail` + `tail->next`). Comment each repair.
- Demonstrate all four operations in `main`, including popping from each end until empty.
- A singly linked list cannot do `pop_back` in `O(1)` — explain why in a comment above `DNode`.
- Use the deque as a stack (`push_front` + `pop_front`) for three values and show LIFO order in the output.
- Extra credit: add `int deque_peek_front(const Deque *d, int *out)` and `deque_peek_back` without duplicating the empty-check logic more than once.

---

## Exercise 5: Simulation (Ticket Line / Printer Spooler)

Pick one scenario and simulate it with either queue implementation. Serve the front, admit new arrivals, and report statistics at the end.

```c
#include <stdio.h>
#include <stdlib.h>

/* ... your queue from exercise 1 or 3 ... */

int main(void) {
    /* arrivals[t] = jobs arriving at step t; service = 1 job per step */
    int arrivals[] = { 3, 0, 2, 1, 0, 4 };
    int steps = 6;
    /* track: served, still waiting, longest line, idle steps */
    return 0;
}
```

```text
# output (ticket line example)
step 0: 3 arrive, serve ticket 0 (line=2)
step 1: 0 arrive, serve ticket 1 (line=1)
step 2: 2 arrive, serve ticket 2 (line=2)
step 3: 1 arrive, serve ticket 200 (line=2)
step 4: 0 arrive, serve ticket 201 (line=1)
step 5: 4 arrive, serve ticket 202 (line=4)
served=6 still waiting=4 longest line=5 idle steps=0
```

Requirements:

- Ticket ids (or page counts) must come out in FIFO order — the first arrival is the first served. Assert this in the output by printing ids.
- Report at least three statistics: jobs served, jobs still waiting, and longest line observed (printer variant: total pages printed and idle steps instead).
- Invalid input (negative arrivals, unknown job size) is rejected with a message and does not abort the simulation.
- State your ownership choice in a comment: who frees the queue, and where.
- Extra credit: serve variable-size jobs (pages per job) and report average wait in steps; or allow 2 servers per step and compare the longest line.

---

## Exercise 6: Sliding Window Maximum or Level-Order

Choose **one** of the two. Both force the queue to do real work.

**Option A — sliding window maximum (simple version).** Given an array and a window size `k`, print the maximum of each window. A deque of indices keeps it `O(n)`; a plain rescan keeps it `O(n*k)` and is acceptable if you say so in a comment.

```text
# input
values: 4 2 12 3 8 7, k = 3

# output
window [4 2 12] -> 12
window [2 12 3] -> 12
window [12 3 8] -> 12
window [3 8 7] -> 8
```

**Option B — level-order print with a queue.** Store pointers to tree nodes in a queue, dequeue-print-enqueue-children, and print the tree breadth-first.

```c
typedef struct TNode {
    int value;
    struct TNode *left;
    struct TNode *right;
} TNode;

/* OWNERSHIP: the queue borrows TNode pointers; it never frees them.
   OWNERSHIP: the tree is freed separately (post-order) after printing. */
```

```text
# output (tree: 1 with children 2 3; 2 with children 4 5)
level order: 1 2 3 4 5
```

Requirements:

- Option A: handle `k <= 0`, `k > n`, and empty input with a message and status return, not a crash. State the complexity of your approach in a comment.
- Option B: the queue holds `TNode *` (borrowed pointers). The tree nodes are freed exactly once, separately from the queue nodes — explain the two ownership layers in comments.
- Either option: the queue from exercise 1, 3, or 4 is reused unchanged, proving the abstraction holds.
- Extra credit: do the option you did not choose, or make option A run in `O(n)` with a deque of indices and show timings for a large input.

---

#### [Previous](./02-part.md) | [Next](../08-hashtables/01-part.md)
