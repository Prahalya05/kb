# Appendix: cheat sheets, tools, books, courses, papers, docs

Everything here was either used in a lesson (commands are the ones run in this track, GCC 13.3 / LLVM 18.1.3 / MLIR 18 / Csmith 2.3.0 / C-Reduce 2.11, Ubuntu 24.04, Nov 2026) or is a primary reference cited by a lesson.

## A. Cheat sheet — GCC and Clang drivers

| Goal | GCC | Clang |
|---|---|---|
| Show driver phases / sub-commands | `gcc -v file.c`, `gcc -### file.c` | `clang -ccc-print-phases file.c`, `clang -### file.c` |
| Preprocess only | `gcc -E file.c` | `clang -E file.c` |
| Assembly | `gcc -S -O2 file.c -o -` | `clang -S -O2 file.c -o -` |
| Object file | `gcc -c file.c` | `clang -c file.c` |
| Tokens / AST | — | `clang -fsyntax-only -Xclang -dump-tokens f.c`; `clang -fsyntax-only -Xclang -ast-dump f.c` |
| IR | `gcc -fdump-tree-gimple`, `-fdump-tree-ssa`, `-fdump-tree-optimized`, `-fdump-tree-all`, `-fdump-rtl-all` | `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c` (optimizable, unoptimized IR) |
| Optimization report | `-fopt-info` | `-Rpass='.*'`, `-Rpass-missed=…`, `-Rpass-analysis=…` |
| Disable one optimization | `-fno-tree-fre`, `-fdisable-tree-<pass>`, `-fdbg-cnt=…` | `-mllvm -opt-bisect-limit=N` |
| UB probes | `-fwrapv`, `-fno-strict-aliasing`, `-fno-delete-null-pointer-checks` | same flags |
| Sanitizers | `-fsanitize=undefined,address` | same (Ubuntu needs `libclang-rt-18-dev` for UBSan runtime) |
| LTO | `-flto` | `-flto -fuse-ld=lld` |
| Cross-target assembly | (needs cross GCC) | `clang --target=aarch64-linux-gnu -S`, `riscv64-linux-gnu`, `nvptx64-nvidia-cuda` |
| CUDA → PTX (no SDK) | — | `clang++ -x cuda --cuda-gpu-arch=sm_70 --cuda-device-only -nocudainc -nocudalib -S k.cu` |

## B. Cheat sheet — LLVM tools

```text
llvm-as x.ll -o x.bc          # text -> bitcode (+ verifier)      llvm-dis x.bc -o -   # bitcode -> text
lli x.ll                      # run IR directly (JIT)
opt -S -passes='mem2reg,instcombine' x.ll -o y.ll            # run passes
opt -S -passes='default<O2>' x.ll                            # whole -O2 pipeline
opt -passes='default<O2>' -print-pipeline-passes -disable-output x.ll
opt -S -passes='cgscc(inline),globaldce' x.ll      opt -S -passes='loop(loop-rotate),loop-mssa(licm)' x.ll
opt -passes='print<domtree>' -disable-output x.ll            # dominator tree
opt -passes='print<uniformity>' -disable-output gpu.ll       # GPU divergence
opt -passes=dot-cfg -disable-output x.ll; dot -Tpng .f.dot -o f.png
opt -passes=verify -disable-output x.ll
llc -O2 x.ll -o x.s   llc -relocation-model=pic x.ll   llc -regalloc=greedy x.ll   # or fast|basic|pbqp
llc -stop-after=finalize-isel x.ll -o -                      # Machine IR after isel
llc -debug-pass=Structure x.ll -o /dev/null                 # which codegen passes run
llvm-reduce --test=./interesting.sh in.ll -o out.ll
clang -O2 -mllvm -print-changed=diff-quiet -mllvm -filter-print-funcs=f -c f.c
```

## C. Cheat sheet — MLIR (18)

```text
mlir-opt in.mlir --canonicalize
mlir-opt in.mlir --mlir-print-op-generic
mlir-opt in.mlir --convert-scf-to-cf --convert-arith-to-llvm --convert-index-to-llvm \
    --convert-func-to-llvm --convert-cf-to-llvm --reconcile-unrealized-casts -o low.mlir
mlir-translate --mlir-to-llvmir low.mlir -o out.ll
mlir-reduce                        # MLIR's test-case reducer
```

## D. Cheat sheet — binary inspection

```text
nm f.o                 # symbols (U undefined, T text, D data, b local bss)
objdump -dr f.o        # disassembly + relocations
readelf -S f.o         # sections        readelf -r prog   # relocations (GLOB_DAT, JUMP_SLOT)
readelf -d prog        # dynamic section (BIND_NOW, NEEDED)
ldd prog               # shared-library dependencies
ar rcs libx.a a.o b.o  # static library
```

## E. Cheat sheet — compiler-testing tools

```text
csmith --seed 42 -o t.c && gcc -O2 -w -I/usr/include/csmith t.c && ./a.out   # prints checksum = <hex>
csmith --no-pointers --no-structs --max-funcs 3 -o t2.c                    # swarm-style feature toggles
creduce --n 4 ./interesting.sh t.c            # C programs
creduce --not-c ./interesting.sh t.txt        # any text (language-agnostic passes only)
gcc -O0 --coverage -c t.c && gcc --coverage t.o -o t && ./t && gcov t.c   # line coverage (EMI)
```

Interestingness-test template (exit 0 = still interesting):

```bash
#!/bin/bash
# file: interesting-template.sh
# Template: keep a C program only if it is UB-free and the two builds still disagree.
gcc -O0 -w t.c -o o0 && clang -O2 -w t.c -o o2 || exit 1                     # must compile
clang -O0 -w -fsanitize=undefined,address -fno-sanitize-recover=all t.c -o san || exit 1
timeout 5 ./san > /dev/null 2>&1 || exit 1                                    # no UB / memory errors
timeout 5 ./o0 > out0.txt 2>&1 || exit 1                                      # terminates
timeout 5 ./o2 > out2.txt 2>&1 || exit 1
! cmp -s out0.txt out2.txt                                                     # interesting: outputs differ
```

## F. Formulae and facts to remember

* ddmin: worst case n² + 3n tests; best case ≤ 2·log₂ n (single culprit).
* Dataflow: forward IN[B] = ⊓ OUT[preds], OUT[B] = f_B(IN[B]); may → ∪ start ∅; must → ∩ start ⊤.
* DF(d) = {y | d dom some pred of y, d does not strictly dom y}; φs at the iterated DF of definition blocks.
* LL(1): FIRST(α) ∩ FIRST(β) = ∅; if β ⇒* ε then FIRST(α) ∩ FOLLOW(A) = ∅.
* Bison: later `%left/%right` = higher precedence; unresolved shift/reduce → shift.
* Refinement: target behaviours ⊆ source behaviours; UB in source ⇒ anything.
* Csmith safe wrappers return the first operand when the operation would be undefined.

## G. Books

* A. V. Aho, M. S. Lam, R. Sethi, J. D. Ullman, *Compilers: Principles, Techniques, and Tools*, 2nd ed. (the Dragon Book) — theory-heavy; best for lexing/parsing/dataflow.
* K. D. Cooper, L. Torczon, *Engineering a Compiler*, 3rd ed., 2022 — engineering-oriented; best for IR, optimization, back end.
* A. W. Appel, *Modern Compiler Implementation in C / Java / ML* — register allocation (graph colouring with coalescing) explained well (recommended by Cornell's CS 4120 notes).
* J. R. Levine, *Linkers and Loaders*, 1999 — linking in depth.
* F. Rastello, F. Bouchez Tichadou (eds.), *SSA-based Compiler Design* (Springer, 2022) — research-level SSA [edition/year unverified].
* R. Nystrom, *Crafting Interpreters* (free online: https://craftinginterpreters.com) — a gentle, complete interpreter/bytecode-VM build.

## H. Courses

* Stanford CS143 Compilers — https://web.stanford.edu/class/cs143/ (classic undergraduate sequence; Cool project).
* Cornell CS 6120 Advanced Compilers (self-guided, Bril IR, LLVM) — https://www.cs.cornell.edu/courses/cs6120/2023fa/self-guided/ (data flow, SSA, LLVM, loops, alias analysis, dynamic compilers; course blog has student write-ups on EMI and Alive).
* Cornell CS 4120 notes (register allocation) — https://www.cs.cornell.edu/courses/cs4120/2022sp/notes/regalloc
* LLVM Kaleidoscope tutorial — https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/index.html
* MLIR Toy tutorial — https://mlir.llvm.org/docs/Tutorials/Toy/

## I. Papers, in suggested reading order

**Foundations**
1. Knuth 1965, "On the translation of languages from left to right" (LR parsing).
2. Kildall 1973, "A unified approach to global program optimization" (dataflow); Kam & Ullman 1977, "Monotone data flow analysis frameworks".
3. Cytron, Ferrante, Rosen, Wegman, Zadeck 1991, "Efficiently computing static single assignment form and the control dependence graph" (TOPLAS 13(4)).
4. Cooper, Harvey, Kennedy, "A simple, fast dominance algorithm".
5. Wegman & Zadeck 1991, "Constant propagation with conditional branches" (TOPLAS 13(2)).
6. Chaitin 1982, "Register allocation & spilling via graph coloring"; Poletto & Sarkar 1999, "Linear scan register allocation".
7. Lattner & Adve 2004, "LLVM: A compilation framework for lifelong program analysis & transformation" (CGO); Lattner et al. 2021, "MLIR: Scaling compiler infrastructure for domain specific computation" (CGO).

**Undefined behaviour**
8. Lattner 2011, "What every C programmer should know about undefined behavior" (LLVM blog, 3 parts).
9. Wang, Zeldovich, Kaashoek, Solar-Lezama 2013, "Towards optimization-safe systems: analyzing the impact of undefined behavior" (SOSP; STACK).

**Compiler testing**
10. McKeeman 1998, "Differential testing for software" (DTJ 10(1)).
11. Yang, Chen, Eide, Regehr 2011, "Finding and understanding bugs in C compilers" (PLDI; Csmith).
12. Zeller & Hildebrandt 2002, "Simplifying and isolating failure-inducing input" (TSE; ddmin).
13. Regehr et al. 2012, "Test-case reduction for C compiler bugs" (PLDI; C-Reduce).
14. Le, Afshari, Su 2014, "Compiler validation via equivalence modulo inputs" (PLDI; EMI/Orion); Le, Sun, Su 2015 (OOPSLA; Athena); Sun, Le, Su 2016 (OOPSLA; Hermes).
15. Lidbury, Lascu, Chong, Donaldson 2015, "Many-core compiler fuzzing" (PLDI; CLsmith).
16. Donaldson, Evrard, Lascu, Thomson 2017, "Automated testing of graphics shader compilers" (OOPSLA; GLFuzz).
17. Sun, Li, Zhang, Gu, Su 2018, "Perses: syntax-guided program reduction" (ICSE).
18. Chen et al. 2020, "A survey of compiler testing" (ACM Computing Surveys 53(1)).
19. Livinskii, Babokin, Regehr 2020, "Random testing for C and C++ compilers with YARPGen" (OOPSLA).
20. Donaldson et al. 2021, "Test-case reduction and deduplication almost for free with transformation-based compiler testing" (PLDI; spirv-fuzz).
21. Liu et al. 2023, "NNSmith: generating diverse and valid test cases for deep learning compilers" (ASPLOS).
22. Xia et al. 2024, "Fuzz4All: universal fuzzing with large language models" (ICSE).

**Verification**
23. Pnueli, Siegel, Singerman 1998, "Translation validation" (TACAS).
24. Leroy 2009, "Formal verification of a realistic compiler" (CACM; CompCert).
25. Lopes, Menendez, Nagarakatte, Regehr 2015, "Provably correct peephole optimizations with Alive" (PLDI).
26. Lopes, Lee, Hur, Liu, Regehr 2021, "Alive2: bounded translation validation for LLVM" (PLDI).
27. Hack, Grund, Goos 2006, "Register allocation for programs in SSA-form" (CC).

## J. Official documentation

* LLVM: LangRef, Passes, NewPassManager, CodeGenerator, GlobalISel, MIRLangRef, OptBisect, LinkTimeOptimization, NVPTXUsage, CompileCudaWithLLVM, ConvergentOperations, OpaquePointers, GetElementPtr, CommandGuide — https://llvm.org/docs/
* Clang: DriverInternals, IntroductionToTheClangAST, UndefinedBehaviorSanitizer, DiagnosticsReference, UsersManual — https://clang.llvm.org/docs/
* GCC: Internals (Passes, GIMPLE, RTL), Developer Options, Optimize Options, Warning Options, Gcov — https://gcc.gnu.org/onlinedocs/
* MLIR: LangRef, Dialects, Passes, Toy — https://mlir.llvm.org/docs/
* NVIDIA: nvcc, PTX ISA, NVVM IR — https://docs.nvidia.com/cuda/
* GNU: flex manual (https://westes.github.io/flex/manual/), Bison manual (https://www.gnu.org/software/bison/manual/), ld manual (https://sourceware.org/binutils/docs/ld/)
* System V AMD64 psABI — https://gitlab.com/x86-psABIs/x86-64-ABI
* Ian Lance Taylor's linker series — https://www.airs.com/blog/archives/38 (part 1; parts 2–20 follow)
* John Regehr's blog (compiler testing, C-Reduce, UB) — https://blog.regehr.org/

## K. Tool sources

Csmith https://github.com/csmith-project/csmith · C-Reduce https://github.com/csmith-project/creduce · YARPGen https://github.com/intel/yarpgen · Perses https://github.com/uw-pluverse/perses · Alive2 https://github.com/AliveToolkit/alive2 · Z3 https://github.com/Z3Prover/z3 · CompCert https://compcert.org/

## Sources

All references above are the sources; each lesson's own Sources section links the specific pages and papers quoted. Book list items without a lesson citation: *Crafting Interpreters* (https://craftinginterpreters.com), Alive2 repository (https://github.com/AliveToolkit/alive2), CompCert site (https://compcert.org/).

## Verify yourself

```bash
# file: verify-appendix.sh
# Spot-check that the cheat-sheet commands exist in your toolchain.
clang -ccc-print-phases -x c /dev/null 2>&1 | tail -1
opt --print-passes | grep -cE "^ *(mem2reg|sccp|licm)"
llc --help-hidden | grep -c "=pbqp "            # the -regalloc=pbqp choice
/usr/lib/llvm-18/bin/mlir-opt --help | grep -c "convert-scf-to-cf"
csmith --help | grep -c "no-pointers"
bash -n interesting-template.sh && echo "template parses"
```

The Rastello/Bouchez *SSA-based Compiler Design* edition details are marked [unverified]; check the publisher's page before citing.
