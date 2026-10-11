# Static Libraries and Make

Part 1 split a program into headers and sources and compiled everything in one `gcc` line. That stops scaling at three or four files: every tiny change recompiles the world, and every project re-copies `vec.c` instead of reusing it.

The two fixes are a static library (compile once, reuse many times) and `make` (recompile only what changed).

---

## Why a Library

Suppose three programs all need `vec.c`. Without a library you compile it three times:

```text
app1:  main1.c + vec.c  -> app1
app2:  main2.c + vec.c  -> app2
tests: test_vec.c + vec.c -> tests
```

With a static library you compile `vec.c` once into `libvec.a`, then link that archive wherever needed:

```text
vec.c -> vec.o -> libvec.a ------+
                                 +--> app1 (main1.o + libvec.a)
                                 +--> app2 (main2.o + libvec.a)
                                 +--> tests (test_vec.o + libvec.a)
```

An archive (`.a`) is just a bundle of `.o` files with an index. Linking against it copies only the object files your program actually uses into the executable. The result is self-contained: users of your program do not need `libvec.a` installed.

| Approach | Recompile vec.c when main changes? | Share vec without source? | Needs lib at runtime? |
| --- | --- | --- | --- |
| `gcc main.c vec.c -o app` | yes, always | no (source required) | no |
| Static lib `libvec.a` | no, link only | yes (`.a` + `.h` is enough) | no |
| Shared lib `libvec.so` (preview) | no | yes | yes, must be installed |

This part covers static libraries only. Shared libraries add versioning and loader paths (`LD_LIBRARY_PATH`, `rpath`) that deserve their own topic.

---

## Building a Static Library by Hand

Files from part 1, reused unchanged:

```c
/* vec.h */
#ifndef VEC_H
#define VEC_H

typedef struct {
    double x;
    double y;
} Vec2;

Vec2 vec_make(double x, double y);
Vec2 vec_add(Vec2 a, Vec2 b);
double vec_len(Vec2 v);

#endif /* VEC_H */
```

```c
/* vec.c */
#include <math.h>

#include "vec.h"

static double square(double x) {
    return x * x;
}

Vec2 vec_make(double x, double y) {
    Vec2 v = { x, y };
    return v;
}

Vec2 vec_add(Vec2 a, Vec2 b) {
    Vec2 r = { a.x + b.x, a.y + b.y };
    return r;
}

double vec_len(Vec2 v) {
    return sqrt(square(v.x) + square(v.y));
}
```

```c
/* main.c */
#include <stdio.h>

#include "vec.h"

int main(void) {
    Vec2 a = vec_make(1.0, 2.0);
    Vec2 b = vec_make(3.0, 4.0);
    Vec2 s = vec_add(a, b);

    printf("sum = (%.1f, %.1f)\n", s.x, s.y);
    printf("len = %.2f\n", vec_len(s));

    return 0;
}
```

Step 1 — compile the library source to an object file (no linking):

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g -c vec.c -o vec.o
```

Step 2 — bundle it into an archive with `ar`:

```bash
ar rcs libvec.a vec.o
```

| Flag | Meaning |
| --- | --- |
| `r` | insert (replace) files into the archive |
| `c` | create the archive silently if missing (without it, `ar` warns) |
| `s` | write the symbol index (so the linker finds `vec_add` fast) |

Inspect the result:

```bash
ar t libvec.a
nm libvec.a
```

```text
vec.o
```

```text
vec.o:
0000000000000000 t square
0000000000000025 T vec_add
0000000000000000 T vec_len
0000000000000056 T vec_make
```

Lowercase `t square` is the `static` helper (local). Uppercase `T` symbols are the public API the linker can resolve. If you do not see your function here with a `T`, linking against the library will fail with `undefined reference` — the archive simply does not contain it.

Step 3 — link the program against the library:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -L. -lvec -o app -lm
./app
```

```text
sum = (4.0, 6.0)
len = 7.21
```

| Flag | Meaning |
| --- | --- |
| `-L.` | "search for libraries in the current directory" (`.`; use `-Llib` for a `lib/` folder) |
| `-lvec` | "link `libvec.a` (or `libvec.so`)" — strip the `lib` prefix and the suffix |
| `-o app` | name the executable |
| `-lm` | math library, still needed because `vec_len` calls `sqrt` |

General form: `-l<name>` finds `lib<name>.a`. To link `libphone.a`, write `-lphone`. To link `libm.a` (math), write `-lm`.

---

## Library Link Order Matters

The GNU linker scans left to right and pulls from an archive only the objects that resolve a *currently* undefined symbol. Libraries therefore go **after** the files that use them:

```bash
# CORRECT: main.o needs vec_add, linker then finds it in libvec.a
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -L. -lvec -o app -lm
```

```bash
# WRONG on GNU ld: library scanned before anyone needs it, nothing pulled in
gcc -std=c11 -Wall -Wextra -pedantic -g -L. -lvec main.c -o app -lm
```

```text
/usr/bin/x86_64-linux-gnu-ld.bfd: /tmp/ccXXXXXX.o: in function `main':
main.c:(.text+0x2b): undefined reference to `vec_add'
main.c:(.text+0x52): undefined reference to `vec_make'
main.c:(.text+0x7e): undefined reference to `vec_len'
collect2: error: ld returned 1 exit status
```

The error is identical to forgetting `vec.c` entirely, which makes this bug hard to spot: the library is right there on the command line, just in the wrong position.

Rules:

1. Put `-l` flags at the **end** of the command line, after all `.c` and `.o` files.
2. If library A uses library B, put A first: `-lphone -lvec` (phone needs vec, so vec comes later).
3. `-lm` almost always goes last, because everybody may use math but math uses nobody.

```text
 link order (left to right):

 main.c  ->  -lphone  ->  -lvec  ->  -lm
  needs       needs vec     needs      needs
  phone_*     vec_*         sqrt       nothing
```

A chain example you will meet in the phonebook project (part 3): `main` uses `phonebook`, `phonebook` uses `vec`-style helpers. The link line reads `main.c -lphonebook -lvec -lm`, most-dependent first.

---

## Make: Rebuild Only What Changed

Recompiling everything on every edit wastes seconds now and minutes later. `make` compares timestamps: a target is rebuilt only if one of its prerequisites is newer.

Minimal `Makefile` for this project:

```make
CC = gcc
CFLAGS = -std=c11 -Wall -Wextra -pedantic -g
LDLIBS = -lm

app: main.o libvec.a
	$(CC) $(CFLAGS) main.o -L. -lvec $(LDLIBS) -o app

main.o: main.c vec.h
	$(CC) $(CFLAGS) -c main.c -o main.o

vec.o: vec.c vec.h
	$(CC) $(CFLAGS) -c vec.c -o vec.o

libvec.a: vec.o
	ar rcs libvec.a vec.o

clean:
	rm -f *.o *.a app

.PHONY: clean
```

Three concepts, nothing more:

| Concept | Meaning | Example above |
| --- | --- | --- |
| **Target: prerequisites** | "to build target, first make sure prerequisites exist, then run the recipe" | `app: main.o libvec.a` |
| **Recipe (TAB-indented)** | shell commands that build the target; must start with a TAB | `$(CC) $(CFLAGS) ... -o app` |
| **Variable** | `NAME = value`, used as `$(NAME)` | `CC = gcc`, `CFLAGS = ...` |

`CC` (which compiler) and `CFLAGS` (which compile flags) are conventions every C programmer recognizes. `LDLIBS` (which libraries) is the third standard one. Use these names so anyone can override from the command line:

```bash
make CC=clang CFLAGS="-std=c11 -Wall -g -O2"
```

How incremental rebuild works:

```bash
make          # builds vec.o, libvec.a, main.o, app
make          # second run: nothing to do
touch vec.c   # pretend vec.c changed
make          # rebuilds vec.o, libvec.a, app — but NOT main.o
```

```text
make: Nothing to be done for 'all'.        # second run output (up to date)
```

```text
gcc -std=c11 -Wall -Wextra -pedantic -g -c vec.c -o vec.o
ar rcs libvec.a vec.o
gcc -std=c11 -Wall -Wextra -pedantic -g main.o -L. -lvec -lm -o app
```

ASCII view of the dependency graph `make` follows:

```text
        app
       /   \
  main.o   libvec.a
   /  \        |
main.c vec.h  vec.o
              /   \
          vec.c  vec.h

change vec.h  -> everything rebuilds (both .o files depend on it)
change main.c -> only main.o + relink
change vec.c  -> vec.o, libvec.a, relink (main.o untouched)
```

That is why headers are listed as prerequisites (`main.o: main.c vec.h`): without them, editing `vec.h` would not trigger a rebuild and `main.o` would keep the old struct layout — a silent binary mismatch.

Two mandatory habits:

1. **Every recipe line starts with a TAB, not spaces.** `make` rejects spaces with the famously unhelpful message `missing separator`. Configure your editor to keep tabs in `Makefile`s.
2. **Mark non-file targets `.PHONY`.** `clean` is not a file; without `.PHONY: clean`, a file named `clean` in the directory would make `make clean` silently do nothing.

```bash
make clean
ls
```

```text
rm -f *.o *.a app
```

```text
main.c  Makefile  vec.c  vec.h
```

`clean` deletes everything the build created, leaving only sources. Anyone cloning the project runs `make` to build and `make clean` to reset — no stale objects, no "works on my machine because of an old `.o`".

---

## Direct Compile vs Library vs Make

| Method | Command | Rebuild cost after editing `main.c` | Best for |
| --- | --- | --- | --- |
| One-liner | `gcc main.c vec.c -o app -lm` | recompiles `vec.c` too | 1–2 files, quick test |
| Manual lib | `ar rcs libvec.a vec.o` + link | recompile `vec.c` by hand, relink | sharing code without source |
| `make` | `make` | recompiles only what changed | everything with 3+ files |

Start with the one-liner, graduate to `make` the moment you have a second `.c` file. The `Makefile` above is enough for the rest of this course.

A final habit worth building now: never commit generated files. After `make`, your directory holds `*.o`, `*.a`, and `app` alongside the sources — all of them rebuildable. A one-line `.gitignore` keeps the repository clean:

```text
*.o
*.a
app
```

With that in place, `git status` shows only `main.c`, `vec.c`, `vec.h`, and `Makefile`. Anyone cloning the project gets a working build from `make` alone, with no stale binaries.

---

## Bugs You Will Meet

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| 1 | `undefined reference to 'vec_add'` despite `-lvec` present | library placed before source on the link line | move `-lvec` after `main.c` / `main.o` |
| 2 | `cannot find -lvec` | wrong `-L` path, or archive named `vec.a` instead of `libvec.a` | `-L.` + file must be `libvec.a`; check with `ls` |
| 3 | `ar: creating libvec.a` warning | forgot `c` flag (`ar rs` instead of `ar rcs`) | use `ar rcs` |
| 4 | `missing separator. Stop.` | recipe indented with spaces | re-indent with a TAB |
| 5 | `make clean` does nothing, file `clean` exists | `clean` not marked `.PHONY` | add `.PHONY: clean` |
| 6 | changed `vec.h`, `make` rebuilt nothing | header missing from prerequisite list | `main.o: main.c vec.h`, `vec.o: vec.c vec.h` |
| 7 | stale crash after changing a struct | old `.o` not rebuilt, layout mismatch | `make clean && make` to rebuild from scratch |

---

## Checklist

1. Is each module compiled to `.o` with `-c` before archiving?
2. Is the archive built with `ar rcs lib<name>.a *.o` and does `nm` show your `T` symbols?
3. Does the link line put `-L<dir> -l<name>` after all sources/objects?
4. Are dependent libraries ordered most-dependent-first (`-lphone -lvec -lm`)?
5. Does the `Makefile` use `CC`, `CFLAGS`, `LDLIBS` and list headers as prerequisites?
6. Are all recipes TAB-indented and is `clean` marked `.PHONY`?
7. Does `make clean && make` build from scratch with zero warnings?

---

#### [Previous](./01-part.md) | [Next](./03-part.md)
