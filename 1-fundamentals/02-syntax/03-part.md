# `const`, `static` and Storage Duration

C gives you a few keywords that control *when* and *whether* a value can change and *where* it lives. These are the tools you use to express intent and protect your own code from mistakes.

---

## `const`: This Value Will Not Change

```c
const int MAX = 100;
int age = 26;
const int *p = &age;   /* pointer to const int — *p cannot be assigned */
int * const q = &age;  /* const pointer to int — q cannot be reassigned */
const int * const r = &age;  /* both */
```

| Declaration | What cannot change |
| --- | --- |
| `const int x;` | the value of `x` |
| `const int *p;` | the value `*p` points to, but `p` itself can point elsewhere |
| `int * const p;` | `p` itself, but `*p` can be modified |
| `const int * const p;` | neither — a fully immutable binding |

Use `const` for **every function parameter you do not modify** and for **every pointer you receive from a caller**. It turns "I hope nobody changes this" into a compile error.

```c
void print_name(const char *name);          /* good */
void print_name(char *name);                 /* weaker, but legal */

#define MAX 100   /* macro: the literal 100 is pasted everywhere */
```

Modern C prefers `enum { MAX = 100 };` or `#define` for constants depending on the codebase convention. What you should avoid is a mutable global:

```c
int counter = 0;      /* anyone can change this from any file */
static int counter = 0;  /* only this file can change it */
```

---

## Storage Duration

Every object in C has a storage duration that determines when its memory exists:

| Duration | Created | Destroyed | Example |
| --- | --- | --- | --- |
| **Automatic** | when the block is entered | when the block exits | local variables |
| **Static** | program start | program end | file-scope variables, `static` locals |
| **Allocated** | by `malloc` | by `free` | dynamic memory |

### Automatic (the default for locals)

```c
int main(void) {
    int x = 5;
    printf("%d\n", x);
}   /* x is destroyed here; the memory is recycled */
```

Local variables live on the **stack**. They are allocated when the block starts and reclaimed when it ends. Fast, but limited in size — the stack is typically a few megabytes.

### Static

```c
static int call_count = 0;   /* zero-initialized automatically */
```

Two flavors:

```c
/* file scope: visible to every function that can see it, lives forever */
static int module_state = 0;

/* block scope: visible only inside the block, but survives across calls */
void tick(void) {
    static int times = 0;   /* initialized to 0, then never reset */
    times++;
    printf("called %d times\n", times);
}
```

```c
tick();  /* called 1 times */
tick();  /* called 2 times  ← note: 'times' was NOT reset */
```

Static locals are initialized **before `main` runs**, so they cannot depend on anything computed at runtime. That also makes them the right place for a **lookup table of string constants**:

```c
const char *get_month_name(int month) {
    static const char *const names[] = {
        "January", "February", "March", ...
    };
    return names[month - 1];
}
```

Using `static const char *const` here gives a read-only table stored in read-only memory, and makes the string literals non-modifiable.

---

## Lifetime vs Scope vs Visibility

These three terms are related but different:

- **Scope** — where the *name* can be used in the source code (lexical scope).
- **Lifetime** — how long the *memory* exists (storage duration).
- **Visibility (linkage)** — how many translation units can refer to the name (`static` at file scope gives *internal* linkage).

A `static` local has block scope but static lifetime — you can only name it inside the block, but it lives for the whole program run.

---

#### [Previous](./02-part.md) | [Next](./exercises.md)