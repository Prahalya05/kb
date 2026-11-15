# Interview questions with model answers

Grouped by level. Model answers are written the way you would say them out loud: the core idea first, then one concrete detail or example that shows you have actually done it. Every factual claim is backed by a lesson (linked) and its sources.

## Fundamentals

**1. Walk me through what happens when you run `gcc -O2 hello.c -o hello`.**
`gcc` is a driver. It runs the preprocessor (macros, `#include`), then `cc1`, the compiler proper — lexing, parsing, semantic analysis, lowering to GIMPLE, SSA-based optimization passes, lowering to RTL, instruction selection, register allocation — producing assembly; then `as` turns that into an object file with relocations; then `collect2`/`ld` links it with the C runtime and libc into an executable. You can see each step with `gcc -v`, and Clang prints its plan with `-ccc-print-phases`. One detail I like: at `-O1`, Ubuntu's GCC replaced my `printf` with `__printf_chk` because the distro enables `_FORTIFY_SOURCE` — the defaults are part of the toolchain. → 01-beginner/01

**2. Why do compilers have an intermediate representation?**
Two reasons. Engineering: with *M* languages and *N* targets, a shared IR needs *M + N* components instead of *M × N* compilers — the LLVM and GCC design. Technical: optimizations need a representation that is simpler than source (no syntax sugar, explicit control flow) and more abstract than machine code (no registers or addressing modes); SSA-based IRs like LLVM IR make def-use relationships explicit. → 01-beginner/01, 02-intermediate/03

**3. What is maximal munch, and where does it show up?**
Take the longest possible match. In lexing it means `letter` is one identifier, not the keyword `let` plus `ter`, and `>=` is one token; flex's rule is "longest match, then earliest rule". In instruction selection it means "cover the tree with the biggest matching tile first", which on x86 folds a scaled-index address calculation and a load into one instruction. → 01-beginner/02, 03-advanced/01

**4. Why can't a recursive-descent parser handle `E → E + T`, and how do you fix it while keeping `+` left-associative?**
`E()` would call `E()` before consuming a token — infinite recursion. Rewrite as `E → T { + T }` and implement the repetition as a loop that folds each new operand into the left result (`left = new_node(left, op, right)`), which keeps `a - b - c` grouped as `(a - b) - c`. Precedence climbing generalises this with a table of binding powers: recurse with `prec + 1` for left-associative operators, `prec` for right-associative ones. → 01-beginner/03

**5. LL(1) vs LALR(1) — what's the difference and which do real compilers use?**
LL(1) is top-down and predicts the production from one lookahead token; it needs FIRST/FOLLOW-disjoint alternatives and no left recursion. LR parsers are bottom-up shift/reduce automata that accept a strictly larger class of grammars, including left recursion; LALR(1) merges LR(1) states with the same core, which is what Bison builds by default. Production C/C++ front ends — GCC since 3.4 (C++) and 4.1 (C), Clang — use hand-written recursive descent for better error messages and control; generators remain common for DSLs and SQL. → 02-intermediate/01

**6. What does semantic analysis do that parsing can't?**
Context-sensitive checks: resolving every name to its declaration with scoping and shadowing, type checking, inserting implicit conversions, and errors like redeclaration or wrong argument counts. Implementation: a stack of scopes for the symbol table and a post-order walk computing each expression's type, with an error type to stop cascades. The classic C trap it exposes: `-1 < 1u` is false because the usual arithmetic conversions turn `-1` into `4294967295` — Clang's AST shows the inserted `ImplicitCastExpr <IntegralCast>`. → 02-intermediate/02

## IR, SSA, analysis

**7. What is SSA and why do optimizers love it?**
Every variable is assigned exactly once in the program text; at control-flow merges a φ-function picks the incoming value by predecessor. Each use has exactly one reaching definition, so def-use chains are pointers and many analyses become sparse and simpler (constant propagation, value numbering, DCE). LLVM front ends don't build it themselves: they emit `alloca`/`load`/`store`, and `mem2reg` inserts the φs. → 02-intermediate/04

**8. How are φ-functions placed?**
Cytron et al.: compute dominators, then dominance frontiers — DF(d) is where d's dominance ends — and place a φ for variable *v* at the iterated dominance frontier of the blocks that define *v*; then rename with a dominator-tree walk and a stack of versions per variable. I implemented it with the Cooper–Harvey–Kennedy algorithms: iterative dominators using postorder-number "fingers", and DF by walking from each join's predecessors up to its idom. → 02-intermediate/04

**9. Explain dataflow analysis using liveness as the example.**
A variable is live at a point if it may be read later before being overwritten. Liveness is a backward "may" analysis: live-out(B) = ∪ live-in(successors), live-in(B) = use(B) ∪ (live-out(B) − def(B)). Solve with a worklist until nothing changes; it terminates because transfer functions are monotone over a finite lattice. Uses: dead-code elimination and register allocation (the number of simultaneously live values is the register pressure). Must-analyses like available expressions use intersection and start at ⊤. → 02-intermediate/06

**10. What's the difference between MOP and MFP?**
Meet-over-all-paths is the ideal: apply the transfer functions along every path and meet the results. The iterative algorithm computes the maximal fixed point, meeting at every join. They coincide when transfer functions are distributive (all gen/kill problems); otherwise — e.g. constant propagation — MFP is a safe approximation. → 02-intermediate/06

## Optimizations and back end

**11. Pick three optimizations and explain when they are *not* allowed.**
LICM: can't hoist something that might trap (a division, a load) out of a loop that might execute zero times, nor a load if a store in the loop may alias it. Strength reduction: `x / 4` → shift is only valid for unsigned or with a correction for negative signed values — Clang emits `lea 3(%rdi)`, `cmovns`, `sar $2`. CSE: two `*p` loads around a store through a possibly-aliasing pointer are not redundant. → 02-intermediate/07

**12. Why does inlining matter so much?**
Not for the call overhead — for what it enables: after inlining, the callee's body is specialised to the caller's arguments, so constant propagation, CSE, LICM and DCE fire. Clang's remark showed `sq` inlined into `inl` with `cost=-15030, threshold=337`, and `globaldce` then deleted the `static` function entirely. → 02-intermediate/07

**13. How does register allocation work?**
Build liveness, then either an interference graph coloured with *k* colours — Chaitin–Briggs: simplify nodes of degree < k, optimistically push a high-degree node when stuck, then select colours, spilling those that get none and iterating — or linear scan over live intervals for speed (JITs). LLVM's default greedy allocator uses a priority queue, eviction, and live-range splitting before spilling. Measurement matters: on an adversarial 20-live-value loop, LLVM 18's greedy allocator produced code about 2× slower than `-regalloc=basic` or `pbqp` in my tests — heuristics have bad cases. → 03-advanced/02

**14. What do the GOT and PLT do?**
Shared libraries and PIEs can't patch addresses into code, so external data addresses live in the GOT, filled by the dynamic linker, and calls to external functions go through PLT stubs that jump via GOT entries. Classic ELF binds lazily on first call, but Ubuntu's GCC passes `-z now -z relro` by default — the binary shows `BIND_NOW` — so the GOT can be made read-only after start-up. → 03-advanced/03

**15. Why does library order matter on the GNU linker command line?**
GNU ld searches an archive once, at its position, extracting only members that define symbols undefined at that moment. `gcc -L. -lutil main.o` fails; `gcc main.o -L. -lutil` works. → 03-advanced/03

## Undefined behaviour and debugging

**16. What is undefined behaviour and why do compilers exploit it?**
The C standard imposes no requirements on it (e.g. signed overflow, null dereference, strict-aliasing violations). Compilers assume it never happens and derive facts from that: `x + 1 > x` folds to true, a loop's trip count is known, a null check after a dereference is deleted (the Linux `tun` bug in the STACK paper). In LLVM, signed C arithmetic carries `nsw`, so overflow yields poison. I've seen Clang print 0 at `-O0` and 1 at `-O2` for `INT_MAX + 1 > INT_MAX` — both conforming. → 03-advanced/04

**17. Undefined vs unspecified vs implementation-defined?**
Implementation-defined: one choice, documented (size of `int`). Unspecified: several allowed outcomes, not documented (argument evaluation order — GCC evaluated `pair(f(), g())` right-to-left, Clang left-to-right). Undefined: no requirements at all. All three can break naive differential testing. → 03-advanced/04

**18. A program behaves differently at -O0 and -O2. How do you debug it?**
First rule out UB: UBSan/ASan, and probes like `-fwrapv`, `-fno-strict-aliasing`, `-fno-delete-null-pointer-checks`. Then localise: with Clang, binary-search `-mllvm -opt-bisect-limit=N` to find the first pass that changes the output and diff the IR before/after it; with GCC, `-fdump-tree-all` plus `-fdisable-tree-<pass>`/`-fno-<opt>`. Then reduce the input (C-Reduce / llvm-reduce) and report with exact versions and a single `opt -passes=…` reproducer. Caveat from practice: the first pass that changes behaviour is a lead, not a verdict — disabling GCC's FRE didn't change a strict-aliasing result because later passes do the same thing. → 03-advanced/06

## Compiler testing and verification

**19. How do you test a compiler when you don't know the expected output?**
Differential testing (McKeeman 1998): compile with several compilers, optimization levels or versions and compare; majority vote identifies the odd one. Metamorphic testing: transform the program so the expected output is known — EMI deletes code not executed on the given inputs, so the output must not change. Crash oracles for ICEs and hangs. And formal methods: translation validation (Alive2) or verified compilers (CompCert). The inputs must be UB-free, deterministic and terminating, otherwise mismatches are false alarms. → 04-pro/01, 04-pro/03, 04-pro/06

**20. How does Csmith avoid undefined behaviour?**
By construction: safe-math wrappers that return the first operand when an operation would be undefined (I read `safe_div` in Csmith's `safe_math.h`: zero divisor or `INT_MIN / -1` → return `si1`), whole-program pointer and effect analyses (no unsequenced read/write conflicts, which also handles unspecified argument order), in-bounds array indexing via modulo and loop structure, explicit initialisation, and a dataflow fixpoint when adding loop back edges. Programs end by printing a checksum of all globals. In my own experiment, a generator with wrappers gave 0/100 UB programs and 0 mismatches; the same generator with raw operators gave 100/100 UB and 44 false "mismatches". → 04-pro/02

**21. What is EMI and why did it find so many bugs?**
Equivalence Modulo Inputs: profile a program on inputs I, delete statements never executed on I; the variant must behave identically on I. The compiler can't know the code is dead on I, so the deletions change what the optimizer sees while the semantics on I stay fixed — a free oracle needing only one compiler. Orion found 147 confirmed GCC/LLVM bugs in 11 months; Athena and Hermes extended it with insertions and live-code mutations. I built a mini-Orion with gcov and validated it with a control experiment (deleting executed statements must produce violations — it did, 24/24). → 04-pro/03

**22. Explain delta debugging. What goes wrong in practice?**
ddmin splits the failing input into n chunks and tries each chunk and each complement; on failure-preservation it recurses, otherwise doubles granularity; it ends with a 1-minimal input. In practice: interestingness tests must check the *same* failure, validity, UB-freedom and termination. I hit both classic accidents — a C-Reduce run stalled because a candidate deleted the loop increment and hung (fix: timeouts on every step), and an llvm-reduce test was vacuously satisfied by the reduced input already containing the target instruction (fix: require the input not to have it). C-Reduce then shrank an 825-byte failing program to `print(0 <= 0);`. → 04-pro/04

**23. What's specific about GPU compilers?**
SIMT: a warp of typically 32 threads executes one instruction at a time, so divergent branches serialise; compilers use uniformity analysis and if-conversion (Clang turned my divergent `if/else` into a `selp`). Convergent operations like barriers must not be made control-dependent on new conditions — moving `__syncthreads()` into both branches is legal on a CPU and wrong on a GPU. PTX is a virtual ISA; `ptxas` or the driver JIT produce SASS. Testing is harder: closed toolchains, races, floating-point contraction; prior work includes CLsmith, GLFuzz and spirv-fuzz. → 04-pro/05

**24. What does CompCert guarantee and what not?**
A Coq proof that generated code behaves as prescribed by the source semantics, for the verified passes. Csmith found no wrong-code bugs in its verified core after about six CPU-years, but did find bugs in the then-unverified front end and at the assembler boundary — the guarantee covers exactly what is verified. → 04-pro/06

**25. What does Alive2 check and what's "refinement"?**
For an LLVM transformation it checks that the target refines the source: every behaviour of the target is a behaviour of the source. If the source is UB/poison for an input, any target behaviour is fine; but the target must not introduce new UB. It uses an SMT solver and is bounded (loops unrolled to a limit). Running it over LLVM's unit tests found 47 new bugs, 28 fixed at publication. In my Z3 mini-Alive, `x / 2 ⇒ x >> 1` fails at x = −49, and `x + 1 > x ⇒ true` is wrong for wrapping add but correct for `add nsw`. → 04-pro/06

## Project-style questions

**26. Describe a compiler-testing tool you would build and how you'd validate it.**
A model answer outline: generator with stated validity guarantees (UB-free by construction, terminating, deterministic) → oracle (cross-compiler × cross-optimization; EMI as a second oracle) → triage (UB re-check, dedup, ddmin/C-Reduce, pass bisection) → validation of the tool itself (plant a bug and show it's caught and reduced; run a control experiment; measure the false-positive rate with UB deliberately allowed) → evaluation (bugs reported/confirmed/fixed/duplicates; comparison to Csmith/YARPGen under equal time; saturation curves). → 01-beginner/05, 04-pro/01–04

**27. How would you extend CPU compiler fuzzing to GPU compilers?**
Generate kernels that are race-free and barrier-uniform by construction; produce a CPU reference by emulating the block phase by phase; compare GPU results from different toolchains/levels against it with tolerances for floating point; check barrier placement with uniformity analysis/post-dominance in the IR before running; reduce with C-Reduce using the CPU oracle. The mini project in `exercises.md` (Project 3) implements the generator and CPU oracle. → 04-pro/05, exercises

## Sources

* Each answer links its lesson; the lessons' Sources sections contain the primary references (C11 N1570, LLVM/GCC/MLIR/NVIDIA documentation, and the papers: Cytron et al. 1991; Cooper–Harvey–Kennedy; Kildall 1973; Chaitin 1982; Poletto & Sarkar 1999; McKeeman 1998; Yang et al. 2011; Zeller & Hildebrandt 2002; Regehr et al. 2012; Le et al. 2014; Wang et al. 2013; Leroy 2009; Lopes et al. 2015, 2021).
* GCC 3.4 / 4.1 release notes (hand-written parsers): https://gcc.gnu.org/gcc-3.4/changes.html, https://gcc.gnu.org/gcc-4.1/changes.html
* LLVM OptBisect: https://llvm.org/docs/OptBisect.html

## Verify yourself

* For each answer that quotes a measured number (questions 1, 13, 14, 17, 20, 21, 22, 25), rerun that lesson's `Verify yourself` block and check the number on your machine — note which ones change with compiler version.
* Practise answering aloud in under two minutes, then check you mentioned (a) the core idea, (b) one limitation, (c) one concrete example.
