# Common mistakes — the consolidated list

Every lesson has its own "Common mistakes" table; this file gathers the most important ones in one place, grouped by theme, each with **symptom → cause → fix → where it is taught**. Several entries are *real surprises hit while testing this track* (marked ⚑) — mistakes the author's own first drafts made, caught by the compiler, a sanitizer or a differential test.

---

## 1. Build and toolchain

| Mistake | Symptom | Cause | Fix | Lesson |
|---|---|---|---|---|
| Compiling without warnings | Uninitialised variables, wrong `printf` formats compile silently | `-Wall -Wextra` not enabled | Always `-Wall -Wextra`; `-Werror` in CI | B1 |
| Not pinning the dialect | Code breaks when the distro moves to GCC 15 (`gnu23` default) | Defaults differ by compiler version | `-std=c17` / `-std=c23` explicitly | B1, A5 |
| Calling a function with no declaration | Error since GCC 14 | Implicit declarations were removed in C99 | Include the header / write a prototype | B1, B3 |
| `int f();` meaning "no arguments" in C17 | No argument checking; breaks in C23 | Old-style declaration | `int f(void);` | B3, A5 |
| Defining variables in headers | `multiple definition` (GCC ≥ 10) | Each `.c` that includes it defines it | `extern` in header, one definition in a `.c` | I5 |
| Libraries before objects on the link line | `undefined reference` despite `-l` | The linker scans left to right | Objects first, then `-l` | I5 |
| ⚑ `CC ?= gcc` in a Makefile | Build runs `cc`, not `gcc` | Make predefines `CC = cc` | `CC := gcc` | I6 |
| ⚑ Trailing `# comment` after a Makefile variable | Paths with trailing spaces, `No rule to make target '%.c'` | Spaces before `#` become part of the value | Comments on their own line | I6 |
| Headers missing from Makefile prerequisites | Stale builds after editing a header | Make doesn't know about `#include` | `-MMD -MP` + `-include $(DEPS)` | I6 |
| ⚑ Missing feature-test macros with `-std=c17` | `MAP_ANONYMOUS undeclared`, implicit `fsync` | Strict mode hides POSIX/BSD APIs | `#define _POSIX_C_SOURCE 200809L` / `_DEFAULT_SOURCE` before includes | P2 |
| ⚑ Assuming the environment doesn't change `malloc` | 1 MiB block came from `brk`, not `mmap` | `MALLOC_MMAP_THRESHOLD_` set in the environment | Treat allocator placement as an implementation detail | B6 |

## 2. Types and arithmetic

| Mistake | Symptom | Cause | Fix | Lesson |
|---|---|---|---|---|
| Assuming `int`/`long` sizes | Overflow/truncation on other platforms | Sizes are implementation-defined | `<stdint.h>` exact-width types | B2 |
| Signed/unsigned comparison | `-1 < 1u` is false; `sizeof(int) > -1` is false | Usual arithmetic conversions | One signedness per comparison; heed `-Wsign-compare` | B2 |
| Integer division where a fraction was meant | `(3 + 4) / 2 == 3` | Both operands are `int` | Make one operand floating | B2 |
| Overflow check after the operation | Check folded away (even at `-O0` in GCC 13) | Signed overflow is UB | Pre-check limits or `ckd_add` | I3, A4, A5 |
| `1 << 31` with 32-bit `int` | UBSan: "cannot be represented in type 'int'" | Left shift overflowing a signed type | `1u << 31` | B2 |
| `return a - b;` in comparators | Wrong order for extreme values (UB) | Subtraction overflow | `(a > b) - (a < b)` | I1 |
| ⚑ `a/2 + b/2 + (a%2 + b%2)/2` as an "overflow-free average" | Off by one for mixed signs | C division truncates toward zero | Handle mixed signs separately; verify with a differential test | P4 |
| ⚑ Wrong hash constant typed from memory | Silently worse hash distribution | Typo in a 20-digit constant | Copy constants from the specification and cite it | exercises |

## 3. Arrays, strings and pointers

| Mistake | Symptom | Cause | Fix | Lesson |
|---|---|---|---|---|
| `i <= n` loops | Out-of-bounds read/write | Off-by-one | `i < n` | B3, B4 |
| `sizeof arr` inside a function | Returns 8 | Array parameter is a pointer | Pass the length | B4 |
| `strcpy`/`sprintf`/`gets` into fixed buffers | Buffer overflow | No size information | `snprintf`, `fgets`, check lengths | B4 |
| No room for `'\0'` | `strlen` runs off the end | String needs n+1 bytes | Size buffers as length + 1 | B4 |
| `==` on strings | Compares addresses | Strings are arrays | `strcmp` | B4 |
| Modifying a string literal | Segfault | Literals are read-only | Copy into `char[]` | B4, B6 |
| Uninitialised / null / dangling / out-of-bounds pointers | Crashes or silent corruption | The four pointer sins | Initialise, check NULL, null after free, stay in bounds | B5 |
| `int* a, b;` | `b` isn't a pointer | `*` binds to the declarator | One declaration per line | B5 |
| Assigning to a pointer parameter to "return" memory | Caller sees nothing; leak | Pass by value | `T **out` or return the pointer | I1 |
| Arithmetic on `void *` | Not standard C | `void` has no size | Cast to `unsigned char *` | I1 |

## 4. Memory management

| Mistake | Symptom | Cause | Fix | Lesson |
|---|---|---|---|---|
| Returning the address of a local | Dangling pointer (GCC may return NULL) | Automatic lifetime ends | Caller buffer / heap / static | B6 |
| Not checking `malloc` | Null dereference under memory pressure | Allocation can fail | Check every allocation | B6, I2 |
| `n * size` overflow in allocation size | Tiny buffer, heap overflow | Unchecked multiplication | `calloc`, `ckd_mul`, or explicit check | I2, A5 |
| `p = realloc(p, n)` | Leak on failure | NULL overwrites the only pointer | Use a temporary | I2 |
| Reading `node->next` after `free(node)` | Use-after-free (ASan: heap-use-after-free) | Wrong loop order | Save `next` first | I2 |
| Double free | glibc abort "double free detected in tcache 2" — or silent corruption | Two owners | Single owner; null after free | I2, I8 |
| Using freed memory "just once" | Reads another object's data (LIFO reuse) | Allocator immediately recycles chunks | Never touch freed memory | A3 |
| `realloc(p, 0)` to free | UB in C23 | Standard change | `free(p)` | I2, A5 |
| Huge local arrays | Stack overflow segfault | 8 MiB stack (test machine) | Heap allocation | B6, A2 |

## 5. Undefined behaviour and the optimiser

| Mistake | Symptom | Cause | Fix | Lesson |
|---|---|---|---|---|
| "It works at `-O0`, so it's correct" | Different results at `-O2` (e.g. `table[4]` loop returned 1) | UB has no defined result | Remove the UB; test at several levels | I3 |
| Dereference before null check | Check deleted | Compiler infers non-null | Check first | I3, A4 |
| `*(float *)&int_var` | Different results at `-O0`/`-O2` | Strict-aliasing UB | `memcpy`, union, char access | A1 |
| `*(uint32_t *)(buf + 1)` | Misaligned UB (faults on some CPUs) | Alignment requirement | `memcpy` / byte shifts | A1 |
| `p + len < p` overflow check | Rewritten to something else | Pointer arithmetic can't wrap in C | Compare lengths | A4 |
| `restrict` on overlapping pointers | Silently wrong (3x instead of 4x) | Broken promise = UB | Use only when provably disjoint | A1 |
| `unreachable()` on an unproven path | `classify(7)` returned 80 | Reaching it is UB | `assert` in debug; prove first | A4 |
| `i = i++ + 1;`, `a[i] = i++;` | Unpredictable | Unsequenced side effects | One modification per statement | I3 |

## 6. Debugging and tools

| Mistake | Symptom | Cause | Fix | Lesson |
|---|---|---|---|---|
| `printf` debugging around crashes | Output vanishes when piped | Full buffering of stdout to pipes | `stderr`, `fflush`, or a debugger | I7 |
| Debugging `-O2` builds | `<optimized out>`, jumping lines | Optimisation | `-O0 -g` / `-Og -g` | I7 |
| ⚑ Conditional breakpoint on a loop-test line never fires | No stop | Breakpoints bind to addresses; loop test placed at the bottom | Use another line in the loop body | I7 |
| ⚑ Expecting ASan to catch a write right before `free` at `-O1` | No report | Dead store eliminated | Debug at `-O0`; use the value | I8 |
| ⚑ Expecting a UBSan alignment report for a cast in the same function | No report | Instrumentation blind spot | Don't rely on one tool; test via a helper function | A1 |
| Valgrind for stack-array overflows | 0 errors | Memcheck only knows heap bounds | ASan | I8 |
| ASan + TSan in one build | Build error | Incompatible runtimes | Separate builds | I8 |
| Trusting "sanitizer clean" as proof | Bugs on untested paths | Dynamic tools check executed paths only | Fuzzing, static analysis | I8, P4 |
| ⚑ Assuming static analysers find everything | Leak not reported by either GCC or Clang analyser | Practical analysers are unsound | Combine layers | P4 |

## 7. Concurrency

| Mistake | Symptom | Cause | Fix | Lesson |
|---|---|---|---|---|
| Unprotected shared counter | Lost updates (e.g. 3.09M of 4M) | Data race | Mutex or atomic | A6 |
| `volatile` as synchronisation | Still racy | No atomicity/ordering | `<stdatomic.h>` | A6 |
| `relaxed` on a publishing flag | TSan race on the payload; stale reads on ARM | No happens-before | release store / acquire load | A6 |
| `if` around `cnd_wait` | Acting on a false condition | Spurious/stolen wake-ups | `while` loop | A6 |
| ⚑ TSan with C11 `thrd_create` | TSan SEGV on the test machine | Observed runtime incompatibility [unverified as general] | Use pthreads under TSan | A6 |
| Counters packed in one cache line | 1.3×–10× slowdown | False sharing | Align/pad to 64 bytes | P5 |

## 8. Systems programming

| Mistake | Symptom | Cause | Fix | Lesson |
|---|---|---|---|---|
| Not flushing before `fork` | Output printed twice when piped | Child inherits the stdio buffer | `fflush(NULL)` before `fork` | P1 |
| `exit` in the child after failed `exec` | Duplicate output, `atexit` handlers run twice | stdio flushed in both processes | `_exit` | P1 |
| Never `waitpid` | Zombies | Unreaped children | Reap (blocking or `WNOHANG` loop) | P1 |
| `printf`/`malloc` in signal handlers | Rare deadlocks/corruption | Not async-signal-safe | Flag + main loop | P1 |
| Single `write` call | Truncated output | Short writes | `write_all` loop, handle `EINTR` | P2 |
| Forgetting to close pipe ends | Reader hangs forever | EOF needs all write ends closed | Close every unused end in every process | P2 |
| Assuming `mmap` is always faster | No gain (sometimes slower) on sequential scans | Page-fault and mapping costs | Measure | P2 |
| Interposing `malloc` and calling `printf` inside | Recursion/deadlock | `printf` may allocate | Minimal wrappers, `write` | P3 |

## 9. Performance

| Mistake | Symptom | Cause | Fix | Lesson |
|---|---|---|---|---|
| Column-major traversal of row-major data | ~13× slower | One int used per cache line | Loop order matches layout | B4, P5 |
| Linked lists in hot loops | ~196× slower than an array (shuffled nodes) | Dependent cache misses | Arrays, indices, pools | I2, P5 |
| ⚑ Assuming `-O3` ≥ `-O2` | `-O3` ~6× slower on a benchmark | Loop interchange created a dependency chain | Measure; read `-fopt-info` | P5 |
| Single-run timings | Conclusions from noise | Variance | Repeat, report ranges | P5 |
| `-march=native` in shipped binaries | Illegal instruction on older CPUs | Non-portable ISA extensions | Target a baseline | P5 |

---

## Sources

- The individual lessons' Sources sections (each row names its lesson).
- SEI CERT C Coding Standard (INT32-C, ARR30-C, STR31-C, MEM30-C, EXP33-C): https://cmu-sei.github.io/secure-coding-standards/sei-cert-c-coding-standard/rules
- cppreference, *Undefined behavior*: https://en.cppreference.com/w/c/language/behavior

## Verify yourself

1. Pick any ⚑ row and reproduce it from the referenced lesson's code.
2. Take a C project you have written and grep it for `strcpy`, `sprintf`, `realloc(`, `gets`, `<= n` loops and casts between pointer types; check each hit against this list.
