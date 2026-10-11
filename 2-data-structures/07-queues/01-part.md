# Queues with Circular Buffers

A stack returns the item you added most recently. Many real problems need the opposite: the item that has been waiting longest goes first. A printer serves the job that arrived first. A ticket line serves the person at the front. A network card transmits the oldest buffered packet first. That discipline is called **FIFO** — first in, first out — and the data structure that implements it is the **queue**.

A queue has two ends and two rules:

- `enqueue` adds at the **back** (tail).
- `dequeue` removes from the **front** (head).

Everything else — `peek`, `is_empty`, `is_full`, `size` — is observation. The contract is small, but the implementation has one decision that matters: where the storage lives, and what happens when the ends move.

Compile everything in this part with:

```text
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer file.c -o file
```

---

## FIFO in One Picture

```text
  enqueue 10          enqueue 20          enqueue 30          dequeue -> 10
  +-----+             +------+------+      +------+------+------+   +------+------+ 
  | 10  |  back       | 10 | 20  | back   | 10 | 20 | 30  | back | 20 | 30  | back  
  +-----+  front      +------+------+      +------+------+------+   +------+------+ 
     ^  front            ^  front              ^  front                ^  front     
```

The front only moves forward. The back only grows. Nothing in the middle ever moves — or at least, nothing should have to.

---

## Why Shifting Is a Bad Queue

The naive queue is an array plus "shift everything left on dequeue":

```c
#include <stdio.h>

#define CAP 8

typedef struct {
    int data[CAP];
    int count;
} NaiveQueue;

static int naive_enqueue(NaiveQueue *q, int value) {
    if (q == NULL) {
        return -1;
    }
    if (q->count >= CAP) {
        return -1; /* full: report, do not abort */
    }
    q->data[q->count] = value;
    q->count++;
    return 0;
}

static int naive_dequeue(NaiveQueue *q, int *out) {
    int i;

    if (q == NULL) {
        return -1;
    }
    if (q->count == 0) {
        return -1; /* empty: report, do not abort */
    }
    if (out != NULL) {
        *out = q->data[0];
    }
    for (i = 1; i < q->count; i++) {
        q->data[i - 1] = q->data[i];
    }
    q->count--;
    return 0;
}
```

It works. It is also `O(n)` on every dequeue, because `n - 1` elements move. Enqueue a thousand jobs and dequeue a thousand jobs and you have performed roughly half a million copies to simulate a line:

| Operation | Naive array queue | What you want |
| --- | --- | --- |
| `enqueue` | `O(1)` — write at `data[count]` | `O(1)` |
| `dequeue` | `O(n)` — shift `n - 1` elements left | `O(1)` |
| `peek` | `O(1)` — read `data[0]` | `O(1)` |
| `n` dequeues in a row | `O(n^2)` total copies | `O(n)` total |

Worse, the shift loop is exactly the loop beginners get backwards (the same bug as `remove_at` in the arrays chapter). Run it right-to-left and each copy overwrites the element the next iteration still needs. The fix is not to write the loop more carefully. The fix is to never move the elements at all: move the **indices** instead.

---

## The Ring Idea: Head, Tail, Count

Keep the array, but stop anchoring the front at index `0`. Track two positions:

- `head`: index of the front element (next to dequeue).
- `tail`: index where the next enqueue will write.
- `count`: how many elements are stored right now.
- `capacity`: how many the block holds.

Both indices advance with wraparound:

```c
q->tail = (q->tail + 1) % q->capacity;
q->head = (q->head + 1) % q->capacity;
```

The `% capacity` is what makes the buffer **circular**: index `capacity - 1` is followed by index `0`, like numbers on a clock face.

```text
  capacity 5, count 3, head at 3, tail at 1 (wrapped)

   index:    0     1     2     3     4
           +-----+-----+-----+-----+-----+
           | 30  |     |     | 10  | 20  |
           +-----+-----+-----+-----+-----+
              ^                    ^
              tail                 head
              (next write)         (front = 10)

   logical order: 10 (head) -> 20 -> 30 -> tail
   physical order: data[3], data[4], data[0]
```

Enqueue writes at `tail` and advances it. Dequeue reads from `head` and advances it. Nothing else moves. Both operations are `O(1)`.

```text
  dequeue 10: head 3 -> 4        enqueue 40: tail 1 -> 2

   index:    0     1     2     3     4      index:    0     1     2     3     4
           +-----+-----+-----+-----+-----+          +-----+-----+-----+-----+-----+
           | 30  |     |     |     | 20  |          | 30  | 40  |     |     | 20  |
           +-----+-----+-----+-----+-----+          +-----+-----+-----+-----+-----+
              ^                       ^                ^  ^                    ^
              tail                    head             |  tail                 head
```

---

## The Struct and Its Ownership

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int *data;        /* heap block of `capacity` ints */
    size_t head;      /* index of front element */
    size_t tail;      /* index of next write */
    size_t count;     /* elements stored */
    size_t capacity;  /* size of the block */
} Queue;

/* OWNERSHIP: the Queue owns `data` after a successful init.
   OWNERSHIP: queue_destroy frees it; every other function borrows it. */
static int queue_init(Queue *q, size_t capacity) {
    if (q == NULL || capacity == 0) {
        return -1;
    }
    q->data = malloc(capacity * sizeof(int));
    if (q->data == NULL) {
        q->head = 0;
        q->tail = 0;
        q->count = 0;
        q->capacity = 0;
        return -1;
    }
    q->head = 0;
    q->tail = 0;
    q->count = 0;
    q->capacity = capacity;
    return 0;
}

static void queue_destroy(Queue *q) {
    if (q == NULL) {
        return;
    }
    free(q->data);
    q->data = NULL;
    q->head = 0;
    q->tail = 0;
    q->count = 0;
    q->capacity = 0;
}

static int queue_empty(const Queue *q) {
    return q == NULL || q->count == 0;
}

static int queue_full(const Queue *q) {
    return q != NULL && q->count == q->capacity;
}

static size_t queue_size(const Queue *q) {
    return (q == NULL) ? 0 : q->count;
}
```

Notes on the shape:

- `malloc` is checked. On failure the struct is left in a state `queue_destroy` can safely handle.
- Read-only observers take `const Queue *`. Anything that writes takes `Queue *`.
- `queue_destroy` sets `data` to `NULL` so a double destroy is harmless.

---

## Enqueue, Dequeue, Peek

Status returns, never aborts: `0` on success, `-1` on overflow, underflow, or a `NULL` queue. User input that overfills or over-drains the queue is an ordinary event the caller handles, not a reason to crash.

```c
static int queue_enqueue(Queue *q, int value) {
    if (q == NULL || q->data == NULL) {
        return -1;
    }
    if (q->count == q->capacity) {
        return -1; /* full: caller decides what to do */
    }
    q->data[q->tail] = value;
    q->tail = (q->tail + 1) % q->capacity;
    q->count++;
    return 0;
}

static int queue_dequeue(Queue *q, int *out) {
    if (q == NULL || q->data == NULL) {
        return -1;
    }
    if (q->count == 0) {
        return -1; /* empty: caller decides what to do */
    }
    if (out != NULL) {
        *out = q->data[q->head];
    }
    q->head = (q->head + 1) % q->capacity;
    q->count--;
    return 0;
}

static int queue_peek(const Queue *q, int *out) {
    if (q == NULL || q->data == NULL) {
        return -1;
    }
    if (q->count == 0) {
        return -1;
    }
    if (out != NULL) {
        *out = q->data[q->head];
    }
    return 0;
}
```

- `peek` takes `const Queue *`: it promises not to move `head` or `tail`.
- `out` may be `NULL`, which means "discard the value, just advance". That keeps the interface usable for queues used purely as counters.
- Every path that touches `q->data[i]` has already proven `i < capacity`, so ASan stays quiet.

A small driver that exercises wraparound:

```c
int main(void) {
    Queue q;
    int v;
    int i;

    if (queue_init(&q, 4) != 0) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    for (i = 1; i <= 4; i++) {
        queue_enqueue(&q, i * 10); /* 10 20 30 40 */
    }
    printf("full? %d size=%zu\n", queue_full(&q), queue_size(&q));
    if (queue_peek(&q, &v) == 0) {
        printf("front=%d\n", v);
    }

    queue_dequeue(&q, &v);
    printf("dequeued %d\n", v);
    queue_dequeue(&q, &v);
    printf("dequeued %d\n", v);

    queue_enqueue(&q, 50); /* wraps into slot 0 */
    queue_enqueue(&q, 60); /* wraps into slot 1 */

    printf("draining: ");
    while (!queue_empty(&q)) {
        queue_dequeue(&q, &v);
        printf("%d ", v);
    }
    printf("\n");

    if (queue_dequeue(&q, &v) != 0) {
        printf("dequeue on empty queue -> status -1, no crash\n");
    }

    queue_destroy(&q);
    return 0;
}
```

```text
full? 1 size=4
front=10
dequeued 10
dequeued 20
draining: 30 40 50 60
dequeue on empty queue -> status -1, no crash
```

The interesting line is `draining: 30 40 50 60`. Physically `50` and `60` sit at indices `0` and `1`, before `30` and `40` in memory. Logically they come after, because `head` says where "first" is.

---

## Full vs Empty: Why Count Resolves the Ambiguity

With only `head` and `tail`, the two states look identical:

```text
  empty: head == tail, nothing stored     full: head == tail, capacity stored

   +---+---+---+---+                      +----+----+----+----+
   |   |   |   |   |  head = 2            | 40 | 50 | 10 | 20 |  head = 2
   +---+---+---+---+  tail = 2            +----+----+----+----+  tail = 2
       ^                                  (tail wrapped all the way around)
```

`head == tail` means "empty" in one case and "full" in the other. There are two classic fixes:

| Strategy | How it works | Cost |
| --- | --- | --- |
| Keep `count` (this part) | `count == 0` is empty, `count == capacity` is full; `head == tail` means nothing by itself | one extra field, no wasted slot |
| Waste one slot | full is `(tail + 1) % capacity == head`; a capacity-`n` block holds at most `n - 1` items | simpler indices, but one slot always unusable |
| Keep a `full` flag | a boolean updated on every operation disambiguates the equality | easy to forget to update — a bug magnet |

`count` wins for teaching code: it also gives you `queue_size` for free, and `is_empty` / `is_full` read as plain comparisons instead of modular arithmetic.

---

## Resize Preserving Order

A fixed ring is fine until it fills. Growing it means allocating a bigger block and copying elements **in logical order** — starting at `head`, wrapping once — then resetting `head` to `0` and `tail` to `count`. Copying index-for-index (`new[i] = old[i]`) is the bug: it preserves physical layout and scrambles logical order whenever the ring has wrapped.

```c
static int queue_resize(Queue *q, size_t new_capacity) {
    int *tmp;
    size_t i;

    if (q == NULL || q->data == NULL || new_capacity == 0) {
        return -1;
    }
    if (new_capacity < q->count) {
        return -1; /* cannot shrink below what is stored */
    }
    tmp = malloc(new_capacity * sizeof(int));
    if (tmp == NULL) {
        return -1; /* old block untouched */
    }
    for (i = 0; i < q->count; i++) {
        tmp[i] = q->data[(q->head + i) % q->capacity];
    }
    free(q->data);
    q->data = tmp;
    q->head = 0;
    q->tail = q->count % new_capacity;
    q->capacity = new_capacity;
    return 0;
}
```

The key expression is `(q->head + i) % q->capacity`: logical position `i` lives at that physical index. After the copy the queue is unwrapped — `head` at `0`, `tail` right after the last element — which is exactly the state a fresh queue with `count` enqueues would be in.

```text
  before: capacity 4, head = 3, logical order 10 20 30

   index:    0     1     2     3
           +-----+-----+-----+-----+
           | 30  |     |     | 10  |  head = 3, tail = 1, count = 3
           +-----+-----+-----+-----+
                       20 is at index... wait, index 4 does not exist.
```

A cleaner picture with capacity 5, `head = 3`, `count = 3`:

```text
  before resize (capacity 5):        after resize (capacity 10):

   0     1     2     3     4           0     1     2     3 ...
  +-----+-----+-----+-----+-----+    +-----+-----+-----+-----+--...
  | 30  |     |     | 10  | 20  |    | 10  | 20  | 30  |     |
  +-----+-----+-----+-----+-----+    +-----+-----+-----+-----+--...
     ^                    ^             ^           ^
     tail = 1             head = 3      head = 0    tail = 3
```

Always assign through a temporary (`tmp`) and only `free` the old block after the copy succeeds. Writing `q->data = realloc(...)` directly leaks the old block on failure, the same lesson as the dynamic memory chapter.

A doubling helper used by auto-growing queues:

```c
static int queue_grow(Queue *q) {
    size_t doubled;

    if (q == NULL) {
        return -1;
    }
    doubled = (q->capacity == 0) ? 4 : q->capacity * 2;
    return queue_resize(q, doubled);
}
```

---

## Common Bugs

**Bug 1: forgetting the modulo.** `q->tail++` without `% capacity` walks off the end after the first wrap. It passes every test that never fills the buffer, then corrupts the heap under ASan:

```c
q->data[q->tail] = value;   /* BUG when tail == capacity */
q->tail++;                  /* should be (q->tail + 1) % q->capacity */
```

```text
==1234==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x...
WRITE of size 4 at 0x... thread T0
    #0 in queue_enqueue queue.c:42
    #1 in main queue.c:110

0x... is located 0 bytes to the right of 16-byte region [0x...,0x...)
allocated by thread T0 here:
    #0 in malloc
    #1 in queue_init queue.c:20
SUMMARY: AddressSanitizer: heap-buffer-overflow queue.c:42 in queue_enqueue
```

**Bug 2: using `head == tail` as the empty test.** Half the states are misclassified: a full queue reads as empty and the next enqueue overwrites the front element. Keep `count` and test `count == 0`.

**Bug 3: resize that copies physical order.** `tmp[i] = q->data[i]` is correct only when `head == 0`. The moment the ring has wrapped, the oldest element is not at index `0` and the "resized" queue comes out rotated. Always copy from `(head + i) % capacity`.

| Symptom | Likely cause |
| --- | --- |
| first dequeue correct, later ones return stale values | `head` advanced without modulo, or advanced twice |
| queue reports empty right after filling it | `head == tail` used for both states |
| order scrambles after a grow | physical copy instead of logical copy |
| ASan heap-buffer-overflow inside enqueue | missing `% capacity` on `tail` |
| `tail` equals `capacity` after several operations | same: modulo applied on read but not on write |

---

## Summary

- A queue is **FIFO**: enqueue at the back, dequeue from the front.
- Shifting an array on every dequeue is `O(n)` per operation. Move the indices, not the elements.
- A circular buffer keeps `head`, `tail`, `count`, and `capacity`; both ends advance with `% capacity`.
- `count` distinguishes full (`count == capacity`) from empty (`count == 0`); `head == tail` alone cannot.
- `enqueue` writes at `tail`, `dequeue` reads from `head`, `peek` reads from `head` through a `const` pointer.
- Overflow and underflow return a status (`-1`). They never abort on user input.
- Growing the ring copies in **logical** order from `(head + i) % capacity`, then resets `head` to `0`.
- `realloc`-style growth always goes through a temporary pointer so failure leaves the old block intact.

Next part: the same interface on linked nodes, and the double-ended generalization.

---

## Checklist

- I can state the FIFO contract in one sentence.
- I can explain why dequeue-by-shifting is `O(n)`.
- I can draw a wrapped ring and label `head`, `tail`, and logical order.
- I use `(index + 1) % capacity` on every advance of `head` and `tail`.
- I test emptiness with `count == 0` and fullness with `count == capacity`.
- `peek` and `size` take `const Queue *`; mutating functions return a status.
- My resize copies `(head + i) % capacity` into `tmp[i]` and resets `head`/`tail`.
- `malloc` results are checked; `realloc` growth uses a temporary pointer.
- The program compiles with `-std=c11 -Wall -Wextra -pedantic` plus ASan/UBSan with zero warnings.

---

#### [Previous](../06-stacks/exercises.md) | [Next](./02-part.md)
