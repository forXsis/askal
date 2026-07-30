# Syntax — Askal Language Syntax Reference

This document describes the complete syntax of the Askal programming language. Askal is designed to be minimal, readable, and easy to learn.

---

## 📝 Program Structure

An Askal program consists of a sequence of statements. Statements are executed in order from top to bottom.

**Example:**
```akl
var int x = 10;
var int y = 20;
var int sum = x + y;
print < sum;
```

---

## 📌 Comments

Askal supports single-line comments only.

```akl
// This is a comment
var int x = 10;  // Inline comment
```

---

## 📦 Variables

### Declaration

Variables must be declared before use. The syntax is:

```akl
var <type> <name>;
var <type> <name> = <expression>;
```

**Examples:**
```akl
var int x;              // Declaration without initialization
var int y = 10;         // Declaration with initialization
var float pi = 3.14;
var str name = "Askal";
var bool flag = true;
```

### Data Types

| Type    | Description                    | Size   | Example           |
|---------|--------------------------------|--------|-------------------|
| `int`   | Signed integer                 | 4 bytes| `42`, `-10`       |
| `float` | Floating-point number          | 4 bytes| `3.14`, `-0.5`    |
| `str`   | String (fixed 64-byte buffer)  | 64 bytes| `"Hello"`        |
| `bool`  | Boolean value                  | 1 byte | `true`, `false`   |

### Assignment

Variables can be reassigned using the `=` operator:

```akl
var int x = 10;
x = 20;           // Reassign
x = x + 5;        // x becomes 25
```

**Note:** The assigned value must match the variable's type.

---

## ➗ Arithmetic Operations

Askal supports four basic arithmetic operators:

| Operator | Operation      | Supported Types       |
|----------|----------------|-----------------------|
| `+`      | Addition       | `int`, `float`, `str` |
| `-`      | Subtraction    | `int`, `float`        |
| `*`      | Multiplication | `int`, `float`        |
| `/`      | Division       | `int`, `float`        |

### Operator Precedence

1. Parentheses `( )` – highest precedence
2. Multiplication `*` and Division `/`
3. Addition `+` and Subtraction `-` – lowest precedence

### Examples

```akl
var int a = 5 + 3 * 2;     // 11 (3*2=6, 5+6=11)
var int b = (5 + 3) * 2;   // 16 (5+3=8, 8*2=16)
var int c = 10 / 2 - 1;    // 4  (10/2=5, 5-1=4)
```

### String Concatenation

The `+` operator concatenates strings:

```akl
var str s1 = "Hello";
var str s2 = " World";
var str s3 = s1 + s2;      // "Hello World"
```

---

## 🖨️ Output

### print Statement

Prints values to the console **without** adding a newline.

```akl
print < expression1 + expression2;
```

**Examples:**
```akl
print < "Hello";
print < "x = " + x;
print < "The result is " + 42;
```

### println Statement

Prints values to the console **with** a newline at the end.

```akl
println < expression1 + expression2;
```

**Examples:**
```akl
println < "Hello, World!";
println < "x = " + x;
println < "Sum: " + a + b;
```

### Multiple Arguments

Both `print` and `println` accept multiple arguments separated by commas:

```akl
print < "Value: " + x + ", " + "Result: " + y;
```

---

## 🔧 Functions

### Declaration

Functions are declared using the `fun` keyword. A function body is enclosed in curly braces `{ }`.

```akl
fun <name> {
    // function body
}
```

**Example:**
```akl
fun greet {
    print < "Hello, World!";
}
```

### Function Calls

Functions are called by their name followed by a semicolon `;`.

```akl
greet;   // Calls the greet function
```

### Complete Example

```akl
fun greet {
    print < "Hello, World!";
}

fun main {
    greet;  // Call the function
}
```

---

## 🔍 Lexical Rules

### Identifiers

- Must start with a letter (a-z, A-Z) or underscore `_`
- Can contain letters, digits, and underscores
- Are case-sensitive

**Valid:**
```
x, _temp, var1, helloWorld, my_var
```

**Invalid:**
```
1var, my-var, var, print
```

### Keywords (Reserved)

These words cannot be used as identifiers:

```
var, print, println, fun, true, false, null
int, float, str, bool
```

### Literals

**Integers:**
```akl
42, -10, 0, 255
```

**Floats:**
```akl
3.14, -0.5, 2.0, .5   // .5 is not allowed (must start with digit)
```

**Strings:**
```akl
"Hello", "x = ", "Line\nBreak"
```
Supported escape sequences:
- `\n` – newline
- `\t` – tab
- `\\` – backslash
- `\"` – double quote

**Booleans:**
```akl
true, false
```

**Null:**
```akl
null
```
`null` is treated as `0` (integer) or empty value.

---

## 📚 Complete Examples

### Hello World

```akl
println < "Hello, World!";
```

### Variables and Arithmetic

```akl
var int a = 10;
var int b = 20;
var int sum = a + b;
var int product = a * b;

println < "Sum: " + um;
println < "Product: " + product;
```

### String Concatenation

```akl
var str firstName = "John";
var str lastName = "Doe";
var str fullName = firstName + " " + lastName;
println < "Full name: " + fullName;
```

### Function with Output

```akl
fun greet < name {
    print < "Hello, "+ name;
}

greet < "Askal";
println < "!";
```

---

## ⚠️ Common Errors

| Error                    | Cause                           | Fix                      |
|--------------------------|---------------------------------|--------------------------|
| `Undefined variable`     | Using a variable before declaration | Declare the variable first |
| `Variable already defined` | Duplicate variable name        | Use a different name     |
| `Unexpected identifier`  | Missing operator or semicolon  | Check syntax             |
| `Reserved name`          | Using a keyword as identifier  | Choose a different name  |
| `Stack empty on PRINT`   | Nothing to print               | Ensure values are on stack |