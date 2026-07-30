# Askal Programming Language

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![C++](https://img.shields.io/badge/C++-17-blue.svg)](https://isocpp.org/)
[![Version](https://img.shields.io/badge/version-1.0-orange.svg)](https://github.com/forXsis/askal/releases)
[![Platform](https://img.shields.io/badge/platform-Windows%20-lightgrey.svg)]()

**Askal** is a minimalistic programming language with its own virtual machine, compiler to bytecode, and a built‑in memory manager. It is designed to be simple, readable, and easy to extend.

---

## Features

- **Variables** – `int`, `float`, `str`, `bool`
- **Arithmetic** – `+`, `-`, `*`, `/` with standard precedence and parentheses
- **Output** – `print` and `println` with multiple arguments
- **Functions** – declaration and calls (under development)
- **Bytecode compiler** – generates compact `.aklp` files
- **Virtual machine** – stack‑based execution with MMD memory management
- **Disassembler** – `ast.py` for inspecting bytecode

---

## Quick Start

### Installation

Install the ready-made zip archive with the compiled exe file and additional files (LICENSE, examples, docs) in the Releases branch.

### Build

Clone the repository and build the project:

```bash
git clone https://github.com/forXsis/askal.git
cd askal
```

**Using Visual Studio** (Windows):  
1. Open the source code in Visual Studio 2022.
2. Create a C++17 Console Application.
3. Add all files from the `src/` directory.
4. Build the project (Ctrl + Shift + B).
5. Add the path to the created program to Path so that you can easily interact with it.

> CMake support is planned for future releases.

### Write Your First Program

Create a file named `hello.akl`:

```akl
var str greeting = "Hello, Askal!";
println < greeting;
```

### Compile and Run

```bash
# Compile to bytecode
askal-manager -compile hello.akl

# Run the bytecode
askal-manager -run hello.aklp

# Or compile and run in one step
askal-manager -build hello.akl
```

Output:
```
Hello, Askal!
```

---

## Language Basics

### Variables

Declare a variable with an explicit type:

```akl
var int x = 10;
var float pi = 3.14;
var str name = "Askal";
var bool flag = true;
```

Variables must be declared before use. Uninitialized variables are allowed (they default to zero/null).

### Arithmetic

Standard arithmetic operators are supported: `+`, `-`, `*`, `/`.  
Parentheses `( )` can be used to change evaluation order.

```akl
var int a = 5 + 3 * 2;   // 11
var int b = (5 + 3) * 2; // 16
```

### Output

Use `print` (no newline) or `println` (with newline). Multiple arguments are separated by commas.

```akl
print < "Hello, " + "world!";
println < "x = ", x;
```

### Functions (under development)

Declare a function using the `fun` keyword:

```akl
fun greet < name {
    print < "Hello, ", name;
}
```

Call a function by its name:

```akl
greet < "Askal";
```

---

## Architecture

```
┌─────────────────────────────────────┐
│        Source code (.akl)           │
└───────────────┬─────────────────────┘
                ▼
┌─────────────────────────────────────┐
│          Compiler                   │
│  • Tokenization                     │
│  • Parsing                          │
│  • Bytecode generation              │
└───────────────┬─────────────────────┘
                ▼
┌─────────────────────────────────────┐
│         Bytecode (.aklp)            │
│  [VARIABLES] [FUNCTIONS] [DEFAULT] │
└───────────────┬─────────────────────┘
                ▼
┌─────────────────────────────────────┐
│       Virtual Machine               │
│  • Loader                           │
│  • Stack‑based executor             │
│  • MMD memory manager               │
└─────────────────────────────────────┘
```

---

## Tools

### Disassembler (`ast.py`)

To inspect the structure of a `.aklp` file:

```bash
python tools/ast.py program.aklp
```

Example output:

```
=== ASKAL BYTECODE DUMP ===
[POD_SEC_VARIABLES]
  VAR id=1 type=INT name="x" weight=4
[END_SEC]
[SEC_DEFAULT]
  INT 10
  STORE id=1
  CALL_VAR id=1
  PRINTLN
  HALT
[END_DEFAULT]
```

---

## Documentation

Detailed documentation is available in the `docs/` folder:

- [Syntax](docs/syntax.md) – full language syntax reference.
- [Compiler](docs/compiler.md) – how the compiler works.
- [Runtime](docs/runtime.md) – virtual machine internals.
- [Logger](docs/logger.md) – logging system.
- [MMD](docs/MMD.md) – memory manager.

---

## License

Copyright 2026 Daniil Tarasov (forXsis)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

---

## Contact

- **Author**: forXsis
- **GitHub**: [github.com/forXsis](https://github.com/forXsis)

---

**If you like this project, please give it a star!**