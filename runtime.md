# Runtime — Askal Virtual Machine

The Askal Runtime is a stack‑based virtual machine that loads and executes `.aklp` bytecode files. It manages memory, executes instructions, and provides I/O operations.

---

## 🏗️ Architecture Overview

```
┌──────────────────────────────────────────────┐
│              .aklp Bytecode File             │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│                Loader                        │
│  • Reads sections: VARIABLES, FUNCTIONS,     │
│    DEFAULT                                   │
│  • Creates variables in MMD                  │
│  • Stores functions for later calls          │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│              Executor                        │
│  • Instruction fetch/decoding                │
│  • Stack operations                          │
│  • Arithmetic computations                   │
│  • Function calls (with call stack)          │
└──────────────────────────────────────────────┘
```

---

## 📦 Components

### 1. Loader

The loader (`Runtime::load()`) parses the `.aklp` file:

- Reads the file into a byte buffer
- Scans for section markers (`0xA4`, `0xA5`, `0xA6`)
- Extracts variable definitions and creates them in the MMD memory pool
- Extracts function bytecode and stores it in the function table
- Copies the `DEFAULT` section into the execution buffer

### 2. Executor

The executor (`Runtime::execute_code()`) runs the bytecode:

- Maintains an instruction pointer (`pc`) per execution context
- Fetches one byte at a time (opcode or data marker)
- Performs the corresponding operation
- Uses a stack for intermediate values

### 3. Memory Manager (MMD)

The MMD (Memory Manager & Defragmenter) provides a dynamic byte pool:

- Variables are stored by `id` (not by name)
- Each variable has: `offset` (in pool), `size` (weight), `type`
- Supports fixed‑size allocation (strings are fixed to 64 bytes)

### 4. Stack

The stack holds values of four types:

```cpp
std::variant<int32_t, float, std::string, bool>
```

Operations push and pop values, and arithmetic instructions take operands from the stack.

---

## 🧩 Instruction Set

### Control Flow

| Opcode | Name   | Description                    |
|--------|--------|--------------------------------|
| `0x00` | HALT   | Stops program execution        |
| `0x04` | RETURN | Returns from a function call   |

### I/O

| Opcode | Name    | Description                                  |
|--------|---------|----------------------------------------------|
| `0x01` | PRINT   | Pops stack top and prints it (no newline)    |
| `0x02` | PRINTLN | Pops stack top and prints it (with newline)  |

### Memory Operations

| Opcode | Name     | Description                                    |
|--------|----------|------------------------------------------------|
| `0x03` | STORE    | Pops value from stack, stores in variable by id |
| `0xC0` | CALL_VAR | Loads variable by id and pushes it onto stack  |

### Arithmetic

| Opcode | Name | Description                          |
|--------|------|--------------------------------------|
| `0x20` | ADD  | Pops two values, pushes sum          |
| `0x21` | SUB  | Pops two values, pushes difference   |
| `0x22` | MUL  | Pops two values, pushes product      |
| `0x23` | DIV  | Pops two values, pushes quotient     |

**Note:** All arithmetic instructions pop two operands from the stack (right operand first, left operand second) and push the result back.

### Function Calls

| Opcode | Name     | Description                              |
|--------|----------|------------------------------------------|
| `0xC1` | CALL_FUN | Calls function by id, switches context   |

---

## 📊 Data Types (Markers)

Before each literal value, the compiler emits a type marker:

| Marker | Type        | Followed by                     |
|--------|-------------|---------------------------------|
| `0xD0` | STRING      | 4‑byte length + UTF‑8 characters |
| `0xD1` | INT         | 4‑byte signed integer           |
| `0xD2` | BOOL_TRUE   | 1‑byte (always `0x01`)          |
| `0xD3` | BOOL_FALSE  | 1‑byte (always `0x00`)          |
| `0xD4` | FLOAT       | 4‑byte IEEE 754 float           |
| `0xD5` | NULLABLE    | No additional data (value = 0)  |

---

## 🔄 Execution Flow

### 1. Main Program

1. The VM starts executing at the beginning of the `DEFAULT` section.
2. Instructions are processed sequentially.
3. When `HALT` is encountered, the program stops.

### 2. Function Calls

1. `CALL_FUN` reads a 4‑byte function id.
2. The VM saves the current `pc` (return address) onto the call stack.
3. It switches to the function's bytecode and resets its `pc` to 0.
4. The function executes until `RETURN` is reached.
5. The VM restores the saved `pc` and continues after the `CALL_FUN`.

**Important:** The VM is currently single‑threaded and uses a simple `std::vector<size_t>` for the call stack (no recursion limit enforced).

---

## 🧪 Example Execution

**Source code:**
```akl
var int x = 10;
print < x;
```

**Bytecode (DEFAULT section):**
```
D1 0A 00 00 00    → INT 10 (push 10)
03 01 00 00 00    → STORE id=1 (pop into x)
C0 01 00 00 00    → CALL_VAR id=1 (push x)
01                → PRINT (pop and print)
00                → HALT
```

**Execution trace:**
1. `INT 10` – push `10` onto stack → `[10]`
2. `STORE id=1` – pop `10`, store in variable with id=1 → `[]`
3. `CALL_VAR id=1` – load variable 1 (`10`), push onto stack → `[10]`
4. `PRINT` – pop `10`, output `10` → `[]`
5. `HALT` – stop program

---

## 🛡️ Error Handling

The VM validates every operation at runtime:

| Error                        | Cause                                   |
|------------------------------|-----------------------------------------|
| `Unexpected end of bytecode` | File is truncated or invalid           |
| `Variable not found`         | Accessing a variable that doesn't exist |
| `Stack empty`                | Popping from an empty stack            |
| `Stack underflow`            | Arithmetic without enough operands     |
| `Division by zero`           | DIV with divisor 0                     |
| `Unknown byte`               | Invalid opcode or data marker          |