# Askal

> A fast, lightweight and educational programming language built from scratch in C++.

Askal is a statically typed programming language with its own compiler, bytecode format (`.aklp`) and lightweight virtual machine.

The project was created to study compiler construction, virtual machines and language design while remaining practical enough for writing real programs.

---

## Features

- Written entirely in modern C++
- Custom compiler
- Custom bytecode format (`.aklp`)
- Lightweight Virtual Machine
- Static typing
- Type inference (`var`)
- Cross-platform architecture
- Fast startup
- Easy-to-read syntax

---

## Hello World

```akl
var str name = "World";

println < "Hello, " < name;
```

---

## Project Structure

```
askal/
│
├── compiler/      # Compiler source
├── vm/            # Virtual Machine
├── include/       # Headers
├── std/           # Standard Library (future)
├── examples/      # Example programs
├── docs/          # Documentation
└── tests/         # Tests
```

---

## Building

### Requirements

- CMake
- C++20 compiler

Build:

```bash
git clone https://github.com/forXiss/askal.git

cd askal

mkdir build
cd build

cmake ..
cmake --build .
```

---

## Roadmap

- [x] Variables
- [x] Arithmetic
- [x] Bytecode generation
- [x] Virtual Machine
- [ ] Functions
- [ ] Loops
- [ ] Arrays
- [ ] Modules
- [ ] Package manager
- [ ] Standard Library
- [ ] C/C++ bindings
- [ ] Native executable packaging

---

## Philosophy

Askal is designed to be:

- simple enough for beginners;
- fast enough for everyday scripting;
- educational enough to understand how programming languages actually work.

---

## Contributing

Contributions, ideas and bug reports are welcome.

If you'd like to improve Askal, feel free to open an Issue or Pull Request.

---

## License

Licensed under the Apache License 2.0.
