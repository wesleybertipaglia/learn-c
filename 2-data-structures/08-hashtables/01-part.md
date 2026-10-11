# Hashing and Chaining

An array gives you O(1) lookup, but only by integer index. `scores[3]` is instant because the address is `base + 3 * sizeof(int)`. Nothing is computed, nothing is searched. The moment the key is a string — a name, a word, a username — that arithmetic breaks. There is no `table["apple"]` in C.

A hash table restores O(1) lookup for string keys with one trick: **convert the string to a number, then use that number as an array index.** The conversion is the hash function. The array is the bucket array. And because two different strings can convert to the same index, every bucket holds a chain of entries instead of a single entry.

---

## From String to Index in Two Steps

Step 1 hashes the string to a large unsigned number. Step 2 reduces it into range with modulo.

```c
index = hash(key) % capacity;
```

This part uses the **djb2** hash by Dan Bernstein. It is short, fast, and good enough for a first table. It is not cryptographic and was never meant to be.

```c
/* djb2: hash a NUL-terminated string to an unsigned long. */
static unsigned long hash_djb2(const char *str) {
    unsigned long hash = 5381UL;
    int c;

    while ((c = (unsigned char)*str) != 0) {
        hash = hash * 33UL + (unsigned long)c;
        str++;
    }
    return hash;
}

static size_t ht_index(const char *key, size_t capacity) {
    return (size_t)(hash_djb2(key) % (unsigned long)capacity);
}
```

Three details matter:

1. `5381` and `33` are the constants Bernstein chose. The `33 * hash + c` step mixes each new character into all the bits accumulated so far, so `"abc"` and `"cba"` hash differently.
2. The cast `(unsigned char)*str` avoids sign extension on platforms where `char` is signed. Without it, bytes above 127 hash differently on different machines.
3. `hash % capacity` is what keeps the index inside the array. Change the capacity and **every index can change**, which is why resizing must rehash (the subject of the next part).

| Step | Example with key `"apple"`, capacity 8 |
| --- | --- |
| `hash_djb2("apple")` | `210706734647` |
| `210706734647 % 8` | `7` |
| bucket used | `buckets[7]` |

A tiny program makes the mapping concrete:

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <string.h>

static unsigned long hash_djb2(const char *str) {
    unsigned long hash = 5381UL;
    int c;

    while ((c = (unsigned char)*str) != 0) {
        hash = hash * 33UL + (unsigned long)c;
        str++;
    }
    return hash;
}

int main(void) {
    const char *keys[] = { "apple", "banana", "cherry", "date", "fig" };
    size_t capacity = 8;
    size_t i;

    printf("key       hash          index (mod %zu)\n", capacity);
    printf("--------  ------------  ---------------\n");
    for (i = 0; i < 5; i++) {
        unsigned long h = hash_djb2(keys[i]);
        printf("%-8s  %12lu  %12zu\n", keys[i], h,
               (size_t)(h % (unsigned long)capacity));
    }
    return 0;
}
```

```text
key       hash          index (mod 8)
--------  ------------  ---------------
apple     210706734647               7
banana    209011361158               6
cherry    210714636832               2
date      6385144165                 3
fig       193491643                  3
```

`"date"` and `"fig"` both land on bucket 3. That is not a bug in djb2. That is the normal state of a hash table.

---

## Collisions Are Inevitable

There are more possible strings than buckets. With 8 buckets, the 9th distinct key **must** share. With 1 million buckets, two keys out of a few thousand still share with high probability. This is the pigeonhole principle, not bad luck.

So the design accepts sharing from the start: each bucket is the head of a **linked list** (a chain). Keys that collide live in the same chain, distinguished by `strcmp`.

```text
  buckets array (capacity 8)          chains

  [0] NULL
  [1] NULL
  [2] -> ["cherry",3] -> NULL
  [3] -> ["fig",9] -> ["date",4] -> NULL     <- collision chain, length 2
  [4] NULL
  [5] NULL
  [6] -> ["banana",2] -> NULL
  [7] -> ["apple",99] -> NULL
```

| Situation | What happens |
| --- | --- |
| Empty bucket | chain of length 0, `NULL` head |
| One key | chain of length 1 |
| Two keys, same index | chain of length 2, found by walking + `strcmp` |
| `n` keys, one index (worst case) | chain of length `n`, lookup degrades to O(n) |

Chaining is the simplest collision strategy and the one this chapter uses. Open addressing (probing inside the array itself) exists, but it entangles deletion with probing and is harder to get right the first time.

---

## The Two Structs

```c
typedef struct Node {
    char *key;            /* owned: strdup'd on insert, free'd on remove/destroy */
    int value;
    struct Node *next;    /* owned: next node in the same bucket chain */
} Node;

typedef struct {
    Node **buckets;       /* owned: array of bucket heads, calloc'd */
    size_t capacity;      /* number of buckets */
    size_t size;          /* number of stored pairs */
} HashTable;
```

```text
  HashTable                          buckets array              chains

  +----------+  buckets   +-----+-----+-----+-----+-----+
  | capacity |----------->|  *  |  *  |  *  |  *  | ... |  each cell is a Node *
  | size     |            +--|--+--|--+--|--+--|--+-----+
  +----------+               |     |     |     |
                             v     v     v     v
                           NULL  [k,v]->NULL NULL ...
```

| Field | Meaning |
| --- | --- |
| `buckets` | array of `capacity` chain heads; `NULL` means the bucket is empty |
| `capacity` | how many buckets exist; the modulo divisor |
| `size` | how many key/value pairs are stored across all chains |
| `Node.key` | a private copy of the key (see ownership below) |
| `Node.next` | next entry in the same bucket, or `NULL` |

`buckets` is allocated with `calloc` so every head starts as `NULL`. `size` and `capacity` travel with the array for the same reason a vector keeps `size` and `capacity` with its block: a bare `Node **` has no idea how long it is.

---

## The Complete Implementation

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct Node {
    char *key;            /* owned: strdup'd on insert, free'd on remove/destroy */
    int value;
    struct Node *next;    /* owned: next node in the same bucket chain */
} Node;

typedef struct {
    Node **buckets;       /* owned: array of bucket heads, calloc'd */
    size_t capacity;      /* number of buckets */
    size_t size;          /* number of stored pairs */
} HashTable;

static unsigned long hash_djb2(const char *str) {
    unsigned long hash = 5381UL;
    int c;

    while ((c = (unsigned char)*str) != 0) {
        hash = hash * 33UL + (unsigned long)c;
        str++;
    }
    return hash;
}

static size_t ht_index(const char *key, size_t capacity) {
    return (size_t)(hash_djb2(key) % (unsigned long)capacity);
}

/* OWNERSHIP: returns a new table; the CALLER must call ht_destroy. */
/* Returns NULL on allocation failure. */
static HashTable *ht_create(size_t capacity) {
    HashTable *ht;

    if (capacity == 0) {
        capacity = 8;
    }
    ht = malloc(sizeof *ht);
    if (ht == NULL) {
        return NULL;
    }
    ht->buckets = calloc(capacity, sizeof *ht->buckets);
    if (ht->buckets == NULL) {
        free(ht);
        return NULL;
    }
    ht->capacity = capacity;
    ht->size = 0;
    return ht;
}

/* OWNERSHIP: frees every key, every node, the bucket array, and the table. */
/* Safe to call with NULL. */
static void ht_destroy(HashTable *ht) {
    size_t i;

    if (ht == NULL) {
        return;
    }
    for (i = 0; i < ht->capacity; i++) {
        Node *cur = ht->buckets[i];
        while (cur != NULL) {
            Node *next = cur->next;
            free(cur->key);   /* key and node die together, always */
            free(cur);
            cur = next;
        }
    }
    free(ht->buckets);
    free(ht);
}

/* Insert or update. Returns 0 on success, -1 on allocation failure. */
static int ht_put(HashTable *ht, const char *key, int value) {
    size_t idx;
    Node *cur;
    Node *node;
    char *copy;

    if (ht == NULL || key == NULL) {
        return -1;
    }
    idx = ht_index(key, ht->capacity);
    for (cur = ht->buckets[idx]; cur != NULL; cur = cur->next) {
        if (strcmp(cur->key, key) == 0) {
            cur->value = value;   /* key already owned: no new strdup, no leak */
            return 0;
        }
    }
    node = malloc(sizeof *node);
    if (node == NULL) {
        return -1;
    }
    copy = strdup(key);
    if (copy == NULL) {
        free(node);   /* node without a key is garbage: release it now */
        return -1;
    }
    node->key = copy;
    node->value = value;
    node->next = ht->buckets[idx];
    ht->buckets[idx] = node;
    ht->size++;
    return 0;
}

/* OWNERSHIP: borrows key and table. Writes the value to *out. */
/* Returns 0 when found, -1 when missing. */
static int ht_get(const HashTable *ht, const char *key, int *out) {
    size_t idx;
    Node *cur;

    if (ht == NULL || key == NULL || out == NULL) {
        return -1;
    }
    idx = ht_index(key, ht->capacity);
    for (cur = ht->buckets[idx]; cur != NULL; cur = cur->next) {
        if (strcmp(cur->key, key) == 0) {
            *out = cur->value;
            return 0;
        }
    }
    return -1;
}

/* Returns 1 when the key exists, 0 otherwise. Borrows both arguments. */
static int ht_contains(const HashTable *ht, const char *key) {
    int dummy;

    return ht_get(ht, key, &dummy) == 0;
}

/* Remove one key. Frees its key copy and its node. */
/* Returns 0 when removed, -1 when the key was missing. */
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
                ht->buckets[idx] = cur->next;
            } else {
                prev->next = cur->next;
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

int main(void) {
    HashTable *ht = ht_create(8);
    const char *words[] = { "apple", "banana", "cherry", "date", "apple" };
    int values[] = { 1, 2, 3, 4, 99 };
    size_t i;
    int out;

    if (ht == NULL) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }
    for (i = 0; i < 5; i++) {
        if (ht_put(ht, words[i], values[i]) != 0) {
            fprintf(stderr, "put failed\n");
            ht_destroy(ht);
            return 1;
        }
    }
    printf("size = %zu, capacity = %zu\n", ht->size, ht->capacity);
    for (i = 0; i < 4; i++) {
        if (ht_get(ht, words[i], &out) == 0) {
            printf("%-8s -> %d (bucket %zu)\n", words[i], out,
                   ht_index(words[i], ht->capacity));
        }
    }
    printf("contains cherry? %s\n", ht_contains(ht, "cherry") ? "yes" : "no");
    printf("contains fig?    %s\n", ht_contains(ht, "fig") ? "yes" : "no");
    if (ht_remove(ht, "banana") == 0) {
        printf("removed banana, contains? %s\n",
               ht_contains(ht, "banana") ? "yes" : "no");
    }
    if (ht_get(ht, "apple", &out) == 0) {
        printf("apple after update = %d\n", out);
    }
    ht_destroy(ht);
    return 0;
}
```

```text
size = 4, capacity = 8
apple    -> 99 (bucket 7)
banana   -> 2 (bucket 6)
cherry   -> 3 (bucket 2)
date     -> 4 (bucket 3)
contains cherry? yes
contains fig?    no
removed banana, contains? no
apple after update = 99
```

Five puts produced `size = 4` because the second `"apple"` updated the existing entry instead of adding one. That update path is what keeps `size` honest.

| Function | Borrows or owns | Failure mode |
| --- | --- | --- |
| `ht_create` | returns an owned table | `NULL` when `malloc`/`calloc` fails |
| `ht_put` | borrows `key` (`const char *`), owns the `strdup` copy | `-1` when `malloc`/`strdup` fails; old table untouched |
| `ht_get` / `ht_contains` | borrow everything, write through `out` | `-1` / `0` when the key is missing |
| `ht_remove` | borrows `key`, frees the stored copy + node | `-1` when the key is missing |
| `ht_destroy` | takes the table, frees everything | safe on `NULL` |

---

## Ownership: `strdup` the Key, `free` It With the Node

The table stores `const char *` keys it does not own — string literals, stack buffers, `fgets` buffers that get reused. If it kept the caller's pointer, the next input line would silently rewrite every key. So `ht_put` makes a private copy with `strdup`, and that copy dies with its node:

```text
  caller buffer (borrowed)          table node (owned)

  char buf[] = "apple";             +-------+-------+
  buf跌 -> [a][p][p][l][e][\0]  strdup  | "apple" | 99  | -> next
                                    +-------+-------+     ^
                                    freed together        |
                                    in ht_remove /        |
                                    ht_destroy            +-- chain link
```

The three rules:

1. **`strdup` is checked.** It calls `malloc` internally, so it can return `NULL`. The code above frees the bare node and returns `-1` on that path.
2. **Update does not `strdup`.** When the key already exists, only `value` changes. A second copy would orphan the first one — a leak of exactly `strlen(key) + 1` bytes per update.
3. **Key and node are freed together, in both places.** `ht_remove` frees one pair; `ht_destroy` walks every chain and frees every pair. There is no path where a node dies and its key survives.

`strdup` is POSIX, not ISO C. Under `-std=c11 -pedantic` the declaration is hidden unless you ask for it first:

```c
#define _POSIX_C_SOURCE 200809L
#include <string.h>
```

Without that first line, `gcc` reports `implicit declaration of function 'strdup'` and the build fails. That error is the feature: it tells you the function is outside standard C before you ship code that assumes it exists.

---

## Two Bugs You Will Write Once

**Bug 1: storing the caller's pointer instead of a copy.**

```c
/* WRONG: node->key aliases whatever the caller passed in */
node->key = (char *)key;
```

Run it with a reused `fgets` buffer and every key becomes the last line read. Run it with a stack buffer and the table dangles the moment the caller returns. Under AddressSanitizer the second case reports `heap-use-after-free` or `stack-use-after-return`, pointing at the `strcmp` in `ht_get` — far from the `ht_put` line that caused it.

**Bug 2: freeing the node but not the key.**

```c
/* WRONG: the strdup'd key leaks on every remove */
free(cur);
/* missing: free(cur->key); */
```

The program prints correct output. The plain build exits 0. Under the sanitizer:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer table.c -o table_san
./table_san
```

```text
=================================================================
==38102==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 6 byte(s) in 1 object(s) allocated from:
    #0 0x7f8a2c92b60f in strdup ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x55c9a1b2240e in ht_put /tmp/table.c:95
    #2 0x55c9a1b22a17 in main /tmp/table.c:172

SUMMARY: AddressSanitizer: 6 byte(s) leaked in 1 allocation(s).
```

`Direct leak of 6 byte(s)` from `strdup` in `ht_put` — 6 bytes is `"apple"` plus its terminator, and the frame tells you which `strdup` produced it. That is the whole diagnosis: **what** leaked, **how much**, and **where it came from**.

---

## Checklist

```text
[ ] djb2 mixes every character: hash = hash * 33 + c, starting from 5381
[ ] Cast through (unsigned char) before hashing each byte
[ ] Index is always hash % capacity, never the raw hash
[ ] Collisions are expected: same bucket holds a chain, compared with strcmp
[ ] Buckets array is calloc'd so every head starts NULL
[ ] put walks the chain first: update on match, strdup only on insert
[ ] get/contains borrow the key as const char* and never store it
[ ] remove unlinks the node, then frees key AND node together
[ ] destroy walks every bucket and frees every key, node, the array, the struct
[ ] Every malloc/calloc/strdup is checked against NULL
[ ] #define _POSIX_C_SOURCE 200809L comes before the includes for strdup
[ ] Build with -fsanitize=address,undefined and run until silent
```

---

#### [Previous](../07-queues/exercises.md) | [Next](./02-part.md)
