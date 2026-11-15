# Appendix — Cheat sheets, references, tools, books, courses and papers

Quick-reference material for the whole track. Version-specific facts are dated (tested November 2026 with GCC 13.3/14.2, Clang 18.1, glibc 2.39, GDB 15.1, Valgrind 3.22, GNU Make 4.3 on Ubuntu 24.04 x86-64).

---

## 1. Compiler flags cheat sheet (GCC; Clang accepts nearly all of these)

| Purpose | Flags | Lesson |
|---|---|---|
| Pick the dialect explicitly | `-std=c17`, `-std=c23` (GCC 14+, Clang 18+), `-std=gnu23` (GCC 15 default) | B1, A5 |
| Warnings (always) | `-Wall -Wextra` (+ `-Wpedantic -Wshadow -Wconversion -Wformat=2` when hardening code) | B1 |
| Warnings as errors (CI) | `-Werror` | B1 |
| Debug info | `-g` (with `-O0` or `-Og` for debugging) | I7 |
| Optimisation | `-O0`, `-Og`, `-O1`, `-O2`, `-O3`, `-Os`, `-flto`, `-march=...` | A4, P5 |
| Stop after a stage | `-E` (preprocess), `-S` (assembly), `-c` (object) | B1, I5 |
| Macros / include paths | `-DNAME=val`, `-UNAME`, `-Idir` | I4 |
| Dependency files for make | `-MMD -MP` | I6 |
| Libraries | `-Ldir -lname` (after objects!), `-shared -fPIC`, `-Wl,-rpath,'$ORIGIN'` | I5 |
| Sanitizers | `-fsanitize=address,undefined` / `-fsanitize=thread` / `-fsanitize=memory` (Clang) + `-fno-omit-frame-pointer` | I8 |
| Stop on first UBSan error | `-fno-sanitize-recover=undefined` | I3, I8 |
| UB policy flags | `-fwrapv`, `-fno-strict-overflow`, `-fno-strict-aliasing`, `-fno-delete-null-pointer-checks`, `-ftrapv` | A4 |
| Hardening | `-D_FORTIFY_SOURCE=3 -fstack-protector-strong -fstack-clash-protection -fcf-protection=full -fPIE -pie -Wl,-z,relro,-z,now`, or `-fhardened` (GCC 14+) | P4 |
| Static analysis | `-fanalyzer` (GCC 10+), `clang --analyze` | P4 |
| What did the optimiser do? | `-fopt-info-vec-optimized`, `-fopt-info-optimized`, `-S -o -` | P5 |
| What does the compiler predefine? | `gcc -dM -E - < /dev/null` | I4 |
| What is enabled at a level? | `gcc -Q --help=optimizers -O2` | A2 |

## 2. `printf` format specifiers you actually need

| Type | Specifier | Type | Specifier |
|---|---|---|---|
| `int` | `%d` / `%i` | `unsigned` | `%u`, `%x`, `%o`, `%b` (C23) |
| `long` / `long long` | `%ld` / `%lld` | `unsigned long` / `unsigned long long` | `%lu` / `%llu` |
| `size_t` | `%zu` | `ptrdiff_t` | `%td` |
| `double` (and `float`, promoted) | `%f`, `%e`, `%g` | `long double` | `%Lf` |
| `char` | `%c` | string (`char *`) | `%s` (`%.*s` with a length) |
| pointer | `%p` with `(void *)` cast | literal `%` | `%%` |
| `int32_t`, `uint64_t`… | `PRId32`, `PRIu64` from `<inttypes.h>` (C23 also `%w32d`) | width from argument | `%*d`, `%*s` |

`scanf` counterparts: `%d`, `%lf` for `double` (not `%f`!), `%31s` with an explicit width for a 32-byte buffer (B7).

## 3. GDB cheat sheet

`run [args]`, `break f` / `break file.c:N` / `break N if cond`, `tbreak`, `watch expr` / `rwatch` / `awatch`, `next`, `step`, `finish`, `until N`, `continue`, `backtrace`, `frame N`, `up`, `down`, `print expr`, `print *p`, `print arr[0]@n`, `display expr`, `info locals`, `info args`, `info breakpoints`, `info frame`, `x/NFU addr` (F: x d c s i; U: b h w g), `ptype expr`, `set var x = 5`, `layout src`, batch mode `gdb -q -batch -x script ./prog`. (Lesson I7.)

## 4. Make cheat sheet

```makefile
# file: Makefile.cheatsheet
# build: true
# run: true
CC      := gcc
CFLAGS  := -std=c17 -Wall -Wextra -g -O2
DEPFLAGS := -MMD -MP
SRCS    := $(wildcard *.c)
OBJS    := $(SRCS:.c=.o)

app: $(OBJS)                  # $@ = target, $^ = all prerequisites
	$(CC) $^ -o $@

%.o: %.c                      # $< = first prerequisite
	$(CC) $(CFLAGS) $(DEPFLAGS) -c $< -o $@

.PHONY: clean
clean:
	rm -f app $(OBJS) $(OBJS:.o=.d)

-include $(OBJS:.o=.d)
```

Traps: recipe lines need a **TAB**; `CC ?= gcc` does nothing (`CC` is predefined as `cc`); a trailing `# comment` after a variable value leaves trailing spaces in the value (lesson I6).

## 5. Valgrind and sanitizer cheat sheet

| Tool | Command | Finds |
|---|---|---|
| Memcheck | `valgrind --leak-check=full --track-origins=yes --error-exitcode=1 ./prog` | heap errors, uninitialised use, leaks (not stack/global arrays) |
| ASan + LSan | `gcc -g -O1 -fsanitize=address -fno-omit-frame-pointer` | heap/stack/global OOB, UAF, double free, leaks |
| UBSan | `gcc -g -fsanitize=undefined -fno-sanitize-recover=undefined` | signed overflow, shifts, null/misaligned, bounds |
| TSan | `gcc -g -O1 -fsanitize=thread` | data races |
| MSan (Clang) | `clang -g -O1 -fsanitize=memory -fsanitize-memory-track-origins` | uninitialised reads |
| Runtime options | `ASAN_OPTIONS=detect_stack_use_after_return=1`, `UBSAN_OPTIONS=print_stacktrace=1` | |

## 6. Two standard-library corners not given their own lesson

### 6.1 Variadic functions (`<stdarg.h>`)

```c
// file: variadic.c
// build: gcc -std=c17 -Wall -Wextra -o variadic variadic.c
// run: ./variadic

#include <stdio.h>
#include <stdarg.h>    // va_list, va_start, va_arg, va_end

// sums `count` ints passed after the count argument
int sum_ints(int count, ...)
{
    va_list ap;
    va_start(ap, count);              // start after the last named parameter
    int res = 0;
    for (int i = 0; i < count; i++)
        res += va_arg(ap, int);       // the caller must really pass ints (promoted types!)
    va_end(ap);                       // every va_start needs a va_end
    return res;
}

// a printf-like logger: forwards the variadic list with vfprintf
__attribute__((format(printf, 2, 3)))  // GCC/Clang: check format strings like printf
void log_msg(const char *level, const char *fmt, ...)
{
    va_list ap;
    va_start(ap, fmt);
    fprintf(stderr, "[%s] ", level);
    vfprintf(stderr, fmt, ap);        // the v-variant takes a va_list
    va_end(ap);
    fputc('\n', stderr);
}

int main(void)
{
    printf("sum = %d\n", sum_ints(4, 10, 20, 30, 40));
    log_msg("info", "loaded %d records from %s", 3, "marks.txt");
    return 0;
}
```

Output (stderr is unbuffered, stdout was a pipe, so the log line came out first):

```text
[info] loaded 3 records from marks.txt
sum = 100
```

Rules: arguments undergo default promotions (`char`/`short` → `int`, `float` → `double`), so `va_arg(ap, float)` is wrong; the function needs some way to know how many arguments and of what types (a count, a format string, or a sentinel); `va_end` must match every `va_start`. C23 relaxes the requirement that a variadic function have a named parameter before `...` (N2975).

### 6.2 Non-local jumps (`<setjmp.h>`)

```c
// file: jmp.c
// build: gcc -std=c17 -Wall -Wextra -o jmp jmp.c
// run: ./jmp

#include <stdio.h>
#include <setjmp.h>    // jmp_buf, setjmp, longjmp

static jmp_buf on_error;              // where to jump back to

// deep inside some parser: bail out to the recovery point on bad input
static int parse_digit(char c)
{
    if (c < '0' || c > '9')
        longjmp(on_error, 1);         // non-local jump: unwinds straight back to setjmp
    return c - '0';
}

int main(void)
{
    const char *inputs[] = {"7", "x"};
    for (int i = 0; i < 2; i++) {
        if (setjmp(on_error) == 0)    // returns 0 when called directly...
            printf("digit %d\n", parse_digit(inputs[i][0]));
        else                          // ...and 1 when longjmp jumps back here
            printf("bad input \"%s\"\n", inputs[i]);
    }
    return 0;
}
```

Output:

```text
digit 7
bad input "x"
```

`setjmp`/`longjmp` jump across function frames without running any cleanup — memory and resources acquired in the skipped frames leak, and local variables of the `setjmp` caller modified after `setjmp` must be `volatile` to have defined values after the jump. They are used for error recovery in some C libraries and interpreters [unverified: specific projects using them, e.g. Lua's error handling, not checked here]; prefer explicit error returns and the `goto cleanup` pattern (P2).

### 6.3 Library areas intentionally not covered in depth

`<math.h>`/`<complex.h>`/`<fenv.h>` (numerics), `<locale.h>`/`<wchar.h>`/`<uchar.h>` (internationalisation), and `<time.h>` beyond monotonic clocks: read them from cppreference's C library pages when needed — they are reference material rather than concepts. Annex K (`strcpy_s` etc.) is optional and not provided by glibc.

---

## 7. Standards and primary documents

- **C23 draft N3220** (identical to C23 except one Annex K footnote, per editor's report N3221): https://open-std.org/JTC1/SC22/WG14/www/docs/n3220.pdf
- **N3221 editor's report:** https://open-std.org/JTC1/SC22/WG14/www/docs/n3221.htm
- **C17 final draft N2176** (mirror): https://teaching.csse.uwa.edu.au/units/CITS2002/resources/n2176.pdf
- **WG14 document index:** https://www.open-std.org/jtc1/sc22/wg14/www/docs/
- **ISO/IEC TS 6010:2025** (provenance-aware memory object model) catalogue entry: https://standards.iteh.ai/catalog/standards/iso/de3583c6-f277-4853-9df7-5f9e9c5e0fa8/iso-iec-ts-6010-2025
- **POSIX.1-2024 (The Open Group Base Specifications Issue 8):** https://pubs.opengroup.org/onlinepubs/9799919799/
- **System V x86-64 psABI** source: https://gitlab.com/x86-psABIs/x86-64-ABI

## 8. Reference documentation

- cppreference C: https://en.cppreference.com/w/c
- GCC manual: https://gcc.gnu.org/onlinedocs/gcc/ ; GCC C status: https://gcc.gnu.org/projects/c-status.html ; porting notes: https://gcc.gnu.org/gcc-14/porting_to.html, https://gcc.gnu.org/gcc-15/porting_to.html
- Clang docs: https://clang.llvm.org/docs/ ; Clang C status: https://clang.llvm.org/c_status.html
- GNU Make manual: https://www.gnu.org/software/make/manual/make.html
- GDB manual: https://sourceware.org/gdb/current/onlinedocs/gdb.html/
- Valgrind manual: https://valgrind.org/docs/manual/manual.html
- Linux man pages: https://man7.org/linux/man-pages/
- glibc manual: https://www.gnu.org/software/libc/manual/
- SEI CERT C Coding Standard: https://cmu-sei.github.io/secure-coding-standards/sei-cert-c-coding-standard/rules

## 9. Books

| Book | Why read it | Link |
|---|---|---|
| Kernighan & Ritchie, *The C Programming Language*, 2nd ed. (1988) | The classic; short and precise (pre-C99, so mind the dialect) | https://www.pearson.com/en-us/subject-catalog/p/c-programming-language/P200000000368/9780131103627 |
| Jens Gustedt, *Modern C*, 3rd ed. (Manning, covers C23) | Modern style, by a WG14 member | https://www.manning.com/books/modern-c-third-edition |
| Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective* | Data representation, assembly, linking, ECF, virtual memory, allocators | https://csapp.cs.cmu.edu/ |
| Michael Kerrisk, *The Linux Programming Interface* | Definitive Linux systems programming | https://man7.org/tlpi/ |
| Stevens & Rago, *Advanced Programming in the UNIX Environment*, 3rd ed. | Classic UNIX systems programming | https://www.pearson.com/en-us/subject-catalog/p/advanced-programming-in-the-unix-environment/P200000000628 |
| John R. Levine, *Linkers and Loaders* | The linking/loading book | https://linker.iecc.com/ |
| Paul E. McKenney, *Is Parallel Programming Hard…?* (free) | Concurrency and memory ordering in depth | https://cdn.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html |

## 10. Courses

- MIT OCW 6.087 *Practical Programming in C* (the coverage baseline of this track): https://ocw.mit.edu/courses/6-087-practical-programming-in-c-january-iap-2010/
- CMU 15-213 *Introduction to Computer Systems* (CS:APP labs: data, bomb, attack, cache, shell, malloc): https://www.cs.cmu.edu/afs/cs/academic/class/15213-f24/www
- Harvard CS61 notes on calling conventions: https://cs61.seas.harvard.edu/site/2018/Asm2

## 11. Papers (the research spine of this track)

| Paper | Topic | Lesson |
|---|---|---|
| D. M. Ritchie, *The Development of the C Language*, HOPL-II 1993 — https://www.nokia.com/bell-labs/about/dennis-m-ritchie/chist.html | History | B1 |
| Yang, Chen, Eide, Regehr, *Finding and Understanding Bugs in C Compilers*, PLDI 2011 — https://www-old.cs.utah.edu/~regehr/papers/pldi11-preprint.pdf | Csmith, differential testing | I3, P4 |
| Wang, Zeldovich, Kaashoek, Solar-Lezama, *Towards Optimization-Safe Systems*, SOSP 2013 — https://pdos.csail.mit.edu/papers/stack:sosp13.pdf | UB-unstable code | I3, A4 |
| Serebryany et al., *AddressSanitizer*, USENIX ATC 2012 — https://www.usenix.org/conference/atc12/technical-sessions/presentation/serebryany | Sanitizers | I8 |
| Nethercote & Seward, *Valgrind*, PLDI 2007 — https://dx.doi.org/10.1145/1273442.1250746 | Dynamic binary instrumentation | I8 |
| Wilson et al., *Dynamic Storage Allocation: A Survey and Critical Review*, IWMM 1995 — https://www.cs.hmc.edu/~oneill/gc-library/Wilson-Alloc-Survey-1995.pdf | Allocators | A3 |
| Leijen, Zorn, de Moura, *Mimalloc*, APLAS 2019 — https://www.microsoft.com/en-us/research/publication/mimalloc-free-list-sharding-in-action | Allocators | A3 |
| Boehm, *Threads Cannot Be Implemented as a Library*, PLDI 2005 — https://courses.cs.washington.edu/courses/cse590p/05au/p261-boehm.pdf | Memory models | A6 |
| Memarian et al., *Exploring C Semantics and Pointer Provenance*, POPL 2019 — https://www.cl.cam.ac.uk/~km569/exploring_provenance.pdf | Semantics | P5 |
| Szekeres, Payer, Wei, Song, *SoK: Eternal War in Memory*, IEEE S&P 2013 — https://www.ieee-security.org/TC/SP2013/papers/4977a048.pdf | Memory safety | A2, P4 |
| Leroy, *Formal Verification of a Realistic Compiler*, CACM 2009 — https://xavierleroy.org/publi/compcert-CACM.pdf | Verified compilation | P4 |
| Kroening & Tautschnig, *CBMC*, TACAS 2014 — https://link.springer.com/10.1007/978-3-642-54862-8_26 | Model checking | P4 |
| Zeller & Hildebrandt, *Simplifying and Isolating Failure-Inducing Input*, TSE 2002 — https://www.st.cs.uni-saarland.de/papers/tse2002/tse2002.pdf | Delta debugging | P4 |
| Crotty, Leis, Pavlo, *Are You Sure You Want to Use MMAP in Your DBMS?*, CIDR 2022 — https://vldb.org/cidrdb/2022/are-you-sure-you-want-to-use-mmap-in-your-database-management-system.html | `mmap` trade-offs | P2 |
| Drepper, *What Every Programmer Should Know About Memory*, 2007 — https://lwn.net/Articles/250967/ | Caches, performance | A1, P5 |
| Lattner, *What Every C Programmer Should Know About Undefined Behavior*, 2011 — https://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html | UB (blog, supporting) | I3, A4 |

## 12. Tools

Compiler Explorer (https://godbolt.org), `cdecl` (https://cdecl.org), `strace`, `ltrace`, `perf` (https://perfwiki.github.io/main/), `pahole` (struct holes), `readelf`/`objdump`/`nm` (GNU binutils: https://sourceware.org/binutils/docs/binutils/), `rr` (https://rr-project.org/), Frama-C (https://frama-c.com/), OSS-Fuzz (https://github.com/google/oss-fuzz), Alive2 (https://github.com/AliveToolkit/alive2).

---

## Sources

All links in sections 7–12 above are the sources of this appendix; flag behaviours are from the GCC manual pages cited in lessons B1, I5, I8, A4, P4 and P5, and were exercised on the test machine.

## Verify yourself

1. Build and run `variadic.c` and `jmp.c`; then call `sum_ints(3, 1, 2)` (one argument missing) under `-fsanitize=address` and think about why no tool can reliably catch it.
2. Run `gcc -Q --help=optimizers -O2 | grep enabled | wc -l` and `-O3` to count enabled passes.
3. Check three rows of the flag table against the GCC manual.
