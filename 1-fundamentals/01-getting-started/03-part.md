# Coding Style and Conventions

Unlike most languages, C does not force a formatting style on you. The language is agnostic about spaces, line breaks and indentation — which is exactly why every C project on the planet looks slightly different. What matters much more than indentation is **whitespace**, because in C whitespace can change the meaning of your program.

## The Rule That Bites Everyone

```c
int main(void) { return 0; }   // ok

int* getPointer(void) { ... }  // this returns an int* ... always
int * getPointer(void) { ... } // ... but THIS returns a pointer to int!
```

It compiles. It may even run. But it is confusing, and it eventually causes a bug in someone else's file. **The convention in the entire C world** is to attach the `*` to the name:

```c
char *name;
void **argv;
const char *path;
```

The same applies to the reference operator `&`:

```c
int  *ptr;   // ptr is a pointer to int
int &ref;    // NOT a C thing at all — this does not compile
```

The rule of thumb: **the type is `type *name`, and the `*` sticks to `name`.**

---

## Indentation

There is no official standard. The two most common styles:

**K&R / "unix" style** — used by the Linux kernel, opens the brace on the same line as the control statement:

```c
if (x > 0) {
    printf("positive\n");
}
```

**Allman style** — the brace goes on its own line, common in C# and Java circles:

```c
if (x > 0)
{
    printf("positive\n");
}
```

Pick one and be consistent. This course uses the K&R style, which you will see everywhere in real C code.

### Spaces around operators, not inside brackets

```c
/* good */
for (int i = 0; i < 10; i++) {
    int result = a + b * 2;
    ptr->field = other->field;
}

/* avoid */
for(int i=0;i<10;i++){
    int result=a+b*2;
}
```

---

## Braces: When You Can Skip Them

C lets you write a control statement with a single statement and no braces:

```c
if (x > 0)
    printf("positive\n");

for (int i = 0; i < 3; i++)
    printf("%d\n", i);
```

This is legal, and in a 1970s code base a single line is all you would write. In real code, always use braces. Adding a second statement to a brace-less block is the single most common source of bugs in C:

```c
if (logged_in)
    printf("Welcome!\n");
    printf("Press 1 to continue\n");   /* ALWAYS runs — not part of the if */
```

The compiler will not warn you about this. Use braces.

---

## Comments

Three styles are available. Use the first two.

```c
/* This is the classic style. It can span
   multiple lines. */

/* Single-line comment. */

// This is a C99+ style comment. Also multi-line.

#if 0
/* Not compiled at all — useful to disable a block temporarily. */
#endif
```

Guidelines:
- Comment the **why**, never the **what**. The code already says what it does.
- `//` requires C99 or later; this course uses C11 so it is always available.
- Do not leave commented-out code behind. Git remembers it; delete it.

---

## Naming Conventions

There is no single standard, but you will see these conventions everywhere:

| Element | Convention | Example |
| --- | --- | --- |
| Functions | `snake_case` (C standard library) or `camelCase` | `strcpy`, `calculateTotal` |
| Types / structs | `PascalCase` or `snake_case` with `_t` | `Vector`, `size_t` |
| Variables | `snake_case` | `total_items` |
| Macros / constants | `UPPER_SNAKE_CASE` | `MAX_SIZE`, `NULL` |
| Struct members | `snake_case` | `point.x`, `list.size` |
| Local variables | any, but short names for short lives | `i`, `n`, `total` |

For types, the C standard library uses the `_t` suffix (`size_t`, `ssize_t`, `uint32_t`) while POSIX libraries often prefix with `pthread_` or nothing at all. Both are fine.

---

## Declaration Order

Unlike Java or C#, C allows you to declare variables anywhere a statement is allowed. The modern style is to declare variables right before first use:

```c
/* old style: everything at the top of the function */
int i, j, total;
char buffer[100];

/* modern style: declare where you need it */
int total = 0;
for (int i = 0; i < 10; i++) {
    total += i;
}
```

The `for (int i = 0; ...)` form is C99+. It is standard, readable, and keeps the variable's scope limited to the loop.

---

## A Note on Headers and Namespaces

C has **no namespaces**. Every identifier in a program is global unless declared `static`. This means that a function named `list_add` in your library will collide with a `list_add` from another library at link time. Common mitigations, all seen in the wild:

- Prefix your functions (`mystring_length`, `mylist_add`).
- Keep them `static` when they are internal to one file.
- Name your types with a prefix (`mylist_t`).

---

#### [Previous](./02-part.md) | [Next](./exercises.md)