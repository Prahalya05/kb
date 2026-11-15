# Exercises — graded by level, with solutions and three mini projects

How to use this file: attempt each exercise **before** opening the solution. Solutions that are programs were compiled and run on the test machine (GCC 13.3 / GCC 14.2, glibc 2.39, x86-64 Linux, November 2026); their `// build:`/`// run:` lines are the exact commands, and the outputs shown are real. Every solution is written in the track's style (`snake_case`, `res`/`ans`, `l`/`r`, every block commented).

Difficulty: ★ quick check · ★★ needs thought · ★★★ project-sized.

---

## Level 1 — Beginner (lessons B1–B7)

### E1 ★ — Predict the output (types and conversions)

What does each line print on x86-64 Linux, and why?

```c
printf("%d\n", 7 / 2 * 2);
printf("%d\n", -7 % 3);
printf("%d\n", (unsigned char)300);
printf("%d\n", -1 < 0u);
printf("%zu\n", sizeof(long));
```

<details>
<summary>Solution</summary>

1. `6` — `7 / 2` is integer division (3), then `* 2`.
2. `-1` — the remainder takes the sign of the dividend (truncation toward zero, B2).
3. `44` — 300 mod 256 (conversion to an unsigned type is modular).
4. `0` — `-1` is converted to `unsigned int` (`UINT_MAX`) by the usual arithmetic conversions, so the comparison is false.
5. `8` — `long` is 64-bit in the LP64 model (it is 4 on 64-bit Windows).

</details>

### E2 ★ — Reverse a string in place

Write `void reverse(char *s)` that reverses a C string in place, handling the empty string.

<details>
<summary>Solution</summary>

```c
// file: e2_reverse.c
// build: gcc -std=c17 -Wall -Wextra -o e2_reverse e2_reverse.c
// run: ./e2_reverse

#include <stdio.h>
#include <string.h>

// reverses s in place using two indices moving toward each other
void reverse(char *s)
{
    size_t n = strlen(s);
    if (n < 2) return;                 // empty or single char: nothing to do
    for (size_t l = 0, r = n - 1; l < r; l++, r--) {
        char tmp = s[l];
        s[l] = s[r];
        s[r] = tmp;
    }
}

int main(void)
{
    char a[] = "coimbatore", b[] = "x", c[] = "";
    reverse(a); reverse(b); reverse(c);
    printf("%s|%s|%s|\n", a, b, c);
    return 0;
}
```

Output:

```text
erotabmioc|x||
```

The `n < 2` guard matters: for `n == 0`, `n - 1` would wrap to `SIZE_MAX` (unsigned) and the loop would run off the array.

</details>

### E3 ★★ — Find the bugs (no code to write)

```c
char *greet(const char *name)
{
    char buf[16];
    sprintf(buf, "hello %s", name);
    return buf;
}
```

<details>
<summary>Solution</summary>

1. **Buffer overflow:** `sprintf` doesn't know `buf` holds 16 bytes; a name longer than 9 characters overflows it (B4). Use `snprintf(buf, sizeof buf, ...)`.
2. **Dangling pointer:** `buf` has automatic storage; returning it returns the address of a dead object (B6). GCC warns with `-Wreturn-local-addr` and may even return `NULL`.
3. Fix: have the caller pass a buffer and its size (`int greet(char *out, size_t n, const char *name)`), or return a heap string the caller must `free` (I2).

</details>

### E4 ★★ — Swap two struct fields through pointers

Write `void swap_marks(struct student *a, struct student *b)` that swaps only the `cgpa` fields. Explain why `void swap_marks(struct student a, struct student b)` cannot work.

<details>
<summary>Solution</summary>

```c
struct student { int roll; double cgpa; };

// swaps only the cgpa members of two students
void swap_marks(struct student *a, struct student *b)
{
    double tmp = a->cgpa;     // -> reads a member through a pointer
    a->cgpa = b->cgpa;
    b->cgpa = tmp;
}
```

The by-value version receives copies of both structs (pass by value, B3/B7); swapping the copies has no effect on the caller's objects.

</details>

### E5 ★★ — Print the bits of an integer

Write `print_bits(unsigned v)` printing 32 bits MSB-first in groups of 8.

<details>
<summary>Solution</summary>

```c
// file: e5_bits.c
// build: gcc -std=c17 -Wall -Wextra -o e5_bits e5_bits.c
// run: ./e5_bits

#include <stdio.h>
#include <limits.h>

// prints the 32 bits of v, most significant first, with a space every 8 bits
void print_bits(unsigned v)
{
    for (int i = (int)(sizeof v * CHAR_BIT) - 1; i >= 0; i--) {
        putchar((v >> i) & 1u ? '1' : '0');      // unsigned shift: always defined for i < width
        if (i % 8 == 0 && i != 0) putchar(' ');
    }
    putchar('\n');
}

int main(void)
{
    print_bits(5u);
    print_bits((unsigned)-5);          // conversion to unsigned is defined: modulo 2^32
    return 0;
}
```

Output:

```text
00000000 00000000 00000000 00000101
11111111 11111111 11111111 11111011
```

The second line is two's complement of −5 viewed as unsigned (B2).

</details>

---

## Level 2 — Intermediate (lessons I1–I8)

### E6 ★★ — Decode declarations

Explain: (a) `char **argv`, (b) `int (*rows)[4]`, (c) `void (*handlers[3])(int)`, (d) `const char *const names[]`.

<details>
<summary>Solution</summary>

(a) pointer to pointer to `char` (an array of strings when used as `argv`). (b) pointer to an array of 4 `int` (one row of a `[N][4]` matrix; `rows + 1` advances 16 bytes). (c) array of 3 pointers to functions taking `int` and returning `void` (a signal-handler-style dispatch table). (d) array of `const` pointers to `const char` — neither the pointers nor the characters may be modified (I1).

</details>

### E7 ★★ — A heap matrix with one allocation and overflow checks

<details>
<summary>Solution</summary>

```c
// file: e7_matrix.c
// build: gcc -std=c17 -Wall -Wextra -g -fsanitize=address,undefined -o e7_matrix e7_matrix.c
// run: ./e7_matrix

#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

// a rows x cols matrix in ONE contiguous allocation (cache-friendly, one free)
typedef struct {
    size_t rows, cols;
    double *data;
} matrix;

// returns 0 on success; refuses sizes whose byte count would overflow
int matrix_init(matrix *m, size_t rows, size_t cols)
{
    if (cols != 0 && rows > SIZE_MAX / cols / sizeof(double))
        return -1;
    m->data = calloc(rows * cols, sizeof *m->data);   // zeroed
    if (m->data == NULL) return -1;
    m->rows = rows;
    m->cols = cols;
    return 0;
}

// element (r, c) lives at r*cols + c (row-major)
double *at(matrix *m, size_t r, size_t c) { return &m->data[r * m->cols + c]; }

void matrix_free(matrix *m)
{
    free(m->data);
    m->data = NULL;
    m->rows = m->cols = 0;
}

int main(void)
{
    matrix m;
    if (matrix_init(&m, 3, 4) != 0) return 1;
    for (size_t r = 0; r < m.rows; r++)
        for (size_t c = 0; c < m.cols; c++)
            *at(&m, r, c) = (double)(r * 10 + c);
    printf("m[2][3] = %.0f, overflow refused: %s\n", *at(&m, 2, 3),
           matrix_init(&(matrix){0}, SIZE_MAX / 2, 4) != 0 ? "yes" : "no");
    matrix_free(&m);
    return 0;
}
```

Output (clean under ASan + UBSan):

```text
m[2][3] = 23, overflow refused: yes
```

One contiguous block (instead of an array of row pointers) means one `free`, no partial-allocation cleanup, and row-major cache-friendly access (P5).

</details>

### E8 ★★ — Classify each as UB, unspecified, implementation-defined, or fine

(a) `int x = INT_MAX; x++;` (b) `unsigned u = 0; u--;` (c) `printf("%d %d", f(), g());` — which runs first? (d) `int i = 0; i = i++;` (e) `char c = 200;` (f) `int a[4]; int *p = a + 4;` (g) `int a[4]; int *p = a + 5;`

<details>
<summary>Solution</summary>

(a) UB (signed overflow). (b) Fine — unsigned wraps to `UINT_MAX`. (c) Unspecified order of evaluation of arguments. (d) UB (unsequenced modifications of `i`). (e) Implementation-defined (out-of-range conversion to signed `char`, which is signed on x86-64). (f) Fine — one-past-the-end may be formed (not dereferenced). (g) UB — pointer arithmetic beyond one-past-the-end (I3).

</details>

### E9 ★★★ — An `ARRAY_LEN` macro that refuses pointers

Write `ARRAY_LEN(a)` that gives the element count of an array and fails **at compile time** if given a pointer (GNU extensions allowed).

<details>
<summary>Solution</summary>

```c
// file: e9_countof.c
// build: gcc -std=c17 -Wall -Wextra -o e9_countof e9_countof.c
// run: ./e9_countof

#include <stdio.h>

// GNU C: __builtin_types_compatible_p tells whether a is an array or a pointer.
// If a is a pointer, the static assertion inside fails at COMPILE time.
#define IS_ARRAY(a) (!__builtin_types_compatible_p(__typeof__(a), __typeof__(&(a)[0])))
#define ARRAY_LEN(a) \
    (sizeof(a) / sizeof((a)[0]) + 0 * sizeof(struct { _Static_assert(IS_ARRAY(a), "ARRAY_LEN on a pointer"); int x; }))

int main(void)
{
    int marks[7] = {0};
    printf("ARRAY_LEN(marks) = %zu\n", ARRAY_LEN(marks));
    // int *p = marks; ARRAY_LEN(p);   // uncomment: compile-time error "ARRAY_LEN on a pointer"
    return 0;
}
```

Output:

```text
ARRAY_LEN(marks) = 7
```

Uncommenting the pointer line produced, on the test machine: `error: static assertion failed: "ARRAY_LEN on a pointer"`. The trick: an array `a` and `&a[0]` have different types; a pointer and `&p[0]` have the same type. (C2y adds a standard `_Countof` operator that does this properly — lesson P5.)

</details>

### E10 ★★ — Write a Makefile

Three files `main.c`, `stack.c`, `stack.h`. Write a Makefile that rebuilds only what changed, tracks header dependencies automatically, and has a `clean` target.

<details>
<summary>Solution</summary>

```makefile
CC       := gcc
CFLAGS   := -std=c17 -Wall -Wextra -g -O2
DEPFLAGS := -MMD -MP
OBJS     := main.o stack.o

app: $(OBJS)
	$(CC) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) $(DEPFLAGS) -c $< -o $@

.PHONY: clean
clean:
	rm -f app $(OBJS) $(OBJS:.o=.d)

-include $(OBJS:.o=.d)
```

Key points (I6): tabs before recipes; `-MMD -MP` + `-include` so editing `stack.h` rebuilds both objects; `.PHONY`; `CC :=` not `CC ?=`. This is the same Makefile pattern that was tested in lesson I6 and appendix.md.

</details>

### E11 ★★ — GDB drill

A program crashes in `__strlen_avx2`. List the GDB commands you would run, in order, and what you are looking for.

<details>
<summary>Solution</summary>

`gdb ./prog` → `run` → `bt` (the crash is in libc; find the first frame in your code) → `frame N` / `up` to that frame → `print` the pointer passed to `strlen` (likely NULL, freed, or a non-terminated buffer) → `info locals`, `ptype` → if the pointer was valid earlier, set `watch` on it or a breakpoint where it is assigned and `run` again. If memory corruption is suspected, switch to ASan or Valgrind (I7, I8).

</details>

---

## Level 3 — Advanced (lessons A1–A6)

### E12 ★★ — Predict `sizeof` and offsets

`struct s { char a; double b; short c; char d; };` on x86-64: give each member's offset and `sizeof`. Then reorder to minimise size.

<details>
<summary>Solution</summary>

`a` 0; padding 1–7; `b` 8; `c` 16; `d` 18; tail padding to a multiple of 8 → `sizeof` = 24. Reordered `{ double b; short c; char a; char d; }`: 8 + 2 + 1 + 1 = 12 → rounded to 16 (A1).

</details>

### E13 ★★ — Overflow-safe integer parsing with C23 `ckd_*`

<details>
<summary>Solution</summary>

```c
// file: e12_ckd.c
// build: gcc-14 -std=c23 -Wall -Wextra -o e12_ckd e12_ckd.c
// run: ./e12_ckd

#include <stdio.h>
#include <stdckdint.h>
#include <stdlib.h>

// parses a non-negative decimal int, refusing overflow; returns 0 on success
int parse_nonneg(const char *s, int *out)
{
    int res = 0;
    if (*s == '\0') return -1;
    for (; *s; s++) {
        if (*s < '0' || *s > '9') return -1;
        if (ckd_mul(&res, res, 10) || ckd_add(&res, res, *s - '0'))
            return -1;                         // would not fit in int
    }
    *out = res;
    return 0;
}

int main(void)
{
    const char *tests[] = {"2147483647", "2147483648", "42", "4x2", ""};
    for (int i = 0; i < 5; i++) {
        int v;
        if (parse_nonneg(tests[i], &v) == 0) printf("\"%s\" -> %d\n", tests[i], v);
        else printf("\"%s\" -> rejected\n", tests[i]);
    }
    return 0;
}
```

Output:

```text
"2147483647" -> 2147483647
"2147483648" -> rejected
"42" -> 42
"4x2" -> rejected
"" -> rejected
```

</details>

### E14 ★★★ — A spinlock from `atomic_flag`

Implement `spin_lock`/`spin_unlock` with `atomic_flag` and the correct memory orders; prove it works with a counter.

<details>
<summary>Solution</summary>

```c
// file: e13_spinlock.c
// build: gcc -std=c17 -O2 -Wall -Wextra -pthread -o e13_spinlock e13_spinlock.c
// run: ./e13_spinlock

#include <stdio.h>
#include <stdatomic.h>
#include <pthread.h>

// a minimal spinlock built on atomic_flag (the one type guaranteed lock-free)
static atomic_flag lock = ATOMIC_FLAG_INIT;
static long counter = 0;               // protected by lock

static void spin_lock(void)
{
    // test_and_set returns the OLD value: loop until we are the one who changed false -> true
    while (atomic_flag_test_and_set_explicit(&lock, memory_order_acquire))
        ;                              // busy-wait (fine only for very short critical sections)
}

static void spin_unlock(void)
{
    atomic_flag_clear_explicit(&lock, memory_order_release);
}

static void *work(void *arg)
{
    (void)arg;
    for (int i = 0; i < 1000000; i++) {
        spin_lock();
        counter++;                     // plain access is safe: acquire/release order it
        spin_unlock();
    }
    return NULL;
}

int main(void)
{
    pthread_t a, b;
    pthread_create(&a, NULL, work, NULL);
    pthread_create(&b, NULL, work, NULL);
    pthread_join(a, NULL);
    pthread_join(b, NULL);
    printf("counter = %ld (expected 2000000)\n", counter);
    return 0;
}
```

Output:

```text
counter = 2000000 (expected 2000000)
```

Acquire on lock / release on unlock creates the happens-before edges that make the plain `counter++` race-free (A6). For anything but tiny critical sections, prefer a mutex: spinning wastes CPU, especially when threads outnumber cores.

</details>

### E15 ★★ — Explain the optimiser

Why may GCC compile `int f(int *p) { int v = *p; if (p == NULL) return 0; return v; }` into just `return *p;`?

<details>
<summary>Solution</summary>

Dereferencing a null pointer is UB, so after `*p` executes the compiler may assume `p != NULL`; the check is dead code (`-fdelete-null-pointer-checks`, on by default). Fix: test before dereferencing. Systems that must keep such checks compile with `-fno-delete-null-pointer-checks` (the Linux kernel does) (I3, A4).

</details>

---

## Level 4 — Pro (lessons P1–P5)

### E16 ★★★ — Run a command with a timeout

Write `run_with_timeout(argv, limit_ms)` returning the exit status, `-SIG` for a signal death, or a timeout code.

<details>
<summary>Solution</summary>

```c
// file: e15_timeout.c
// build: gcc -std=c17 -Wall -Wextra -o e15_timeout e15_timeout.c
// run: ./e15_timeout

#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <signal.h>
#include <time.h>
#include <unistd.h>
#include <sys/wait.h>

// runs argv in a child; kills it if it runs longer than limit_ms. Returns a description code:
// >= 0 exit status, -SIG if killed by a signal, -1000 on timeout, -1 on error
int run_with_timeout(char *const argv[], long limit_ms)
{
    pid_t pid = fork();
    if (pid < 0) return -1;
    if (pid == 0) { execvp(argv[0], argv); _exit(127); }

    struct timespec nap = {0, 10 * 1000 * 1000};          // poll every 10 ms
    for (long waited = 0; ; waited += 10) {
        int st;
        pid_t r = waitpid(pid, &st, WNOHANG);             // non-blocking check
        if (r == pid) {
            if (WIFEXITED(st)) return WEXITSTATUS(st);
            if (WIFSIGNALED(st)) return -WTERMSIG(st);
            return -1;
        }
        if (r < 0) return -1;
        if (waited >= limit_ms) {                          // too slow: kill and reap
            kill(pid, SIGKILL);
            waitpid(pid, NULL, 0);
            return -1000;
        }
        nanosleep(&nap, NULL);
    }
}

int main(void)
{
    char *fast[] = {"true", NULL};
    char *slow[] = {"sleep", "5", NULL};
    char *crash[] = {"sh", "-c", "kill -ABRT $$", NULL};
    printf("true      -> %d\n", run_with_timeout(fast, 500));
    printf("sleep 5   -> %d (timeout)\n", run_with_timeout(slow, 300));
    printf("SIGABRT   -> %d\n", run_with_timeout(crash, 500));
    return 0;
}
```

Output:

```text
true      -> 0
sleep 5   -> -1000 (timeout)
SIGABRT   -> -6
```

Polling with `WNOHANG` is simple and portable; production code would use `signalfd`/`pidfd` or `SIGCHLD` with a self-pipe to avoid the 10 ms polling granularity (P1).

</details>

### E17 ★★ — Why does this hang?

```c
int fds[2]; pipe(fds);
if (fork() == 0) { dup2(fds[0], 0); execlp("wc", "wc", "-l", NULL); }
write(fds[1], "a\nb\n", 4);
wait(NULL);
```

<details>
<summary>Solution</summary>

`wc` waits for end-of-file on its stdin, which happens only when **every** write end of the pipe is closed. The child still holds `fds[1]` (inherited, never closed), and the parent never closes its `fds[1]` before `wait`. Fix: in the child `close(fds[0]); close(fds[1]);` after `dup2`; in the parent `close(fds[0])`, then after writing `close(fds[1])`, then `wait` (P2).

</details>

### E18 ★★ — Interposition thought experiment

You `LD_PRELOAD` a library defining `malloc` that counts calls, and the count is lower than the number of `malloc` calls in the source. Give two reasons.

<details>
<summary>Solution</summary>

(1) The optimiser removed allocations whose memory was never used (`malloc`/`free` pairs elided at `-O2`, observed in P3). (2) Calls made inside a shared library to its own hidden/internal symbols, or inlined/statically linked allocation paths, don't go through the interposable dynamic symbol. (Also: a statically linked binary ignores `LD_PRELOAD` entirely; setuid binaries restrict it.)

</details>

### E19 ★★ — Performance reasoning

Summing a `struct { double x, y, z; int tag; }` array's `x` fields is slower than summing a separate `double xs[]` array of the same length. Why, and what is the layout called?

<details>
<summary>Solution</summary>

Each struct is 32 bytes (with padding); summing `x` uses 8 of every 32 bytes fetched, so 4× more cache lines are loaded than for a dense `double` array, and the strided access vectorises worse. Splitting fields into separate arrays is the **structure-of-arrays** (SoA) layout, vs **array-of-structures** (AoS) (A1, P5).

</details>

---

## Mini projects

### Project 1 ★★★ — `wordfreq`: a word-frequency counter with your own hash table

**Spec:** read text from stdin, count case-insensitive words (letters only), print the top N by count (ties alphabetical). Requirements: chained hash table with FNV-1a hashing and resizing at load factor 0.75; the table owns all strings; zero leaks and zero sanitizer reports. Exercises lessons B4, B7, I1 (`qsort`), I2 (ownership), I8.

<details>
<summary>Reference solution</summary>

```c
// file: wordfreq.c
// build: gcc -std=c17 -O2 -Wall -Wextra -g -fsanitize=address,undefined -o wordfreq wordfreq.c
// run: printf 'the cat and the hat\nThe CAT sat; the end.\n' | ./wordfreq 3

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <stdint.h>

// one key/count pair in a chained hash table; the table OWNS each word string
typedef struct entry {
    char *word;
    long count;
    struct entry *next;
} entry;

typedef struct {
    entry **buckets;
    size_t nbuckets;     // always a power of two
    size_t size;         // number of distinct words
} table;

// FNV-1a 64-bit hash: simple, decent distribution for short strings
static uint64_t fnv1a(const char *s)
{
    uint64_t h = 14695981039346656037ULL;   // FNV-1a 64-bit offset basis
    for (; *s; s++) {
        h ^= (unsigned char)*s;
        h *= 1099511628211ULL;
    }
    return h;
}

static int table_init(table *t, size_t nbuckets)
{
    t->buckets = calloc(nbuckets, sizeof *t->buckets);
    t->nbuckets = nbuckets;
    t->size = 0;
    return t->buckets ? 0 : -1;
}

// doubles the bucket array and re-links every entry (no string copies needed)
static int table_grow(table *t)
{
    size_t nb = t->nbuckets * 2;
    entry **nbk = calloc(nb, sizeof *nbk);
    if (nbk == NULL) return -1;
    for (size_t i = 0; i < t->nbuckets; i++) {
        entry *e = t->buckets[i];
        while (e) {
            entry *next = e->next;
            size_t j = fnv1a(e->word) & (nb - 1);
            e->next = nbk[j];
            nbk[j] = e;
            e = next;
        }
    }
    free(t->buckets);
    t->buckets = nbk;
    t->nbuckets = nb;
    return 0;
}

// adds one occurrence of word; copies the word on first sight
static int table_add(table *t, const char *word)
{
    size_t i = fnv1a(word) & (t->nbuckets - 1);
    for (entry *e = t->buckets[i]; e; e = e->next)
        if (strcmp(e->word, word) == 0) { e->count++; return 0; }
    entry *e = malloc(sizeof *e);
    char *copy = malloc(strlen(word) + 1);
    if (!e || !copy) { free(e); free(copy); return -1; }
    strcpy(copy, word);                      // safe: copy has exactly strlen + 1 bytes
    e->word = copy;
    e->count = 1;
    e->next = t->buckets[i];
    t->buckets[i] = e;
    if (++t->size > t->nbuckets * 3 / 4)     // keep load factor <= 0.75
        return table_grow(t);
    return 0;
}

static void table_free(table *t)
{
    for (size_t i = 0; i < t->nbuckets; i++) {
        entry *e = t->buckets[i];
        while (e) { entry *next = e->next; free(e->word); free(e); e = next; }
    }
    free(t->buckets);
}

// qsort comparator: higher count first, then alphabetical
static int by_count(const void *a, const void *b)
{
    const entry *x = *(const entry *const *)a, *y = *(const entry *const *)b;
    if (x->count != y->count) return (y->count > x->count) - (y->count < x->count);
    return strcmp(x->word, y->word);
}

int main(int argc, char *argv[])
{
    long top = argc > 1 ? atol(argv[1]) : 10;
    table t;
    if (table_init(&t, 8) != 0) return 1;

    char word[64];
    size_t len = 0;
    int c;
    // read characters; letters build a lowercase word, anything else ends it
    while ((c = getchar()) != EOF) {
        if (isalpha(c)) {
            if (len < sizeof word - 1) word[len++] = (char)tolower(c);
        } else if (len > 0) {
            word[len] = '\0';
            if (table_add(&t, word) != 0) { table_free(&t); return 1; }
            len = 0;
        }
    }
    if (len > 0) { word[len] = '\0'; if (table_add(&t, word) != 0) { table_free(&t); return 1; } }

    // collect pointers to entries, sort them, print the top N
    entry **all = malloc(t.size * sizeof *all);
    if (t.size && !all) { table_free(&t); return 1; }
    size_t k = 0;
    for (size_t i = 0; i < t.nbuckets; i++)
        for (entry *e = t.buckets[i]; e; e = e->next) all[k++] = e;
    qsort(all, k, sizeof *all, by_count);
    for (size_t i = 0; i < k && (long)i < top; i++)
        printf("%-8s %ld\n", all[i]->word, all[i]->count);
    printf("(%zu distinct words, %zu buckets)\n", t.size, t.nbuckets);
    free(all);
    table_free(&t);
    return 0;
}
```

Output (clean under ASan + UBSan, including LeakSanitizer at exit):

```text
the      4
cat      2
and      1
(6 distinct words, 8 buckets)
```

Extensions: read a large text and compare speed against `tr -cs A-Za-z '\n' | sort | uniq -c | sort -rn`; replace chaining with open addressing and measure (P5).

</details>

### Project 2 ★★★ — `minish`: a tiny shell with pipelines and exit statuses

**Spec:** read lines; run simple commands and two-stage pipelines `a | b`; support double-quoted words with `\n`; print `[status N]` after each command using shell conventions (127 for not found, 128+signal for signal deaths); `exit` quits. Exercises P1 (fork/exec/wait, buffering), P2 (pipes, `dup2`, closing ends), B4 (string parsing).

<details>
<summary>Reference solution</summary>

```c
// file: minish.c
// build: gcc -std=c17 -Wall -Wextra -g -fsanitize=address,undefined -o minish minish.c
// run: printf 'echo hello world\nprintf "b\\na\\nc\\n" | sort\nfalse\nno_such_cmd_xyz\nexit\n' | ./minish

#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

#define MAX_ARGS 32

// splits line in place into words; handles "double quoted" words and the escape \n inside them
static int split(char *line, char *argv[], int max)
{
    int argc = 0;
    char *p = line;
    while (*p && argc < max - 1) {
        while (*p == ' ' || *p == '\t') p++;            // skip blanks
        if (*p == '\0') break;
        if (*p == '"') {                                // quoted word: copy until the closing quote
            char *out = ++p;
            argv[argc++] = out;
            while (*p && *p != '"') {
                if (p[0] == '\\' && p[1] == 'n') { *out++ = '\n'; p += 2; }
                else *out++ = *p++;
            }
            if (*p == '"') p++;
            *out = '\0';
        } else {                                        // plain word: up to the next blank
            argv[argc++] = p;
            while (*p && *p != ' ' && *p != '\t') p++;
            if (*p) *p++ = '\0';
        }
    }
    argv[argc] = NULL;                                  // execvp needs a NULL-terminated array
    return argc;
}

// runs one command in a child; returns a shell-style status (128+sig for signals)
static int wait_status(pid_t pid)
{
    int st;
    if (waitpid(pid, &st, 0) < 0) return 1;
    if (WIFEXITED(st)) return WEXITSTATUS(st);
    if (WIFSIGNALED(st)) return 128 + WTERMSIG(st);
    return 1;
}

// executes "cmd" or "left | right"; returns the status of the last command
static int run_line(char *line)
{
    char *bar = strchr(line, '|');
    char *left_argv[MAX_ARGS], *right_argv[MAX_ARGS];

    if (bar == NULL) {                                  // simple command
        if (split(line, left_argv, MAX_ARGS) == 0) return 0;
        fflush(stdout);                                 // avoid duplicated buffered output (lesson P1)
        pid_t pid = fork();
        if (pid < 0) { perror("fork"); return 1; }
        if (pid == 0) { execvp(left_argv[0], left_argv); perror(left_argv[0]); _exit(127); }
        return wait_status(pid);
    }

    *bar = '\0';                                        // two-stage pipeline
    if (split(line, left_argv, MAX_ARGS) == 0 || split(bar + 1, right_argv, MAX_ARGS) == 0) {
        fprintf(stderr, "minish: empty pipeline stage\n");
        return 2;
    }
    int fds[2];
    if (pipe(fds) != 0) { perror("pipe"); return 1; }
    fflush(stdout);
    pid_t a = fork();
    if (a == 0) {
        dup2(fds[1], STDOUT_FILENO); close(fds[0]); close(fds[1]);
        execvp(left_argv[0], left_argv); perror(left_argv[0]); _exit(127);
    }
    pid_t b = fork();
    if (b == 0) {
        dup2(fds[0], STDIN_FILENO); close(fds[0]); close(fds[1]);
        execvp(right_argv[0], right_argv); perror(right_argv[0]); _exit(127);
    }
    close(fds[0]); close(fds[1]);                       // parent must close both ends (lesson P2)
    wait_status(a);
    return wait_status(b);                              // a pipeline's status is its last command's
}

int main(void)
{
    char line[1024];
    int last = 0;
    while (1) {
        if (isatty(STDIN_FILENO)) { printf("minish$ "); fflush(stdout); }
        if (fgets(line, sizeof line, stdin) == NULL) break;   // EOF ends the shell
        line[strcspn(line, "\n")] = '\0';
        if (strcmp(line, "exit") == 0) break;
        last = run_line(line);
        printf("[status %d]\n", last);
        fflush(stdout);
    }
    return last;
}
```

Output:

```text
hello world
[status 0]
a
b
c
[status 0]
[status 1]
no_such_cmd_xyz: No such file or directory
[status 127]
```

Extensions: `cd` as a built-in (why can't it be an external program?), `>` redirection with `open` + `dup2`, N-stage pipelines, `SIGINT` handling so Ctrl-C kills the child but not the shell.

</details>

### Project 3 ★★★ — `difffuzz`: differential fuzzing of a hand-written integer parser

**Spec:** implement `my_parse` (optional `-`, digits only, full `int` range, reject overflow) and test it against a `strtol`-based reference on millions of generated inputs biased toward edge cases; build with ASan + UBSan; on mismatch print the input. The reference implementation below contains a **planted bug**; your job is to (1) watch the tools catch it, (2) fix it, (3) confirm 2,000,000 agreeing tests. Exercises I3, A4, A5, P4 — and is a miniature of the *compiler-fuzz* architecture (generator → two implementations → oracle → sanitizer).

<details>
<summary>Reference solution (with the planted bug)</summary>

```c
// file: difffuzz.c
// build: gcc -std=c17 -O1 -g -Wall -Wextra -fsanitize=address,undefined -fno-sanitize-recover=undefined -o difffuzz difffuzz.c
// run: ./difffuzz 2>&1 | tail -3

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <limits.h>
#include <stdbool.h>

// REFERENCE: strtol-based parse of a whole decimal int; false if invalid or out of range
static bool ref_parse(const char *s, int *out)
{
    if (*s == '\0' || *s == ' ' || *s == '+') return false;   // our spec: optional '-', digits only
    char *end;
    errno = 0;
    long v = strtol(s, &end, 10);
    if (errno == ERANGE || *end != '\0' || v < INT_MIN || v > INT_MAX) return false;
    if (end == s) return false;
    *out = (int)v;
    return true;
}

// CANDIDATE: a hand-written parser with the same spec (version 1 contains a planted bug)
static bool my_parse(const char *s, int *out)
{
    bool neg = false;
    if (*s == '-') { neg = true; s++; }
    if (*s == '\0') return false;
    int res = 0;
    for (; *s; s++) {
        if (*s < '0' || *s > '9') return false;
        int d = *s - '0';
        // accumulate as a NEGATIVE number so INT_MIN is representable
        if (res < (INT_MIN + d) / 10 - 1) return false;      // PLANTED BUG: the "- 1" lets res*10 - d overflow
        res = res * 10 - d;
    }
    if (!neg) {
        if (res == INT_MIN) return false;                     // +2147483648 does not fit
        res = -res;
    }
    *out = res;
    return true;
}

// random test string: biased toward edge cases (long digit runs, a leading '-', junk)
static void gen(char *buf, size_t cap)
{
    static const char *edges[] = {"2147483647", "2147483648", "-2147483648", "-2147483649", "0", "-0", "-"};
    if (rand() % 4 == 0) { snprintf(buf, cap, "%s", edges[rand() % 7]); return; }
    size_t len = (size_t)rand() % 12, i = 0;
    if (rand() % 2) buf[i++] = '-';
    for (; i < len && i < cap - 1; i++)
        buf[i] = (rand() % 20 == 0) ? 'x' : (char)('0' + rand() % 10);
    buf[i] = '\0';
}

int main(void)
{
    srand(2026);
    char buf[32];
    for (long it = 1; it <= 2000000; it++) {
        gen(buf, sizeof buf);
        int a = 0, b = 0;
        bool ra = ref_parse(buf, &a), rb = my_parse(buf, &b);
        if (ra != rb || (ra && a != b)) {
            printf("MISMATCH at test %ld on \"%s\": ref=%s(%d) mine=%s(%d)\n",
                   it, buf, ra ? "ok" : "reject", a, rb ? "ok" : "reject", b);
            return 1;
        }
    }
    printf("2000000 tests agreed\n");
    return 0;
}
```

Output with sanitizers (the build line's flags make UBSan stop at the first error):

```text
difffuzz.c:37:13: runtime error: signed integer overflow: -2147483640 - 9 cannot be represented in type 'int'
```

Output of the same program built *without* sanitizers (`gcc -std=c17 -O2 -o dfp difffuzz.c && ./dfp`):

```text
MISMATCH at test 15 on "-2147483649": ref=reject(0) mine=ok(2147483647)
```

Two oracles, two views of one bug: UBSan reports the overflow at the exact line; the differential check shows its visible consequence (an out-of-range input accepted with a nonsense value). **Fix:** remove the `- 1`, i.e. `if (res < (INT_MIN + d) / 10) return false;` — with it, the test machine ran all 2,000,000 tests without a mismatch or sanitizer report. (Why is that bound exact? `res * 10 - d >= INT_MIN` ⇔ `res >= (INT_MIN + d) / 10` rounded toward +∞, and C's truncating division of a negative number rounds toward zero, i.e. toward +∞.)

</details>

---

## Sources

- Every exercise maps to the lesson cited in its solution; see that lesson's Sources for the standard clauses and documentation.
- FNV-1a 64-bit parameters (offset basis 14695981039346656037, prime 1099511628211): IETF draft *The FNV Non-Cryptographic Hash Algorithm*: https://datatracker.ietf.org/doc/html/draft-eastlake-fnv
- POSIX `waitpid`, `pipe`, `dup2`, `execvp`: https://pubs.opengroup.org/onlinepubs/9799919799/
- cppreference `strtol` (range errors via `errno == ERANGE`): https://en.cppreference.com/w/c/string/byte/strtol

## Verify yourself

1. Compile and run every solution with its `// build:` / `// run:` lines.
2. Run `wordfreq`, `minish` and `difffuzz` under Valgrind (rebuild without sanitizers) and confirm zero leaks.
3. For E8, write each snippet into a file, build with `-fsanitize=undefined`, and record which ones UBSan flags and which it does not — then explain each miss using lesson I8's description of what UBSan checks.
4. For Project 3, fix the bug and confirm `2000000 tests agreed`.
