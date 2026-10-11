# Resizing and Performance

Part 1 built a table with a fixed capacity. That table has a slow leak of a different kind: not memory, but speed. Every insert makes the chains longer, and longer chains mean slower lookups. This part adds the mechanism that keeps chains short — resizing with a full rehash — and then measures what it buys.

---

## Load Factor: the One Number That Predicts Speed

```c
load factor = (double)size / (double)capacity;
```

Size counts pairs. Capacity counts buckets. Their ratio is the **average chain length**, and the average chain length is the cost of a lookup.

```text
  size = 3, capacity = 8, load = 0.375        size = 7, capacity = 8, load = 0.875

  [0] NULL                                     [0] -> [k] -> NULL
  [1] -> [k] -> NULL                           [1] -> [k] -> [k] -> NULL
  [2] NULL                                     [2] -> [k] -> NULL
  [3] -> [k] -> NULL                           [3] -> [k] -> NULL
  [4] NULL                                     [4] -> [k] -> NULL
  [5] NULL                                     [5] -> NULL
  [6] -> [k] -> NULL                           [6] -> [k] -> NULL
  [7] NULL                                     [7] -> NULL
  mostly empty: ~1 strcmp per lookup           crowded: some lookups walk 2-3 nodes
```

| Load factor | What it means | What to do |
| --- | --- | --- |
| `0.0 – 0.5` | roomy; chains are length 0 or 1 | nothing |
| `0.5 – 0.75` | healthy; occasional length-2 chain | watch it |
| `> 0.75` | crowded; chains grow, O(1) erodes | **resize now** |
| `> 1.0` | more pairs than buckets; some chain must hold 2+ | overdue; lookups visibly slower |

`0.75` is the conventional threshold (Java's `HashMap` uses it). It is not magic — it is the point where the memory saved by staying small stops paying for the time lost walking chains.

```c
static double ht_load_factor(const HashTable *ht) {
    return (double)ht->size / (double)ht->capacity;
}
```

Both casts matter. `size / capacity` without them is integer division: `7 / 8` is `0`, and the table would never resize.

---

## Resizing: Double the Buckets, Rehash Every Key

Resizing allocates a bigger bucket array and moves every node into it. The critical point: **the index must be recomputed with the new capacity.** `hash % 8` and `hash % 16` give different buckets for most keys, so copying a chain from old bucket `i` to new bucket `i` silently loses entries — they sit in a bucket `ht_get` will never search.

```text
  WRONG: copy bucket i -> bucket i         RIGHT: rehash each key with new capacity

  old cap 8, key hashes to 27              old cap 8 -> new cap 16
  27 % 8 = 3                               27 % 16 = 11  (moved!)
  [3] -> [k]                               [3] -> NULL     [11] -> [k]

  copying [3] -> [3] keeps it findable     rehashing puts it where ht_get looks:
  ONLY if 27 % 16 were also 3.             idx = hash(key) % new_capacity
  Usually it is not.                       for EVERY node.
```

```c
/* Grow (or shrink) the bucket array and rehash every live node into it. */
/* Returns 0 on success, -1 on allocation failure (old table untouched). */
static int ht_resize(HashTable *ht, size_t new_capacity) {
    Node **new_buckets;
    size_t i;

    if (ht == NULL || new_capacity == 0) {
        return -1;
    }
    new_buckets = calloc(new_capacity, sizeof *new_buckets);
    if (new_buckets == NULL) {
        return -1;   /* old buckets still valid: caller keeps using them */
    }
    for (i = 0; i < ht->capacity; i++) {
        Node *cur = ht->buckets[i];
        while (cur != NULL) {
            Node *next = cur->next;   /* save before relinking */
            size_t idx;

            /* REHASH: the modulo divisor changed, so the index changes too. */
            idx = (size_t)(hash_djb2(cur->key) % (unsigned long)new_capacity);
            cur->next = new_buckets[idx];
            new_buckets[idx] = cur;
            cur = next;
        }
    }
    free(ht->buckets);   /* the array moves; the nodes and keys are reused */
    ht->buckets = new_buckets;
    ht->capacity = new_capacity;
    return 0;
}
```

No `malloc` per node, no second `strdup` per key. The nodes themselves move — only the array is new, and only the old array is freed. That is why `ht_resize` cannot leak keys: it never duplicates them.

The call site goes at the end of `ht_put`, after `size` has been incremented:

```c
    node->key = copy;
    node->value = value;
    node->next = ht->buckets[idx];
    ht->buckets[idx] = node;
    ht->size++;

    if (ht_load_factor(ht) > 0.75) {
        if (ht_resize(ht, ht->capacity * 2) != 0) {
            return -1;   /* pair is stored; only the growth failed */
        }
        printf("[resize] load factor exceeded 0.75 -> capacity now %zu\n",
               ht->capacity);
    }
    return 0;
```

Note the contract on growth failure: the pair is already in the table, so `ht_put` returns `-1` but the table is still valid and still owns everything. The caller reports the error; nothing needs unwinding.

This program inserts 10 keys into a capacity-4 table and prints the load after each put:

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* ... Node, HashTable, hash_djb2, ht_index, ht_create, ht_destroy, */
/* ... ht_load_factor, ht_resize, ht_put, ht_get as defined above ... */

int main(void) {
    HashTable *ht = ht_create(4);
    char name[32];
    int i;
    int out;

    if (ht == NULL) {
        return 1;
    }
    for (i = 0; i < 10; i++) {
        snprintf(name, sizeof name, "key%d", i);
        if (ht_put(ht, name, i * 10) != 0) {
            fprintf(stderr, "put failed\n");
            ht_destroy(ht);
            return 1;
        }
        printf("put %-6s size=%2zu cap=%2zu load=%.2f\n",
               name, ht->size, ht->capacity, ht_load_factor(ht));
    }
    for (i = 0; i < 10; i++) {
        snprintf(name, sizeof name, "key%d", i);
        if (ht_get(ht, name, &out) != 0 || out != i * 10) {
            printf("MISMATCH for %s\n", name);
            ht_destroy(ht);
            return 1;
        }
    }
    printf("all 10 keys survive rehash\n");
    ht_destroy(ht);
    return 0;
}
```

```text
put key0   size= 1 cap= 4 load=0.25
put key1   size= 2 cap= 4 load=0.50
put key2   size= 3 cap= 4 load=0.75
[resize] load factor exceeded 0.75 -> capacity now 8
put key3   size= 4 cap= 8 load=0.50
put key4   size= 5 cap= 8 load=0.62
put key5   size= 6 cap= 8 load=0.75
[resize] load factor exceeded 0.75 -> capacity now 16
put key6   size= 7 cap=16 load=0.44
put key7   size= 8 cap=16 load=0.50
put key8   size= 9 cap=16 load=0.56
put key9   size=10 cap=16 load=0.62
all 10 keys survive rehash
```

Two doublings (4 to 8 to 16) absorbed 10 inserts, and the final verification loop proves every key is still retrievable after both rehashes. The `all 10 keys survive rehash` line is the test: without recomputing `hash % new_capacity`, several of those `ht_get` calls would print `MISMATCH`.

The resize bug, shown so you recognize it:

```c
/* WRONG: copies chains without rehashing. Keys land in buckets that */
/* ht_get (which uses hash % NEW capacity) will never visit. */
for (i = 0; i < old_capacity && i < new_capacity; i++) {
    new_buckets[i] = ht->buckets[i];
}
```

It compiles, it keeps `size` correct, and it loses keys. The symptom is `ht_get` returning `-1` for a key `ht_put` just stored — but only for the keys whose index changed, which makes it look intermittent.

---

## Performance: O(1) Average, O(n) Worst Case

With a good hash and resizing at 0.75, the average chain length stays under 1, so put/get/remove each do one hash, one modulo, and about one `strcmp`. That is **O(1) average**.

The worst case is all `n` keys in one bucket — a hostile hash, or an attacker feeding colliding strings. Then every operation walks the full chain: **O(n)**.

| Operation | Average (load <= 0.75, decent hash) | Worst case (all keys collide) |
| --- | --- | --- |
| `put` (new key) | O(1): hash + insert at head | O(n): full chain scan finds no match |
| `put` (update) | O(1): match near the head | O(n): match at the tail |
| `get` / `contains` | O(1): one `strcmp` | O(n): `n` `strcmp` calls |
| `remove` | O(1): unlink near the head | O(n): walk to the tail, then unlink |
| `resize` (rehash all) | O(n): one pass over every node | O(n): same pass |

Resize itself costs O(n), but it happens rarely: after a doubling from `m` to `2m`, the next `m` inserts need no resize at all. Spread over those inserts, the per-insert cost is O(1) amortized — the same doubling argument as the dynamic array in chapter 4.

---

## Removal Done Right: Free Key and Node Together

Removal has three steps and the order is fixed: **unlink, then free key, then free node.**

```text
  bucket[3] before removing "date"

  [3] -> ["fig",9] -> ["date",4] -> NULL
          prev         cur

  1. unlink:  prev->next = cur->next
  [3] -> ["fig",9] -> NULL        ["date",4] -> NULL   (detached)

  2. free(cur->key)   3. free(cur)

  [3] -> ["fig",9] -> NULL        (gone; size--)
```

```c
static int ht_remove(HashTable *ht, const char *key) {
    size_t idx;
    Node *cur;
    Node *prev = NULL;

    if (ht == NULL || key == NULL) {
        return -1;
    }
    idx = ht_index(key, ht->capacity);
    for (cur = ht->buckets[idx]; cur != NULL; cur = cur->next) {
        if (strcmp(cur->key, key) == 0) {
            if (prev == NULL) {
                ht->buckets[idx] = cur->next;   /* head of chain */
            } else {
                prev->next = cur->next;         /* middle or tail */
            }
            free(cur->key);   /* key and node die together, always */
            free(cur);
            ht->size--;
            return 0;
        }
        prev = cur;
    }
    return -1;
}
```

Two edge cases hide here. Removing the head must update `buckets[idx]`, not `prev->next` (there is no `prev` yet). And `prev` advances only on non-match — advancing it before the `strcmp` skips the head on the next iteration and corrupts the chain on removal.

Freeing in the wrong order is the classic double-fault: `free(cur)` first, then `free(cur->key)` reads `cur->key` through a dangling pointer. ASan reports it as `heap-use-after-free` inside `ht_remove`, which at least points at the right function.

---

## A Concrete Rehash Walkthrough

Take three keys in a capacity-4 table and double it to 8. The hashes stay the same; the modulo divisor changes, so the buckets change:

```text
  hashes (djb2, fixed forever)     capacity 4 (hash % 4)      capacity 8 (hash % 8)

  hash("key0") = ...40             40 % 4 = 0                  40 % 8 = 0  (stayed)
  hash("key1") = ...41             41 % 4 = 1                  41 % 8 = 1  (stayed)
  hash("key2") = ...42             42 % 4 = 2                  42 % 8 = 2  (stayed)
  hash("key3") = ...43             43 % 4 = 3                  43 % 8 = 3  (stayed)
  hash("key4") = ...44             44 % 4 = 0  <- shares      44 % 8 = 4  (MOVED)
```

The first four keys happen to stay put, which is exactly why the copy-without-rehash bug survives small tests. The fifth key (`"key4"`) is the one that exposes it: bucket 0 under capacity 4, bucket 4 under capacity 8. Copying bucket 0 to bucket 0 leaves `"key4"` stranded where `ht_get` (computing `hash % 8 = 4`) never looks.

The `ht_resize` loop handles this by never trusting the old position:

```c
Node *cur = ht->buckets[i];
while (cur != NULL) {
    Node *next = cur->next;   /* 1. save the rest of the old chain */
    size_t idx = (size_t)(hash_djb2(cur->key) % (unsigned long)new_capacity);
    cur->next = new_buckets[idx];  /* 2. prepend into the NEW bucket */
    new_buckets[idx] = cur;
    cur = next;               /* 3. continue with the saved pointer */
}
```

| Line | Why it is written that way |
| --- | --- |
| `Node *next = cur->next` | relinking overwrites `cur->next`; without the save, the rest of the old chain is lost |
| `hash_djb2(cur->key) % new_capacity` | reuses the stored key copy — no `strdup`, no caller buffer needed |
| `cur->next = new_buckets[idx]` | prepend is O(1); order inside a chain is meaningless, so no tail pointer is needed |
| `free(ht->buckets)` after the loop | frees the old array only; nodes and keys were moved, not copied |

Build the full program and prove it is leak-free:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer table.c -o table_san
./table_san
echo "exit = $?"
```

```text
put key0   size= 1 cap= 4 load=0.25
...
[resize] load factor exceeded 0.75 -> capacity now 8
...
all 10 keys survive rehash
exit = 0
```

No `ERROR`, no `SUMMARY`, exit 0. Two resizes allocated two new bucket arrays and freed two old ones; ten `strdup` copies were freed by the single `ht_destroy` at the end. If any one of those frees were missing, the sanitizer would print a `Direct leak` report instead of silence.

---

## When NOT to Use a Hash Table

A hash table is the wrong tool whenever order, memory, or key count makes its strengths irrelevant:

| Need | Better choice | Why the hash loses |
| --- | --- | --- |
| Sorted output (`print all keys in order`) | sorted array, or a tree | buckets are in hash order, which is meaningless; you would copy all keys out and sort them anyway |
| Range queries (`all scores between 70 and 90`) | sorted array + binary search | a hash cannot find neighbors; every query scans all `n` keys |
| Fewer than ~20 keys | plain array, linear search | 20 `strcmp` calls beat one hash + modulo + `malloc` per key + bucket array overhead |
| Deterministic iteration order | array or linked list | bucket order changes with capacity, insertion order, and rehash; tests that print a table will flake |
| No allocation allowed (interrupt handler, tiny embedded) | fixed array | every `put` calls `strdup`/`malloc`; failure paths and leaks come with the territory |

The rule of thumb: **fewer than 20 keys, or any requirement with the word "sorted" or "range" in it — reach for the array first.** Hash tables earn their complexity past that point, where O(1) average beats O(n) scan by enough to pay for the allocator traffic.

---

## Leak Checklist With the Sanitizer

Every program in this chapter must pass this sequence. The build command for the rest of the module is:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer table.c -o table_san
./table_san
```

A clean run prints your output and nothing else. A leak prints a report naming the allocating function:

```text
=================================================================
==38102==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 16 byte(s) in 1 object(s) allocated from:
    #0 0x7f8a2c92a6cf in realloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:81
    #1 0x55c9a1b22671 in ht_resize /tmp/table.c:88

SUMMARY: AddressSanitizer: 16 byte(s) leaked in 1 allocation(s).
```

`ht_resize /tmp/table.c:88` with `realloc` in the trace means the old bucket array was replaced without freeing it — the `free(ht->buckets)` line is missing or unreachable. Leaks in `strdup` mean a key survived its node; leaks in `malloc` with `ht_put` in the trace mean a node survived its bucket.

```text
[ ] One ht_destroy at the end frees keys, nodes, buckets, and struct
[ ] ht_remove frees key AND node, decrements size, unlinks head correctly
[ ] ht_resize frees the OLD bucket array and nothing else (nodes move, keys stay)
[ ] strdup failure frees the bare node before returning -1
[ ] calloc/realloc failure leaves the old table usable; caller checks -1
[ ] Update path (existing key) calls no malloc and no strdup
[ ] Load factor uses (double) casts: size / capacity alone is always 0
[ ] Rehash recomputes hash % new_capacity for EVERY node, never copies bucket i to i
[ ] Clean run under -fsanitize=address,undefined prints zero diagnostics
```

---

#### [Previous](./01-part.md) | [Next](./exercises.md)
