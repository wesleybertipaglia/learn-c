# Final Challenge of Module 2

You have finished data structures: arrays, strings, structs, dynamic memory, linked lists, stacks, queues and hash tables. This project uses all of them, and nothing from module 3. It is the last thing you build before recursion, bits and files change how you think about programs.

---

## Objective

Write a **command-line phonebook manager** in C that stores contacts in a linked list, indexes them in a hash table for instant lookup, supports undo through a stack, and keeps a history of recent searches in a queue.

The program must compile with zero warnings and run sanitizer-clean:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g phonebook.c -o phonebook
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer phonebook.c -o phonebook_san
```

A run with no diagnostics from `./phonebook_san` is the passing state. Any leak, use-after-free or overflow is a failure, even if the output looks right.

---

## Features

The menu, printed once at startup and again after every operation:

```text
  1) Add contact
  2) Find contact
  3) Remove contact
  4) List all contacts
  5) Undo last add/remove
  6) Show recent searches
  7) Exit
```

| Option | Behaviour |
| --- | --- |
| 1 | Ask for a name and a phone number. Reject an empty name, a duplicate name, and anything that does not fit the fixed buffers. Insert into the list **and** the index |
| 2 | Ask for a name. Look it up in the hash table in O(1). On success print the contact and record the name in the recent-searches queue (capacity 5, oldest dropped) |
| 3 | Ask for a name. Remove from **both** the list and the index, freeing the key and the node. Push the removed contact onto the undo stack |
| 4 | Walk the linked list in insertion order and print every contact |
| 5 | Pop the undo stack and restore the contact (undo an add removes it again; undo a remove inserts it back). An empty stack reports an error instead of crashing |
| 6 | Print the contents of the recent-searches queue, oldest first |
| 7 | Print a summary (how many contacts, how many undos available, how many recent searches) and exit with status 0 |

---

## Requirements

### Storage

Every structure from the module appears, each one doing the job it is good at:

```c
#define NAME_MAX  64
#define PHONE_MAX 32
#define HASH_CAP  53
#define RECENT_CAP 5

typedef struct {
    char name[NAME_MAX];
    char phone[PHONE_MAX];
} Contact;

typedef struct Node {
    Contact contact;
    struct Node *next;
} Node;

typedef struct {
    Node *buckets[HASH_CAP]; /* chaining: each bucket is a list */
    size_t size;
} PhoneIndex;

typedef struct {
    Node *top;               /* linked stack of undone contacts */
} UndoStack;

typedef struct {
    char items[RECENT_CAP][NAME_MAX]; /* circular buffer */
    size_t head;
    size_t count;
} RecentQueue;
```

Rules:

- The linked list is the **owner** of the insertion order. The hash table nodes may be the same nodes (intrusive index) or separate entries that `strdup` the name — pick one design and document it in a comment. Either way, every `malloc`/`strdup` has exactly one `free`.
- The undo stack stores **copies** of contacts (`Contact` by value), never pointers into the live list. Undoing must not create a dangling pointer.
- The recent-searches queue stores **copies** of names in a fixed circular buffer. No heap allocation inside it.
- Nothing in the heap outlives `main`. Option 7 frees the list, the index, and the undo stack before returning.

### Functions

Every menu option is a function. No option may be implemented inline in `main`.

```text
add_contact   find_contact   remove_contact   list_contacts
undo_last     show_recent    print_menu
```

Helpers you will need (all `static`):

```text
hash_djb2        index_put   index_get   index_remove   index_destroy
list_append      list_remove_by_name     list_destroy
stack_push       stack_pop   queue_push  queue_print
read_line        read_name   read_phone
```

### Error handling

1. **Every fallible function returns a status** and the caller checks it:
   ```c
   typedef enum {
       STATUS_OK = 0,
       STATUS_INVALID,
       STATUS_DUPLICATE,
       STATUS_NOT_FOUND,
       STATUS_EMPTY,
       STATUS_NOMEM
   } status_t;
   ```
   with a `status_text` function. No bare `-1` anywhere.
2. **Errors go to `stderr`**, results to `stdout`. Prove it with `./phonebook > out.txt 2> err.txt`.
3. `malloc`, `strdup`, `calloc` and `realloc` are checked against `NULL` on every call. `realloc` goes through a temporary pointer.
4. Underflow (undo on an empty stack, search with no contacts) and overflow (duplicate name, over-long input) report a status. They never `assert` and never abort on user input.

### Input validation and strings

- Read whole lines with `fgets` into a fixed buffer, then strip the newline. Never `scanf("%s")` — it stops at spaces and overflows the buffer.
- Reject names longer than `NAME_MAX - 1` and phones longer than `PHONE_MAX - 1` **before** copying. Copy with `snprintf` or a length check plus `strcpy`, never an unbounded `strcpy` of user input.
- Document the ownership of every pointer parameter: `/* borrows */` vs `/* takes ownership */` vs `/* returns heap memory, caller must free */`.
- The hash function is djb2 from the hash tables chapter. The bucket index is `hash % HASH_CAP`, computed on the **resized** capacity if you implement rehashing (optional extra).

### Style

1. `const` on every pointer parameter you do not modify (`const char *name`, `const PhoneIndex *idx`).
2. `static` on every helper. One declaration per line.
3. `size_t` for sizes and counts, `int` only for statuses and menu choices.
4. A comment above every struct saying who owns its memory and who frees it.

---

## Example of Operation

```text
$ ./phonebook
========================================
   Phonebook Manager (module 2)
========================================

  1) Add contact
  2) Find contact
  3) Remove contact
  4) List all contacts
  5) Undo last add/remove
  6) Show recent searches
  7) Exit
========================================
Choice: 1
Name: Wesley
Phone: 555-0101
Added Wesley.

Choice: 1
Name: Ana
Phone: 555-0102
Added Ana.

Choice: 1
Name: Wesley
Phone: 555-9999
error: duplicate contact 'Wesley'

Choice: 2
Name: Ana
Ana — 555-0102

Choice: 2
Name: Bruno
error: no contact named 'Bruno'

Choice: 4
Contacts (2):
  Wesley — 555-0101
  Ana — 555-0102

Choice: 3
Name: Ana
Removed Ana.

Choice: 4
Contacts (1):
  Wesley — 555-0101

Choice: 5
Undid remove of Ana.

Choice: 4
Contacts (2):
  Wesley — 555-0101
  Ana — 555-0102

Choice: 6
Recent searches (2):
  Ana
  Bruno

Choice: 7
----------------------------------------
Summary
Contacts: 2
Undos available: 1
Recent searches: 2
Goodbye.
```

Note in the transcript:

- The duplicate add and the missed search both print to `stderr` and leave the data untouched.
- Undo restores `Ana` with the same phone number — the stack held a copy, not a pointer.
- `Bruno` appears in recent searches even though the lookup failed: the queue records what was asked, not only what was found. Document whichever policy you chose.

---

## Tips

### One node, two structures — pick a design and stick to it

Either the hash buckets point at the list nodes (one allocation per contact, freed once) or each `index_put` allocates its own entry with a `strdup` of the name (two allocations, freed together). Both are correct. Mixing them — freeing the list node and then freeing the index entry that points at it — is a double-free. Draw the boxes and arrows for your choice before writing code.

### Undo stores values, not pointers

```c
typedef struct UndoNode {
    Contact contact;          /* a COPY: safe after the original is freed */
    int was_add;              /* 1 if the original op was an add */
    struct UndoNode *next;
} UndoNode;
```

If the stack held a `Node *` into the live list, removing the contact would free the very memory the stack points at. The next undo would be a use-after-free. Copies sidestep the whole problem.

### The queue is the easiest part — keep it that way

```c
static void recent_push(RecentQueue *q, const char *name) {
    size_t pos = (q->head + q->count) % RECENT_CAP;
    if (q->count == RECENT_CAP) {
        q->head = (q->head + 1) % RECENT_CAP; /* drop oldest */
        pos = (q->head + q->count - 1) % RECENT_CAP;
    } else {
        q->count++;
    }
    snprintf(q->items[pos], NAME_MAX, "%s", name);
}
```

No `malloc`, no free, no leak possible. If your queue needs the heap, you overbuilt it.

---

## Extra challenge

If the required set is done, sanitizer-clean and warning-free, pick one:

1. **Rehashing.** Grow `HASH_CAP` to the next prime when `size / capacity > 0.75`, reinserting every entry with `hash % new_capacity`. Print the load factor before and after.
2. **Prefix search.** List every contact whose name starts with a given prefix using `strncmp`, without touching the hash internals — walk the list, not the buckets.
3. **Persistent phonebook.** Save to / load from a text file (one `name;phone` line each). This previews module 3; validate every line on load and skip malformed ones with a message.
4. **Sorted listing.** Print contacts alphabetically without reordering the list: collect pointers into a temporary array and `qsort` it with `strcmp`.
5. **Collision report.** Add a hidden option `0` that dumps every non-empty bucket and its chain length, plus the longest chain. Use it to argue whether 53 was a good capacity.

---

## Grading Checklist

### Compilation

```text
[ ] gcc -std=c11 -Wall -Wextra -pedantic -g phonebook.c -o phonebook, zero warnings
[ ] Also builds clean with -Werror
[ ] Runs clean under -fsanitize=address,undefined (no leaks, no UAF, no overflow)
```

### Structure

```text
[ ] Contacts live in a linked list (insertion order preserved)
[ ] Lookup goes through a hash table with chaining (not a linear scan)
[ ] Undo is a stack of Contact copies; recent searches are a bounded queue
[ ] Every menu option is a function; helpers are static
[ ] Pointer parameters not modified are const; sizes are size_t
[ ] Ownership documented on every struct and every heap-returning function
```

### Error handling

```text
[ ] Every fallible function returns status_t and every caller checks it
[ ] Statuses come from the enum with a status_text function
[ ] Errors on stderr, results on stdout (proved with redirection)
[ ] Empty undo, unknown name, duplicate name, over-long input all handled
[ ] No assert on anything the user typed
```

### Memory and strings

```text
[ ] Every malloc/strdup checked for NULL; realloc via temporary
[ ] Every allocation freed exactly once before exit (sanitizer proves it)
[ ] No use-after-free between list, index and undo stack
[ ] fgets + length check before every copy; no unbounded strcpy of input
[ ] Hash is djb2 (or documented alternative); index is hash % capacity
```

### Behaviour

```text
[ ] Option 1 rejects empty/duplicate/over-long input without corrupting data
[ ] Option 2 records the search regardless of hit or miss (documented)
[ ] Option 3 removes from both structures and pushes a copy onto undo
[ ] Option 5 restores the exact contact on either kind of undo
[ ] Option 7 prints the summary and exits 0 with nothing leaked
```

A submission that runs, finds contacts instantly, undoes correctly, remembers recent searches, and exits sanitizer-clean has met the bar. One that also rehashes or reports collisions is well above it.

---

#### [Previous](./08-hashtables/exercises.md) | [Next](../3-advanced/01-recursion-and-algorithms/01-part.md)
