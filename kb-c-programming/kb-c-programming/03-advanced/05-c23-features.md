# Lesson A5 — C23: what changed, why, and how to use it today

> Level: advanced. Prerequisites: all beginner and intermediate lessons; A1, A4. Time: about 2.5 hours.
> C23 = ISO/IEC 9899:2024 (published 31 October 2024). Examples compiled with **GCC 14.2** (`gcc-14 -std=c23`) and glibc 2.39 on x86-64 Linux, November 2026. Features needing **GCC 15 / Clang 19** (`#embed`, and in Clang `constexpr`) are shown but marked *not compiled here*.

---

## 1. Why this lesson exists

C changes slowly — C17 was a bug-fix release, so C23 is the first revision with significant new features since C11. It matters to you now because **GCC 15 compiles C as `gnu23` by default** (GCC *Porting to GCC 15*): code that names a variable `bool`, declares `int f();` meaning "any arguments", or relies on K&R-style definitions may stop compiling as distributions move to GCC 15.

C23's theme is "remove old footguns, adopt proven practice": many features were long-standing GCC/Clang extensions (`typeof`, binary literals, `__has_include`) or borrowed from C++ (`nullptr`, `constexpr`, `auto`, attributes, digit separators), and several directly target the UB and overflow problems of lessons I3 and A4 (`ckd_add`, `unreachable`, `memset_explicit`).

---

## 2. Intuition: a renovation, not a rebuild

Think of C23 as renovating an old house you still live in:

- **Removing hazards:** K&R function definitions, non-two's-complement integers and trigraphs are gone; `()` now means `(void)`.
- **Labelling the switches properly:** `bool`, `true`, `false`, `nullptr`, `static_assert`, `alignas`, `thread_local` become real keywords instead of macros from headers.
- **New tools in the shed:** checked arithmetic, bit utilities, `#embed`, `typeof`, `constexpr`, `auto`, fixed-underlying-type enums, `_BitInt(N)`.

Everything old that was valid and not hazardous still works.

---

## 3. Precise definitions: the feature map

(Feature names and proposal numbers from cppreference's *C23* page; compiler versions from the GCC *C Status* and Clang *C Support* pages.)

| Area | Feature | Proposal | GCC | Clang |
|---|---|---|---|---|
| Keywords | `bool`, `true`, `false`, `alignas`, `alignof`, `static_assert`, `thread_local` | N2934, N2935 | 13 | 15 for `true`/`false` (N2935); others [unverified] |
| Null | `nullptr` and `nullptr_t` | N3042 | 13 | 17 |
| Constants | `constexpr` objects | N3018 | 13 | 19 |
| Inference | `auto` for object definitions | N3007 | 13 | 18 |
| Types | `typeof`, `typeof_unqual` | N2927, N2930 | 13 | 16 |
| Types | `_BitInt(N)` bit-precise integers | N2763 | 14 (IA-32, x86-64, AArch64 LE) | 15 |
| Enums | fixed underlying type `enum e : unsigned char` | N3030 | 13 [unverified] | 20 |
| Literals | binary `0b1010`, digit separator `1'000'000` | N2549, N2626 | 4.3 / 12 | — |
| Init | empty initialiser `= {}` | N2900 | — | — |
| Attributes | `[[nodiscard]]`, `[[maybe_unused]]`, `[[deprecated]]`, `[[fallthrough]]`, `[[noreturn]]`, `[[unsequenced]]`, `[[reproducible]]` | N2267 etc. | 10 (syntax) | — |
| Functions | unnamed parameters in definitions; `()` = `(void)`; K&R removed | N2480, N2841, N2432 | — | — |
| Preprocessor | `#embed`, `__has_include`, `#elifdef`, `#warning`, `__VA_OPT__` | N3017, N2799, N2645, N2686, N3033 | 15 (`#embed`) | 19 (`#embed`) |
| Library | `<stdckdint.h>` (`ckd_add/sub/mul`), `<stdbit.h>`, `memset_explicit`, `strdup`/`strndup`, `memccpy`, `%b` printf, `unreachable()`, `free_sized` | — | GCC 14 provides `<stdckdint.h>`; the rest come from the C library | — |
| Removed | trigraphs; non-two's-complement signed integers; K&R definitions | — | — | — |
| Changed | `realloc(p, 0)` is now UB; `__STDC_VERSION__` = `202311L` | — | — | — |

(Dashes mean "not individually listed in the sources consulted"; check the status pages for your exact version. Note: GCC 13/14 report `__STDC_VERSION__` as `202000L` in `-std=c2x`/`-std=c23` mode on the test machine; GCC 15 reports `202311L` — lesson B1.)

---

## 4. Step by step

### Step 1 — the core language upgrades in one program

```c
// file: c23_core.c
// build: gcc-14 -std=c23 -Wall -Wextra -o c23_core c23_core.c
// run: ./c23_core

#include <stdio.h>
#include <stddef.h>    // nullptr_t

// enum with a fixed underlying type: guaranteed 1 byte, values checked against it
enum colour : unsigned char { RED, GREEN, BLUE };

// unnamed parameter: no need for (void)unused tricks
static int second(int, int b) { return b; }

int main(void)
{
    constexpr int N = 1'000;            // a true compile-time constant object; ' separates digits
    static int table[N];                // constexpr objects can size arrays (not a VLA)
    auto mask = 0b1111'0000;            // auto infers int; binary literal
    typeof(mask) copy = mask;           // typeof names the type of an expression
    const int ci = 5;
    typeof_unqual(ci) not_const = ci;   // same type minus qualifiers: plain int
    not_const++;

    int *p = nullptr;                   // nullptr has its own type nullptr_t, never an int
    bool ok = true;                     // bool/true/false are keywords: no <stdbool.h> needed
    int zeros[4] = {};                  // empty initialiser: all zero

    static_assert(sizeof(enum colour) == 1);           // message is optional now
    static_assert(sizeof table == N * sizeof(int), "sizes match");

    printf("N=%d mask=%d copy=%d not_const=%d p=%p ok=%d zeros[3]=%d second=%d\n",
           N, mask, copy, not_const, (void *)p, ok, zeros[3], second(1, 2));
    return 0;
}
```

Output:

```text
N=1000 mask=240 copy=240 not_const=6 p=(nil) ok=1 zeros[3]=0 second=2
```

Why each matters:

- **`constexpr`** replaces many `#define` constants (lesson I4) with typed, scoped objects the compiler checks. Unlike a `const int`, a `constexpr int` is an integer constant expression, so `static int table[N]` is a normal array, not a VLA. `constexpr` file-scope objects have internal linkage (lesson I5).
- **`nullptr`**: `NULL` may be `0` or `(void*)0`, which goes wrong in variadic calls and `_Generic`; `nullptr` is always a null *pointer* constant.
- **`auto`** is deliberately limited: only for object definitions with an initialiser (no `auto` functions or parameters like C++).
- **Fixed-underlying-type enums** fix the "how big is my enum?" question (implementation-defined before C23) — useful for compact structs and ABIs.

### Step 2 — attributes: telling the compiler your intent

```c
// file: attrs.c
// build: gcc-14 -std=c23 -Wall -Wextra -o attrs attrs.c 2>&1 | grep -o "warning: .*"
// run: ./attrs

#include <stdio.h>
#include <stdlib.h>

// the caller must use the result: an ignored error code is a bug
[[nodiscard]] int parse_port(const char *s)
{
    int v = atoi(s);
    return (v > 0 && v < 65536) ? v : -1;
}

[[deprecated("use parse_port")]] int old_parse(const char *s) { return atoi(s); }

// noreturn: lets the compiler (and readers) know control never comes back
[[noreturn]] void die(const char *msg)
{
    fprintf(stderr, "fatal: %s\n", msg);
    exit(1);
}

int level_bonus(int level)
{
    int bonus = 0;
    switch (level) {
    case 3:
        bonus += 30;
        [[fallthrough]];                 // intentional: level 3 also gets level 2's bonus
    case 2:
        bonus += 20;
        break;
    default:
        break;
    }
    return bonus;
}

int main(void)
{
    [[maybe_unused]] int debug_only = 0;     // no "unused variable" warning
    parse_port("8080");                      // WARNING: result ignored
    int legacy = old_parse("1");             // WARNING: deprecated
    int port = parse_port("8080");
    if (port < 0) die("bad port");
    printf("port=%d bonus(3)=%d bonus(2)=%d legacy=%d\n", port, level_bonus(3), level_bonus(2), legacy);
    return 0;
}
```

Output (compiler warnings, then the program):

```text
warning: ignoring return value of 'parse_port', declared with attribute 'nodiscard' [-Wunused-result]
warning: 'old_parse' is deprecated: use parse_port [-Wdeprecated-declarations]
port=8080 bonus(3)=50 bonus(2)=20 legacy=1
```

(The `[[...]]` syntax is portable across conforming compilers; unknown attributes are ignored rather than being errors, and `__has_c_attribute(nodiscard)` lets you test support.)

### Step 3 — checked arithmetic: the end of "overflow-check after the fact"

```c
// file: ckd.c
// build: gcc-14 -std=c23 -Wall -Wextra -o ckd ckd.c
// run: ./ckd

#include <stdio.h>
#include <stdckdint.h>    // ckd_add, ckd_sub, ckd_mul
#include <stdint.h>
#include <limits.h>
#include <stdlib.h>

// allocate rows * cols elements of `size` bytes, refusing on overflow
void *alloc_matrix(size_t rows, size_t cols, size_t size)
{
    size_t cells, bytes;
    if (ckd_mul(&cells, rows, cols) || ckd_mul(&bytes, cells, size))
        return NULL;                            // the true product doesn't fit in size_t
    return malloc(bytes);
}

int main(void)
{
    int res;
    // ckd_add returns true if the mathematically correct result didn't fit in res
    bool ov1 = ckd_add(&res, INT_MAX, 1);
    printf("INT_MAX + 1 overflows int? %d (res = %d, the wrapped value)\n", ov1, res);

    bool ov2 = ckd_add(&res, 40, 2);
    printf("40 + 2 overflows? %d (res = %d)\n", ov2, res);

    // mixed types are fine: the check is against the result type
    uint8_t small;
    printf("200 + 100 fits in uint8_t? %s\n", ckd_add(&small, 200, 100) ? "no" : "yes");

    void *m1 = alloc_matrix(1000, 1000, sizeof(double));
    void *m2 = alloc_matrix(SIZE_MAX / 2, 3, 1);
    printf("1000x1000 doubles: %s; (SIZE_MAX/2)x3 bytes: %s\n", m1 ? "allocated" : "refused", m2 ? "allocated" : "refused");
    free(m1);
    free(m2);
    return 0;
}
```

Output:

```text
INT_MAX + 1 overflows int? 1 (res = -2147483648, the wrapped value)
40 + 2 overflows? 0 (res = 42)
200 + 100 fits in uint8_t? no
1000x1000 doubles: allocated; (SIZE_MAX/2)x3 bytes: refused
```

`ckd_add(&r, a, b)` computes the *mathematical* result, stores it wrapped into `*r`, and returns whether it fit — no UB anywhere. This is the standard version of GCC/Clang's `__builtin_add_overflow`, and the right answer to lesson I3's "check before you add" problem and to the `malloc(n * size)` overflow of lesson I2.

### Step 4 — `<stdbit.h>`, `%b`, `_BitInt`, and standardised POSIX string functions

```c
// file: bits23.c
// build: gcc-14 -std=c23 -Wall -Wextra -o bits23 bits23.c
// run: ./bits23

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbit.h>      // C23 bit utilities (provided by glibc 2.39 here)

int main(void)
{
    unsigned x = 0xF0u;
    printf("x = %b (binary via C23 %%b)\n", x);
    printf("count_ones=%u leading_zeros(1u)=%u trailing_zeros=%u bit_width(1000)=%u\n",
           stdc_count_ones(x), stdc_leading_zeros(1u), stdc_trailing_zeros(x), stdc_bit_width(1000u));
    printf("has_single_bit(64)=%d bit_ceil(100)=%u\n", stdc_has_single_bit(64u), stdc_bit_ceil(100u));

    // _BitInt(N): an integer of exactly N bits (here 12): range -2048..2047
    _BitInt(12) t = 2047;
    unsigned _BitInt(3) three_bits = 7;
    printf("_BitInt(12)=%d unsigned _BitInt(3)=%u\n", (int)t, (unsigned)three_bits);

    // strdup/strndup: long-time POSIX, now ISO C
    char *s = strndup("vanakkam world", 8);
    printf("strndup -> \"%s\"\n", s);
    free(s);
    return 0;
}
```

Output:

```text
x = 11110000 (binary via C23 %b)
count_ones=4 leading_zeros(1u)=31 trailing_zeros=4 bit_width(1000)=10
has_single_bit(64)=1 bit_ceil(100)=128
_BitInt(12)=2047 unsigned _BitInt(3)=7
strndup -> "vanakkam"
```

Before C23 these bit operations needed compiler builtins (`__builtin_popcount`) or slow loops; now they are portable and compile to single instructions (`popcnt`, `lzcnt`) where available.

### Step 5 — what *breaks* when you move to C23

```c
// file: breaks.c
// build: gcc -std=c17 -c breaks.c -o breaks_c17.o && echo "C17: compiles"; gcc-14 -std=c23 -c breaks.c -o breaks_c23.o 2>&1 | grep -E "error" | head -4
// run: true

int f();                  // C17: "unknown parameters"; C23: means f(void)
int bool = 1;             // C17: ordinary identifier; C23: bool is a keyword

int call_it(void)
{
    return f(1, 2);       // C17 accepts (no checking); C23 rejects: f takes no arguments
}

int old_style(a, b)       // K&R definition: removed in C23
    int a; int b;
{
    return a + b + bool;
}
```

Output (GCC 14.2; line numbers count the three comment lines at the top of the file):

```text
C17: compiles
breaks.c:6:5: error: two or more data types in declaration specifiers
breaks.c:6:10: error: expected identifier or '(' before '=' token
breaks.c:10:12: error: too many arguments to function 'f'
breaks.c:16:20: error: expected expression before 'bool'
```

A surprise from testing: **GCC 14 in `-std=c23` mode still accepts the K&R definition**, with only a warning (`old-style function definition [-Wold-style-definition]`) when tested on its own; the errors above are all about `bool` and `f`. Clang 18 with `-std=c23` rejects the same K&R definition outright (`unknown type name 'a'`). So "removed from the standard" does not mean "every compiler errors" — another reason to compile with more than one compiler. Migration rules: give every declaration a prototype, rename identifiers that are now keywords, rewrite K&R definitions, replace `realloc(p, 0)` with `free(p)`, and pass `-std=` explicitly in builds (GCC "Porting to GCC 15" also warns that `bool` is not the same as `int` at the ABI level if you change a type across a boundary).

### Step 6 — `#embed` (not compiled here)

```c
/* NOT COMPILED ON THE TEST MACHINE: #embed needs GCC 15+ or Clang 19+.
   Embeds the bytes of a file as an initialiser list at compile time. */
static const unsigned char cert[] = {
#embed "server.der"
};
static const size_t cert_len = sizeof cert;
```

Before `#embed`, projects used `xxd -i` or linker tricks to turn binary files into C arrays; `#embed` does it in the preprocessor (lesson I4), with `limit(...)`, `prefix(...)`, `suffix(...)`, `if_empty(...)` parameters and `__has_embed` for detection (cppreference *#embed*).

### Step 7 — `memset_explicit`, `free_sized`, and the library gap

Some C23 library functions are not yet in glibc 2.39's headers on the test machine (`grep memset_explicit /usr/include/string.h` finds nothing). `memset_explicit` exists for wiping secrets: an ordinary `memset` of a buffer that is never read again is a *dead store* the optimiser may delete (exactly what deleted the `a[4] = 99` store in lesson I8). Until your libc ships it, the common alternatives are `explicit_bzero` (glibc, BSD) or a write through a `volatile` pointer. Lesson for real projects: **a language standard is only as available as your compiler *and* your C library.**

---

## 5. Diagram: C23 at a glance

```mermaid
mindmap
  root((C23))
    Removed
      K&R definitions
      () as unknown params
      non-two's-complement
      trigraphs
    Keywords
      bool true false
      nullptr
      static_assert alignas alignof thread_local
    Types & constants
      constexpr
      auto
      typeof typeof_unqual
      enum : type
      _BitInt N
    Syntax
      attributes [[...]]
      0b literals, digit '
      = {} init
      unnamed params
      labels before decls
    Preprocessor
      #embed
      __has_include
      #elifdef #warning
      __VA_OPT__
    Library
      stdckdint.h ckd_add
      stdbit.h
      memset_explicit
      strdup strndup
      %b printf
      unreachable
      free_sized
```

---

## 6. Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Relying on the compiler's default dialect | Behaviour changes when GCC 15 defaults to `gnu23` | Always pass `-std=c17` or `-std=c23` |
| `int f();` in headers | C23: means no parameters → call sites with arguments fail | Full prototypes |
| Using `bool`/`true`/`false`/`nullptr` as identifiers or macros | Compile errors in C23 | Rename; use `<stdbool.h>` only for C99–C17 |
| Expecting C++ `constexpr` functions or `auto` return types | Not in C23 (objects only) | Use `static inline` functions |
| Assuming a library feature exists because the compiler accepts `-std=c23` | Link/compile errors (`memset_explicit` missing) | Check libc version; `__has_include` |
| Treating `ckd_add`'s stored value as valid after overflow | It's the wrapped value | Check the returned `bool` first |
| `_BitInt` on targets the compiler doesn't support | Errors (GCC 14 limits it to IA-32, x86-64, AArch64 LE) | Check `__BITINT_MAXWIDTH__` |
| Using `unreachable()` as an assertion | UB if reached (lesson A4) | `assert` in debug builds |

---

## 7. Connections

- **C++ track:** `nullptr`, `constexpr`, `auto`, attributes, digit separators, `static_assert` and `typeof`'s cousin `decltype` came from C++; compare how far each goes in each language.
- **Compilers track:** `_BitInt(N)` maps directly to LLVM's arbitrary-width `iN` types; attributes are hints consumed by the optimiser and warning passes.
- **Security track:** `ckd_*` (overflow-safe size computations) and `memset_explicit` (secret wiping that survives dead-store elimination) are security features.
- **Hardware / CUDA track:** `_BitInt` targets FPGA/HLS and ML quantisation use cases where odd widths matter [unverified: the motivation is attributed to the `_BitInt` proposal N2763's rationale, not checked here].
- **Rust track:** Rust's `checked_add`/`overflowing_add` are the same idea as `ckd_add`.

### Where this shows up in my projects

- **compiler-fuzz:** GCC 15.2 defaults to `gnu23` while older compilers default to `gnu17`, so a differential tester comparing GCC 15.2 and Clang 21.1 must pin `-std=` for both — otherwise a difference could come from different *dialects* (e.g. the meaning of `()`), not from a miscompilation. C23's `ckd_add` is also a clean way for a generator to emit overflow-safe arithmetic without hand-written safe-math helpers.

---

## 8. Self-check questions

1. What is the difference between `const int N = 10;` and `constexpr int N = 10;` for `int a[N];` at block scope?
2. Why is `nullptr` better than `NULL` in a call to a variadic function?
3. What does `ckd_mul(&r, a, b)` return, and what is stored in `r` on overflow?
4. List three things that compile in C17 but not in C23.
5. Why might `memset(secret, 0, n)` at the end of a function be removed, and what does C23 offer?
6. Which C23 feature is supported by GCC 14 but not by the test machine's Clang 18, among `constexpr`, `auto`, `typeof`?

<details>
<summary>Answers</summary>

1. With `const int`, `N` is not an integer constant expression in C, so `int a[N]` is a variable-length array; with `constexpr int`, `N` is a constant expression and `a` is an ordinary fixed-size array.
2. `NULL` may expand to plain `0` (an `int`), which is passed as an `int` to a variadic function expecting a pointer — wrong size/representation on 64-bit. `nullptr` always has pointer type semantics (`nullptr_t`).
3. It returns `true` if the mathematical product does not fit in `r`'s type; `r` receives the product wrapped to that type.
4. Calling a function declared `int f();` with arguments; using `bool`/`true`/`false`/`nullptr` as identifiers; K&R-style function definitions (also: trigraphs).
5. If `secret` is never read afterwards, the store is dead and may be eliminated as an optimisation. C23's `memset_explicit` must perform the write (alternatives today: `explicit_bzero`, volatile writes).
6. `constexpr` (Clang added it in Clang 19; GCC has it since 13).

</details>

---

## 9. Mnemonic (Tamil + English)

**C23's big five for daily use — "Null, Const, Check, Type, Bits" → "N-C-C-T-B":**

> **"Naan Chennai Central-la Train Booking"** — **N**ullptr, **C**onstexpr, **C**kd_add (checked arithmetic), **T**ypeof/auto, **B**its (`<stdbit.h>`, `_BitInt`, `0b`).

**What died in C23 — "K-T-O" (K&R, Trigraphs, Ones'-complement/sign-magnitude):**

> **"KTO — Kadaisi Thadavai Ozhinjudhu."** — *Kadaisi thadavai ozhinjudhu*: "finished for the last time".

---

## Sources

- cppreference, *C23* (feature list with proposal numbers): https://www.cppreference.com/w/c/23.html
- WG14 N3220 (C23-equivalent draft; `<stdckdint.h>` §7.20, `<stdbit.h>` §7.18, `<stddef.h>` §7.21, type inference §6.7.10, binary resource inclusion §6.10.4): https://open-std.org/JTC1/SC22/WG14/www/docs/n3220.pdf
- WG14 N3221, editor's report (N3220 vs N3219): https://open-std.org/JTC1/SC22/WG14/www/docs/n3221.htm
- Wikipedia, *C23 (C standard revision)* (publication 31 Oct 2024, `202311L`, removals): https://en.wikipedia.org/wiki/C23_(C_standard_revision)
- GCC, *C Status* (per-feature GCC versions): https://gcc.gnu.org/projects/c-status.html
- Clang, *C Support in Clang* (per-feature Clang versions, `-std=c23` from Clang 18): https://clang.llvm.org/c_status.html
- GCC, *Porting to GCC 15* (default `-std=gnu23`; `()` meaning; new keywords; `bool` ABI note): https://gcc.gnu.org/gcc-15/porting_to.html
- GCC, *GCC 14 Release Series Changes* (`-std=c23` option): https://gcc.gnu.org/gcc-14/changes.html
- cppreference, *#embed*: https://en.cppreference.com/w/c/preprocessor/embed ; *unreachable*: https://en.cppreference.com/w/c/program/unreachable ; *ckd_add*: https://en.cppreference.com/w/c/numeric/ckd_add
- Jens Gustedt, *Modern C*, Third Edition (Manning, covers C23): https://www.manning.com/books/modern-c-third-edition

## Verify yourself

1. Build every block with `gcc-14 -std=c23` (or GCC 15 / Clang 19+) and compare with the outputs above. For `attrs.c`, note the exact last line your program prints.
2. Check feature-by-feature support: `echo '__STDC_VERSION__' | gcc -std=c23 -E -x c - | tail -1`, then look up each feature on the GCC and Clang status pages.
3. Compile `breaks.c` with `-std=c17` and `-std=c23` and record the exact error messages.
4. Look for C23 library functions in your libc: `grep -E "memset_explicit|free_sized" /usr/include/*.h`.
5. Try `#embed` on Compiler Explorer with GCC 15 (`-std=c23`).
