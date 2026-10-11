# Exercise List 8 - Hash Tables

Six exercises on hashing with chaining, plus one instruction that applies to all of them. Every program must compile with zero warnings and run clean under both sanitizers:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer main.c -o main_san
./main_san
```

Silence from the sanitizer is the passing state. A correct-looking output with a leak report underneath is a failure.

Keys you only read are always `const char *`. Keys the table keeps are always `strdup` copies that die with their node. Every `malloc`, `calloc`, and `strdup` is checked against `NULL`.

---

## Exercise 1: djb2 Plus Put and Get

Implement the hash, the index, and the two core operations. Everything else is given.

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct Node {
    char *key;            /* owned: strdup'd on insert, free'd on remove/destroy */
    int value;
    struct Node *next;
} Node;

typedef struct {
    Node **buckets;       /* owned: calloc'd array of chain heads */
    size_t capacity;
    size_t size;
} HashTable;

/* TODO: implement hash_djb2 (start 5381, step hash * 33 + c). */
/* TODO: implement ht_index as hash_djb2(key) % capacity. */
/* TODO: implement ht_put (walk, update on match, else malloc+strdup+prepend). */
/* TODO: implement ht_get (walk with strcmp, write *out, 0/-1). */
/* The two (void) casts below only silence -Wunused-parameter; delete them */
/* as soon as the parameters are used. */

static unsigned long hash_djb2(const char *str) {
    (void)str;
    return 0;
}

static HashTable *ht_create(size_t capacity) {
    HashTable *ht = malloc(sizeof *ht);
    if (ht == NULL) {
        return NULL;
    }
    if (capacity == 0) {
        capacity = 8;
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

/* OWNERSHIP: frees every key, every node, the bucket array, the struct. */
static void ht_destroy(HashTable *ht) {
    size_t i;
    if (ht == NULL) {
        return;
    }
    for (i = 0; i < ht->capacity; i++) {
        Node *cur = ht->buckets[i];
        while (cur != NULL) {
            Node *next = cur->next;
            free(cur->key);
            free(cur);
            cur = next;
        }
    }
    free(ht->buckets);
    free(ht);
}

/* OWNERSHIP: borrows key (const char*); owns the strdup copy on insert. */
static int ht_put(HashTable *ht, const char *key, int value) {
    (void)ht;
    (void)key;
    (void)value;
    return -1;
}

/* OWNERSHIP: borrows table and key; writes through out. */
static int ht_get(const HashTable *ht, const char *key, int *out) {
    (void)ht;
    (void)key;
    (void)out;
    return -1;
}

int main(void) {
    HashTable *ht = ht_create(8);
    int out;

    if (ht == NULL) {
        return 1;
    }
    ht_put(ht, "apple", 1);
    ht_put(ht, "banana", 2);
    ht_put(ht, "apple", 99);   /* update, not a second entry */
    if (ht_get(ht, "apple", &out) == 0) {
        printf("apple -> %d\n", out);
    }
    if (ht_get(ht, "fig", &out) != 0) {
        printf("fig missing\n");
    }
    printf("size = %zu\n", ht->size);
    ht_destroy(ht);
    return 0;
}
```

```text
apple -> 99
fig missing
size = 2
```

Requirements:

- `hash_djb2` starts at `5381UL`, steps `hash * 33UL + (unsigned long)c`, casts each byte through `(unsigned char)`. Write in comments why the cast exists.
- `ht_put` with an existing key updates `value` and calls neither `malloc` nor `strdup`. Write in comments what would leak if it duplicated the key.
- `ht_put` checks both `malloc` and `strdup` against `NULL`, freeing the bare node when `strdup` fails. Write in comments why the order (node first, then key) matters.
- `size` counts pairs, so two puts of `"apple"` leave `size = 2` for three puts total. Print `size` to prove it.
- Run under the sanitizer and confirm zero diagnostics. Then delete one `free` in `ht_destroy` and paste the real LeakSanitizer report.
- Extra credit: print `hash_djb2(key)` and `hash % capacity` for each key, and point at two keys sharing a bucket.

---

## Exercise 2: Count Collisions and Print Chains

Make the invisible visible: dump every bucket and count how full the table really is.

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* ... same Node, HashTable, hash_djb2, ht_create, ht_destroy, ... */
/* ... ht_put, ht_get as finished in Exercise 1 ...                */

/* TODO: implement ht_dump: print each bucket index, then its chain as */
/* [key:value] -> [key:value] -> NULL. Print "(empty)" for NULL heads. */
/* TODO: implement ht_stats: count used buckets, longest chain, and the */
/* number of keys sitting past position 1 (the collision count). */

/* OWNERSHIP: borrows the table; prints keys without keeping them. */
static void ht_dump(const HashTable *ht) {
    (void)ht;
    /* your code here */
}

int main(void) {
    HashTable *ht = ht_create(4);
    const char *keys[] = { "apple", "banana", "cherry", "date", "fig", "grape" };
    size_t i;

    if (ht == NULL) {
        return 1;
    }
    for (i = 0; i < 6; i++) {
        if (ht_put(ht, keys[i], (int)i) != 0) {
            ht_destroy(ht);
            return 1;
        }
    }
    ht_dump(ht);
    ht_destroy(ht);
    return 0;
}
```

Target output (bucket indexes depend on your capacity; the shape does not):

```text
bucket 0: (empty)
bucket 1: [grape:5] -> NULL
bucket 2: [cherry:2] -> NULL
bucket 3: [fig:4] -> [date:3] -> NULL
collisions: 1 key past position 1, longest chain 2, 3 of 4 buckets used
```

Requirements:

- `ht_dump` takes `const HashTable *` because it borrows the table. Write in comments why `const` is a promise here, not just decoration.
- Report three numbers: buckets used, longest chain, keys past position 1. Define "collision" in a comment as a key stored anywhere past the head of its chain.
- Try capacities 4, 8, and 64 with the same 6 keys and paste all three dumps. Write in comments what happened to the longest chain as capacity grew.
- Extra credit: add `size_t ht_bucket_len(const HashTable *ht, size_t idx)` with a bounds check `idx >= capacity`, and use it inside `ht_dump`.

---

## Exercise 3: Remove and Contains With Correct Free

Removal is where ownership bugs live. Implement it, then prove the memory is gone.

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* ... same Node, HashTable, hash, create, destroy, put, get ... */

/* TODO: implement ht_contains via ht_get (1 present, 0 absent). */
/* TODO: implement ht_remove: unlink, then free key, then free node. */
/* Handle the head-of-chain case (prev == NULL) separately. */

/* OWNERSHIP: borrows table and key; returns 1 present, 0 absent. */
static int ht_contains(const HashTable *ht, const char *key) {
    (void)ht;
    (void)key;
    return 0;
}

/* OWNERSHIP: borrows key; frees the stored key copy AND its node. */
/* Returns 0 removed, -1 missing. */
static int ht_remove(HashTable *ht, const char *key) {
    (void)ht;
    (void)key;
    return -1;
}

int main(void) {
    HashTable *ht = ht_create(8);

    if (ht == NULL) {
        return 1;
    }
    ht_put(ht, "apple", 1);
    ht_put(ht, "banana", 2);
    ht_put(ht, "cherry", 3);
    printf("contains banana? %s\n", ht_contains(ht, "banana") ? "yes" : "no");
    ht_remove(ht, "banana");
    printf("contains banana? %s\n", ht_contains(ht, "banana") ? "yes" : "no");
    printf("remove again: %d\n", ht_remove(ht, "banana"));
    printf("size = %zu\n", ht->size);
    ht_destroy(ht);
    return 0;
}
```

```text
contains banana? yes
contains banana? no
remove again: -1
size = 2
```

Requirements:

- `ht_remove` unlinks before freeing, and frees `cur->key` before `free(cur)`. Write in comments why freeing the node first would make `free(cur->key)` a use-after-free.
- The head case writes `ht->buckets[idx] = cur->next`; the rest write `prev->next = cur->next`. Write in comments what breaks if both cases use `prev->next`.
- Every `remove` decrements `size` exactly once, and a missing key changes nothing. Print `size` before and after to prove it.
- Run under the sanitizer: full script must be silent. Then comment out `free(cur->key)` once, re-run, and paste the real `Direct leak ... allocated from ... strdup` report.
- Extra credit: remove the head, the tail, and the middle of a 3-long chain (force one with capacity 1), printing `ht_dump` after each step.

---

## Exercise 4: Word Frequency Counter

Read a line, split it into words, count each word. This is the first program where the table earns its keep.

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* ... same Node, HashTable, hash, create, destroy, put, get ... */

/* TODO: in main, read ONE line with fgets into char buf[256]. */
/* TODO: split it with strtok(buf, " \t\n") and for each word do: */
/*   ht_get -> found: ht_put(word, count + 1); missing: ht_put(word, 1). */
/* TODO: dump every pair at the end, then ht_destroy. */

int main(void) {
    /* OWNERSHIP: buf is a stack buffer, reused every call; the table must */
    /* never store pointers into it -- ht_put strdup's each word. */
    char buf[256];

    printf("enter words: ");
    fflush(stdout);
    if (fgets(buf, sizeof buf, stdin) == NULL) {
        return 1;
    }
    /* your code here: strtok loop + print + destroy */
    return 0;
}
```

```text
enter words: apple banana apple cherry banana apple
apple: 3
banana: 2
cherry: 1
```

Requirements:

- Input uses `fgets(buf, sizeof buf, stdin)` and splitting uses `strtok` with `" \t\n"`. Write in comments why `scanf("%s")` would be wrong here (spaces) and what `strtok` does to the buffer (writes `'\0'` separators into it).
- Keys passed to `ht_put` are `const char *` views into `buf`. Write in comments why the table must `strdup` them: `buf` is reused, so a stored pointer would alias the next input.
- Increment path: `ht_get` first, then `ht_put(word, count + 1)`. A missing word starts at 1. Show both paths running in the output above.
- Handle empty input (blank line prints nothing, still destroys cleanly) and overlong input (note in comments that `fgets` truncates at `sizeof buf - 1`).
- Extra credit: print the pairs sorted by count descending (copy pointers out, `qsort`, print, free the pointer array only — not the keys).

---

## Exercise 5: Resize and Rehash Past Load 0.75

Grow the table under load and prove no key is lost. The failing version (copy bucket `i` to bucket `i` without rehashing) must be tried and its symptom pasted.

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* ... same Node, HashTable, hash, create, destroy, put-with-resize, get ... */

/* TODO: implement ht_load_factor as (double)size / (double)capacity. */
/* TODO: implement ht_resize: calloc new buckets, rehash EVERY node with */
/*   hash % new_capacity, free only the old ARRAY (nodes and keys move). */
/* TODO: call it from ht_put when load factor > 0.75, doubling capacity. */

static double ht_load_factor(const HashTable *ht) {
    (void)ht;
    return 0.0;
}

static int ht_resize(HashTable *ht, size_t new_capacity) {
    (void)ht;
    (void)new_capacity;
    return -1;
}

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

Requirements:

- `ht_load_factor` casts both operands to `double`. Write in comments what `7 / 8` evaluates to without the casts and why the table would then never resize.
- `ht_resize` recomputes `hash_djb2(cur->key) % new_capacity` per node and saves `cur->next` before relinking. Write in comments why each half is required.
- First implement the wrong version (copy `new_buckets[i] = ht->buckets[i]`), run the 10-key test, and paste which keys print `MISMATCH`. Then fix it with real rehashing and show `all 10 keys survive rehash`.
- On `calloc` failure `ht_resize` returns `-1` and the old table stays usable. Write in comments why the old array must not be freed on that path.
- Extra credit: count resizes over 1,000 inserts from capacity 4 and confirm capacity never exceeds roughly `4/3 * size` (the 0.75 threshold guarantees `capacity > size / 0.75`).

---

## Exercise 6: Phonebook With Struct Values

Same table, richer value: map a name to a phone string. Both strings are owned, both are freed, and one `destroy` frees everything.

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct PhoneEntry {
    char *name;               /* owned: strdup'd, free'd in pb_destroy/remove */
    char *phone;              /* owned: strdup'd, free'd in pb_destroy/remove */
    struct PhoneEntry *next;  /* owned: next entry in the same bucket */
} PhoneEntry;

typedef struct {
    PhoneEntry **buckets;     /* owned: calloc'd array of chain heads */
    size_t capacity;
    size_t size;
} PhoneBook;

/* TODO: implement hash_djb2, pb_create, pb_destroy, pb_put, pb_get. */
/* pb_put borrows name+phone (const char*) and strdup's BOTH on insert. */
/* On update (name exists), strdup the new phone FIRST, then free the old. */
/* TODO: implement pb_remove with the same unlink-then-free-key-then-free */
/* pattern, extended to two owned strings. */

static PhoneBook *pb_create(size_t capacity) {
    (void)capacity;
    return NULL;
}

/* OWNERSHIP: frees every name, every phone, every node, buckets, struct. */
static void pb_destroy(PhoneBook *pb) {
    (void)pb;
}

/* OWNERSHIP: borrows name+phone; owns the copies. 0 ok, -1 on failure. */
static int pb_put(PhoneBook *pb, const char *name, const char *phone) {
    (void)pb;
    (void)name;
    (void)phone;
    return -1;
}

/* OWNERSHIP: borrows book+name; *out borrows the stored phone (do NOT free). */
static int pb_get(const PhoneBook *pb, const char *name, const char **out) {
    (void)pb;
    (void)name;
    (void)out;
    return -1;
}

int main(void) {
    PhoneBook *pb = pb_create(8);
    const char *found;

    if (pb == NULL) {
        return 1;
    }
    pb_put(pb, "Ada", "555-0101");
    pb_put(pb, "Grace", "555-0102");
    pb_put(pb, "Ada", "555-9999");   /* update: new number wins */
    if (pb_get(pb, "Ada", &found) == 0) {
        printf("Ada -> %s\n", found);
    }
    if (pb_get(pb, "Linus", &found) != 0) {
        printf("Linus not found\n");
    }
    printf("size = %zu\n", pb->size);
    pb_destroy(pb);
    return 0;
}
```

```text
Ada -> 555-9999
Linus not found
size = 2
```

Requirements:

- `pb_put` duplicates **both** strings and checks **both** `strdup` results. If the second fails, free the first copy and the bare node. Write in comments which bytes leak if either check is skipped.
- Update path: `strdup` the new phone before freeing the old one, so a failed `strdup` keeps the old number intact. Write in comments why the reverse order (free first, then `strdup`) loses the number on failure.
- `pb_get` returns the stored phone through `const char **out` that the caller must **not** free. Write that promise in an `OWNERSHIP` comment and restate it in `main`.
- `pb_destroy` frees `name`, then `phone`, then the node, for every entry, then the array and the struct. One `pb_destroy` at the end must leave zero sanitizer diagnostics.
- Extra credit: implement `pb_remove` and call it for head, middle, and missing names, printing `size` after each call.

---

## Sanitize Everything

Do this for **every** exercise above, and for every exercise in the whole course from now on:

1. Compile with warnings as errors and with both sanitizers:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -Werror -g \
       -fsanitize=address,undefined -fno-omit-frame-pointer main.c -o main_san
   ```

2. Run it with several different inputs, not just one. A collision that only appears with 7 keys needs 7 keys to be reachable.
3. Fix everything the sanitizer reports. It is not a suggestion.
4. Re-run until the output contains no sanitizer diagnostics at all. Silence is the passing state.
5. Then compile the plain version and confirm it still behaves identically:

   ```bash
   gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
   ```

6. If a program legitimately cannot be leak-free (a library that keeps state until exit), write down **why** in a comment, and check that the leaked bytes are bounded and documented.

Report, for each program, in a comment at the top of the file:

```text
[ ] compiles with -Werror, zero warnings
[ ] no sanitizer diagnostics
[ ] every malloc/calloc/strdup has a matching free
[ ] every free argument came from malloc/calloc/realloc/strdup
[ ] no pointer used after free
[ ] keys read as const char*, keys kept as strdup copies
[ ] ownership documented on every pointer-returning function
```

That checklist is the habit the whole module is trying to build. A C program that passes it will not be the fastest, but it will still be working next month.

---

#### [Previous](./02-part.md) | [Next](../challenge.md)
