# Interview questions — with model answers

Questions systems, embedded, compiler and security interviewers commonly ask about C, grouped by depth. Each model answer is written the way you would *say* it: the core fact first, then the reasoning, then (where useful) a concrete example from this track. Lesson references point to where the full explanation, code and sources live.

---

## A. Fundamentals

### 1. What happens when you run `gcc main.c -o app`?

**Model answer.** The `gcc` driver runs four stages in order: the **preprocessor** expands `#include`s and macros into a translation unit; the **compiler proper** translates it to assembly; the **assembler** produces an object file with machine code, a symbol table and relocations; the **linker** combines object files and libraries, resolves every undefined symbol to exactly one definition, and patches addresses. For a dynamically linked program, the final binding of library functions like `printf` happens at load time in `ld.so`. You can stop after each stage with `-E`, `-S`, `-c`. (B1, I5, P3)

### 2. What is the difference between a declaration and a definition?

**Model answer.** A declaration tells the compiler a name and its type; a definition also creates the object or provides the function body. `extern int count;` and `int f(void);` declare; `int count = 0;` and a function with a body define. Headers should contain declarations; exactly one translation unit provides each external definition — otherwise you get `undefined reference` (none) or `multiple definition` (more than one, an error by default since GCC 10's `-fno-common`). (I5)

### 3. What does `static` mean in C?

**Model answer.** Two different things depending on where it appears. On a **block-scope variable** it gives static storage duration: one object for the whole program, initialised once, keeping its value between calls. On a **file-scope variable or function** it gives internal linkage: the name is private to that translation unit, which avoids clashes and gives the optimiser more freedom. (B3, I5)

### 4. Why is C "pass by value", and how does `swap` work then?

**Model answer.** Every argument is copied into a new parameter object. `swap(int a, int b)` swaps copies. `swap(int *a, int *b)` still receives copies — of the *addresses* — and writing through them changes the caller's variables. To change a pointer in the caller you pass a pointer to the pointer. (B3, B5, I1)

### 5. Arrays vs pointers — are they the same?

**Model answer.** No. An array is a contiguous block of elements; in most expressions it *converts* to a pointer to its first element, which is why they look alike (`a[i]` is defined as `*(a + i)`). The differences show with `sizeof` (whole array vs one pointer), `&a` (pointer to the whole array, so `&a + 1` jumps the entire array), and assignment (arrays aren't assignable). When passed to a function the array decays, so the function must receive the length separately. (B4, I1)

### 6. What is the difference between stack and heap memory?

**Model answer.** The stack holds automatic objects — locals and parameters — allocated on function entry and released on return, fast but small (8 MiB by default on the test machine). The heap holds allocated objects you create with `malloc` and must release with `free`; it's large and outlives functions, but costs allocator bookkeeping and is where leaks, double frees and use-after-free live. The C standard itself speaks of storage durations — automatic, static, thread, allocated — and the stack/heap is how implementations realise them. (B6)

---

## B. Undefined behaviour and correctness

### 7. What is undefined behaviour, and why does C have it?

**Model answer.** UB means the standard imposes no requirements on the program's behaviour. It exists so compilers can generate fast code on every machine without inserting checks: they may assume signed arithmetic doesn't overflow, pointers are valid, types don't alias incompatibly, and so on. The consequence is that a program with UB can do anything — including behave correctly at `-O0` and differently at `-O2`. It's distinct from implementation-defined behaviour (documented choice, like `sizeof(int)`) and unspecified behaviour (one of several outcomes, like argument evaluation order). (I3)

### 8. Give five common kinds of UB.

**Model answer.** Signed integer overflow; out-of-bounds access or pointer arithmetic beyond one-past-the-end; null-pointer dereference; using an object after its lifetime (dangling pointers, use-after-free); unsequenced modifications like `i = i++ + 1`. Others: reading uninitialised locals, shifting by ≥ the width, division by zero, strict-aliasing violations, data races. (I3)

### 9. Why might `if (x + 1 < x)` never be true even when `x == INT_MAX`?

**Model answer.** For signed `x`, the comparison can only be true through overflow, which is UB, so the compiler may fold it to false — GCC 13 did so even at `-O0` on the test machine. Check before the operation (`x > INT_MAX - 1`) or use C23 `ckd_add`. With `-fwrapv` GCC defines wrapping and the check "works", but that's not portable C. (I3, A4, A5)

### 10. Explain the strict aliasing rule.

**Model answer.** An object may only be accessed through an lvalue of a compatible type (ignoring qualifiers and signedness), an aggregate or union containing such a type, or a character type. Anything else — like reading an `int` through a `float *` — is UB, and at `-O2` GCC uses the rule to assume such pointers don't alias: in the track's example a function returned 1 at `-O2` and 0 at `-O0`. To reinterpret bytes legally, use `memcpy`, a union (in C), or `unsigned char` access. The Linux kernel opts out with `-fno-strict-aliasing`. (A1)

### 11. What does `volatile` do, and why isn't it enough for threads?

**Model answer.** `volatile` forces every access to actually happen in program order relative to other volatile accesses — needed for memory-mapped hardware registers and `sig_atomic_t` flags in signal handlers. It provides no atomicity and no ordering of ordinary memory between threads, so concurrent access is still a data race (UB). Use `_Atomic`/`<stdatomic.h>` or mutexes. Java's `volatile` is different: it has acquire/release semantics. (A6, P1)

### 12. How do you find memory bugs in C code?

**Model answer.** Layered: compiler warnings (`-Wall -Wextra`), static analysis (`-fanalyzer`, Clang Static Analyzer), then dynamic tools on tests — AddressSanitizer for out-of-bounds, use-after-free and double free (fast, needs a rebuild, catches stack overflows too), Valgrind Memcheck for heap errors, uninitialised reads and leaks without a rebuild (20–30× slower, misses stack arrays), UBSan for UB arithmetic, TSan for races — and finally fuzzing with sanitizers to explore inputs tests never try. (I8, P4)

---

## C. Memory and data structures

### 13. What's wrong with `p = realloc(p, n);`?

**Model answer.** If `realloc` fails it returns NULL and leaves the original block allocated; assigning that NULL to `p` loses the only pointer, leaking the block. Use a temporary: `tmp = realloc(p, n); if (!tmp) handle; else p = tmp;`. Also remember the block may move, so other pointers into it become invalid, and `realloc(p, 0)` is UB in C23. (I2)

### 14. How does `malloc` work internally?

**Model answer.** It manages chunks carved from memory obtained from the OS (`brk` for the main heap, `mmap` for large requests — above 128 KiB by default in glibc). Each chunk has a header with its size and flags; freed chunks go into free lists. glibc first checks a per-thread cache (tcache, up to 7 chunks per size class by default), then size-segregated bins in an arena, splitting and coalescing blocks to fight fragmentation. That's why `malloc(1)` actually uses a 32-byte chunk and why a freed block is often returned by the very next same-size `malloc` — which makes use-after-free exploitable. (A3)

### 15. Why can a struct be bigger than the sum of its members?

**Model answer.** Alignment: each member must start at a multiple of its alignment, so the compiler inserts padding, and adds tail padding so arrays of the struct keep every element aligned. `struct { char; double; char; int; char; }` was 32 bytes on x86-64, and 16 after sorting members by decreasing alignment. Padding bytes have unspecified values, so don't compare structs with `memcmp` or dump them raw to files. (B7, A1)

### 16. When would you use a linked list over an array in C?

**Model answer.** Rarely, for performance-critical traversal: arrays are contiguous, cache- and prefetcher-friendly, and on the test machine summing a shuffled linked list was about 196× slower than summing an array of the same length. Lists win when you need O(1) insertion/removal in the middle given a node pointer, stable addresses for elements, or intrusive lists (like the Linux kernel's), and traversal cost doesn't dominate. (I2, P5)

---

## D. Systems and concurrency

### 17. Explain `fork`, `exec` and `wait`.

**Model answer.** `fork` duplicates the process — copy-on-write memory, inherited file descriptors — returning the child's PID in the parent and 0 in the child. `exec` replaces the current process image with a new program (same PID, same open descriptors unless close-on-exec). `waitpid` reaps a child and returns its status, decoded with `WIFEXITED`/`WEXITSTATUS` or `WIFSIGNALED`/`WTERMSIG`; un-reaped children are zombies. Gotchas: flush stdio before `fork` or buffered output is printed twice; use `_exit` in the child after a failed `exec`. (P1)

### 18. What can you safely do in a signal handler?

**Model answer.** Only async-signal-safe operations: set a `volatile sig_atomic_t` flag, call functions like `write` or `_exit`. Not `printf`, not `malloc`, nothing that takes locks, because the signal may have interrupted the main program in the middle of the same function. The robust pattern is: the handler sets a flag (or writes to a self-pipe), the main loop does the work. (P1)

### 19. What is a data race, and how do you prevent one?

**Model answer.** Two threads access the same memory location, at least one writes, at least one access isn't atomic, and neither happens-before the other. In C11 that's UB. Prevent it with a mutex around the shared data, atomics for single variables (relaxed for pure counters), and release/acquire when one thread publishes data to another. TSan detects races dynamically. (A6)

### 20. Explain acquire and release semantics.

**Model answer.** A release store guarantees that the thread's earlier reads and writes aren't reordered after it; an acquire load guarantees later reads and writes aren't reordered before it. When an acquire load reads the value written by a release store, everything before the store happens-before everything after the load — that's how a producer safely publishes a payload with a flag. On x86 both compile to plain `mov` (the hardware is strongly ordered), while a seq_cst store needs `xchg`; on ARM64 they typically become `stlr`/`ldar` [unverified on the test machine — no ARM compiler was available], which is why x86 testing can't validate memory orders. (A6)

### 21. What is false sharing?

**Model answer.** Two threads write different variables that happen to share a cache line; every write forces the line to bounce between cores, so the threads slow each other down despite sharing no data. On the test VM the penalty ranged from 1.3× to 10× depending on how the hypervisor placed the vCPUs. Fix by padding or aligning hot per-thread data to the cache-line size (64 bytes), or accumulating locally and combining at the end. (P5)

### 22. How does a call to `printf` in your program reach libc?

**Model answer.** The compiler emits `call printf@plt`. The PLT stub jumps indirectly through a GOT entry that the dynamic loader fills with libc's address; with `BIND_NOW` that happens at load time and RELRO then makes the GOT read-only, otherwise it's resolved lazily on first call. Because resolution goes through the loader's search order, `LD_PRELOAD` can interpose your own `printf`. (P3)

---

## E. Expert and compiler-focused

### 23. How do optimisers exploit UB? Give a concrete example.

**Model answer.** They treat "no UB happens" as an axiom and derive facts from it. Example: `int sum_to(int n) { for (int i = 0; i <= n; i++) ... }` — since `i++` can't overflow, the loop runs exactly n+1 times, so GCC widened `i` to a 64-bit register and could compute a trip count; the `unsigned` version must preserve 32-bit wrap and might even be infinite. Another: `(x * 2) / 2` folds to `x` for `int` but becomes `x & 0x7fffffff` for `unsigned`. A third: dereferencing `p` before checking it lets the compiler delete the null check. (A4)

### 24. How would you test a C compiler?

**Model answer.** Differential testing: generate random programs that are free of undefined behaviour (like Csmith), compile each with several compilers or optimisation levels, run them, and compare their outputs — usually a checksum of final global state. Any disagreement is a bug candidate; you first rule out UB in the test (sanitizers), then minimise the program with delta debugging (ddmin, C-Reduce) and report it. Complements: EMI (equivalence modulo inputs), translation validation (Alive2), and formally verified compilers (CompCert) at the far end. (P4)

### 25. What changed in C23 that could break existing code?

**Model answer.** `bool`, `true`, `false`, `nullptr`, `static_assert`, `alignas`, `alignof`, `thread_local` became keywords, so code using them as identifiers breaks; `int f();` now means `int f(void)`, so calls with arguments to old-style declarations fail; K&R function definitions were removed (though GCC 14 still accepted one with a warning in testing, while Clang 18 rejected it); `realloc(p, 0)` became UB. Practically, GCC 15 compiles as `gnu23` by default, so builds should pin `-std=`. (A5)

### 26. What is pointer provenance and why does it matter?

**Model answer.** Provenance is the idea that a pointer carries not just an address but the identity of the object it was derived from, and may only access that object. Compilers rely on it for alias analysis, but C17 never defined it precisely; WG14 published ISO/IEC TS 6010:2025 with the PNVI-ae-udi model, building on the Cerberus research (POPL 2019). It matters for pointer–integer casts, comparisons of pointers to different objects, and hardware like CHERI where pointers are capabilities. (P5)

---

## Sources

- The lessons referenced after each answer (each lists its primary sources: WG14 N3220, GCC/Clang manuals, man7.org pages, glibc manual, and the papers cited).
- WG14 N3220: https://open-std.org/JTC1/SC22/WG14/www/docs/n3220.pdf
- cppreference, *Undefined behavior*: https://en.cppreference.com/w/c/language/behavior
- cppreference, *memory_order*: https://en.cppreference.com/w/c/atomic/memory_order

## Verify yourself

1. For each answer, open the referenced lesson and re-run its example so you can cite your own observation in an interview ("on GCC 13 at `-O2` this returned 1").
2. Practise answering aloud in under two minutes, then check you mentioned the core fact, the reason, and one example.
