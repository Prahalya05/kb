# Common mistakes, collected

Every mistake below appears (with its fix and explanation) in the lesson named after it. Items marked **(hit while writing)** are mistakes that actually happened while building and testing this track — kept because real failures teach best.

## Front end

1. **Thinking `gcc` is one program** — it's a driver for `cpp`/`cc1`/`as`/`ld`; inspect with `gcc -v` or `clang -ccc-print-phases`. → 01-beginner/01
2. **Expecting `-O0` to mean "no transformations"** — GCC folded `x + 1 > x` to `return 1;` in its front end even at `-O0`. → 03-advanced/04
3. **Reporting compiler behaviour without version, flags and distro defaults** — Ubuntu's GCC swapped `printf` for `__printf_chk` and links with `-pie -z now -z relro` by default. → 01-beginner/01, 03-advanced/03
4. **Macro arguments without parentheses** — `SQ(1+2)` = `1+2*1+2`. → 01-beginner/01, exercises B-1
5. **Checking keywords before finishing the identifier** — `letter` lexed as `let` + `ter`; in flex, putting the identifier rule above keywords. → 01-beginner/02
6. **Passing a plain `char` to `isdigit`** — UB for negative values; cast to `unsigned char`. → 01-beginner/02
7. **Backtracking regexes on untrusted input** — exponential blow-up; DFA matching is linear. → 01-beginner/02
8. **Left recursion in recursive descent** — infinite recursion; convert to a loop. → 01-beginner/03
9. **Removing left recursion and accidentally making operators right-associative** — build the tree in the loop. → 01-beginner/03
10. **Not checking for trailing input** after parsing. → 01-beginner/03
11. **Silent shift/reduce conflicts in Bison; precedence lines in the wrong order** (later lines bind tighter). → 02-intermediate/01
12. **Checking redeclaration against all scopes** (breaks legal shadowing) / no error type (cascading errors). → 02-intermediate/02
13. **Assuming C's signed/unsigned conversions are intuitive** — `-1 < 1u` is false. → 02-intermediate/02
14. **Unbounded table in a toy interpreter** — a `let` inside a loop appended a new variable each iteration, overflowed a 256-entry array, corrupted memory and printed `1` forever. **(hit while writing)** → 01-beginner/04
15. **Computing with `long` in an interpreter whose language defines wrap-around** — makes the interpreter itself execute C UB; compute in `unsigned long`. → 01-beginner/04

## Code generation for the toy compiler

16. **Misaligned stack at `call`** (`%rsp` must be 16-byte aligned) and **forgetting `%al = 0`** before variadic calls. → 01-beginner/05
17. **Emitting a bare `idiv`** — traps on `x / 0` *and* `LONG_MIN / -1`. → 01-beginner/05
18. **Emitting `sdiv MIN, -1` and discarding the result with `select`** — still immediate UB in LLVM IR; divide by a safe divisor instead. **(hit while writing)** → exercises, Project 1

## IR, SSA, LLVM

19. **Forgetting the instruction after a branch starts a new basic block / missing fall-through edges.** → 02-intermediate/03
20. **Assuming GCC dump file numbers are stable** across versions. → 02-intermediate/03
21. **"SSA = assigned once at run time"** — it's once in the text. → 02-intermediate/04
22. **Placing φs only in the DF, not the iterated DF; treating φs in a block as sequential** (swap problem). → 02-intermediate/04
23. **Running `opt` on `clang -O0` IR** — `optnone` blocks passes, and `-O0` also adds `noinline` to every function (the inliner refused to inline `sq`). Use `-O1 -Xclang -disable-llvm-passes`. **(hit while writing)** → 02-intermediate/04, 02-intermediate/07
24. **Wrong string-constant length in hand-written IR** — `[12 x i8]` for an 11-byte string; the parser rejects it. **(hit while writing)** → 02-intermediate/05
25. **Using a value its definition doesn't dominate** — "Instruction does not dominate all uses!". → 02-intermediate/05
26. **Linking `llc` output into a PIE without `-relocation-model=pic`** — `R_X86_64_32 … recompile with -fPIE`. **(hit while writing)** → 02-intermediate/05
27. **Using typed-pointer syntax (`i32*`)** — removed in LLVM 17; use `ptr`. → 02-intermediate/05
28. **Running `licm` outside `loop-mssa(...)`** — "LICM requires MemorySSA (loop-mssa)"; module passes inside a cgscc pipeline (`globaldce`). **(hit while writing)** → 02-intermediate/07
29. **Feeding x86-targeted IR to `llc -march=riscv64`** — "'x86-64' is not a recognized processor…", then "RV64 target requires an RV64 CPU". Generate IR per target. **(hit while writing)** → 03-advanced/01

## Analysis and optimization

30. **Wrong dataflow direction or instruction order within a block; initialising a must-analysis to ∅.** → 02-intermediate/06
31. **Expecting identical warnings at every `-O`** — GCC 13's `-Wmaybe-uninitialized` printed nothing at `-O0` for a clear case; Clang's warns at all levels. → 02-intermediate/06
32. **Hoisting trapping instructions; strength-reducing signed division as a plain shift; CSE across aliasing stores.** → 02-intermediate/07
33. **Judging register allocators by static spill counts** — measure the hot loop; on an adversarial kernel LLVM 18's default greedy was ~2× slower than `basic`/`pbqp`. → 03-advanced/02

## Linking

34. **Libraries before objects with GNU ld; defining globals in headers (`-fno-common` since GCC 10); mismatched `extern` types across files.** → 03-advanced/03
35. **Assuming lazy binding** — check `readelf -d` for `BIND_NOW`. → 03-advanced/03

## Undefined behaviour

36. **Calling an `-O0`/`-O2` difference a compiler bug without ruling out UB.** → 03-advanced/04, 03-advanced/06
37. **Overflow checks that overflow; checking a pointer after dereferencing it; type punning through casts.** → 03-advanced/04
38. **Confusing unspecified with undefined** — GCC and Clang evaluate function arguments in different orders. → 03-advanced/04
39. **Trusting one sanitizer run** — GCC's UBSan reported nothing because the overflow had been folded away; Clang's UBSan needs `libclang-rt-18-dev` on Ubuntu. **(hit while writing)** → 03-advanced/04

## MLIR

40. **Missing conversion passes before `mlir-translate`** ("Dialect `arith' not found…"); trusting pass names from tutorials for other versions. → 03-advanced/05

## Debugging the optimizer

41. **Treating the first pass that changes behaviour as the culprit** — disabling GCC's FRE didn't change the strict-aliasing result; later passes repeat the transformation. → 03-advanced/06
42. **Forgetting which passes `opt-bisect-limit` can't skip** (analyses, `-O0` codegen passes, regalloc-required passes; LTO passes ignore driver `-mllvm`). → 03-advanced/06

## Compiler testing

43. **Generators that emit UB** — measured: raw operators gave UB in 100/100 programs and 44 false mismatches. → 04-pro/02
44. **No timeouts** — UB programs and reduced candidates can loop forever; a C-Reduce run stalled on "(Interestingness test killed by timeout at 300 seconds.)". **(hit while writing)** → 04-pro/02, 04-pro/04
45. **Treating run timeouts of generated programs as compiler bugs** — mark them inconclusive. → 04-pro/01
46. **Vacuous interestingness tests** — an llvm-reduce test satisfied by the reduced input alone. **(hit while writing)** → 04-pro/04
47. **Matching "any crash" during reduction** — the reducer drifts to a different bug. → 04-pro/04, exercises P-4
48. **EMI: deleting executed code, profiling optimized builds, line-granularity coverage traps.** → 04-pro/03
49. **gcov notes-file naming** — one-step `gcc --coverage f.c -o exe` (observed with GCC 13.3) names it `exe-f.gcno`, so `gcov f.c` can't find it; compile and link separately. **(hit while writing)** → 04-pro/03
50. **Never validating the oracle** — plant a bug or run a control experiment so "0 bugs found" means something. → 01-beginner/05, 04-pro/03

## GPU and verification

51. **Treating PTX as the final binary; shipping no PTX for future GPUs; expecting bit-identical floating point between CPU and GPU.** → 04-pro/05
52. **Barriers under divergent control flow; data races in generated kernels.** → 04-pro/05, exercises Project 3
53. **Equality instead of refinement in verification; forgetting target UB; proving at one bit-width only.** → 04-pro/06
54. **Mis-modelling semantics in an SMT encoding** — `BVMulNoOverflow(…, True)` alone gave a bogus counterexample; signed underflow needs `BVMulNoUnderflow`. **(hit while writing)** → exercises P-5
55. **Wrong builtin names without the CUDA SDK** — `__builtin_ptx_read_tid_x` was rejected by Clang 18; `__nvvm_read_ptx_sreg_tid_x` works. **(hit while writing)** → 04-pro/05

## Sources

Each item's lesson contains the explanation and its primary source (C11 N1570; LLVM LangRef, Passes, NewPassManager, OptBisect, OpaquePointers; GCC 10 release notes; GNU ld manual; Clang UBSan docs; C-Reduce and ddmin papers; Z3 API: https://z3prover.github.io/api/html/namespacez3py.html).

## Verify yourself

Pick any item marked **(hit while writing)** and reproduce it from the named lesson's `Verify yourself` block — e.g. compile the `[12 x i8]` version of `fact.ll` with `llvm-as`, or run `opt -passes=licm` on `base.ll` to see the MemorySSA error.
