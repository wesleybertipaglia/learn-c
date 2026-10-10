# Character Pointers, Strings and `const`

Everything so far used `int`. Strings are where pointers show up in real programs constantly, and where the rules about ownership, read-only memory and `const` stop being optional.

---

## A String Is a `char` Array Ending in `\0`

There is no string type in C. A "string" is a sequence of `char` values with a zero byte at the end:

```text
  "Wesley"

   'W'  'e'  's'  'l'  'e'  'y'  '\0'
   0x57 0x65 0x73 0x6c 0x65 0x79  0x00
   ^                          ^
   |                          +-- terminator: printf stops here
   +-- the characters
```

Because a string is just an array, `char *` can point at the first character — which is exactly what `printf("%s")` expects. This is why `sizeof("Wesley")` is `7` and not `6`.

```c
#include <stdio.h>

void show_rodata(void) {
    const char *name = "Wesley";
    int i;

    printf("name = %s\n", name);
    printf("name[0] = %c\n", name[0]);

    for (i = 0; name[i] != '\0'; i++) {
        printf("  name[%d] = %c  (address %p)\n", i, name[i],
               (void *)&name[i]);
    }

    printf("literal size = %zu bytes (including the \\0 terminator)\n",
           sizeof("Wesley"));
}

int main(void) {
    show_rodata();

    return 0;
}
```

```text
name = Wesley
name[0] = W
  name[0] = W  (address 0x55f992f88008)
  name[1] = e  (address 0x55f992f88009)
  name[2] = s  (address 0x55f992f8800a)
  name[3] = l  (address 0x55f992f8800b)
  name[4] = e  (address 0x55f992f8800c)
  name[5] = y  (address 0x55f992f8800d)
literal size = 7 bytes (including the \0 terminator)
```

Note the addresses: 1 byte apart, because a `char` is 1 byte. This is the clearest possible picture of what a string in memory looks like.

---

## String Literals Are Read-Only

This declaration looks harmless:

```c
char *name = "Wesley";
```

It is not. `"Wesley"` is not a variable you can modify. It is a constant block that the compiler places in a read-only section of the program (usually called `.rodata`). The pointer you get is real and readable — it is just not writable.

**This is undefined behaviour. Do not run it:**

```c
#include <stdio.h>

int main(void) {
    char *name = "Wesley";
    name[0] = 'J';        /* writing to read-only memory */
    printf("%s\n", name);
    return 0;
}
```

It compiles with zero warnings. Then:

```text
Segmentation fault (core dumped)
```

The kernel refuses the write because the page is mapped read-only. On other systems, or with a different compiler, you might instead see the string silently change, or the program die somewhere else entirely. None of those outcomes are guaranteed, which is what "undefined" means.

### `const char *`: The Fix

Adding `const` converts the runtime crash into a compile error:

```c
#include <stdio.h>

int main(void) {
    const char *name = "Wesley";
    name[0] = 'J';
    printf("%s\n", name);
    return 0;
}
```

```text
p5_const.c:5:13: error: assignment of read-only location '*name'
    5 |     name[0] = 'J';
      |             ^
```

Now the compiler catches it before you run anything. **Bug and fix side by side:**

```c
char       *name = "Wesley";    /* WRONG: promises writability you do not have */
const char *name = "Wesley";    /* RIGHT: promises read-only, and it is       */
```

### When You Need a Writable String, Copy It

```c
char buffer[7] = "Wesley";   /* a real array: your own, writable memory */

buffer[0] = 'J';
printf("%s\n", buffer);      /* prints: Jesley */
```

```text
+---------+     +---------+
|  'W' 'e'|     |  'J' 'e'|
|  's' 'l'| --> |  's' 'l'|
|  'e' 'y'|     |  'e' 'y'|
|  '\0'    |    |  '\0'   |
+---------+     +---------+
  read-only       writable
  .rodata         stack
```

The sizes line up because a `char[7]` and the literal `"Wesley"` hold identical bytes. The difference is ownership: the literal belongs to the program image, the array belongs to you.

---

## `const char *` in Function Parameters

The `const` version is not only for literals. Use it for **every** string pointer you receive:

```c
void print_label(const char *label) {
    printf("%s\n", label);
}
```

`const char *label` means "I promise not to modify the bytes at `label`." If the function body tries, the compiler refuses:

```c
void broken(const char *label) {
    label[0] = 'X';   /* error: assignment of read-only location */
}
```

This costs nothing and buys three things:

1. The compiler stops you from corrupting the caller's data.
2. The caller can pass **both** `char *` and `const char *` — a `const char *` parameter accepts any string.
3. The signature documents the contract.

```c
char  buffer[20] = "hi";
const char *lit = "hi";

print_label(buffer);   /* fine */
print_label(lit);      /* fine */
```

The reverse does not work:

```c
void needs_writable(char *s);
needs_writable(lit);   /* warning: discards 'const' qualifier */
```

> Full `const` for pointers is covered in the `const` section of the syntax chapter. The rule of thumb here: **if you receive a `char *` and do not write through it, add `const`.**

---

## Arrays of Strings

There are two shapes, and the difference matters.

### Shape 1: an array of pointers

```c
char *names[] = {"Ana", "Bruno", "Carla"};
```

```text
  names[3]  (array of 3 pointers, 24 bytes on a 64-bit machine)
  +--------+--------+--------+
  | ptr 0  | ptr 1  | ptr 2  |
  +---+----+---+----+---+----+
      |        |        |
      v        v        v
   "Ana\0" "Bruno\0" "Carla\0"    <-- three separate literals in .rodata
```

- Compact: only as many bytes as the pointers, plus the literals.
- Every row must have the same length if they were real arrays; here they do not need to.
- Writing through `names[i]` modifies a literal → undefined. For that you need `char *copies[] = { strdup("Ana"), ... };` or a real 2D array.

### Shape 2: a 2D array

```c
char grid[3][20];
```

```text
  grid  (one contiguous block: 3 rows x 20 bytes = 60 bytes)

  row 0: [ first\0................ ]  <-- grid[0], decays to char *
  row 1: [ second\0............... ]  <-- grid[1]
  row 2: [ third\0................ ]  <-- grid[2]
  +------------------------------------------------+
  | 20 bytes                                        |
  +------------------------------------------------+
```

- Fixed size for every row: 20 bytes each, always.
- Contiguous, cache friendly, and every row is writable.
- Costs the full 60 bytes even if the strings are short.

Rule of thumb: **fixed number of items with a known maximum length → 2D array.** Variable names coming from a lookup table or config → array of pointers.

A complete, runnable program using both shapes:

```c
#include <stdio.h>
#include <string.h>

void print_label(const char *label) {
    printf("%s\n", label);
}

int main(void) {
    char  buffer[8] = "Wesley";
    char *names[]  = {"Ana", "Bruno", "Carla"};
    char  grid[3][20];
    int   i;

    printf("buffer = %s (a real array, %zu bytes, writable)\n",
           buffer, sizeof(buffer));

    buffer[0] = 'X';
    printf("after buffer[0] = 'X': %s\n", buffer);

    strcpy(grid[0], "first");
    strcpy(grid[1], "second");
    strcpy(grid[2], "third");

    printf("grid[0] = %s\n", grid[0]);
    printf("grid[2] = %s\n", grid[2]);
    printf("sizeof(grid[0]) = %zu bytes per row\n", sizeof(grid[0]));
    printf("sizeof(grid) = %zu bytes total\n", sizeof(grid));

    printf("names is an array of %zu pointers, each one a literal:\n",
           sizeof(names) / sizeof(names[0]));
    for (i = 0; i < 3; i++) {
        print_label(names[i]);
    }

    print_label("passed as const char *");

    return 0;
}
```

```text
buffer = Wesley (a real array, 8 bytes, writable)
after buffer[0] = 'X': Xesley
grid[0] = first
grid[2] = third
sizeof(grid[0]) = 20 bytes per row
sizeof(grid) = 60 bytes total
names is an array of 3 pointers, each one a literal:
Ana
Bruno
Carla
passed as const char *
```

One detail worth pausing on: `strcpy(grid[0], "first")` is needed because `grid[0]` is an **array** of 20 chars, and you cannot assign to an array. `grid[0] = "first"` is a compile error:

```text
error: assignment to expression with array type
   21 |     grid[0] = "first";
      |             ^
```

`strcpy` is the right tool. Copying is a whole topic of its own — the Strings chapter covers `strlen`, `strcpy`, `strncpy` and why `strcpy` is dangerous.

---

## Pointers to Pointers

One last layer. If `int *p` means "a pointer to an int", then `int **pp` means "a pointer to a pointer to an int". It sounds useless until you notice what it lets you do: **rebind the caller's pointer.**

```text
  **pp  <-- can change what p points at
   |
   v
   p    <-- points at x
   |
   v
   x = 42
```

```c
#include <stdio.h>

void set_value(int **target, int value) {
    **target = value;
}

int main(void) {
    int x = 42;
    int *p = &x;
    int **pp = &p;
    int y = 99;

    printf("x    = %d at %p\n", x, (void *)&x);
    printf("p    = %p (points at x)\n", (void *)p);
    printf("pp   = %p (points at p)\n", (void *)(void *)pp);
    printf("*p   = %d\n", *p);
    printf("**pp = %d\n", **pp);

    *pp = &y;
    printf("\nafter *pp = &y: *p = %d, p now points at %p\n", *p, (void *)p);

    set_value(&p, 7);
    printf("after set_value(&p, 7): **pp = %d, y = %d\n", **pp, y);

    return 0;
}
```

```text
x    = 42 at 0x7ffe4fbc9360
p    = 0x7ffe4fbc9360 (points at x)
pp   = 0x7ffe4fbc9368 (points at p)
*p   = 42
**pp = 42

after *pp = &y: *p = 99, p now points at 0x7ffe4fbc9364
after set_value(&p, 7): **pp = 7, y = 7
```

Line by line:

- `*pp = &y;` assigns through two levels, so the **caller's `p` now points at `y`**. You cannot do that with `int *` alone — remember from the previous part that assigning to a pointer parameter only changes the function's copy.
- `set_value(&p, 7)` takes `int **` so it can reach the caller's `p` and write `7` through it into `y`.

And the bug, side by side:

```c
void set_to(int **target, int value) {
    *target = &value;    /* WRONG: makes target point at this
                            function's local 'value', which dies
                            at return — the caller gets a dangling
                            pointer                                   */
    **target = value;    /* RIGHT: writes the value through the pointer */
}
```

### When You Actually Need `int **`

Three situations, in rough order of how often you will meet them:

1. **2D arrays.** A `char grid[3][20]` decays to a pointer to rows, `char (*)[20]`. To pass the whole grid you need `char (*grid)[20]` as the parameter type, and to pass a *pointer to a pointer to a row* you need `char **`.
2. **Out-parameters for pointers.** `fgets(buf, size, fp)` writes through `fp`. A function that must let the caller choose *which* buffer to fill takes `char **out` or `FILE **out`.
3. **Linked lists and trees.** Each node holds a pointer to the next node, so the nodes are manipulated through `Node *` and swapping links needs `Node **`. The Linked Lists chapter is built entirely on this.

### How to Read Multi-Level Declarators

Right to left, starting at the name:

```c
int *p;         /* p is a pointer to int                     */
int **pp;       /* pp is a pointer to (pointer to int)       */
int *arr[3];    /* arr is an array of 3 pointers to int      */
int (*ap)[3];   /* ap is a pointer to (array of 3 ints)      */
char **argv;    /* argv is a pointer to (pointer to char)    */
```

The rule: `*` binds tighter than `[ ]`, so wrap the array in parentheses when you want a pointer to an array.

---

## Rules to Survive Pointers

This is the chapter that decides whether the rest of C feels natural or hostile. Print this out.

```text
1.  A pointer is a variable holding an address. Nothing more.

2.  &x  gives the address of x.   *p  gives the thing at p.
    They are opposites, not synonyms.

3.  Initialize every pointer at declaration:
        int *p = &x;      or      int *p = NULL;
    An uninitialized pointer holds garbage and will eventually crash.

4.  Check for NULL before you follow a pointer that might not be set.

5.  If a function must change the caller's variable, take a pointer.
    If it must change the caller's pointer, take a pointer to a pointer.

6.  A pointer parameter is still passed by value.
        p = something   changes only the function's copy
        *p = something  changes the caller's data

7.  Arrays are not pointers, but they decay into pointers in almost
    every expression. sizeof does not decay. Never assume sizeof in a
    function parameter tells you the array length.

8.  Pointer arithmetic counts ELEMENTS, not bytes: p + 1 moves
    sizeof(*p) bytes, and only stays inside the array it came from.

9.  A string literal is read-only. Use const char * for anything you
    receive, and a real char array when you need to write.

10. If you do not own the memory, do not free it, and do not write
    through a pointer to it.
```

### The Habit That Prevents Most Bugs

For every line that contains a `*`, ask one question: **is this the pointer, or the thing it points at?**

```c
int *p = &x;      /* p  is the POINTER        */
*p = 5;           /* *p is the THING          */
printf("%d", *p); /* *p is the THING          */
printf("%p", (void *)p);   /* p  is the POINTER */
```

If you cannot answer that question about a line, do not run the program yet. Sit with it until the answer is obvious. Every serious C programmer has learned this the slow way; the habit is the whole shortcut.

### Debugging Pointers When It Goes Wrong

You have `-g` in your compile flags, so use the tools:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g prog.c -o prog
gdb ./prog
```

```text
(gdb) break main
(gdb) run
(gdb) next
(gdb) print x
$1 = 42
(gdb) print &x
$2 = (int *) 0x7ffd977ac9a8
(gdb) print p
$3 = (int *) 0x7ffd977ac9a8
(gdb) print *p
$4 = 42
(gdb) continue
```

`print &x`, `print p`, `print *p` in that order answers almost every "why is my pointer wrong" question. When a program segfaults, `gdb ./prog` and then `run` tells you the exact line. The Debugging chapter covers this properly.

---

## Where Pointers Go Next

You now have the whole model:

- An **array** is a contiguous run of values, accessed as `arr[i]`, which is `*(arr + i)`.
- A **pointer** is an address, adjusted by element size, and it does not know how long the run is.
- **Strings** are `char` arrays with a `\0`, usually reached through a `char *`.
- **Functions** take pointers so they can write into the caller's memory.
- **`malloc`** hands you a pointer to memory you must `free` — the Dynamic Memory chapter.

From here, arrays, strings, structs and dynamic memory are all pointer mechanics with different payload types. That is the payoff: you are learning one idea, not eight.

---

#### [Previous](./04-part.md) | [Next](../09-errors/01-part.md)