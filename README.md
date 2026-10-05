# decac — A Compiler for Deca, a Java Subset

A multi-pass compiler, written in Java, that turns Deca source code (an object-oriented subset of Java) into assembly for the IMA abstract machine. It also has an experimental ARM backend.

> Academic project, Ensimag (Grenoble INP), December 2021 – January 2022. Software Engineering ("Projet GL") team project: team gl47 "JuNGLE", five developers.

## Overview

`decac` is a full compiler pipeline built on top of a provided skeleton:

1. **Lexing and parsing** with ANTLR 4 grammars, which build an abstract syntax tree (AST).
2. **Contextual verification** in three passes over classes (hierarchy, then members, then bodies), followed by the main program: type checking, scoping, inheritance and visibility rules from the Deca specification.
3. **Code generation** for IMA, a register-and-stack abstract machine, with runtime error checks.

The work was organised in three agile sprints, one per language level: "Hello world", Deca without objects, then full Deca. It followed a validation-driven process: a large non-regression suite, GitLab CI, and JaCoCo coverage. The design, validation and user documentation (in French) are in `docs/`.

## What was implemented

- **Lexer and parser** (`src/main/antlr4/.../DecaLexer.g4`, `DecaParser.g4`, about 850 lines of grammar):
  - the full Deca syntax: classes, fields with `protected` visibility, methods, `new`, casts, `instanceof`, `this`, `null`
  - `#include` directives, with detection of circular and missing includes (`CircularInclude`, `IncludeFileNotFound`)
- **AST** (`fr.ensimag.deca.tree`): about 85 node classes. Every node can decompile itself back to Deca source (`decac -p`).
- **Contextual analysis** (`fr.ensimag.deca.context`):
  - types: `IntType`, `FloatType`, `BooleanType`, `ClassType`, `NullType` and others, plus a built-in `Object` class with `equals`
  - environments (`EnvironmentExp`, `EnvironmentType`), method `Signature`s, field and method definitions
  - located error messages (`ContextualError`)
- **IMA code generation**:
  - *Deca without objects*: variables, int/float arithmetic with implicit `ConvFloat`, boolean logic, `if`/`while`, I/O (`print`, `println`, `readInt`, `readFloat`), and int/float casts
  - *objects*: virtual method tables built at program start, object allocation (`new`) with field initialisation that chains to the superclass initialiser, field access and assignment, and method calls dispatched through the method table
  - runtime error handlers: arithmetic overflow, division by zero, stack/heap overflow (`TSTO`/`BOV`) and invalid input
- **Parallel compilation** (`decac -P`): compiles each file as a task on an `ExecutorService` thread pool sized to the number of CPU cores.
- **Experimental ARM backend** (`fr.ensimag.arm.pseudocode`, `decac -a`):
  - emits GNU ARM assembly that uses Linux syscalls (`Write`, `Exit`)
  - `arm-env.sh` installs a cross-toolchain and runs the result under `qemu-arm`
  - it currently handles printing string literals. The design for full Deca is in `docs/Documentation_de_l_extension.pdf`.

**Known limitations at submission:** IMA code generation is not implemented for `return`, `this`, `null` or assembly-bodied methods. `instanceof` code generation is incomplete. The `Math` library (`Math.decah`) is a stub.

## Technical highlights

- **Composite / Interpreter-pattern AST**: each node implements its own `verify*`, `decompile` and `codeGen` methods.
- **One AST, two targets**: the `CodeGen` interface has overloads for `IMAProgram` and `ARMProgram`, behind a common `OutputProgram` abstraction.
- **Expected-output test suite**:
  - 298 `.deca` test programs, sorted by stage (`syntax`, `context`, `codegen`) and by outcome (`valid`, `invalid`, `perf`)
  - expected results are stored as `.lis` traces and compared by a Python runner (`src/test/script/test.py`) that Maven calls during `verify`
- **Unit tests**: about 30 JUnit 5 / Mockito test classes, for example `EnvironmentExpTest`, `OpArithTest` and `TestPlusAdvanced`.
- **CI**: the GitLab pipeline (`.gitlab-ci.yml`) compiles, runs every test, and publishes a JaCoCo report with the instruction coverage printed in the job log.

## My contribution

My work was mainly on validation:

- wrote a large share of the syntax and contextual-verification test programs, from the "hello world" level up to classes (inheritance, duplicate fields, `protected` access, casts, `instanceof`)
- wrote the first version of the Python test runner (`test.py`)
- wrote the script that generates the `.lis` expected outputs (`organiser.sh`)
- organised the test tree by stage and validity, and wired the suite into the Maven build

## Tech stack

Java 11 · ANTLR 4.9 · Maven · JUnit 5 · Mockito · JaCoCo · log4j · Python · Shell · GitLab CI · Docker · IMA abstract machine · ARM assembly / QEMU

## Project structure

```
src/main/antlr4/fr/ensimag/deca/syntax/   ANTLR lexer and parser grammars
src/main/java/fr/ensimag/deca/
├── DecacMain.java, DecacCompiler.java    CLI entry point, compilation pipeline
├── syntax/                               parser support, includes, syntax errors
├── tree/                                 AST nodes (verify / decompile / codeGen)
├── context/                              types, definitions, environments
└── codegen/                              CodeGen and OutputProgram interfaces
src/main/java/fr/ensimag/ima/pseudocode/  IMA instruction model
src/main/java/fr/ensimag/arm/pseudocode/  ARM instruction model (extension)
src/test/deca/                            .deca test programs and expected .lis outputs
src/test/java/                            JUnit / Mockito unit tests
src/test/script/                          test runner and launchers
docs/                                     user manual, design, validation, ARM extension docs
tictac.deca                               sample program: tic-tac-toe in Deca
```

## Build and run

Requirements: JDK 11+, Maven, Python 3, and the IMA interpreter (Linux x86_64 binaries are in `ima/`).

```bash
mvn compile                       # generate the ANTLR parser and compile
./src/main/bin/decac prog.deca    # write prog.ass (IMA assembly)
ima/ima prog.ass                  # run it on the IMA machine

mvn verify                        # unit tests + .deca suite + JaCoCo report (target/site)
```

`decac` options:

| Option | Effect |
|---|---|
| `-b` | Print the team banner |
| `-p` | Parse only, then decompile the AST |
| `-v` | Stop after contextual verification |
| `-n` | Disable runtime overflow checks |
| `-r N` | Limit the number of registers |
| `-d` | Increase log verbosity (repeatable) |
| `-P` | Compile several files in parallel |
| `-a` | Emit ARM assembly (experimental) |

To try the ARM backend: run `./arm-env.sh install`, then `./arm-env.sh run prog.s` (requires `qemu-arm`).
