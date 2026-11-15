# Glossary (A–Z)

Each entry: **term** — plain meaning. *Precise:* technical definition. → where it is taught.

## A

**ABI (application binary interface)** — the rules compiled code must follow to call other compiled code. *Precise:* register usage for arguments/returns, callee/caller-saved registers, stack alignment, data layout (e.g. System V AMD64: integer args in `rdi, rsi, rdx, rcx, r8, r9`, result in `rax`, 16-byte stack alignment at calls). → 01-beginner/05, 03-advanced/01

**Abstract syntax tree (AST)** — the program's structure as a tree, without punctuation. *Precise:* tree of operators/constructs (interior nodes) and operands (leaves), omitting parentheses and chain productions; usually annotated with types and source locations after semantic analysis. → 01-beginner/04

**Alias analysis** — figuring out whether two pointers can refer to the same memory. *Precise:* an analysis returning no/may/must-alias for pairs of memory accesses; type-based alias analysis (TBAA) uses C's strict-aliasing rules. → 02-intermediate/07, 03-advanced/04, 03-advanced/06

**Alive / Alive2** — tools that prove LLVM optimizations correct. *Precise:* Alive (PLDI 2015) verifies peephole rewrites written in a DSL using Z3; Alive2 (PLDI 2021) performs bounded translation validation of LLVM IR functions via refinement checking. → 04-pro/06

**Ambiguous grammar** — a grammar that allows two readings of one sentence. *Precise:* some string has two distinct parse trees (equivalently, two leftmost derivations). → 01-beginner/03

**Attribute grammar** — a grammar where symbols carry computed values. *Precise:* Knuth (1968); synthesized attributes flow up the parse tree, inherited attributes flow down. → 02-intermediate/02

**Available expressions** — expressions already computed on every path. *Precise:* forward must-analysis with intersection meet; gen = expressions computed in a block, kill = expressions whose operands the block redefines. → 02-intermediate/06

## B

**Back edge** — an edge that jumps backwards to form a loop. *Precise:* a CFG edge n → h where h dominates n; h is the natural loop's header. → 02-intermediate/03–04

**Basic block** — a straight run of instructions with no jumps in or out of the middle. *Precise:* maximal instruction sequence with a single entry (first instruction) and single exit (last instruction). → 02-intermediate/03

**Bison / yacc** — parser generators. *Precise:* produce LALR(1) (by default) shift/reduce parsers from a grammar with semantic actions; resolve conflicts via `%left/%right/%prec`, defaulting to shift. → 02-intermediate/01

**Block arguments** — MLIR's replacement for φ-functions. *Precise:* blocks declare typed arguments; branch terminators pass values for them. → 03-advanced/05

## C

**C-Reduce** — an automatic shrinker for failing C programs. *Precise:* a fixpoint over pluggable transformations (line deletion, balanced-delimiter removal, ~30 source-to-source transformations) guided by a user-supplied interestingness test (Regehr et al., PLDI 2012). → 04-pro/04

**Calling convention** — see *ABI*. → 01-beginner/05

**CFG (control-flow graph)** — a map of which block can run after which. *Precise:* directed graph of basic blocks with an entry node; an edge B₁→B₂ means control can pass directly from the end of B₁ to the start of B₂. → 02-intermediate/03

**Coalescing** — removing register-to-register copies by merging their values. *Precise:* merging non-interfering nodes connected by a copy in the interference graph; conservative coalescing only merges when colourability is preserved. → 03-advanced/02

**CompCert** — a formally verified C compiler. *Precise:* compiler from (a large subset of) C to assembly with a Coq proof that generated code behaves as prescribed by the source semantics (Leroy, CACM 2009). → 04-pro/06

**Constant folding / propagation** — computing constant expressions at compile time / replacing variables known to be constant. *Precise:* folding evaluates operations on constant operands; propagation substitutes known constant values into uses (SCCP also prunes non-executable edges). → 02-intermediate/07

**Context-free grammar (CFG — grammar sense)** — rules describing a language's syntax. *Precise:* 4-tuple (N, Σ, P, S) of nonterminals, terminals, productions A → α, start symbol. → 01-beginner/03

**Convergent operation** — a GPU operation whose meaning depends on which threads reach it together. *Precise:* inter-thread communication (barrier, ballot, shuffle) where "the set of threads which participate … is implicitly affected by control flow"; LLVM's `convergent` attribute forbids making such calls control-dependent on new conditions. → 04-pro/05

**Csmith** — a random generator of UB-free C programs for compiler testing. *Precise:* grammar-aided generator with safe-math wrappers, pointer/effect analyses and a checksum-of-globals oracle hook (Yang et al., PLDI 2011). → 04-pro/01–02

## D

**Dataflow analysis** — computing facts that hold along program paths. *Precise:* solve equations IN/OUT per block (direction, meet, transfer functions) by fixpoint iteration over a lattice; terminates for monotone transfer functions on finite-height lattices. → 02-intermediate/06

**ddmin (delta debugging)** — an algorithm that shrinks a failing input. *Precise:* Zeller & Hildebrandt (TSE 2002): test subsets and complements of n chunks, increase granularity when neither fails; yields a 1-minimal failing input; worst case n² + 3n tests. → 04-pro/04

**Dead-code elimination (DCE)** — deleting code whose results are never used or that can't run. → 02-intermediate/07

**DFA / NFA** — finite automata that recognise patterns. *Precise:* deterministic (one transition per state and symbol) vs nondeterministic (several, plus ε-moves); equivalent in power (regular languages); NFA→DFA by subset construction. → 01-beginner/02

**Dialect (MLIR)** — a namespace of MLIR operations, types and attributes, e.g. `arith`, `scf`, `llvm`, `gpu`. → 03-advanced/05

**Differential testing** — compare two implementations on the same input. *Precise:* (McKeeman 1998) any output disagreement indicates a bug in at least one; variants: cross-compiler, cross-optimization, cross-version. → 01-beginner/05, 04-pro/01

**Divergence (GPU)** — threads of one warp taking different branches. *Precise:* the warp executes each taken path serially with inactive threads masked; a divergent value may differ between threads of a warp. → 04-pro/05

**Dominator / dominance frontier** — "always passes through" / "where that stops being true". *Precise:* d dominates n if every path from entry to n contains d; DF(d) = {y | d dominates a predecessor of y but does not strictly dominate y}. → 02-intermediate/04

## E

**EMI (Equivalence Modulo Inputs)** — a metamorphic compiler-testing technique. *Precise:* P and Q are EMI w.r.t. inputs I if ∀ i ∈ I, ⟦P⟧(i) = ⟦Q⟧(i); Orion creates Q by deleting code not executed on I (Le, Afshari, Su, PLDI 2014). → 04-pro/03

## F

**FIRST / FOLLOW sets** — which tokens can start / follow a grammar symbol. *Precise:* FIRST(α) = terminals beginning strings derived from α (+ε); FOLLOW(A) = terminals that can appear immediately after A in a sentential form; computed by fixpoint iteration. → 02-intermediate/01

**Freeze** — LLVM instruction that turns undef/poison into an arbitrary but fixed value. *Precise:* "used to stop propagation of undef and poison values". → 03-advanced/04

## G

**GEP (getelementptr)** — LLVM's address-computation instruction. *Precise:* computes the address of a sub-element of an aggregate; "does not access memory". → 02-intermediate/05

**GIMPLE / GENERIC / RTL** — GCC's IRs. *Precise:* GENERIC = language-independent trees; GIMPLE = "a three-address representation derived from GENERIC" (and its SSA form); RTL = low-level "Register Transfer Language". → 02-intermediate/03, 03-advanced/01

**GlobalISel / SelectionDAG** — LLVM's instruction selectors. *Precise:* SelectionDAG builds a per-basic-block DAG; GlobalISel works on whole functions in Machine IR (IRTranslator → Legalizer → RegBankSelect → InstructionSelect). → 03-advanced/01

**GOT / PLT** — tables that let position-independent code reach shared-library symbols. *Precise:* the GOT holds addresses filled by the dynamic linker; PLT stubs jump through GOT entries for function calls (lazily or at start-up with `-z now`). → 03-advanced/03

**GVN (global value numbering)** — finding redundant computations across a function. → 02-intermediate/07

## I

**Implementation-defined behaviour** — behaviour each implementation must document (e.g. `sizeof(int)`). *Precise:* C11 §3.4.1. → 03-advanced/04

**Inlining** — replacing a call with the callee's body. → 02-intermediate/07

**Instruction selection** — choosing target instructions for IR operations. *Precise:* tree/DAG tiling with target patterns (maximal munch, dynamic programming, SelectionDAG/GlobalISel). → 03-advanced/01

**Interestingness test** — the predicate a reducer uses to keep a candidate. *Precise:* returns success iff the candidate still triggers *the* failure and is valid (compiles, UB-free, terminates). → 04-pro/04

**Interference graph** — which values can't share a register. *Precise:* nodes = values; edge if one is live at the other's definition. → 03-advanced/02

## L

**Lattice** — an ordered set where any two elements have a meet and a join; the domain of dataflow facts. → 02-intermediate/06

**LCSSA (loop-closed SSA)** — LLVM form where values leaving a loop pass through φs in exit blocks. → 02-intermediate/05

**Leaders** — first instructions of basic blocks: the first instruction, jump targets, instructions after jumps. → 02-intermediate/03

**Left recursion** — a rule that refers to itself first (`E → E + T`). *Precise:* fatal for recursive descent (infinite recursion); removed by A → βA′, A′ → αA′ | ε. → 01-beginner/03

**Lexer (scanner)** — turns characters into tokens. → 01-beginner/02

**LICM (loop-invariant code motion)** — hoisting unchanging computations out of loops into the preheader. → 02-intermediate/07

**Linear scan** — fast register allocation over live intervals (Poletto & Sarkar, TOPLAS 1999). → 03-advanced/02

**Linker** — combines object files, resolves symbols, applies relocations. → 03-advanced/03

**Liveness** — whether a value may still be used later. *Precise:* backward may-analysis; gen = used before defined, kill = defined. → 02-intermediate/06

**LL(1) / LR(1) / LALR(1)** — parser classes. *Precise:* LL = left-to-right, leftmost derivation (top-down); LR = left-to-right, rightmost derivation in reverse (bottom-up); (1) = one token of lookahead; LALR merges LR(1) states with equal cores. → 02-intermediate/01

**LLVM IR** — LLVM's SSA-based, typed intermediate language (text `.ll`, bitcode `.bc`, in-memory). → 02-intermediate/05

**LTO (link-time optimization)** — optimizing across files at link time using IR stored in object files. → 03-advanced/03

## M

**Maximal munch** — always take the longest match (lexing) / the biggest tile (instruction selection). → 01-beginner/02, 03-advanced/01

**mem2reg** — LLVM pass promoting stack slots (`alloca`) to SSA registers with φs. → 02-intermediate/04

**Metamorphic testing** — testing via known relations between outputs of related inputs (EMI is one). → 04-pro/03

**MIR (Machine IR)** — LLVM's post-isel, target-specific representation with virtual and physical registers. → 03-advanced/01

**MLIR** — a framework for multi-level, extensible IRs (dialects, regions, progressive lowering). → 03-advanced/05

**Monotone framework** — the theory of dataflow analysis (Kam & Ullman 1977): monotone transfer functions over a lattice. → 02-intermediate/06

## N

**`nsw` / `nuw`** — LLVM flags promising no signed/unsigned wrap; violation yields poison. → 02-intermediate/05, 03-advanced/04

**NVPTX** — LLVM's back end producing NVIDIA PTX. → 04-pro/05

## O

**Opaque pointer (`ptr`)** — LLVM's pointer type without a pointee type (only form since LLVM 17). → 02-intermediate/05

**`opt-bisect-limit`** — LLVM option running only the first N skippable passes, used to find the pass that changes behaviour. → 03-advanced/06

**Oracle (test oracle)** — the mechanism deciding whether a test passed. → 01-beginner/05, 04-pro/01

## P

**Parse tree** — tree recording the grammar derivation, including punctuation and chain rules. → 01-beginner/03–04

**Pass / pass manager** — one transformation or analysis / the infrastructure that schedules passes and caches analyses. → 02-intermediate/07

**Perses** — syntax-guided program reducer (ICSE 2018). → 04-pro/04

**φ (phi) function** — SSA merge: selects a value depending on which predecessor control came from. → 02-intermediate/04

**PIC / PIE** — position-independent code / executable. → 03-advanced/03

**Poison / undef** — LLVM's "erroneous result" values. *Precise:* undef = each use may see any bit-pattern; poison = result of an erroneous operation, propagates, and becomes immediate UB in certain uses; poison is stronger. → 03-advanced/04

**Precedence climbing / Pratt parsing** — one function parses all binary operators using a binding-power table. → 01-beginner/03

**PTX / SASS** — NVIDIA's virtual ISA / real GPU machine code. → 04-pro/05

## R

**Reaching definitions** — which assignments may reach a point. *Precise:* forward may-analysis with union. → 02-intermediate/06

**Refinement** — the correctness notion for compilation: every behaviour of the target is a behaviour of the source. → 02-intermediate/07, 04-pro/06

**Register allocation** — mapping unlimited virtual registers to physical registers, spilling the rest. → 03-advanced/02

**Relocation** — a hole in an object file plus instructions for filling it. *Precise:* "a computation to perform on the contents" (typically S + A − P). → 03-advanced/03

## S

**Saturation** — when a test generator stops finding new bugs in a compiler (YARPGen). → 04-pro/01

**SCCP** — sparse conditional constant propagation (Wegman & Zadeck, TOPLAS 1991). → 02-intermediate/07

**Scope** — the region of program text where a declaration is visible. → 02-intermediate/02

**Semantic analysis** — checking meaning: name resolution, type checking, implicit conversions. → 02-intermediate/02

**SIMT** — single instruction, multiple threads (GPU execution model). → 04-pro/05

**Spill / reload** — storing a value to memory when registers run out / loading it back. → 03-advanced/02

**SSA (static single assignment)** — every variable assigned exactly once in the program text. → 02-intermediate/04

**Strict aliasing** — C's rule that objects may be accessed only through compatible types (enables TBAA). → 03-advanced/04

**Symbol table** — compiler (or linker) map from names to their information. → 02-intermediate/02, 03-advanced/03

## T

**Three-address code (TAC)** — IR where each instruction has at most one operator and named temporaries. → 02-intermediate/03

**Token / lexeme** — classified unit / its source text. → 01-beginner/02

**Translation validation** — checking each compilation's output against its input instead of verifying the compiler (Pnueli et al. 1998; Alive2). → 04-pro/06

## U

**UBSan** — Clang/GCC run-time undefined-behaviour detector (`-fsanitize=undefined`). → 03-advanced/04

**Undefined behaviour (UB)** — behaviour for which the standard "imposes no requirements" (C11 §3.4.3). → 03-advanced/04

**Uniformity analysis** — determining which values are the same across a GPU warp. → 04-pro/05

**Unspecified behaviour** — the standard allows several outcomes without documenting which (e.g. argument evaluation order). → 03-advanced/04

## V

**Value numbering** — giving equal computations the same number to detect redundancy (local LVN, global GVN). → 02-intermediate/07

**Verifier** — checker that rejects malformed IR (LLVM, MLIR). → 02-intermediate/05, 03-advanced/05

## W

**Warp** — group of GPU threads executing together ("Typically, a warp has 32 threads"). → 04-pro/05

**Weak symbol** — a definition that a strong one may override at link time. → 03-advanced/03

**Worklist algorithm** — iterate only on items whose inputs changed. → 02-intermediate/06

## Y

**YARPGen** — random C/C++ program generator using generation policies and analysis-based UB avoidance (OOPSLA 2020). → 04-pro/02

## Sources

* LLVM Language Reference: https://llvm.org/docs/LangRef.html
* GCC Internals: https://gcc.gnu.org/onlinedocs/gccint/
* MLIR Language Reference: https://mlir.llvm.org/docs/LangRef/
* ISO C N1570 draft (§3.4 behaviours): https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf
* NVIDIA PTX ISA: https://docs.nvidia.com/cuda/parallel-thread-execution/index.html
* LLVM Convergent Operation Semantics: https://llvm.org/docs/ConvergentOperations.html
* Aho, Lam, Sethi, Ullman, *Compilers: Principles, Techniques, and Tools*, 2nd ed.: https://www.pearson.com/store/en-us/p/compilers-principles-techniques-and-tools/P200000003472
* Each lesson's own Sources section gives the primary source for its terms.

## Verify yourself

* Pick five terms at random and find the precise definition in the linked lesson and in its primary source (e.g. `freeze` → LangRef; dominance frontier → Cytron et al. 1991).
* Run `opt --print-passes | grep -E "^ *(sccp|gvn|licm|mem2reg|instcombine)"` to confirm the pass names used in this glossary exist in your LLVM version.
