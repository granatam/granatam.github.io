---
title: "How compilation works"
date: "2025-04-27"
layout: "single"
tags:
  - compilers
tldr: "It's not as simple as you might think!"
description: "What exactly happens when you click on that green Build button? Let's figure it out!"
---

****

{{< callout type="custom" 
    emoji="⚡️" title="Disclaimer"
    text="JIT compilation will not be covered in this post. If you are interested in this topic, please wait for a future post or look for an information elsewhere."
    style="background-color: transparent; border: 3px solid #d340e0;"
>}}

## Introduction

When you enter a build command in the terminal or click the Build button
in an IDE, you're triggering a process that translates human-readable source
code into an executable program. Terms like *compiler*, *preprocessor*,
*assembler*, and *linker* may sound familiar, but do you know exactly what each
of these tools does? Let's explore what actually takes place at each stage of
the compilation process!

Before we start, I'd like to clarify some terminology. What we typically call a
*compiler* (such as `gcc`, `clang`, etc.), is, more precisely, a **compiler
driver**: a tool that coordinates several utilities of the compiler
toolchain. The compiler driver manages these tools, invoking them in the
correct order with the appropriate options so that users don't have to run each
tool manually or specify all the details themselves. *Still, outside of
compiler toolchain development teams, it's common to refer to the
entire toolchain as the compiler.*

In this post we will refer to a `g++` compiler driver as an example.

## Preprocessor

The **preprocessor** is the first to get to work, processing the source code:
1. Replaces macros with their defined values (`#define true false`).
2. Recursively replaces `#include` directives with the content of those files.
3. Removes conditional directives `#if`, `#ifdef`, etc., and leaves only the
   code that matches the conditions.
4. Handles escape sequences and concatenates string literals.

The output is a `.i` (C) or `.ii` (C++) file with the preprocessed source code.

> Depending on the toolchain, the preprocessor may be part of the compiler (as
> in GNU Toolchain, LLVM, etc.) or operate as a standalone utility (e.g., the
> `cpp` preprocessor).

## Compiler

Next, the **compiler** steps in; for `g++`, the actual compiler is the
`cc1plus` tool. The compiler is logically divided into three parts:
the **frontend**, **middle-end** and **backend**. The frontend analyzes the
source code and transforms it into an intermediate representation, the
middle-end performs machine-independent optimizations on this IR, and the
backend is responsible for machine-dependent optimizations and code generation.

### Frontend

The first (or the second, if preprocessor is integrated in the compiler) stage
of the compiler's work is **lexical analysis** or scanning. The meaning of the
constructs isn't important yet; the goal is to break the code into **lexemes**,
classify them, and check for invalid characters. This is done in a single pass.
For example, if we input the code `int main() { return 0; }`, the lexical
analyzer will output the following sequence: `int`, `main`, `(`, `)`, `{`,
`return`, `0`, `;`, `}`. During this phase, the initial **symbol table** is
also created, recording identifiers.

> Although this post uses GNU Toolchain as an example, LLVM with its compiler
> drivers (`clang` or `clang++`) offers additional features, such as detailed
> token diagnostics. To inspect the output of the LLVM lexical analyzer, use
> e.g. `clang -fsyntax-only -Xclang -dump-tokens foo.c`.

Next comes **syntax analysis** or **parsing**. At this stage, the compiler
transforms the sequence of lexemes into a tree structure. **The parser**
identifies expressions, statements, and declarations from the tokens according
to language's grammar rules, producing an **Abstract Syntax Tree (AST)** --- a
program representation that links operations with their data dependencies.
During parsing, identifiers in the symbol table are also bound to their context
and scope. As with lexical analysis, we can get an AST in LLVM with `clang
-fsyntax-only -Xclang -ast-dump foo.c`.

> This might sound like an *LLVM is better than GNU Toolchain* claim, but the
> reality is more nuanced. We'll explore this comparison in another post!

**Semantic analysis** takes place **alongside** parsing and builds on the
partially constructed AST to interpret language constructs and refine the tree
further. During this phase, the compiler checks things like variable
declarations, type correctness, function overloads, scopes, specific language
features and many more to ensure the program is semantically correct.

The next and transitional step between the frontend and middle-end is
**intermediate representation (IR) generation**. IR helps in code
optimizations, analysis and synthesis, and it allows to build a family of
compilers much easier: instead of building $N * M$ compilers for $N$ languages
and $M$ platforms, you only need $N$ frontends and $M$ backends. While AST is
technically an IR, this phase is specifically called IR generation because ASTs
are language-specific, whereas IR becomes a language- and platform-independent
foundation for the middle-end.

<p style="text-align: center;">
  <img src="/images/compilation-ir.svg" alt="Modular architecture" style="width:500px; display: inline-block;">
</p>

> Did you know that GCC is not just for C and C++? Over the years, the GNU
> Compiler Collection has supported a wide range of languages. For example,
> until GCC 7, it included a Java compiler `gcj`. As of version 15.1, GCC
> features Go, Ada, COBOL (since GCC 15.1 0_0) and many other frontends.

In GCC, most frontends first convert their AST into GENERIC --- a
language-independent IR. This is followed by **gimplification**, where GENERIC
is translated into GIMPLE, a simplified three-address code IR.

> Why not convert AST directly to GIMPLE? The answer is modularity and code
> deduplication. The conversion from GENERIC to GIMPLE is handled by a separate
> module called `gimplifier`. This allows any changes to GIMPLE processing to
> be made in one place, without touching the frontends. Without GENERIC, each
> frontend would have to implement its own logic for lowering complex
> constructs and converting them to three-address code, which would lead to
> redundant work.

However, for C and C++, as of now, this mechanism isn't used:  GIMPLE is
generated directly from the AST, since these languages are easier to translate
straight into three-address code.

To view the GIMPLE representation of a program, use the `-fdump-tree-gimple`
flag. The general syntax for GCC IR dumps is `-fdump-tree-<type>`.

### Middle-end

The **middle-end** or **optimizer** performs **machine-independent**
optimizations on the IR. Its work can be divided into two parts:
1. **Analysis** --- the compiler gathers information about a program, such as
   building a Control Flow Graph, determining variables lifetimes and more.
2. **Optimization** --- a sequence of optimization passes is performed, each
   solving a specific problem. The order of these optimizations is defined in
   the compiler’s source code. Examples of such passes include: dead code
   elimination, constant folding, copy propagation and many more.

To inspect optimized IR at specific stages, use GCC's `-fdump-tree-<pass>`
flags. For example, `-fdump-tree-stdarg` shows GIMPLE after optimizing variadic
functions.

After middle-end optimizations, GCC transitions from GIMPLE to RTL (Register
Transfer Language) --- low-level, Lisp-like IR that incorporates
platform-specific details.

> Although GCC’s middle-end primarily works with the GIMPLE IR, some of its
> optimization passes leverage target hooks to access backend-specific
> information, enabling architecture-aware optimizations while preserving a
> separation between generic and platform-dependent code.

### Backend

Just as there are **machine-independent** optimizations, there are also
**machine-dependent** ones. The backend handles these: it allocates registers
(initially using pseudo-registers, which are later mapped to the CPU’s physical
registers), chooses and reorders instructions, and performs various other
architecture-specific optimizations.

The compiler's last step is **code generation**, where the IR is translated
into assembly code. In `cc1plus`, this involves matching RTL patterns
defined in `.md` (Machine Description) files to generate architecture-specific
instructions. These files ensure deterministic code generation for a given
target platform. During code generation, pattern matching is performed,
registers are allocated (RTL pseudo-registers are replaced with CPU registers),
and, finally, assembly code is generated.

> In the case of C++, code generation also includes name mangling ---
> identifiers are transformed into  **unique** strings encoding parameter
> types, namespaces, and other details. This enables features like function
> overloading and templates. C, however, doesn't mangle names, which is why C++
> code using C functions must declare them with `extern "C"` to avoid linkage
> errors.

While the compiler's job ends with the `.s` file, the process isn't over yet!

## Assembler

The next tool in the toolchain is the **assembler**. In the GCC ecosystem, this
is the GNU Assembler or `gas`.

During the assembly stage, assembly mnemonics are translated into machine code
--- each instruction is replaced by the corresponding binary code for the CPU
command. An object file is created, which contains sections such as:
- `.text` --- machine instructions,
- `.data`, `.rodata`, `.bss` --- global variables, read-only data, and
  uninitialized data, respectively,
- `.symtab` and `.strtab` --- the symbol table and string table for symbol
  names,
- `.rel.text`, `.rel.data` --- relocation information for code and data.

If the program is built with debugging information (the `-g` flag in GCC), the
object file will also include `.debug_*` sections. The object file may also
contain architecture- and language-specific sections, but that is beyond the
scope of this discussion. The assembler also processes assembler directives,
such as those that define the program’s entry point, code sections, external
function declarations, and similar instructions.

## Linker

The last tool in the toolchain, but by no means the least important, is the
**linker** (`ld` in GCC, `lld` in LLVM). While the compiler processes
individual translation units, the linker combines object files into a single
executable or library.

In GCC, the linker stage involves an utility called `collect2`, which is a `ld`
wrapper. Its primary role is to handle global constructors (e.g., C++ static
initializers) and generate initialization code --- a temporary `.c` file with
a table of constructors, which is added as an extra object module. These
constructors are then called before `main()` is executed.

In addition to compiler optimizations, there are **Link Time Optimizations**
(LTO). These optimizations involve cross-module analysis, which does things
like removing dead code, inlining functions or constants from other modules,
and similar optimization passes. When LTO is enabled, then object files
generated during assembly contain IR instead of machine code. A linker plugin
(e.g., GCC's `liblto_plugin.so`) is then used, which optimizes these files and
convert them into final machine code.

After LTO (if enabled), the linker collects and merges the object files, and
then symbol resolution takes place: the linker creates a global symbol table
from the object files and libraries, searches for matches for each undefined 
symbol in other modules, and, if a symbol cannot be found, or if it is found
twice, reports an error.

After all symbols have been resolved, the linker performs **address
relocation** to ensure that all parts of program are correctly connected.

> **Why is relocation necessary?** When the compiler and assembler generate
> object files, final memory locations of functions or variables are unknown.
> Instead of real addresses, they leave placeholders and create entries in a
> relocation table.

The linker assigns final addresses to each section, then goes through the
relocation table and replaces all placeholders in the code and data with the
actual addresses.

The linker also handles libraries:
- For **static** libraries, the linker extracts only the required symbols from
  the library and merges them with the other object files.
- For **dynamic** libraries, a reference to the library is added to a special
  section (for example, `.dynamic` in ELF files), and the library is loaded at
  runtime using this reference.

The next stage of linking is generating the executable format --- the linker
creates the structure of the executable file according to the target format.
For example, when producing an ELF file, the linker generates the ELF header,
as well as the section and segment tables.

🥳 And finally, we’ve reached this point --- the **executable file** is
created! 🥳 The linker writes the machine code, symbol table, relocation
information, and dynamic library references into the executable. The result is
a ready-to-use program that can be run on the target system.

