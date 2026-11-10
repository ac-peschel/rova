# Rova v0 Spec

Status: **frozen** (2026-11-10). Changes go to the "Later" list in ROADMAP.md, not into this file, until v0 is done.

## Goal and scope

v0 is a complete compiler for a tiny C-like language: integers, booleans, local variables, functions, `if`/`else` and `while`, producing a native Linux x86-64 executable. It is small on purpose, not unfinished: everything listed here works, is tested, and has good error messages.

The syntax is C at the core, with C# strictness: `int x = 12;`, `int add(int a, int b) { ... }`, parentheses around conditions, semicolons, braces. No `fn`, `let` or `var`.

**v0 is done when:**

- Every example program in the test suite compiles, runs, and produces the expected stdout and exit code.
- Every error test produces the expected diagnostic at the expected line and column.
- The compiler is a single binary (C#, Native AOT) and this spec matches what it does.

## Decisions

| Topic | Decision |
| --- | --- |
| Name | Rova, source files `.rv`, compiler binary `rova` |
| Size of `int` | 32 bit on every machine; wider types are explicit (`long`, after v0) |
| Braces | Always required; no braceless one-liners |
| `++` / `--` | Not in v0, possibly never |
| Shadowing | Forbidden, as in C# |

## Lexical structure

Source files are UTF-8; v0 accepts ASCII only outside comments.

- **Comments:** `// to end of line` and `/* block */`. Block comments do not nest.
- **Identifiers:** a letter or `_`, then letters, digits or `_`. Case-sensitive.
- **Keywords:** `int` `bool` `void` `if` `else` `while` `break` `continue` `return` `true` `false`. Reserved for later: `struct` `class` `for` `const`.
- **Integer literals:** decimal only, e.g. `0`, `42`, `2147483647`. A literal that does not fit in `int` is a compile error. `_` separators (`1_000_000`) are allowed.
- **Boolean literals:** `true`, `false`.
- **Whitespace** is insignificant except as a separator.

## Types

v0 has three types and no implicit conversions between them.

| Type | Meaning | Size | Notes |
| --- | --- | --- | --- |
| `int` | signed integer, two's complement | 32 bit | Same as C# `int`. Overflow wraps; never undefined behavior. |
| `bool` | `true` or `false` | 8 bit | Not a number: `if (x)` with `int x` is an error, write `if (x != 0)`. |
| `void` | no value | — | Only as a function return type. |

Division or modulo by zero aborts the program (the CPU traps). Dividing the smallest `int` (-2147483648) by -1 also aborts, as on x86-64.

## Variables

Local variables only; there are no globals in v0.

```
int count = 0;
bool done = false;
```

- A declaration always has an initializer. `int x;` is an error, so there are no uninitialized reads.
- Scope is the enclosing block `{ ... }`. A variable is visible from its declaration to the end of the block.
- Shadowing is an error: declaring a name that is already visible (including a parameter) is rejected, as in C#.
- Variables live on the stack. Parameters are passed by value and are assignable inside the function.

## Functions

Functions are declared at the top level, in any order. There are no prototypes and no headers: the compiler collects all signatures before checking bodies, so a function can call one defined further down, and recursion works.

```
int add(int a, int b) {
    return a + b;
}

void report(int value) {
    print(value);
}
```

- Return type first, then name, then parameters, as in C and C#.
- A non-`void` function must return a value on every path; the compiler checks this.
- A `void` function may use a bare `return;`.
- No overloading in v0: function names are unique.
- **Entry point:** `int main()`. Its return value is the process exit code (0–255 on Linux).

## Statements

Conditions sit in parentheses and must be `bool`. Bodies always use braces; braceless `if (x) y;` is an error, which also removes the dangling-`else` problem.

```
if (n < 2) {
    return n;
} else if (n == 2) {
    return 1;
} else {
    return fib(n - 1) + fib(n - 2);
}

while (i < 10) {
    if (i % 2 == 0) { i += 1; continue; }
    print(i);
    i += 1;
}
```

| Statement | Form |
| --- | --- |
| Declaration | `int x = expr;` |
| Assignment | `x = expr;` and `+=` `-=` `*=` `/=` `%=` |
| Call | `f(a, b);` |
| If | `if (cond) { } else if (cond) { } else { }` |
| While | `while (cond) { }` |
| Break / continue | `break;` `continue;` inside a `while` only |
| Return | `return expr;` or `return;` |
| Block | `{ ... }` opens a new scope |

Assignment is a statement, not an expression, so `if (x = 5)` cannot happen. An expression statement must be a call; `x + 1;` is an error.

## Expressions and operators

Precedence follows C and C#, highest first. Operands are evaluated left to right, always, so `f() + g()` calls `f` first.

| Level | Operators | Operand types | Result |
| --- | --- | --- | --- |
| 1 | `f(args)`, `( expr )` | — | — |
| 2 | unary `-`, `!` | `int`, `bool` | same |
| 3 | `*` `/` `%` | `int` | `int` |
| 4 | `+` `-` | `int` | `int` |
| 5 | `<` `<=` `>` `>=` | `int` | `bool` |
| 6 | `==` `!=` | both `int` or both `bool` | `bool` |
| 7 | `&&` | `bool` | `bool`, short-circuit |
| 8 | `\|\|` | `bool` | `bool`, short-circuit |

All binary operators are left-associative. Division truncates toward zero; `%` takes the sign of the left operand, matching C#.

## Built-ins, toolchain and target

v0 has exactly one built-in, so programs can show results without strings or a standard library:

- `void print(int value)` writes the number in decimal plus a newline to stdout. It is implemented in a tiny runtime the compiler emits itself (integer-to-text plus a `write` system call). It is a temporary intrinsic and is replaced by a real standard library later.

**Toolchain:**

- One source file per program: `rova hello.rv` produces an executable.
- The compiler is written in C# and published with Native AOT as a single binary.
- Pipeline: lexer → parser → syntax tree → type checker → x86-64 assembly text.
- Target: Linux x86-64, statically linked, no libc. The program starts at `_start`, calls `main`, and exits via system call.
- NASM and `ld` assemble and link the output in v0. They are scaffolding: the roadmap replaces them with the compiler's own encoder and ELF writer, after which the toolchain depends on nothing.

## Errors and diagnostics

Every error names the file, line and column, shows the source line, and points at the problem:

```
fib.rv:7:12: error: cannot use 'int' as a condition, expected 'bool'
    if (n) {
        ^
hint: compare explicitly, e.g. 'n != 0'
```

- v0 stops at the first error per compiler phase (lexing, parsing, type checking). Error recovery comes later.
- The compiler never crashes on bad input: an internal failure prints "internal compiler error" with the position, and is treated as a bug.
- Diagnostics are part of the test suite, so their wording and positions are checked like any other output.

## Not in v0

Each item below is deliberately left out of v0 and has a place on the roadmap. Listing it here keeps it from creeping in.

- Other number types: `long`, unsigned, floats, `char`; casts between types
- Strings, arrays, pointers, structs, classes
- Heap allocation, refcounting, destructors
- Global variables and constants
- `for`, `switch`, `++` and `--`, the ternary `?:`
- Bitwise operators `& | ^ ~ << >>`
- Overloading, generics, closures
- Multiple files, modules, imports, a standard library
- Windows, the PE format, other CPUs
- Optimizations beyond what falls out of simple codegen

## Test suite

The test suite defines v0. Each test is a source file plus its expected stdout and exit code, or its expected error.

```
int fib(int n) {
    if (n < 2) {
        return n;
    }
    return fib(n - 1) + fib(n - 2);
}

int main() {
    int i = 0;
    while (i < 10) {
        print(fib(i));
        i += 1;
    }
    return 0;
}
```

| Program | Exercises | Expected |
| --- | --- | --- |
| `exit42` | `main` returning a value | exit code 42 |
| `arith` | precedence, `/` and `%` with negatives, wrapping | printed values |
| `fib` | recursion, `if`, `while` | 0 1 1 2 3 5 8 13 21 34 |
| `factorial` | loops, overflow wrapping at 13! | printed values |
| `gcd` | `%`, `while`, multiple functions | printed values |
| `primes` | nested loops, `break`, `continue`, `&&` | primes below 100 |
| `shortcircuit` | `&&` and `\|\|` skip calls | print order |
| `scopes` | block scoping, parameters as locals | printed values |
| `forward_call` | calling a function defined later | printed values |

Error tests cover each diagnostic: missing semicolon, undeclared name, shadowing, `int` as a condition, missing return path, wrong argument count, uninitialized declaration, braceless `if`, and an out-of-range literal.
