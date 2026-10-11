# Errors, Paths and Cleanup

Opening a file can fail. Writing can fail later than you expect. Paths resolve against a working directory you did not choose. This part is about handling all three: reading `errno` with `perror`/`strerror`, structuring every file function around a single `goto cleanup` exit with NULL-safe `fclose`, using temporary files safely, understanding relative paths versus the current working directory, and never ignoring the return value of `fwrite`/`fclose` when the write matters.

Compile every example with:

```bash
gcc -std=c11 -Wall -Wextra -pedantic -g main.c -o main
```

---

## 1. `errno`, `perror`, `strerror`

When a standard library call fails, it usually signals the failure in its return value (`NULL`, `-1`, `EOF`) and records the *reason* in the global `errno` (declared in `<errno.h>`). Two functions translate it:

```c
#include <stdio.h>    /* perror */
#include <string.h>   /* strerror */
#include <errno.h>    /* errno */

perror("fopen");                  /* prints "fopen: <reason>\n" to stderr */
fprintf(stderr, "%s\n", strerror(errno));  /* same reason as a string */
```

Important: `errno` is only meaningful **immediately after a documented failure**. Any successful library call may overwrite it. Capture or print it before doing anything else.

### The Errors You Will Actually Meet

| `errno` value | Meaning | Typical cause |
| --- | --- | --- |
| `ENOENT` | No such file or directory | typo in path, missing directory, wrong `cwd` |
| `EACCES` | Permission denied | read without `r`, write in read-only dir |
| `EEXIST` | File exists | `O_CREAT \| O_EXCL` race guard tripped |
| `ENOTDIR` | Not a directory | path uses `file.txt/sub` where `file.txt` is a file |
| `EISDIR` | Is a directory | `fopen("somedir", "w")` |
| `ENOSPC` | No space left | disk full — writes fail at flush/`fclose` time |
| `EROFS` | Read-only filesystem | USB stick / mount with `ro` |
| `EMFILE` / `ENFILE` | Too many open files | `fclose` missing on some path (leak) |
| `EINTR` | Interrupted call | signal arrived mid-call; retry if documented |
| `EIO` | I/O error | failing disk, unplugged drive |

Example: print both the code and the message:

```c
#include <errno.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    FILE *fp = fopen("/root/nope.txt", "r");
    if (fp == NULL) {
        int code = errno;
        perror("fopen");
        fprintf(stderr, "errno=%d (%s)\n", code, strerror(code));
        return 1;
    }
    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }
    return 0;
}
```

```text
fopen: No such file or directory
errno=2 (No such file or directory)
```

(On a root-owned box the same path may fail with `EACCES` instead — `Permission denied`, errno 13. The code path is identical; only the reason changes. That is exactly why you print it.)

Never do this:

```c
/* WRONG: errno without a failure means nothing */
fopen("a.txt", "r");
if (errno != 0) { ... }   /* meaningless: fopen may have succeeded */

/* WRONG: another call clobbers errno before you read it */
FILE *fp = fopen("a.txt", "r");
printf("trying...\n");    /* may reset errno */
perror("fopen");          /* possibly the wrong reason now */
```

---

## 2. The `goto cleanup` Pattern (NULL-Safe `fclose`)

Any function that opens a file has at least two exits: success and failure. Real functions have five. Without discipline, one of them forgets `fclose`, or closes twice, or returns without flushing.

The course pattern:

```c
static int do_work(const char *path) {
    FILE *fp = NULL;      /* 1. init to NULL */
    int rc = -1;          /* assume failure */

    fp = fopen(path, "r");
    if (fp == NULL) {
        perror("fopen");
        goto cleanup;     /* 2. every error jumps, never returns directly */
    }

    /* ... work, each failure: perror + goto cleanup ... */

    rc = 0;               /* 3. only reached when everything succeeded */

cleanup:                  /* 4. single exit */
    if (fp != NULL) {     /* 5. NULL-safe: fclose(NULL) is UB, guard it */
        if (fclose(fp) != 0) {
            perror("fclose");
            rc = -1;
        }
    }
    return rc;
}
```

Why each line exists:

1. `FILE *fp = NULL` — the cleanup block must be able to tell "never opened" from "opened". Uninitialized pointer + conditional close = undefined behavior.
2. `goto cleanup` instead of `return` — guarantees the close block runs on every path, including the middle of a ten-step sequence.
3. `rc = 0` at the end — success is explicit; forgetting it leaves the function reporting failure.
4. One label at the bottom — one place to audit for leaks instead of N scattered `fclose` calls.
5. `if (fp != NULL)` — `fclose(NULL)` is undefined behavior per the standard (glibc segfaults). The guard is not style; it is correctness.

With two resources the pattern scales by opening in order and releasing in reverse, each guarded:

```c
#include <errno.h>
#include <stdio.h>
#include <string.h>

static int copy_text(const char *src, const char *dst) {
    FILE *in = NULL;
    FILE *out = NULL;
    int rc = -1;
    char buf[1024];

    in = fopen(src, "r");
    if (in == NULL) {
        perror("fopen src");
        goto cleanup;
    }

    out = fopen(dst, "w");
    if (out == NULL) {
        perror("fopen dst");
        goto cleanup;
    }

    while (fgets(buf, sizeof buf, in) != NULL) {
        if (fputs(buf, out) == EOF) {
            perror("fputs");
            goto cleanup;
        }
    }
    if (ferror(in)) {
        perror("fgets");
        goto cleanup;
    }

    /* Critical write: the close is part of the write. */
    if (fclose(out) != 0) {
        perror("fclose dst");
        out = NULL;
        goto cleanup;
    }
    out = NULL;
    rc = 0;

cleanup:
    if (in != NULL) {
        fclose(in);
    }
    if (out != NULL) {
        /* We are already failing; report but keep the original error. */
        if (fclose(out) != 0) {
            perror("fclose dst (cleanup)");
        }
    }
    return rc;
}

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "usage: %s <src> <dst>\n", argv[0]);
        return 1;
    }
    return copy_text(argv[1], argv[2]) == 0 ? 0 : 1;
}
```

Control-flow picture:

```text
  fopen src --fail--> cleanup (in==NULL, out==NULL: close nothing)
     |
  fopen dst --fail--> cleanup (close in only)
     |
  loop fails -------> cleanup (close in + out)
     |
  fclose(out) fails-> out=NULL, cleanup (close in only, keep error)
     |
  success: rc=0 ---> cleanup (both already NULL / closed normally)
```

After a successful `fclose`, set the pointer to `NULL` so the cleanup block does not close it twice. Double `fclose` is undefined behavior (often a heap double-free inside stdio).

---

## 3. Temporary Files

Scratch files need unique names, safe creation, and guaranteed removal. Three levels:

| Function | What it does | Cleanup |
| --- | --- | --- |
| `tmpnam` | generates a name only | **never use** — race between name and open |
| `tmpfile()` | anonymous `FILE *`, auto-deleted on close | `fclose` (checked); file vanishes |
| `mkstemp(template)` | creates + opens `fd`, unique name | `close(fd)` + `unlink(path)` |

`tmpfile()` is the right default for "I need scratch space and nothing else needs the name":

```c
#include <stdio.h>

int main(void) {
    FILE *tmp = tmpfile();
    if (tmp == NULL) {
        perror("tmpfile");
        return 1;
    }

    fputs("scratch\n", tmp);
    rewind(tmp);

    char buf[64];
    while (fgets(buf, sizeof buf, tmp) != NULL) {
        fputs(buf, stdout);
    }
    if (ferror(tmp)) {
        perror("fgets tmp");
        fclose(tmp);
        return 1;
    }

    if (fclose(tmp) != 0) {
        perror("fclose tmp");
        return 1;
    }
    /* file is gone: no unlink needed */
    return 0;
}
```

```text
scratch
```

When another process must open the file by name (rare in this course), use `mkstemp`, which creates the file with `0600` permissions atomically — no window where an attacker can plant a symlink:

```c
#define _POSIX_C_SOURCE 200809L   /* mkstemp, fdopen declarations */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    char path[] = "/tmp/learn-c-XXXXXX";   /* must be mutable, XXXXXX suffix */
    int fd = mkstemp(path);
    if (fd == -1) {
        perror("mkstemp");
        return 1;
    }

    FILE *fp = fdopen(fd, "w+");
    if (fp == NULL) {
        perror("fdopen");
        close(fd);
        unlink(path);
        return 1;
    }

    fprintf(fp, "temp path was %s\n", path);
    rewind(fp);

    char buf[128];
    if (fgets(buf, sizeof buf, fp) != NULL) {
        fputs(buf, stdout);
    }

    int close_rc = fclose(fp);   /* also closes fd */
    unlink(path);                /* always remove, success or not */
    if (close_rc != 0) {
        perror("fclose");
        return 1;
    }
    return 0;
}
```

Rules: the template must be a writable array (not a string literal), exactly six trailing `X`s; `unlink` the path as soon as no one needs the name; never `tmpnam`/`tempnam` in new code.

---

## 4. Relative Paths vs the Current Working Directory

A relative path like `"data.txt"` or `"../out.bin"` resolves against the process **current working directory (cwd)** — the directory the program was *started from*, not the directory the binary or source lives in:

```text
  /home/user/proj/
  |-- main            <- binary
  |-- data.txt        <- sits next to binary
  |-- build/
      |-- main        <- you run THIS one

  $ ./build/main
  fopen("data.txt") looks in /home/user/proj/build/, NOT /home/user/proj/
  -> ENOENT even though the file "exists" next to the source
```

Demonstration:

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    char cwd[1024];
    if (getcwd(cwd, sizeof cwd) != NULL) {
        printf("cwd: %s\n", cwd);
    }

    FILE *fp = fopen("data.txt", "r");
    if (fp == NULL) {
        perror("fopen data.txt");
        return 1;
    }
    printf("opened data.txt relative to cwd\n");

    if (fclose(fp) != 0) {
        perror("fclose");
        return 1;
    }
    return 0;
}
```

```text
$ ./main
cwd: /home/user/proj
opened data.txt relative to cwd
$ ./build/main
cwd: /home/user/proj/build
fopen data.txt: No such file or directory
```

Practical rules:

1. Treat every relative path as "relative to wherever the user ran me". Document which directory your program expects.
2. For course exercises, run from the directory containing the data file, or pass the path as `argv[1]` so the caller decides.
3. Never build paths with unbounded `strcat` into a fixed buffer — use `snprintf` and check its return:

```c
char full[512];
int n = snprintf(full, sizeof full, "%s/%s", dir, name);
if (n < 0 || (size_t)n >= sizeof full) {
    fprintf(stderr, "path too long\n");
    goto cleanup;
}
```

4. `argv[0]` is not a reliable base directory (it can be a bare name found via `PATH`). For real applications, accept an explicit directory argument or environment variable.

---

## 5. Never Ignore `fwrite` / `fclose` on a Critical Write

stdio buffers output. `fwrite` usually copies into memory and returns success; the bytes reach the disk later, at the next buffer flush or at `fclose`. That means a full disk, a quota, or a dead network mount often surfaces **only in `fclose`**:

```text
  fwrite("...")        fclose(fp)
  +-----------+       +-----------+
  | memcpy to |       | flush to  | --fail--> EOF + errno=ENOSPC
  | buffer    | ----> | disk      |     (THE error appears here)
  | (fast,    |       | (slow,    |
  |  "ok")    |       |  real)    |
  +-----------+       +-----------+
```

Both halves must be checked when the data matters (configs, saves, journals — everything in the exercises):

```c
#include <stdio.h>

static int write_critical(const char *path, const char *text) {
    FILE *fp = NULL;
    int rc = -1;

    fp = fopen(path, "w");
    if (fp == NULL) {
        perror("fopen");
        goto cleanup;
    }

    size_t len = 0;
    while (text[len] != '\0') {
        len++;
    }
    if (len > 0 && fwrite(text, 1, len, fp) != len) {
        perror("fwrite");   /* short write: disk full or quota */
        goto cleanup;
    }

    if (fclose(fp) != 0) {
        perror("fclose");   /* flush failure surfaces here */
        fp = NULL;
        goto cleanup;
    }
    fp = NULL;
    rc = 0;

cleanup:
    if (fp != NULL) {
        fclose(fp);
    }
    return rc;
}

int main(void) {
    if (write_critical("important.txt", "do not lose me\n") != 0) {
        fprintf(stderr, "write failed: data NOT safe\n");
        return 1;
    }
    printf("write confirmed (fwrite + fclose both ok)\n");
    return 0;
}
```

```text
write confirmed (fwrite + fclose both ok)
```

For extra safety on truly critical data (the journal exercise), add `fflush(fp)` + `fsync(fileno(fp))` before `fclose` so the OS confirms the bytes are on stable storage. That is a one-line upgrade once this pattern is in place.

Note the `fwrite(text, 1, len, fp)` shape: size 1, count `len`, so the return value is bytes written and comparable to `len`. Either argument order works if you check consistently; this one makes short-write arithmetic trivial.

---

## Common Bugs

| Bug | Symptom | Fix |
| --- | --- | --- |
| reading `errno` after success | phantom error handling | only read `errno` after a documented failure, immediately |
| `perror` after another call | wrong message printed | `perror` first, or save `int code = errno` |
| `fclose(NULL)` | segfault (undefined behavior) | `FILE *fp = NULL` + `if (fp != NULL)` guard |
| double `fclose` | heap corruption in stdio | set `fp = NULL` right after a successful close |
| early `return` skipping close | descriptor leak (`EMFILE` eventually) | `goto cleanup`, single exit |
| `tmpnam` for temp files | symlink race, security hole | `tmpfile()` or `mkstemp` + `unlink` |
| relative path vs binary dir | `ENOENT` for a file that "exists" | run from the data dir or pass path via `argv` |
| ignoring `fwrite` return | truncated output looks fine | compare count, `perror` on short write |
| ignoring `fclose` after write | full disk reported as success | `if (fclose(fp) != 0) perror(...)` and fail |
| unbounded path `strcat` | stack overflow on long names | `snprintf` + length check |

---

## Checklist

- [ ] `errno` read only right after a failure; `perror`/`strerror` used for the message.
- [ ] Common codes recognized: `ENOENT` (missing), `EACCES` (permission), `ENOSPC` (full), `EISDIR`/`ENOTDIR` (wrong kind).
- [ ] Every file function initializes `FILE *` to `NULL` and exits via one `cleanup` label.
- [ ] Cleanup block guards with `if (fp != NULL)`; pointers nulled after successful close.
- [ ] Scratch files use `tmpfile()` (anonymous) or `mkstemp` + `unlink`; `tmpnam` absent.
- [ ] Relative paths documented as cwd-relative; programs print or accept the expected directory.
- [ ] Paths built with bounded `snprintf`, length checked.
- [ ] Critical writes check **both** `fwrite` count and `fclose` return before reporting success.
- [ ] Everything builds with `gcc -std=c11 -Wall -Wextra -pedantic -g` with zero warnings.

---

#### [Previous](./02-part.md) | [Next](./exercises.md)
