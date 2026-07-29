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
- Fast startup
- Easy-to-read syntax

---

## Hello World

```akl
var str name = "World";

println < "Hello, " + name;
```

---

## Project Structure

```
askal/
│
├── main/ # current version
│  │
│  ├── Release x64/ # a ready-made and compiled program (.exe)
│  ├── src/ # program sources including codes
│  ├── README.md # brief description of the program version
│  ├── CHANGELOG.md # a brief description of the changes in the version
├── version-XX.XX # other versions of the program
```

---

## Building

At the moment Askal is developed with **Visual Studio 2022** using the **C++17** standard.

To build the project:

1. Open the source code in Visual Studio 2022.
2. Create a C++17 Console Application.
3. Add all files from the `src/` directory.
4. Build the project (Ctrl + Shift + B).

> CMake support is planned for future releases.

---

## Roadmap

- [x] Variables
- [x] Arithmetic (Base: +,-,*,/)
- [x] Bytecode generation (Compiler)
- [x] Virtual Machine (Runtime)
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
