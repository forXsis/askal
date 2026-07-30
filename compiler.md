# Compiler — Askal Language Compiler

The Askal compiler transforms source code (`.akl`) into platform‑independent bytecode (`.aklp`). The compilation pipeline is split into three clear phases: tokenization, parsing, and bytecode generation.

---

## 🔧 Compilation Pipeline

```
Source (.akl)  →  Tokenizer  →  Parser  →  Bytecode Generator  →  .aklp file
```

---

## 1️⃣ Tokenization

The tokenizer (`Compiler::tokenize()`) reads the source text character by character and produces a stream of tokens.

### Recognized Tokens

- **Keywords:** `var`, `print`, `println`, `fun`, `true`, `false`, `null`
- **Identifiers:** variable and function names
- **Literals:** integers, floats, strings (with `\n`, `\t`, `\\`, `\"` escapes)
- **Operators:** `+`, `-`, `*`, `/`, `<`, `>`, `<=`, `>=`, `==`, `!=`, `=`
- **Punctuation:** `;`, `,`, `(`, `)`, `{`, `}`

### Error Handling

If an unexpected character is encountered, the tokenizer throws an error with line and column information:

```
Tokenize error at 5:12 - Unexpected character '^'
```

---

## 2️⃣ Parsing

The parser (`Compiler::parse_program()`) builds the program structure using a recursive‑descent approach.

### Grammar (Simplified)

```
program      ::= statement*
statement    ::= var_decl | print_stmt | fun_decl | assignment | call
var_decl     ::= "var" type identifier ["=" expression] ";"
type         ::= "int" | "float" | "str" | "bool"
print_stmt   ::= ("print" | "println") "<" expression ("," expression)* ";"
fun_decl     ::= "fun" identifier "{" statement* "}"
assignment   ::= identifier "=" expression ";"
call         ::= identifier ";"
expression   ::= add_sub (("==" | "!=" | "<" | ">" | "<=" | ">=") add_sub)*
add_sub      ::= mul_div (("+" | "-") mul_div)*
mul_div      ::= unary (("*" | "/") unary)*
unary        ::= "-" unary | primary
primary      ::= NUMBER | FLOAT | STRING | "true" | "false" | "null" | identifier | "(" expression ")"
```

### Scope Management

The compiler maintains a symbol table (`Symbol`) for variables and functions. Each symbol stores:

- `id` – unique numeric identifier
- `name` – source name
- `type` – data type (`int`, `float`, `str`, `bool`)
- `weight` – size in bytes (4 for int/float, 1 for bool, 64 for string)

Function symbols are also tracked to allow forward references (calls before declaration).

---

## 3️⃣ Bytecode Generation

The generator emits a binary `.aklp` file. The file is organized into three sections, each preceded by a marker.

### Bytecode Structure

| Offset | Marker   | Section         | Description |
|--------|----------|-----------------|-------------|
| 0      | `0xA4`   | POD_SEC_VARIABLES | Variable definitions |
| ...    | `0xA3`   | END_SEC         | End of variables |
| ...    | `0xA5`   | POD_SEC_FUNCTIONS | Function definitions |
| ...    | `0xA3`   | END_SEC         | End of functions |
| ...    | `0xA6`   | SEC_DEFAULT     | Executable code |
| ...    | `0x00`   | HALT            | Program termination |

### Variable Record Format

Each variable is encoded as:

```
[0xB0]           → VAR marker
[id]             → 4 bytes, unique identifier
[0xA0]           → DATA_ID marker
[type]           → 1 byte (0xD0..0xD5)
[name]           → string (4‑byte length + chars)
[0xA2]           → NAME marker
[weight]         → 4 bytes, size in bytes
[0xA1]           → WEIGHT marker
```

### Function Record Format

Each function is encoded as:

```
[0xB1]           → FUN marker
[id]             → 4 bytes, unique identifier
[0xA0]           → DATA_ID marker
[name]           → string (4‑byte length + chars)
[0xA2]           → NAME marker
[body bytes]     → instructions (including implicit RETURN)
[0xAE]           → END_FUN marker
```

### Instruction Encoding

Executable code in the `DEFAULT` section is a stream of opcodes and data markers. Each instruction either:

- Has no operands (`HALT`, `PRINT`, `PRINTLN`, `ADD`, `SUB`, `MUL`, `DIV`, `RETURN`)
- Has a 4‑byte operand (`STORE`, `CALL_VAR`, `CALL_FUN`)
- Is preceded by a data‑type marker (`INT`, `FLOAT`, `STRING`, `BOOL_TRUE`, `BOOL_FALSE`, `NULLABLE`)

### Example: `var int x = 5;`

**Variables section:**
```
B0 01 00 00 00 A0 D1 78 00 00 00 A2 04 00 00 00 A1
```
- `B0` – VAR marker
- `01 00 00 00` – id = 1
- `A0` – DATA_ID
- `D1` – type INT
- `78 00 00 00` – name "x" (length 1 + 'x')
- `A2` – NAME
- `04 00 00 00` – weight = 4
- `A1` – WEIGHT

**Default section:**
```
D1 0A 00 00 00 03 01 00 00 00
```
- `D1 0A 00 00 00` – INT literal 10
- `03` – STORE opcode
- `01 00 00 00` – variable id 1

---

## 🔍 Special Cases

### Implicit Return

The compiler automatically adds a `RETURN` (0x04) instruction at the end of every function body. This ensures functions always return cleanly even if no explicit `return` is written.

### Reserved Names

The following names are reserved and cannot be used as identifiers:

```
print, println, var, fun, int, float, str, bool, true, false, null
```

### Type Weights

| Type   | Weight (bytes) |
|--------|----------------|
| `int`  | 4              |
| `float`| 4              |
| `bool` | 1              |
| `str`  | 64 (fixed)     |

---

## 📁 File Output

The compiler writes the final bytecode to a file with the same base name as the input but with the `.aklp` extension.

**Example:**
```
input:  program.akl
output: program.aklp
```

---

## ⚠️ Error Reporting

All compilation errors are reported with:
- A descriptive message
- Line and column number (where available)

Example:
```
Compilation error: Undefined variable 'y'
```