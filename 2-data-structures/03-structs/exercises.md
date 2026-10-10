# Exercise List 3 - Structs

Six exercises on grouping data, passing it around, and understanding the bytes behind it. Every program must compile with `gcc -std=c11 -Wall -Wextra -pedantic -g file.c -o file` and produce **zero** warnings.

Add `-Werror` if you want the discipline enforced by the compiler.

---

## Exercise 1: Points, by Value and by Return

Write a `Point` struct with two `int` members, `x` and `y`, and implement:

- `int distance_squared(Point a, Point b)` — the squared distance, without calling `sqrt`.
- `Point midpoint(Point a, Point b)` — the point halfway between the two.

```c
#include <stdio.h>

typedef struct {
    int x;
    int y;
} Point;

/* by value: the function only reads the two points and returns a number */
static int distance_squared(Point a, Point b) {
    int dx = a.x - b.x;
    int dy = a.y - b.y;

    return dx * dx + dy * dy;
}

/* by value: the result is 8 bytes, so copying is cheaper than a pointer */
static Point midpoint(Point a, Point b) {
    Point m;

    m.x = (a.x + b.x) / 2;
    m.y = (a.y + b.y) / 2;

    return m;
}

int main(void) {
    Point a = {0, 0};
    Point b = {4, 6};
    Point m;

    printf("distance_squared = %d\n", distance_squared(a, b));

    m = midpoint(a, b);
    printf("midpoint = (%d, %d)\n", m.x, m.y);

    return 0;
}
```

```text
distance_squared = 52
midpoint = (2, 3)
```

Requirements:

- Write a comment above **each** function explaining why it takes its arguments by value and not by pointer. The reasoning: `Point` is 8 bytes, both functions only read their arguments, and returning by value avoids making the caller manage memory.
- Then add `void translate(Point *p, int dx, int dy)` that must modify the caller's point. Write a comment explaining that this one **cannot** take `Point` by value, because a copy would be changed and discarded.
- Demonstrate both: call `translate(&a, 5, 5)` and show that `a` really changed.
- Extra credit: write a `distance` function that returns a `double` using `sqrt` from `<math.h>` (compile with `-lm`) and confirm the two agree.
- Extra credit: a struct with a `double x, y` returns 16 bytes. Re-measure `sizeof(Point)` and explain in comments whether passing by value is still the right choice.

---

## Exercise 2: A Rectangle

Write a `Rectangle` struct with `width` and `height` (use `int`), plus:

- `double area(Rectangle r)`
- `double perimeter(Rectangle r)`

```c
#include <stdio.h>

typedef struct {
    int width;
    int height;
} Rectangle;

static double area(Rectangle r) {
    return (double)r.width * (double)r.height;
}

static double perimeter(Rectangle r) {
    return 2.0 * ((double)r.width + (double)r.height);
}

int main(void) {
    Rectangle r = {5, 3};

    printf("area      = %.1f\n", area(r));
    printf("perimeter = %.1f\n", perimeter(r));

    return 0;
}
```

```text
area      = 15.0
perimeter = 16.0
```

Requirements:

- `area` takes the rectangle **by value**. Add a comment saying the caller keeps its own copy.
- Then write `void scale(Rectangle *r, int factor)` that doubles the size in place, and call it with `scale(&r, 2)` to prove the modification reached the caller.
- Print `sizeof(Rectangle)` and explain in comments why it is 8 bytes and not 2.
- Then add a `char label;` member, print `sizeof` again, and explain the difference using the padding rules from the third part.
- Read the dimensions with `scanf("%d %d", &r.width, &r.height)` and check that both reads succeeded before using the values.

---

## Exercise 3: A Student List in a Fixed Array

A `Student` has a `name` (`char[32]`), an `age` (`int`). Build an array of **5** students and implement:

- `int add_student(Student *list, int count, const Student *s)` — append, return the new count or `-1` if full.
- `void print_all(const Student *list, int count)`
- `int find_by_name(const Student *list, int count, const char *name)` — index or `-1`.
- `int remove_at(Student *list, int count, int index)` — shift the tail left, return the new count.

```c
#include <stdio.h>
#include <string.h>

#define CAPACITY 5

typedef struct {
    char name[32];
    int  age;
} Student;

static int add_student(Student *list, int count, const Student *s) {
    if (count >= CAPACITY) {
        return -1;   /* the array is full */
    }

    list[count] = *s;
    return count + 1;
}

static void print_all(const Student *list, int count) {
    for (int i = 0; i < count; i++) {
        printf("%d: %-10s %d\n", i, list[i].name, list[i].age);
    }
}

static int find_by_name(const Student *list, int count, const char *name) {
    for (int i = 0; i < count; i++) {
        if (strcmp(list[i].name, name) == 0) {
            return i;
        }
    }
    return -1;
}

static int remove_at(Student *list, int count, int index) {
    if (index < 0 || index >= count) {
        return count;
    }

    for (int i = index; i < count - 1; i++) {
        list[i] = list[i + 1];
    }
    return count - 1;
}

int main(void) {
    Student list[CAPACITY];
    int count = 0;
    Student s;

    s.age = 19; strcpy(s.name, "Ana");    count = add_student(list, count, &s);
    s.age = 21; strcpy(s.name, "Bruno");  count = add_student(list, count, &s);
    s.age = 20; strcpy(s.name, "Carla");  count = add_student(list, count, &s);

    print_all(list, count);

    int idx = find_by_name(list, count, "Carla");
    printf("\nfind 'Carla' -> %d\n", idx);

    count = remove_at(list, count, idx);
    print_all(list, count);

    return 0;
}
```

```text
0: Ana        19
1: Bruno      21
2: Carla      20

find 'Carla' -> 2
0: Ana        19
1: Bruno      21
```

Requirements:

- `add_student` takes `const Student *` because it only reads the source. Say so in a comment.
- `remove_at` takes `Student *` because it must write. Say so.
- Read the students from the user with `scanf`, using `snprintf` or a width-limited `scanf("%31s")` for the name. Never write past the 32 bytes.
- Print a message when `add_student` returns `-1`, and when `find_by_name` returns `-1`.
- Try to add a 6th student and show the message.
- Explain in comments why the count must be passed separately instead of computed with `sizeof list / sizeof list[0]` inside the functions. (It would work in `main`, and not in a function: inside `add_student` the parameter is a pointer, so `sizeof` would give 8.)
- Extra credit: make `remove_at` return `-1` on a bad index instead of leaving the count untouched, and handle that in `main`.

---

## Exercise 4: Times, and Normalizing Past Midnight

A `Time` struct stores `h`, `m`, `s` as `int`s. Write:

- `int in_seconds(Time t)` — seconds since midnight.
- `Time from_seconds(int total)` — the inverse, **wrapping around** a 24-hour day.
- `Time time_add(Time a, Time b)` — adding two times.

```c
#include <stdio.h>

#define SECONDS_PER_DAY 86400

typedef struct {
    int h;
    int m;
    int s;
} Time;

static int in_seconds(Time t) {
    return t.h * 3600 + t.m * 60 + t.s;
}

static Time from_seconds(int total) {
    Time t;

    total = ((total % SECONDS_PER_DAY) + SECONDS_PER_DAY) % SECONDS_PER_DAY;

    t.h = total / 3600;
    t.m = (total % 3600) / 60;
    t.s = total % 60;

    return t;
}

static Time time_add(Time a, Time b) {
    return from_seconds(in_seconds(a) + in_seconds(b));
}

int main(void) {
    Time a = {23, 30, 0};
    Time b = {2, 45, 15};
    Time r;

    r = time_add(a, b);
    printf("%02d:%02d:%02d + %02d:%02d:%02d = %02d:%02d:%02d\n",
           a.h, a.m, a.s, b.h, b.m, b.s, r.h, r.m, r.s);

    return 0;
}
```

```text
23:30:00 + 02:45:15 = 02:15:15
```

Requirements:

- Explain in comments why `from_seconds` uses `((x % N) + N) % N` and not just `x % N`. Then prove it: call `from_seconds(-100)` and show that you get `23:58:20`, not a negative `Time`.
- Add a `time_diff(Time a, Time b)` returning the number of **seconds** between two times, handling the case where `b` is earlier than `a` (it should roll forward into the next day). Show it with `01:00:00` and `23:00:00`.
- Read both times from the user and reject anything outside `0..23`, `0..59`, `0..59` with a clear message.
- Extra credit: add a `day` field to the struct that counts how many days the value is from midnight, so `from_seconds` can report more than 24 hours without losing the count.

---

## Exercise 5: A Tagged Union

Build a variant value that can hold an `int`, a `double`, or a string:

- An `enum` for the tag.
- A struct containing the tag and a **named** union member.
- `void print_value(const Value *v)` that switches on the tag.

```c
#include <stdio.h>

typedef enum {
    V_INT,
    V_DOUBLE,
    V_STRING
} ValueType;

typedef struct {
    ValueType type;
    union {
        int    i;
        double d;
        const char *s;
    } as;
} Value;

static void print_value(const Value *v) {
    switch (v->type) {
        case V_INT:
            printf("int    : %d\n", v->as.i);
            break;
        case V_DOUBLE:
            printf("double : %.2f\n", v->as.d);
            break;
        case V_STRING:
            printf("string : %s\n", v->as.s);
            break;
        default:
            printf("<invalid tag %d>\n", (int)v->type);
            break;
    }
}

int main(void) {
    Value values[3];
    int i;

    values[0].type = V_INT;
    values[0].as.i  = 42;

    values[1].type = V_DOUBLE;
    values[1].as.d  = 3.5;

    values[2].type = V_STRING;
    values[2].as.s  = "hello";

    for (i = 0; i < 3; i++) {
        print_value(&values[i]);
    }

    printf("\nsizeof(Value) = %zu bytes\n", sizeof(Value));

    /* reading the wrong member gives nonsense, which is why the tag exists */
    values[0].type = V_DOUBLE;
    values[0].as.d = 1.5;
    print_value(&values[0]);

    return 0;
}
```

```text
int    : 42
double : 3.50
string : hello

sizeof(Value) = 16 bytes
double : 1.50
```

Requirements:

- Print `sizeof` of the union alone and of the whole struct. Explain in comments why the union is 8 bytes (the `double`, plus pointer alignment) and why the struct is bigger than 8.
- Write a deliberately wrong line: read `values[1].as.i` when the tag says `V_DOUBLE`, print the result, and comment that it is the raw bits of the `double`, not a number.
- Add a `V_CHAR` case with a `char c` member. Check whether the struct got bigger. Explain in comments why adding a `char` did **not** shrink the union.
- Then remove the `union` and replace it with three separate members. Print the new `sizeof` and explain the cost of not knowing in advance which type you need.
- Extra credit: write `void print_all(const Value *list, int n)` and call `print_value` on each element in a loop.

---

## Exercise 6: Where Do the Bytes Go?

Take a struct with **4** members and print the size and the `offsetof` of every one:

```c
#include <stdio.h>
#include <stddef.h>

typedef struct {
    char   id;
    int    count;
    double total;
    char   active;
} Record;

static void report(void) {
    printf("sizeof(Record)          = %zu bytes\n", sizeof(Record));
    printf("sum of the member types = %zu bytes\n",
           sizeof(char) + sizeof(int) + sizeof(double) + sizeof(char));
    printf("padding                 = %zu bytes\n",
           sizeof(Record) - (sizeof(char) + sizeof(int) + sizeof(double) + sizeof(char)));
    printf("\n");

    printf("offsetof(Record, id)     = %zu\n", offsetof(Record, id));
    printf("offsetof(Record, count)  = %zu\n", offsetof(Record, count));
    printf("offsetof(Record, total)  = %zu\n", offsetof(Record, total));
    printf("offsetof(Record, active) = %zu\n", offsetof(Record, active));
}

int main(void) {
    Record r = {'A', 42, 3.5, 1};

    printf("r = {'%c', %d, %.1f, %d}\n\n", r.id, r.count, r.total, (int)r.active);

    report();

    return 0;
}
```

```text
r = {'A', 42, 3.5, 1}

sizeof(Record)          = 24 bytes
sum of the member types = 14 bytes
padding                 = 10 bytes

offsetof(Record, id)     = 0
offsetof(Record, count)  = 4
offsetof(Record, total)  = 8
offsetof(Record, active) = 16
```

Requirements:

- The total is **24** while the members only need **14**. Write a comment above each member explaining every byte of the gap: 3 bytes after `id` so `count` lands on a 4-byte boundary, 7 bytes after `total`... check the numbers and describe each gap from the `offsetof` values you printed.
- Draw the layout as an ASCII diagram in a comment, one row per member, marking the padding.
- Reorder the members from largest to smallest alignment and print `sizeof` again. Report the saving in comments.
- Then replace `double total` with `float total` and print again. Explain in comments why the struct shrank by less than 4 bytes.
- Finally, add a fifth member `short flag` and print again. Explain the alignment rule that decides its offset.
- Extra credit: write a function `void dump(const void *p, size_t n)` that prints `n` bytes in hex, call it on a `Record`, and identify the padding bytes by their values.

---

## Extra Challenge: A Small Library of Structs

Write one program that puts the whole chapter together. No input validation shortcuts.

```text
[ ] A Point struct with translate, distance and midpoint
[ ] A Rectangle struct that embeds a Point as its top-left corner
[ ] A Student array with add / print / find / remove
[ ] A Time struct with addition that normalizes past midnight
[ ] A tagged union holding either a Time or a Point
[ ] A function that receives a pointer to an array of structs and its count
[ ] Every malloc is paired with a free; run under the sanitizer to prove it
```

Then answer these in comments at the top of the file:

1. Which functions take a struct by value and which take a pointer? Give the one-sentence rule you used.
2. Which structs contain padding, and how much?
3. Which structs would you pass as `const T *` in a real API, and why?
4. Which function is the first place a use-after-free could appear if someone later changes an array to a heap allocation?

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -fsanitize=address,undefined -fno-omit-frame-pointer main.c -o main
```

---

#### [Previous](./03-part.md) | [Next](../04-dynamic-memory/01-part.md)
