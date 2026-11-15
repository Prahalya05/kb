# Glossary — C Programming, Zero to Expert

Each entry: **term** — *plain meaning* — precise meaning — where it is taught. Lesson codes: B = beginner, I = intermediate, A = advanced, P = pro (see [README.md](README.md) for file names). Standard references are to WG14 draft N3220 (C23 content).

---

## A

- **ABI (Application Binary Interface)** — *the binary-level contract between compiled pieces of code.* Rules for argument passing, register usage, stack alignment, struct layout and symbol naming that let separately compiled code interoperate (System V AMD64 ABI on x86-64 Linux). — [A2](03-advanced/02-the-stack-up-close.md), [I5](02-intermediate/05-compilation-and-linking.md)
- **Acquire / release** — *"I'll see everything you did before you raised the flag."* Memory orders: a release store and an acquire load that reads its value create a synchronizes-with edge, so writes before the store happen-before reads after the load. — [A6](03-advanced/06-concurrency-and-atomics.md)
- **Address operator `&`** — *"where does this live?"* Yields a pointer to its operand (N3220 §6.5.4.2). — [B5](01-beginner/05-pointers-first-look.md)
- **AddressSanitizer (ASan)** — *a compiler-inserted memory checker.* Instrumentation plus shadow memory and red zones that detect out-of-bounds accesses (heap, stack, global), use-after-free, double free; ~2× slowdown (Clang docs). — [I8](02-intermediate/08-valgrind-and-sanitizers.md)
- **Aliasing** — *two names for the same memory.* Two lvalues alias if they designate overlapping storage; the strict aliasing rule limits which types may alias. — [A1](03-advanced/01-object-representation-alignment-aliasing.md)
- **Alignment** — *allowed starting addresses for a type.* A power of two; objects of a type must start at addresses that are multiples of it; queried with `alignof`, raised with `alignas`. — [A1](03-advanced/01-object-representation-alignment-aliasing.md)
- **Arena (allocator)** — *one big region carved up and freed all at once.* Also glibc's term for a set of heaps with its own bookkeeping. — [A3](03-advanced/03-heap-allocator-internals.md)
- **Array-to-pointer conversion (decay)** — *an array "becomes" a pointer to its first element in most expressions.* Except as the operand of `sizeof`, `&`, `typeof`, `alignof`, or a string-literal initialiser. — [B4](01-beginner/04-arrays-and-strings.md), [I1](02-intermediate/01-pointers-deep-dive.md)
- **As-if rule** — *the compiler may change anything you can't observe.* Implementations need only reproduce the observable behaviour of the abstract machine for programs without UB (N3220 §5.1.2.4). — [A4](03-advanced/04-ub-and-the-optimizer.md)
- **Async-signal-safe** — *safe to call from inside a signal handler.* Functions listed in `signal-safety(7)` such as `write`, `_exit`; not `printf` or `malloc`. — [P1](04-pro/01-processes-and-signals.md)
- **Atomic** — *indivisible.* Operations on `_Atomic` objects that no other thread can observe half-done; data races on atomics are impossible. — [A6](03-advanced/06-concurrency-and-atomics.md)
- **Automatic storage duration** — *lives as long as its block.* Local non-`static` variables and parameters. — [B6](01-beginner/06-memory-layout-stack-vs-heap.md)

## B

- **`_BitInt(N)`** — *an integer of exactly N bits.* C23 bit-precise integer type (GCC 14 on x86-64; Clang 15). — [A5](03-advanced/05-c23-features.md)
- **Bit-field** — *a struct member occupying a given number of bits.* Layout details are implementation-defined. — [A1](03-advanced/01-object-representation-alignment-aliasing.md)
- **bss** — *the zero-initialised data segment.* Holds static-storage objects without explicit initialisers; costs no file space. — [B6](01-beginner/06-memory-layout-stack-vs-heap.md)
- **Buffering (stdio)** — *output collected before being written.* Fully buffered for files/pipes, line-buffered for terminals, unbuffered for stderr; explains lost or duplicated output on crash or `fork`. — [I7](02-intermediate/07-debugging-with-gdb.md), [P1](04-pro/01-processes-and-signals.md)

## C

- **C23** — *the 2024 edition of the C standard.* ISO/IEC 9899:2024, published 31 Oct 2024, `__STDC_VERSION__` 202311L. — [A5](03-advanced/05-c23-features.md)
- **Cache line** — *the block of memory moved between caches and RAM.* 64 bytes on the test machine. — [P5](04-pro/05-performance-and-research-frontiers.md)
- **Calling convention** — *who puts arguments where.* On x86-64 SysV: integer args in `rdi, rsi, rdx, rcx, r8, r9`, return in `rax`. — [A2](03-advanced/02-the-stack-up-close.md)
- **Canary (stack protector)** — *a secret value guarding the return address.* Inserted by `-fstack-protector*` and checked before return. — [A2](03-advanced/02-the-stack-up-close.md)
- **`ckd_add` / `ckd_sub` / `ckd_mul`** — *arithmetic that tells you if it overflowed.* C23 `<stdckdint.h>`; returns `true` on overflow, stores the wrapped result. — [A5](03-advanced/05-c23-features.md)
- **Coalescing** — *merging adjacent free blocks.* Allocator technique against fragmentation. — [A3](03-advanced/03-heap-allocator-internals.md)
- **Condition variable** — *a way to sleep until something becomes true.* Used with a mutex; always wait in a `while` loop. — [A6](03-advanced/06-concurrency-and-atomics.md)
- **`const`** — *read-only through this name.* A type qualifier; modifying an object defined `const` is UB. — [B5](01-beginner/05-pointers-first-look.md)
- **`constexpr`** — *a true compile-time constant object.* C23; usable in constant expressions such as array sizes. — [A5](03-advanced/05-c23-features.md)
- **Copy-on-write** — *share pages until someone writes.* How `fork` and `MAP_PRIVATE` avoid copying memory up front. — [P1](04-pro/01-processes-and-signals.md), [P2](04-pro/02-file-io-pipes-mmap-ipc.md)

## D

- **Dangling pointer** — *a pointer to something that no longer exists.* Its value is indeterminate after the object's lifetime ends (N3220 §6.2.4). — [B6](01-beginner/06-memory-layout-stack-vs-heap.md), [I2](02-intermediate/02-dynamic-memory-and-data-structures.md)
- **Data race** — *two threads touching the same memory unsafely.* Conflicting accesses, at least one non-atomic, not ordered by happens-before; UB (N3220 §5.1.2.5). — [A6](03-advanced/06-concurrency-and-atomics.md)
- **Declaration vs definition** — *"it exists" vs "here it is".* A definition also creates the object or provides the function body. — [I5](02-intermediate/05-compilation-and-linking.md)
- **Differential testing** — *compare two implementations; disagreement = bug signal.* Basis of Csmith-style compiler testing. — [P4](04-pro/04-hardening-fuzzing-verification.md)
- **`dlopen` / `dlsym`** — *load a library and find a symbol at run time.* POSIX dynamic loading. — [P3](04-pro/03-linkers-loaders-elf.md)
- **Double free** — *returning the same block twice.* UB; corrupts allocator state. — [I2](02-intermediate/02-dynamic-memory-and-data-structures.md), [I8](02-intermediate/08-valgrind-and-sanitizers.md)
- **Dynamic loader (`ld.so`)** — *the program that loads shared libraries before `main`.* Named by `PT_INTERP`. — [P3](04-pro/03-linkers-loaders-elf.md)

## E

- **Effective type** — *the type memory "currently has" for aliasing purposes.* Declared type, or for allocated memory the type of the first non-character store (N3220 §6.5.1). — [A1](03-advanced/01-object-representation-alignment-aliasing.md)
- **ELF** — *Linux's executable/object file format.* Sections for linkers, segments for loaders. — [P3](04-pro/03-linkers-loaders-elf.md)
- **`#embed`** — *include a binary file as data.* C23 preprocessor directive (GCC 15, Clang 19). — [I4](02-intermediate/04-preprocessor.md), [A5](03-advanced/05-c23-features.md)
- **Endianness** — *byte order of multi-byte values.* x86-64 is little-endian. — [A1](03-advanced/01-object-representation-alignment-aliasing.md)
- **`errno`** — *"why did the last call fail?"* Thread-local error number set by failing library/system calls. — [B7](01-beginner/07-structs-enums-and-file-io.md), [P2](04-pro/02-file-io-pipes-mmap-ipc.md)
- **`exec`** — *replace the running program with another.* Same PID, new image. — [P1](04-pro/01-processes-and-signals.md)

## F

- **False sharing** — *threads slowing each other via one cache line.* Different variables, same line. — [P5](04-pro/05-performance-and-research-frontiers.md)
- **File descriptor** — *a small integer naming an open file/pipe/socket.* 0, 1, 2 = stdin, stdout, stderr. — [P2](04-pro/02-file-io-pipes-mmap-ipc.md)
- **Flexible array member** — *a struct's last member with no size.* `char data[];`, storage provided by `malloc`. — [A1](03-advanced/01-object-representation-alignment-aliasing.md)
- **`fork`** — *duplicate the current process.* Returns child PID in parent, 0 in child. — [P1](04-pro/01-processes-and-signals.md)
- **FORTIFY_SOURCE** — *glibc's checked versions of string/memory functions.* Aborts on detectable overflows; needs optimisation. — [P4](04-pro/04-hardening-fuzzing-verification.md)
- **Fragmentation** — *wasted memory inside (internal) or between (external) blocks.* — [A3](03-advanced/03-heap-allocator-internals.md)
- **Function pointer** — *a variable holding which function to call.* E.g. `int (*cmp)(const void *, const void *)`. — [I1](02-intermediate/01-pointers-deep-dive.md)
- **Fuzzing** — *feeding many generated inputs to find crashes.* Random or coverage-guided (libFuzzer, AFL++). — [P4](04-pro/04-hardening-fuzzing-verification.md)

## G

- **GDB** — *the GNU debugger.* Breakpoints, watchpoints, backtraces, memory inspection. — [I7](02-intermediate/07-debugging-with-gdb.md)
- **GOT / PLT** — *the address table and call stubs for shared-library symbols.* Filled by the dynamic loader. — [P3](04-pro/03-linkers-loaders-elf.md)
- **`goto cleanup`** — *one exit path that releases resources.* Idiomatic C error handling. — [P2](04-pro/02-file-io-pipes-mmap-ipc.md)

## H

- **Happens-before** — *a guaranteed ordering between actions, possibly across threads.* — [A6](03-advanced/06-concurrency-and-atomics.md)
- **Header file** — *a file of declarations shared between translation units.* Protected by include guards. — [I4](02-intermediate/04-preprocessor.md), [I5](02-intermediate/05-compilation-and-linking.md)
- **Heap** — *memory you allocate and free explicitly.* Allocated storage duration. — [B6](01-beginner/06-memory-layout-stack-vs-heap.md), [A3](03-advanced/03-heap-allocator-internals.md)

## I

- **Implementation-defined behaviour** — *the compiler chooses and documents.* E.g. `sizeof(int)` (N3220 §3.5.1). — [B2](01-beginner/02-types-variables-operators.md), [I3](02-intermediate/03-undefined-behavior.md)
- **Include guard** — *`#ifndef X_H / #define X_H / #endif`.* Prevents double inclusion within one translation unit. — [I4](02-intermediate/04-preprocessor.md)
- **Integer promotion** — *small integer types become `int` in arithmetic.* — [B2](01-beginner/02-types-variables-operators.md)
- **Interposition** — *replacing a library function by defining it earlier in search order.* `LD_PRELOAD`. — [P3](04-pro/03-linkers-loaders-elf.md)

## L

- **Leak** — *allocated memory you can no longer free.* Valgrind's "definitely lost". — [I2](02-intermediate/02-dynamic-memory-and-data-structures.md), [I8](02-intermediate/08-valgrind-and-sanitizers.md)
- **Linkage** — *whether a name refers to the same thing across files.* External, internal (`static`), none (N3220 §6.2.2). — [I5](02-intermediate/05-compilation-and-linking.md)
- **Linker** — *combines object files, resolves symbols, applies relocations.* — [I5](02-intermediate/05-compilation-and-linking.md)
- **Lvalue** — *an expression that designates an object.* Can appear on the left of `=` if modifiable. — [B5](01-beginner/05-pointers-first-look.md), [A1](03-advanced/01-object-representation-alignment-aliasing.md)

## M

- **Macro** — *a named text substitution performed by the preprocessor.* Object-like or function-like. — [I4](02-intermediate/04-preprocessor.md)
- **`make`** — *a build tool that rebuilds only out-of-date targets.* — [I6](02-intermediate/06-make.md)
- **`malloc` / `calloc` / `realloc` / `free`** — *the C heap API.* — [B6](01-beginner/06-memory-layout-stack-vs-heap.md), [I2](02-intermediate/02-dynamic-memory-and-data-structures.md)
- **Memory order** — *how much ordering an atomic operation guarantees.* relaxed, acquire, release, acq_rel, seq_cst. — [A6](03-advanced/06-concurrency-and-atomics.md)
- **`mmap`** — *map a file or anonymous memory into the address space.* — [P2](04-pro/02-file-io-pipes-mmap-ipc.md)
- **Mutex** — *a lock allowing one thread at a time.* — [A6](03-advanced/06-concurrency-and-atomics.md)

## N

- **`nullptr`** — *the null pointer constant with its own type.* C23 (`nullptr_t`). — [A5](03-advanced/05-c23-features.md)
- **Null pointer** — *a pointer that points to no object.* Dereferencing it is UB. — [B5](01-beginner/05-pointers-first-look.md)

## O

- **Object file** — *compiled but unlinked machine code.* ELF relocatable with symbols and relocations. — [I5](02-intermediate/05-compilation-and-linking.md)
- **Object representation** — *the bytes of an object.* Viewable as `unsigned char[sizeof obj]`. — [A1](03-advanced/01-object-representation-alignment-aliasing.md)
- **Ownership** — *who is responsible for freeing.* A design convention, not a language rule. — [I2](02-intermediate/02-dynamic-memory-and-data-structures.md)

## P

- **Padding** — *unused bytes inserted for alignment.* Unspecified values. — [B7](01-beginner/07-structs-enums-and-file-io.md), [A1](03-advanced/01-object-representation-alignment-aliasing.md)
- **PIE / PIC** — *code that works at any load address.* `-fPIE`/`-fPIC`; enables ASLR for executables. — [I5](02-intermediate/05-compilation-and-linking.md), [P3](04-pro/03-linkers-loaders-elf.md)
- **Pipe** — *a one-way kernel byte channel between descriptors.* — [P2](04-pro/02-file-io-pipes-mmap-ipc.md)
- **Pointer** — *a variable holding an address, with a type.* — [B5](01-beginner/05-pointers-first-look.md), [I1](02-intermediate/01-pointers-deep-dive.md)
- **Pointer provenance** — *which object a pointer is allowed to access.* Formalised in ISO/IEC TS 6010:2025. — [P5](04-pro/05-performance-and-research-frontiers.md)
- **Preprocessor** — *the text-rewriting stage before compilation.* Translation phase 4. — [I4](02-intermediate/04-preprocessor.md)

## R

- **Red zone** — *128 bytes below `rsp` a leaf function may use without adjusting `rsp`.* SysV x86-64. — [A2](03-advanced/02-the-stack-up-close.md)
- **Relocation** — *a placeholder the linker/loader patches with a final address.* — [I5](02-intermediate/05-compilation-and-linking.md), [P3](04-pro/03-linkers-loaders-elf.md)
- **RELRO** — *making relocated data read-only after loading.* With `BIND_NOW`: "full RELRO". — [P3](04-pro/03-linkers-loaders-elf.md)
- **`restrict`** — *a promise that no other pointer accesses this object.* Enables optimisation; violation is UB. — [A1](03-advanced/01-object-representation-alignment-aliasing.md)

## S

- **Sanitizer** — *compiler instrumentation that checks behaviour at run time.* ASan, UBSan, TSan, MSan, LSan. — [I8](02-intermediate/08-valgrind-and-sanitizers.md)
- **Sequenced-before / unsequenced** — *ordering of evaluations within one thread.* Two unsequenced side effects on one scalar are UB. — [I3](02-intermediate/03-undefined-behavior.md)
- **Shadow memory** — *metadata memory describing the state of application memory.* Used by ASan. — [I8](02-intermediate/08-valgrind-and-sanitizers.md)
- **Signal** — *an asynchronous notification to a process.* `SIGSEGV`, `SIGINT`, `SIGTERM`… — [P1](04-pro/01-processes-and-signals.md)
- **`size_t`** — *the unsigned type of sizes.* Result of `sizeof`; printed with `%zu`. — [B2](01-beginner/02-types-variables-operators.md)
- **Stack** — *automatic memory for calls.* Frames with locals and return addresses; grows down on x86-64. — [B6](01-beginner/06-memory-layout-stack-vs-heap.md), [A2](03-advanced/02-the-stack-up-close.md)
- **Static analysis** — *finding bugs without running the code.* `-fanalyzer`, Clang Static Analyzer. — [P4](04-pro/04-hardening-fuzzing-verification.md)
- **Static storage duration** — *lives for the whole program.* Globals, `static` locals, string literals. — [B6](01-beginner/06-memory-layout-stack-vs-heap.md)
- **Strict aliasing** — *the rule limiting which lvalue types may access an object.* GCC assumes it at `-O2`. — [A1](03-advanced/01-object-representation-alignment-aliasing.md)
- **String** — *chars up to and including the first null character.* (N3220 §7.1.1) — [B4](01-beginner/04-arrays-and-strings.md)

## T

- **tcache** — *glibc's per-thread cache of recently freed chunks.* Default 7 per size class. — [A3](03-advanced/03-heap-allocator-internals.md)
- **ThreadSanitizer (TSan)** — *a data-race detector.* Incompatible with ASan in one build. — [I8](02-intermediate/08-valgrind-and-sanitizers.md), [A6](03-advanced/06-concurrency-and-atomics.md)
- **Translation unit** — *one source file after preprocessing.* — [B1](01-beginner/01-why-c-and-first-program.md), [I5](02-intermediate/05-compilation-and-linking.md)
- **Two's complement** — *the signed-integer bit encoding.* Mandatory since C23. — [B2](01-beginner/02-types-variables-operators.md)
- **`typeof`** — *the type of an expression.* Standard in C23. — [A5](03-advanced/05-c23-features.md)

## U

- **Undefined behaviour (UB)** — *the standard imposes no requirements.* Optimisers assume it never happens (N3220 §3.5.3). — [I3](02-intermediate/03-undefined-behavior.md), [A4](03-advanced/04-ub-and-the-optimizer.md)
- **UndefinedBehaviorSanitizer (UBSan)** — *run-time checks for many UB kinds.* — [I3](02-intermediate/03-undefined-behavior.md), [I8](02-intermediate/08-valgrind-and-sanitizers.md)
- **Union** — *members sharing the same storage.* Type punning through unions is defined in C. — [B7](01-beginner/07-structs-enums-and-file-io.md), [A1](03-advanced/01-object-representation-alignment-aliasing.md)
- **`unreachable()`** — *a C23 promise that this point is never reached.* Reaching it is UB. — [A4](03-advanced/04-ub-and-the-optimizer.md)
- **Unspecified behaviour** — *one of several allowed outcomes, undocumented.* (N3220 §3.5.4) — [I3](02-intermediate/03-undefined-behavior.md)
- **Use-after-free** — *accessing memory after `free`.* UB, often exploitable. — [I2](02-intermediate/02-dynamic-memory-and-data-structures.md), [I8](02-intermediate/08-valgrind-and-sanitizers.md)

## V

- **Valgrind (Memcheck)** — *runs a binary on a synthetic CPU and checks every memory access.* 20–30× slower; no stack-array checks. — [I8](02-intermediate/08-valgrind-and-sanitizers.md)
- **Variable-length array (VLA)** — *an array whose size is a run-time value.* Optional (automatic VLAs) in C11–C23 via `__STDC_NO_VLA__`. — [B4](01-beginner/04-arrays-and-strings.md), [A5](03-advanced/05-c23-features.md)
- **Vectorisation** — *turning loops into SIMD instructions.* `-fopt-info-vec`. — [P5](04-pro/05-performance-and-research-frontiers.md)
- **`volatile`** — *every access must really happen.* For hardware registers and signal flags; not for thread synchronisation. — [A6](03-advanced/06-concurrency-and-atomics.md), [P1](04-pro/01-processes-and-signals.md)

## W

- **Watchpoint** — *a breakpoint triggered by a value changing.* `watch`, `rwatch`, `awatch` in GDB. — [I7](02-intermediate/07-debugging-with-gdb.md)
- **W^X** — *memory is writable or executable, never both.* Visible in `/proc/self/maps`. — [B6](01-beginner/06-memory-layout-stack-vs-heap.md)

## Z

- **Zombie process** — *a finished child whose status hasn't been collected.* Reap with `waitpid`. — [P1](04-pro/01-processes-and-signals.md)

---

## Sources

- WG14 N3220 (C23-equivalent draft): https://open-std.org/JTC1/SC22/WG14/www/docs/n3220.pdf
- cppreference C language reference: https://en.cppreference.com/w/c/language
- Linux man pages (man7.org): https://man7.org/linux/man-pages/
- Each lesson's own Sources section (linked above per term).

## Verify yourself

1. Pick five terms and check the "precise meaning" against the lesson's cited source (the N3220 clause or man page).
2. For every term you cannot explain without looking, open the linked lesson and redo its self-check questions.
