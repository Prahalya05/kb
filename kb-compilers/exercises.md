# Exercises (graded by level) and mini projects

Each exercise names the lesson it practises. Solutions are in collapsible blocks — try first. Every code solution was compiled and run (GCC 13.3, Clang/LLVM/MLIR 18.1.3, Python 3.13, Z3 5.4.1; Nov 2026).

## Level 1 — Beginner

### B-1 · The preprocessor is text (lesson 01)
What does `#define SQ(x) x*x` followed by `int r = SQ(1+2);` produce, and what value does `r` get? How do you fix the macro?

<details><summary>Solution</summary>

The preprocessor substitutes text: `SQ(1+2)` → `1+2*1+2`, which is `1 + 2 + 2 = 5`, not 9. Fix by parenthesising every use of the parameter and the whole body: `#define SQ(x) ((x)*(x))`. Check: `printf '#define SQ(x) x*x\nint r = SQ(1+2);\n' | gcc -E -P -` prints `int r = 1+2*1+2;`. (Even the fixed macro evaluates its argument twice — `SQ(i++)` is still wrong; a `static inline` function is better.)

</details>

### B-2 · Run the DFA by hand (lesson 02)
Using the `if`/identifier DFA from lesson 02 (states A, B, C, D), give the state sequence and the token(s) produced for the input `if1 ix`.

<details><summary>Solution</summary>

`[a-z]+` in the lesson's DFA has no digits, so `1` is not part of any token pattern there. Lexing proceeds token by token with maximal munch:

* `i` → B (accept IDENT), `f` → D (accept IF), `1` → no transition → stop. Last accepting state is D at length 2 → token **IF** (`if`).
* `1` → no token in this tiny language → lexical error (a real lexer would have an INT rule).
* space skipped; `i` → B, `x` → C (IDENT), end → token **IDENT** (`ix`).

Lesson learned: maximal munch picks the longest *accepting* prefix, then restarts after it.

</details>

### B-3 · Remove left recursion and code it (lesson 03)
Rewrite `L → L , x | x` without left recursion and implement a recursive-descent parser that counts the items in `"x, x, x"` and rejects `"x,,x"`.

<details><summary>Solution</summary>

`L → x { , x }` (equivalently `L → x L'`, `L' → , x L' | ε`). The `{…}` is a loop:

```c
// file: list.c
// Exercise B-3 solution: parse "x, x, x" with  L -> x { , x }  and count the items.
#include <stdio.h>
#include <stdlib.h>
static const char *p;
static void skip(void) { while (*p == ' ') p++; }
static void expect_x(void) {                     // one list element
    skip();
    if (*p != 'x') { fprintf(stderr, "error at '%s': expected x\n", p); exit(1); }
    p++;
}
static int list(void) {                          // L -> x { , x }   (a loop, no left recursion)
    int n = 1;
    expect_x();
    for (skip(); *p == ','; skip()) { p++; expect_x(); n++; }
    return n;
}
int main(int argc, char **argv) {
    p = argc > 1 ? argv[1] : "x, x, x";
    int n = list();
    skip();
    if (*p) { fprintf(stderr, "error: trailing '%s'\n", p); return 1; }
    printf("%d items\n", n);
    return 0;
}
```

```text
$ ./list "x, x, x"   → 3 items
$ ./list "x"         → 1 items
$ ./list "x,,x"      → error at ',x': expected x
```

</details>

### B-4 · Add `%` to Tiny everywhere (lessons 02–05)
Add the remainder operator `%` (same precedence as `*`) to the Tiny lexer/parser, interpreter and x86-64 compiler, with Tiny semantics: `x % 0` is a run-time error, `LONG_MIN % -1` is `0`, otherwise C's `%` (sign follows the dividend). Test that interpreter and compiler agree on `-7 % 3`, `7 % -3`, `LONG_MIN % -1`, `10 % 4 * 3`.

<details><summary>Solution</summary>

Three edits.

`tiny_front.h` — lexer accepts `%`, and `mul` loops over it:

```c
    if (strchr("+-*/%<>=(){};", *src)) {   /* lexer: '%' added to the single-char operator set; body unchanged */
    while (is_op("*") || is_op("/") || is_op("%")) {   /* mul(): same level as * and /; body unchanged */
```

`tiny_interp.c` — inside `eval`'s operator switch (needs `<limits.h>`, already included):

```c
        case '%': if (r == 0) { fprintf(stderr, "line %d: division by zero\n", x->line); exit(1); }
                  if (l == LONG_MIN && r == -1) return 0;            // LONG_MIN % -1 is 0 (would trap)
                  return l % r;                                      // C99: sign follows the dividend
```

`tinyc.c` — inside `gen_expr`'s operator switch; `idivq` leaves the remainder in `%rdx`, and it traps for `LONG_MIN % -1` exactly like division, so special-case `-1`:

```c
        case '%': {                          // remainder: idiv leaves it in rdx
            int l = label_id; label_id += 2;
            printf("    testq %%rcx, %%rcx\n    jz .Ldivzero\n");
            printf("    cmpq $-1, %%rcx\n    jne .L%d\n    xorl %%eax, %%eax\n    jmp .L%d\n", l, l + 1);
            printf(".L%d:\n    cqto\n    idivq %%rcx\n    movq %%rdx, %%rax\n.L%d:\n", l, l + 1);
            return;
        }
```

Test program and results (both implementations print the same):

```text
let a = -7;
print a % 3;            → -1
print 7 % -3;           →  1
let m = 9223372036854775807;
let n = -m - 1;
print n % -1;           →  0   (no SIGFPE)
print 10 % 4 * 3;       →  6   (left-associative: (10 % 4) * 3)
```

Finally extend `difftest.py`'s operator list with `'%'` (using the same odd-divisor trick as `/`) and rerun it.

</details>

### B-5 · Why reproducible seeds? (lesson 05)
Your differential tester found a mismatch overnight but printed only "mismatch found". What three things must it record so you can debug it in the morning?

<details><summary>Solution</summary>

(1) The random seed (or the generated program itself); (2) the exact tool versions and flags for every configuration compared; (3) the full observable behaviour of each configuration (stdout, stderr, exit status). With those, the failure is reproducible and can be reduced (04-pro/04) and bisected (03-advanced/06).

</details>

## Level 2 — Intermediate

### I-1 · FIRST and FOLLOW (lesson 01)
Grammar: `S → A B`, `A → a A | ε`, `B → b`. Compute FIRST and FOLLOW for S, A, B; is the grammar LL(1)?

<details><summary>Solution</summary>

FIRST(A) = {a, ε}; FIRST(B) = {b}; FIRST(S) = FIRST(A) − {ε} ∪ FIRST(B) = {a, b}. FOLLOW(S) = {$}; FOLLOW(A) = FIRST(B) = {b}; FOLLOW(B) = FOLLOW(S) = {$}. LL(1): for `A → a A | ε`, FIRST(aA) = {a} and FOLLOW(A) = {b} are disjoint → yes. Verify by replacing `GRAMMAR` in `ll1.py` with `{"S": [["A","B"]], "A": [["a","A"], [EPS]], "B": [["b"]]}` and `START = "S"`, then parsing `["a","a","b"]`.

</details>

### I-2 · Dominators and frontiers of a diamond (lesson 04)
CFG: entry E → A, E → B, A → J, B → J, J → X. Give idom of every node and DF of every node. Where do φs for a variable assigned in A and B go?

<details><summary>Solution</summary>

idom(A) = idom(B) = idom(J) = E; idom(X) = J. DF(A) = {J}, DF(B) = {J}, DF(E) = DF(J) = DF(X) = ∅. A variable assigned in A and B gets one φ at J (the iterated DF of {A, B} is {J}; J's own DF is empty). Check with `ssa.py` by replacing `CFG` with the five blocks.

</details>

### I-3 · Liveness by hand (lesson 06)
Block: `a = b + c; d = a * 2; b = d - 1; return b + a`. Nothing is live after the return. Compute live-in at each instruction (bottom-up).

<details><summary>Solution</summary>

Going up: before `return b + a`: {a, b}. Before `b = d - 1`: remove b, add d → {a, d}. Before `d = a * 2`: remove d, add a → {a}. Before `a = b + c`: remove a, add b, c → {b, c}. So the block's live-in is {b, c}. At most two values are live at once → two registers suffice.

</details>

### I-4 · max(a, b) in LLVM IR, two ways (lesson 05)
Write `max` once with `select` and once with branches and a `phi`; run both with `lli`; then see what `simplifycfg,instcombine` make of the branchy one.

<details><summary>Solution</summary>

```llvm
; file: max.ll
; Exercise I-4 solution: signed max(a, b) two ways, plus a main that checks both.
define i32 @max_select(i32 %a, i32 %b) {
  %c = icmp sgt i32 %a, %b
  %r = select i1 %c, i32 %a, i32 %b        ; branch-free
  ret i32 %r
}
define i32 @max_branch(i32 %a, i32 %b) {
entry:
  %c = icmp sgt i32 %a, %b
  br i1 %c, label %take_a, label %join
take_a:
  br label %join
join:
  %r = phi i32 [ %a, %take_a ], [ %b, %entry ]   ; which predecessor did we come from?
  ret i32 %r
}
define i32 @main() {
  %x = call i32 @max_select(i32 -3, i32 7)
  %y = call i32 @max_branch(i32 -3, i32 7)
  %s = add i32 %x, %y                            ; 7 + 7 = 14 becomes the exit status
  ret i32 %s
}
```

`lli max.ll; echo $?` → `14`. `opt -S -passes=simplifycfg,instcombine max.ll` turns `@max_branch` into a single `call i32 @llvm.smax.i32(i32 %a, i32 %b)`: the branch became a select, then instcombine recognised the select pattern as the `smax` intrinsic.

</details>

### I-5 · Which pass did that? (lesson 07)
In `opts.c` from lesson 07, which LLVM pass removes `int unused = a * 100;`, and which turns `x / 4` (unsigned) into a shift? Why would the shift be wrong for `int x`?

<details><summary>Solution</summary>

`dce` (or `instcombine`/`adce` in the full pipeline) removes the dead multiply; `instcombine` rewrites `udiv x, 4` to `lshr x, 2`. For signed `int`, C division truncates toward zero but an arithmetic shift rounds toward −∞ (`-7 / 4 == -1`, `-7 >> 2 == -2`), so the compiler must add a correction (it adds 3 to negative values before shifting). Verify: `clang -O2 -S -o - ` on `int f(int x){return x/4;}` shows the correction sequence.

</details>

## Level 3 — Advanced

### A-1 · Register colouring by hand (lesson 02)
Values with intervals a[0,3), b[1,4), c[2,6), d[3,5), e[4,5). With k = 2 registers, which value would Chaitin–Briggs spill, and why?

<details><summary>Solution</summary>

Interference: a–b, a–c, b–c, b–d, c–d, c–e, d–e. Degrees: a 2, b 3, c 4, d 3, e 2. With k = 2, no node has degree < 2 initially except none (a and e have degree 2, not < 2), so a spill candidate is chosen — the highest degree (cost/degree heuristic with equal costs) is **c** (degree 4, longest interval). After removing c, degrees drop (a 1, b 2, d 2, e 1) and the rest simplify and colour with 2 registers. This matches `regalloc.py`'s k = 2 result for the lesson's example, where `c` was also spilled.

</details>

### A-2 · Fix the link (lesson 03)
`gcc -lm prog.o -o prog` fails with "undefined reference to `sqrt`" on some systems, while `gcc prog.o -lm -o prog` works. Explain using the GNU ld rule.

<details><summary>Solution</summary>

GNU ld "will search an archive only once, at the location where it is specified on the command line", extracting only members that define *currently undefined* symbols. Before `prog.o` is read, `sqrt` isn't undefined yet, so nothing is taken from `libm`. Put libraries after the objects that use them. (With a *shared* `libm.so` and `--as-needed`, a library listed too early is likewise dropped as unneeded.)

</details>

### A-3 · Spot the UB (lesson 04)
Which lines have undefined, unspecified or implementation-defined behaviour? (`int` is 32-bit.)

```text
1: int a = INT_MAX; a++;
2: unsigned u = UINT_MAX; u++;
3: int s = -8 >> 1;
4: int b = 1 << 31;
5: f(g(), h());          // g and h print something
6: int *p = NULL; if (x) p = &y; *p = 3;
7: char c = 200;
```

<details><summary>Solution</summary>

1: UB (signed overflow). 2: defined (unsigned wraps to 0). 3: implementation-defined (right shift of a negative value; GCC/Clang do arithmetic shift → −4). 4: UB in C11 (1 shifted into the sign bit of a signed int; C23 changed integer representation rules but check your standard mode). 5: unspecified order of `g`/`h` (03-advanced/04 measured GCC right-to-left, Clang left-to-right). 6: UB when `x` is false (null dereference) — and the compiler may assume `x` is true. 7: implementation-defined (out-of-range conversion to `char` if `char` is signed).

</details>

### A-4 · Bisect a pass (lesson 06)
Using `bisect.py`, find which pass changes the output of `ub_overflow.c` (lesson 04) under Clang. What do you expect, and what does it tell you?

<details><summary>Solution</summary>

Clang `-O0` prints 0 and `-O2` prints 1, so `bisect.py ub_overflow.c` will name the first pass that folds `x + 1 > x` (an InstCombine/InstSimplify-style peephole on `add nsw` + `icmp`) applied to `bigger`. It tells you *where* the UB is exploited — not that the pass is buggy: the program has signed overflow, and the `nsw` flag makes the fold legal (03-advanced/04 §4.1, 04-pro/06 §5).

</details>

### A-5 · `scf.if` in MLIR (lesson 05)
Write `|a − b|` with `scf.if` returning a value, lower it to LLVM IR, run it with `lli` (exit status = result), and inspect the `cf` form.

<details><summary>Solution</summary>

```text
// file: absdiff.mlir
// Exercise A-5 solution: |a - b| with scf.if, lowered to LLVM and run (exit status = result).
func.func @absdiff(%a: i64, %b: i64) -> i64 {
  %lt = arith.cmpi slt, %a, %b : i64
  %r = scf.if %lt -> (i64) {          // scf.if is an operation with two regions that yield a value
    %d = arith.subi %b, %a : i64
    scf.yield %d : i64
  } else {
    %d = arith.subi %a, %b : i64
    scf.yield %d : i64
  }
  return %r : i64
}
func.func @main() -> i64 {
  %x = arith.constant 3 : i64
  %y = arith.constant 45 : i64
  %r = func.call @absdiff(%x, %y) : (i64, i64) -> i64
  return %r : i64
}
```

```text
$ mlir-opt absdiff.mlir --convert-scf-to-cf --convert-arith-to-llvm --convert-func-to-llvm \
    --convert-cf-to-llvm --reconcile-unrealized-casts | mlir-translate --mlir-to-llvmir > absdiff.ll
$ lli absdiff.ll; echo "exit=$?"
exit=42
$ mlir-opt absdiff.mlir --convert-scf-to-cf | grep -E "cf\.|\^bb"
    cf.cond_br %0, ^bb1, ^bb2
  ^bb1:  // pred: ^bb0
    cf.br ^bb3(%1 : i64)
  ^bb2:  // pred: ^bb0
    cf.br ^bb3(%2 : i64)
  ^bb3(%3: i64):  // 2 preds: ^bb1, ^bb2
```

The `scf.yield` values became the block argument `%3` of `^bb3` — the MLIR form of a φ.

</details>

## Level 4 — Pro

### P-1 · Read a results table (lesson 01, 03)
EMI's paper reports, for GCC/LLVM: reported 111/84, duplicates 28/7, confirmed 79/68, fixed 56/54. What fraction of all reports were confirmed? Why do papers report all four numbers?

<details><summary>Solution</summary>

Confirmed / reported = 147 / 195 ≈ 75%. Reporting all four separates *effort* (reports), *noise* (duplicates, invalid), *validity* (confirmed by developers) and *impact* (fixed). A tool that files many reports but few get confirmed is producing false alarms (often UB) or duplicates.

</details>

### P-2 · Unsigned arithmetic in the generator (lesson 02)
You want `minismith.py` to also generate `uint32_t` expressions. Which wrappers are still needed, and which operations need none?

<details><summary>Solution</summary>

Unsigned `+ - *` wrap by definition — no wrapper. Still needed: division and remainder by zero; shifts by an amount ≥ 32 (UB for unsigned too); and **integer promotion**: `uint8_t`/`uint16_t` operands are promoted to *signed* `int`, so `(uint16_t)a * (uint16_t)b` can overflow signed `int` (UB) — cast to `uint32_t` first. Mixing signed and unsigned operands triggers the usual arithmetic conversions (02-intermediate/02), which is defined but easy to mis-model in the generator's expected types.

</details>

### P-3 · ddmin by hand (lesson 04)
Input `[1..8]`; the failure occurs iff the input contains both 2 and 7. Trace ddmin's first steps.

<details><summary>Solution</summary>

n = 2: chunks [1–4], [5–8]; neither alone fails; complements are the other halves (same sets) → increase to n = 4: chunks [1,2], [3,4], [5,6], [7,8]. No single chunk fails. Complements: removing [1,2] loses 2 → passes; removing [3,4] → [1,2,5,6,7,8] fails → recurse with n = max(4−1, 2) = 3 on 6 elements: chunks [1,2], [5,6], [7,8]; complement without [5,6] → [1,2,7,8] fails → n = 2: chunks [1,2], [7,8]; neither alone fails; complements equal the chunks → n = 4: [1],[2],[7],[8]; complement without [1] → [2,7,8] fails → n = 3: [2],[7],[8]; complement without [8] → [2,7] fails → n = 2: [2],[7]; neither alone, complements are singletons → n = min(4, 2) = 2 = |c| → done: **[2, 7]**. Run `ddmin.py` with `lambda xs: 2 in xs and 7 in xs` to compare test counts.

</details>

### P-4 · An interestingness test for a crash (lesson 04)
Write the core lines of an interestingness test that keeps a C file only if `clang -O2` still crashes with the *same* assertion (`Assertion `isa<X>(Val)' failed`).

<details><summary>Solution</summary>

```bash
#!/bin/bash
# file: crash-interesting.sh
# Keep the candidate only if clang -O2 crashes with THE SAME assertion message.
timeout 30 clang -O2 -c -w t.c -o /dev/null > log.txt 2>&1
grep -q "Assertion \`isa<X>(Val)' failed" log.txt
```

Matching the specific message (not just "any non-zero exit") prevents the reducer from drifting to a different crash or to a plain syntax error.

</details>

### P-5 · Verify three rewrites with Z3 (lesson 06)
Check (a) `(x ^ -1) + 1 ⇒ -x`, (b) `x * 2 / 2 ⇒ x` with wrapping multiply, (c) the same with `mul nsw`.

<details><summary>Solution</summary>

```python
# file: ex_alive.py
# Exercise P-5 solution: check three rewrites with Z3 (8-bit), in the style of mini_alive.py.
from z3 import BitVec, BitVecVal, Solver, And, Not, sat, BVMulNoOverflow, BVMulNoUnderflow

def check(name, src, tgt, x, src_ub=None):
    s = Solver()
    bad = src != tgt
    if src_ub is not None:
        bad = And(Not(src_ub), bad)          # UB/poison in the source: anything is allowed
    s.add(bad)
    if s.check() == sat:
        print(f"WRONG    {name:28} x={s.model().eval(x, model_completion=True).as_signed_long()}")
    else:
        print(f"CORRECT  {name}")

x = BitVec("x", 8)
check("(x ^ -1) + 1  =>  -x", (x ^ -1) + 1, -x, x)                     # two's complement negation
check("x * 2 / 2  =>  x   (wrapping)", (x * 2) / 2, x, x)               # wrong: x*2 may wrap
check("x *nsw 2 / 2  =>  x", (x * 2) / 2, x, x,
      src_ub=Not(And(BVMulNoOverflow(x, BitVecVal(2, 8), True),        # z3: signed overflow above MAX
                     BVMulNoUnderflow(x, BitVecVal(2, 8)))))           # ... and below MIN are separate
```

```text
CORRECT  (x ^ -1) + 1  =>  -x
WRONG    x * 2 / 2  =>  x   (wrapping) x=-96
CORRECT  x *nsw 2 / 2  =>  x
```

Pitfall met while writing this solution: modelling `nsw` with `BVMulNoOverflow(..., True)` alone gave a bogus counterexample (x = −127), because Z3 checks signed overflow *above* the maximum and *below* the minimum with two different functions (`BVMulNoOverflow`, `BVMulNoUnderflow`). Encoding the semantics wrongly is the most common way verification tools give wrong answers.

</details>

## Mini projects

### Project 1 · A Tiny → LLVM IR back end, differentially tested (lessons 01-beginner/04–05, 02-intermediate/04–05)

**Goal.** Replace the hand-written x86 code generator with an LLVM IR emitter, run the output with `lli` and with `clang -O2`, and differential-test both against the interpreter.

**Acceptance criteria.** `fact.tiny` prints `3628800 0 1 1 2 3 5 8 13 21 34 55 89 1`; wrap-around and `LONG_MIN / -1` match the interpreter; `x / 0` exits with status 1; 100 random programs from `difftest.py`'s generator show 0 mismatches for both `lli` and `clang -O2`; after `opt -O2` no `alloca` remains.

<details><summary>Reference solution</summary>

```c
// file: tiny2ll.c
// Mini project 1 reference solution: Tiny -> LLVM IR (text). Uses tiny_front.h (lesson 01-beginner/04).
// Strategy from the Kaleidoscope tutorial: every variable gets an alloca in the entry block and
// is accessed with load/store; mem2reg/-O2 turns that into SSA later.
// Usage: ./tiny2ll prog.tiny > prog.ll && lli prog.ll
#include "tiny_front.h"

static char names[256][32];
static int nnames, tmp, lbl;                 // variable table, SSA temp counter, label counter

static int slot(const char *n, int line) {   // compile-time name lookup (as in tinyc.c)
    for (int i = 0; i < nnames; i++) if (!strcmp(names[i], n)) return i;
    fprintf(stderr, "line %d: undefined variable '%s'\n", line, n); exit(1);
}

// emit code for an expression; returns the number N of the SSA value %tN holding it (i64)
static int emit_expr(Node *x) {
    int a, b, t;
    switch (x->kind) {
    case N_NUM: t = ++tmp; printf("  %%t%d = add i64 0, %ld\n", t, x->val); return t;
    case N_VAR: t = ++tmp; printf("  %%t%d = load i64, ptr %%v%d\n", t, slot(x->name, x->line)); return t;
    case N_NEG: a = emit_expr(x->a); t = ++tmp; printf("  %%t%d = sub i64 0, %%t%d\n", t, a); return t;  // wraps: no nsw
    case N_BIN:
        a = emit_expr(x->a); b = emit_expr(x->b);
        if (strchr("+-*", x->op[0]) && !x->op[1]) {          // wrapping arithmetic: no nsw flags
            const char *ins = x->op[0] == '+' ? "add" : x->op[0] == '-' ? "sub" : "mul";
            t = ++tmp; printf("  %%t%d = %s i64 %%t%d, %%t%d\n", t, ins, a, b); return t;
        }
        if (x->op[0] == '/') {                                // x/0 -> runtime error; MIN/-1 -> MIN
            int z = ++tmp, ok = lbl++, m1 = ++tmp, q = ++tmp, neg = ++tmp;
            printf("  %%t%d = icmp eq i64 %%t%d, 0\n", z, b);
            printf("  br i1 %%t%d, label %%divzero, label %%L%d\n", z, ok);
            printf("L%d:\n", ok);
            // sdiv MIN, -1 is immediate UB in LLVM IR even if the result is later discarded,
            // so divide by a SAFE divisor (1 when b == -1) and select the negation instead.
            int d = ++tmp;
            printf("  %%t%d = icmp eq i64 %%t%d, -1\n", m1, b);
            printf("  %%t%d = select i1 %%t%d, i64 1, i64 %%t%d\n", d, m1, b);
            printf("  %%t%d = sdiv i64 %%t%d, %%t%d\n", q, a, d);
            printf("  %%t%d = sub i64 0, %%t%d\n", neg, a);
            t = ++tmp; printf("  %%t%d = select i1 %%t%d, i64 %%t%d, i64 %%t%d\n", t, m1, neg, q);
            return t;
        }
        {                                                     // comparisons -> i1 -> zext to i64
            const char *p = !strcmp(x->op, "<") ? "slt" : !strcmp(x->op, "<=") ? "sle" :
                            !strcmp(x->op, ">") ? "sgt" : !strcmp(x->op, ">=") ? "sge" :
                            !strcmp(x->op, "==") ? "eq" : "ne";
            int c = ++tmp; printf("  %%t%d = icmp %s i64 %%t%d, %%t%d\n", c, p, a, b);
            t = ++tmp; printf("  %%t%d = zext i1 %%t%d to i64\n", t, c); return t;
        }
    default: fprintf(stderr, "not an expression\n"); exit(1);
    }
}

static void emit_stmt(Node *s) {
    int c, l1, l2, l3, v;
    switch (s->kind) {
    case N_LET:
        for (int i = 0; i < nnames; i++)
            if (!strcmp(names[i], s->name)) { fprintf(stderr, "line %d: '%s' already declared\n", s->line, s->name); exit(1); }
        v = emit_expr(s->a);
        strcpy(names[nnames], s->name);
        printf("  store i64 %%t%d, ptr %%v%d\n", v, nnames++);
        return;
    case N_ASSIGN: v = emit_expr(s->a); printf("  store i64 %%t%d, ptr %%v%d\n", v, slot(s->name, s->line)); return;
    case N_PRINT:
        v = emit_expr(s->a);
        printf("  call i32 (ptr, ...) @printf(ptr @fmt, i64 %%t%d)\n", v);
        return;
    case N_IF:
        c = emit_expr(s->a); l1 = lbl++; l2 = lbl++; l3 = lbl++;
        printf("  %%t%d = icmp ne i64 %%t%d, 0\n", ++tmp, c);
        printf("  br i1 %%t%d, label %%L%d, label %%L%d\nL%d:\n", tmp, l1, l2, l1);
        emit_stmt(s->b); printf("  br label %%L%d\nL%d:\n", l3, l2);
        if (s->c) emit_stmt(s->c);
        printf("  br label %%L%d\nL%d:\n", l3, l3);
        return;
    case N_WHILE:
        l1 = lbl++; l2 = lbl++; l3 = lbl++;
        printf("  br label %%L%d\nL%d:\n", l1, l1);              // header
        c = emit_expr(s->a);
        printf("  %%t%d = icmp ne i64 %%t%d, 0\n", ++tmp, c);
        printf("  br i1 %%t%d, label %%L%d, label %%L%d\nL%d:\n", tmp, l2, l3, l2);
        emit_stmt(s->b);
        printf("  br label %%L%d\nL%d:\n", l1, l3);              // back edge, then exit block
        return;
    case N_BLOCK: for (int i = 0; i < s->n; i++) emit_stmt(s->list[i]); return;
    default: emit_expr(s); return;
    }
}

int main(int argc, char **argv) {
    if (argc != 2) { fprintf(stderr, "usage: %s file.tiny\n", argv[0]); return 1; }
    Node *prog = parse_program(read_file(argv[1]));
    printf("@fmt = private constant [5 x i8] c\"%%ld\\0A\\00\"\n");
    printf("@dz = private constant [18 x i8] c\"division by zero\\0A\\00\"\n");
    printf("declare i32 @printf(ptr, ...)\ndeclare void @exit(i32)\n");
    printf("define i32 @main() {\nentry:\n");
    for (int i = 0; i < 256; i++) printf("  %%v%d = alloca i64\n", i);   // one slot per possible variable
    emit_stmt(prog);
    printf("  ret i32 0\ndivzero:\n  call i32 (ptr, ...) @printf(ptr @dz)\n  call void @exit(i32 1)\n  unreachable\n}\n");
    return 0;
}
```

Design notes: variables are `alloca`s + `load`/`store` (the Kaleidoscope approach), so no φs are needed in the front end — `opt -O2` promotes all 256 slots (`grep -c alloca` on the optimized output gives 0). **No `nsw`** on arithmetic, because Tiny defines overflow as wrapping. Division needs care: `sdiv i64 MIN, -1` is undefined behaviour in LLVM IR *even if the result is later discarded by a `select`*, so the emitter divides by a **safe divisor** (`select` 1 when the divisor is −1) — an early draft of this solution did `sdiv` first and `select` afterwards, which is exactly the kind of UB-introducing codegen bug Alive2 catches.

```python
# file: difftest_ll.py
# Mini project 1: differential test of tiny2ll (via lli, and via clang -O2) against tiny_interp,
# reusing the random program generator from lesson 01-beginner/05 (difftest.py).
import random, subprocess, os, tempfile, sys
from difftest import gen_program, run           # generator + runner from lesson 05

def main():
    n = int(sys.argv[1]) if len(sys.argv) > 1 else 100
    rng = random.Random(7)
    tmp = tempfile.mkdtemp()
    bad = 0
    for i in range(n):
        path = os.path.join(tmp, f"t{i}.tiny")
        with open(path, "w") as f:
            f.write(gen_program(rng))
        expected = run(["./tiny_interp", path])
        ll = subprocess.run(["./tiny2ll", path], capture_output=True, text=True).stdout
        with open(path + ".ll", "w") as f:
            f.write(ll)
        got_lli = run(["lli", path + ".ll"])                       # LLVM's JIT, unoptimized IR
        subprocess.run(["clang", "-O2", "-w", path + ".ll", "-o", path + ".bin"], check=True)
        got_o2 = run([path + ".bin"])                              # full LLVM -O2 pipeline + codegen
        if not (expected == got_lli == got_o2):
            bad += 1
            print(f"MISMATCH {path}: interp={expected} lli={got_lli} clang-O2={got_o2}")
    print(f"{n} programs, {bad} mismatches")

if __name__ == "__main__":
    main()
```

```text
$ ./tiny2ll fact.tiny > fact.ll && lli fact.ll | tr '\n' ' '
3628800 0 1 1 2 3 5 8 13 21 34 55 89 1
$ ./tiny2ll wrap.tiny > w.ll && lli w.ll        # three times -9223372036854775808
$ printf 'print 7 / 0;\n' > dz.tiny && ./tiny2ll dz.tiny > dz.ll && lli dz.ll; echo $?
division by zero
1
$ python3 difftest_ll.py 100
100 programs, 0 mismatches
```

Extensions: emit `nsw` for a "fast mode" and show (with the differential tester) how programs that overflow now diverge between `lli` and `-O2`; plant a bug in the emitter and reduce the failing program with `reduce_tiny.py`.

</details>

### Project 2 · An EMI + reduction pipeline (lessons 04-pro/02–04)

**Goal.** Build an end-to-end campaign: generate UB-free seeds with `minismith.py`, create EMI variants with `emi.py`, compile with GCC/Clang × `-O0…-O3`, and automatically reduce any violation with C-Reduce using a UB-checking interestingness test.

**Acceptance criteria.** (1) A driver loops over seeds, logs per-seed results as JSON lines (seed, variant, configuration, outputs); (2) the control mode (`--delete-executed`) produces violations and the pipeline reduces at least one of them to a few lines — proving the reducer path works end-to-end; (3) every reduced program passes UBSan (Clang, `-fno-sanitize-recover=all`) and a second compiler's `-O0` build; (4) a summary reports seeds, variants, compilations, violations, and reduction sizes.

<details><summary>Reference design</summary>

* **Driver**: for seed *s*: `python3 minismith.py s > seed.c`; run `emi.py seed.c N`; parse its `<-- BUG?` lines; for each, write the variant to `case_<s>_<v>.c`.
* **Interestingness test** for a variant `t.c` that showed configuration *X* disagreeing with the seed's expected output *E*: compile with UBSan and run (must be clean, must terminate within 5 s); compile with *X*, run with timeout, require output ≠ *E*; compile with GCC `-O0`, require output = the *EMI-expected* output computed from the **seed program** (store *E* in a file next to the test); otherwise exit 1. (Template: appendix §E.)
* **Reduction**: `creduce --n 2 ./interesting.sh t.c` (C mode — its Clang-based passes are much stronger than `--not-c`).
* **Control experiment**: run the pipeline once with `--delete-executed` so that violations exist; check that reductions finish and stay UB-free. The result will be programs where an *executed* assignment was removed — expected, but it proves the plumbing.
* **Dedup**: group reduced cases by (configuration, normalised reduced text) before reporting.
* **Report**: totals plus, for each real violation, the reduced program, the configuration, both outputs, compiler versions.

</details>

### Project 3 · A race-free CUDA kernel generator with a CPU oracle (lesson 04-pro/05)

**Goal.** Generate random CUDA kernels that are race-free and barrier-safe by construction, compile them to PTX with Clang at several optimization levels, and emit a CPU reference program that computes the expected output — so that, on a machine with an NVIDIA GPU, any GPU result that differs from the CPU reference indicates a compiler bug.

**Acceptance criteria.** (1) Kernels compile to PTX with `clang++ -x cuda --cuda-device-only -nocudainc -nocudalib`; (2) the CPU reference gives the same checksum with GCC `-O0`, GCC `-O2` and Clang `-O2`; (3) barriers appear only at the top level of the kernel (check the IR: the block containing `llvm.nvvm.barrier0` must post-dominate the entry); (4) on a GPU machine: a launcher with 64 threads compares `out[]` to the CPU checksum.

<details><summary>Reference starter (generator + CPU oracle)</summary>

```python
# file: kgen.py
# Mini project 3 starter: generate random CUDA kernels that are race-free and barrier-safe
# BY CONSTRUCTION, then compile them to PTX with Clang (no GPU needed).
# Rules: (1) thread t writes only out[t] and sh[t]; (2) reads of sh[] by other threads happen
# only after a barrier that is at the top level of the kernel (never under a branch);
# (3) integer ops use wrapping unsigned arithmetic (no UB); (4) loops have constant bounds.
# With --cpu it emits a C program that EMULATES the kernel for one 64-thread block (phase 1 for
# every thread, then the barrier, then phase 2) -- a reference oracle for later GPU runs.
import random, sys

HDR = """#define __global__ __attribute__((global))
#define __shared__ __attribute__((shared))
#define TID __nvvm_read_ptx_sreg_tid_x()
"""

def expr(rng, depth, names):
    if depth == 0 or rng.random() < 0.3:
        return rng.choice(names + [f"{rng.randint(0, 999)}u"])
    a, b = expr(rng, depth - 1, names), expr(rng, depth - 1, names)
    op = rng.choice(["+", "-", "*", "^", "&", "|"])        # unsigned: all well-defined
    return f"({a} {op} {b})"

def parts(seed):
    """Return (phase1 lines, phase2 line): the code before and after the barrier."""
    rng = random.Random(seed)
    names = ["t", "in[t]", "v"]
    body = [f"    unsigned t = TID;", f"    unsigned v = in[t] ^ {rng.randint(0, 999)}u;"]
    for _ in range(rng.randint(2, 5)):
        if rng.random() < 0.5:                             # divergent branch: fine WITHOUT barriers
            body.append(f"    if ({expr(rng, 2, names)} & 1u) v = {expr(rng, 3, names)}; else v = {expr(rng, 3, names)};")
        else:
            k = rng.randint(1, 8)
            body.append(f"    for (unsigned i = 0; i < {k}u; i++) v = {expr(rng, 3, names + ['i'])};")
    body.append("    sh[t] = v;")
    after = f"    out[t] = v + sh[(t + {rng.randint(1, 63)}u) % 64u];   // read another thread's value AFTER the barrier"
    return body, after

def kernel(seed):
    body, after = parts(seed)
    body = body + ["    __syncthreads();                       // top level: every thread reaches it", after]
    return (HDR + "__global__ void k(const unsigned *in, unsigned *out) {\n"
            "    __shared__ unsigned sh[64];             // launch with exactly 64 threads per block\n"
            + "\n".join(body) + "\n}\n")

def cpu_reference(seed):
    body, after = parts(seed)
    p1 = "\n".join("        " + l.strip() for l in body if not l.strip().startswith("unsigned t"))
    return ("#include <stdio.h>\nint main(void) {\n"
            "    unsigned in[64], out[64], sh[64], vs[64], crc = 0;\n"
            "    for (unsigned i = 0; i < 64; i++) in[i] = i * 2654435761u;   /* fixed inputs */\n"
            "    for (unsigned t = 0; t < 64; t++) {        /* phase 1: every thread up to the barrier */\n"
            + p1 + "\n        vs[t] = v;\n    }\n"
            "    for (unsigned t = 0; t < 64; t++) {        /* phase 2: after the barrier */\n"
            "        unsigned v = vs[t];\n        " + after.strip() + "\n    }\n"
            "    for (unsigned t = 0; t < 64; t++) crc = crc * 31u + out[t];\n"
            '    printf("checksum = %08X\\n", crc);\n    return 0;\n}\n')

if __name__ == "__main__":
    seed = int(sys.argv[1]) if len(sys.argv) > 1 else 1
    sys.stdout.write(cpu_reference(seed) if "--cpu" in sys.argv else kernel(seed))
```

```text
$ for s in 1 2 3 4 5; do python3 kgen.py $s > k$s.cu && clang++ -x cuda --cuda-gpu-arch=sm_70 \
    --cuda-device-only -nocudainc -nocudalib -O2 -S k$s.cu -o k$s.ptx; done; ls k*.ptx | wc -l
5
$ python3 kgen.py 1 --cpu > ref1.c && gcc -O0 ref1.c -o r0 && gcc -O2 ref1.c -o r2 && clang -O2 ref1.c -o r3
$ ./r0; ./r2; ./r3
checksum = E8160900
checksum = E8160900
checksum = E8160900
```

Why the CPU emulation is valid: each thread writes only `sh[t]` and `out[t]`, other threads' `sh` values are read only after a top-level barrier, and arithmetic is unsigned — so running phase 1 for all threads, then phase 2 for all threads, gives exactly the result any correct GPU execution must produce. Next steps: add a host launcher (needs the CUDA toolkit and a GPU); compare `clang -O0/-O2` and `nvcc` builds; add warp-level operations only under uniform control flow; reduce failures with C-Reduce on the `.cu` file using the CPU checksum as the oracle.

</details>

## Sources

* Lesson sources apply to each exercise; key ones: N1570 (C11 behaviours), LLVM LangRef (`sdiv` UB, `select`, `smax`), GNU ld manual (`-l` order), Zeller & Hildebrandt 2002 (ddmin), Le et al. 2014 (EMI table), Z3 API (`BVMulNoOverflow`, `BVMulNoUnderflow`): https://z3prover.github.io/api/html/namespacez3py.html
* LLVM Kaleidoscope ch. 7 (allocas + mem2reg strategy used in Project 1): https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl07.html
* Clang "Compiling CUDA with clang" (Project 3): https://llvm.org/docs/CompileCudaWithLLVM.html

## Verify yourself

```bash
# file: verify-exercises.sh
# Run from a directory containing the lesson files mentioned (tiny_front.h, tiny_interp.c, difftest.py, ...).
gcc -Wall -Wextra -o list list.c && ./list "x, x, x" && ./list "x,,x"
lli max.ll; echo "exit=$?"                                         # 14
opt -S -passes=simplifycfg,instcombine max.ll | grep smax
python3 ex_alive.py
export PATH=/usr/lib/llvm-18/bin:$PATH
mlir-opt absdiff.mlir --convert-scf-to-cf --convert-arith-to-llvm --convert-func-to-llvm \
  --convert-cf-to-llvm --reconcile-unrealized-casts | mlir-translate --mlir-to-llvmir > absdiff.ll && lli absdiff.ll; echo "exit=$?"   # 42
gcc -Wall -Wextra -std=c11 -o tiny2ll tiny2ll.c && gcc -O2 -w -o tiny_interp tiny_interp.c
python3 difftest_ll.py 100                                          # 0 mismatches
python3 kgen.py 1 --cpu > ref1.c && gcc -O2 ref1.c -o r && ./r      # checksum = E8160900
```
