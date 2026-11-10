# Rova Roadmap

Rova is a hobby language: C at the core, C#-style syntax, compiled to native code with refcounting, and a self-reliant Go-like toolchain (no external compiler, assembler or linker in the end).

The v0 language is defined in [SPEC.md](SPEC.md). This file says how we get there and what comes after.

## Principles

- **Small and complete.** Every milestone ends in something finished and tested at its own scope, never a broken version of the final product.
- **The spec is frozen per milestone.** New ideas go to "Later" at the bottom, not into the current milestone.
- **Tests first-class.** Each test is a `.rv` file plus expected stdout and exit code, or an expected diagnostic. A milestone is done when its tests pass.
- **Verify, don't trust.** Generated binaries are checked with `objdump`, `readelf` and `gdb`, especially byte-level work (instruction encoding, ELF headers).
- **Understand every piece.** No code goes in that the author can't explain. AI helpers explain the concept first, then write it together.
- **Linux x86-64 first.** Windows is its own later milestone.

## Fixed technical choices

| Area | Choice |
| --- | --- |
| Compiler language | C#, published as a single Native AOT binary |
| Editor setup | Neovim (Roslyn LSP via roslyn.nvim) |
| First target | Linux x86-64, ELF, static, no libc, direct system calls |
| v0 assembler/linker | NASM + `ld` as temporary scaffolding, removed in milestone 5 |
| Memory model (later) | Refcounting with RAII-style destructors by default, opt-in arenas |
| Calling convention | System V AMD64 (Linux); Win64 added with Windows support |

## Milestones

### 0. Hand-written assembly
Learn what the compiler will emit before writing the compiler. Build with NASM and `ld`.

- [ ] Program that exits with code 42 (`_start`, `exit` system call)
- [ ] Program that prints a fixed string via the `write` system call
- [ ] Function that converts an int to decimal text and prints it (this becomes the `print` runtime)
- [ ] Function call with arguments and a return value, using the System V convention and a stack frame
- [ ] Inspect each result with `objdump -d` and step through it in `gdb`

Done when: all of the above build, run, and you can explain every line.

### 1. Tiny compiler (v0, part 1)
Lexer → parser → syntax tree → assembly text for the simplest programs.

- [ ] C# project skeleton, `rova file.rv` CLI, test runner that compiles and runs every test
- [ ] Lexer with positions (line, column) on every token
- [ ] Parser for functions, `int` locals, `return`, arithmetic expressions
- [ ] Codegen to NASM: stack frame per function, locals in stack slots, expressions via a simple stack-based scheme
- [ ] `print` runtime emitted into every program

Done when: `exit42`, `arith` and `forward_call` pass.

### 2. Complete v0
Everything else in SPEC.md.

- [ ] `bool`, comparisons, `&&` / `||` with short-circuit
- [ ] `if` / `else if` / `else`, `while`, `break`, `continue`
- [ ] Type checker: no implicit conversions, `bool` conditions, argument counts, return-path checking
- [ ] Scopes and the shadowing rule
- [ ] Diagnostics in the spec's format (file:line:col, source line, caret, hint)
- [ ] All example and error tests from SPEC.md

Done when: the whole SPEC.md test suite passes. **v0 released.**

### 3. Own IR
A mid-level intermediate representation between the syntax tree and codegen.

- [ ] Explicit control flow (basic blocks, jumps), explicit temporaries
- [ ] Lower the checked syntax tree to IR; generate assembly from IR instead
- [ ] IR printer for debugging

Done when: the v0 suite passes through the IR path unchanged.

### 4. Real types and memory
Spec v1, written and frozen before starting.

- [ ] `long`, unsigned types, `char`, explicit casts
- [ ] Structs, arrays, strings
- [ ] Heap allocation (own allocator on `mmap`, no libc)
- [ ] Refcounting: compiler-inserted retain/release in the IR
- [ ] Destructors at scope exit, RAII style
- [ ] Opt-in arenas

### 5. Drop NASM and `ld`
The toolchain becomes self-reliant on Linux.

- [ ] Own x86-64 instruction encoder (check each instruction against NASM output byte for byte)
- [ ] Own ELF writer for static executables
- [ ] Remove NASM and `ld` from the pipeline; the test suite passes with zero external tools

### 6. Language growth
- [ ] Generics (monomorphized)
- [ ] Modules and imports across multiple files (Go-style: compile each module once, importers read only its exported interface)
- [ ] Small standard library (I/O, strings, collections)
- [ ] `for` loops, constants, globals as needed

### 7. Windows
- [ ] PE/COFF writer
- [ ] Win64 calling convention
- [ ] Generate import tables directly (no Windows SDK `.lib` files, as Go does)

### 8. Later stages
- [ ] Register allocation and basic optimizations
- [ ] Debug info (DWARF), so `gdb` shows Rova source lines
- [ ] Optionally: self-hosting (Rova compiler written in Rova)

## Later (ideas parked during frozen milestones)

Add ideas here instead of changing the current milestone's scope.

-

## References

- *Writing a C Compiler*, Nora Sandler (No Starch Press, 2024): incremental compiler to x86-64 assembly; closest match to milestones 1–3.
- *Crafting Interpreters*, Robert Nystrom (free online): lexing, parsing, error reporting.
- System V AMD64 ABI and the Linux system call table for x86-64.
