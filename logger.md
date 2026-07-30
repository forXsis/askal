# Logger — Askal Logging System

The Askal Logger provides a unified, flexible logging interface for both the compiler and the runtime. It supports multiple log levels, colored console output, and both basic and full (debug) logging modes.

---

## 🎯 Purpose

The logger is designed to:
- Provide consistent logging across all components
- Allow detailed debugging without cluttering normal output
- Support color‑coded messages for better readability
- Enable progressive logging detail (from none to full DEBUG)

---

## 📊 Log Levels

| Level     | Purpose                                                      |
|-----------|--------------------------------------------------------------|
| `DEBUG`   | Detailed diagnostic information (only with full logging)     |
| `INFO`    | General progress and status messages                         |
| `WARNING` | Non‑critical issues that might cause problems               |
| `ERROR`   | Critical failures that halt execution                        |

---

## ⚙️ Configuration

The logger uses static flags for global control:

### Flags

```cpp
static bool log_enabled;       // Enable/disable all logging
static bool full_log_enabled;  // Enable DEBUG level (implies log_enabled)
static bool colors_enabled;    // Enable ANSI color codes
```

### Methods

```cpp
// Enable basic logging (INFO, WARNING, ERROR)
Logger::enable_logging(true);

// Enable full logging (includes DEBUG)
Logger::enable_full_logging(true);

// Enable colored output (default: true)
Logger::enable_colors(true);
```

---

## 📝 Usage

### Basic Logging

```cpp
Logger::info("Compilation started", input_file);
Logger::warning("Unused variable", var_name);
Logger::error("Cannot open file", filename);
```

### Debug Logging

```cpp
Logger::debug("Token", "keyword=" + word);
Logger::debug("Emit byte", "0x" + std::to_string(b));
```

### With Details

```cpp
Logger::info("Variable declaration", 
             "id=" + std::to_string(id) + ", name=" + name);
```

---

## 🎨 Output Format

### Basic Mode (`full_log_enabled = false`)

```
[INFO] Loading bytecode file → program.aklp
[WARN] Unused variable → x
[ERROR] Compilation error → Undefined variable 'y'
```

### Full Mode (`full_log_enabled = true`)

Colored output (if enabled):

```
DEBUG read_byte → 0x164 (ip=0)
INFO Reading VARIABLES section → ip=1
DEBUG read_int → value=1 (ip=2)
WARN END_SEC marker → skipping
ERROR Runtime error → Unknown section marker: 0x164
```

### Color Scheme

| Level     | Color (ANSI) |
|-----------|--------------|
| `DEBUG`   | Gray         |
| `INFO`    | Cyan         |
| `WARNING` | Yellow       |
| `ERROR`   | Red          |

Colors are disabled automatically on non‑terminal output.

---

## 🛠️ Helper Functions

### Stack Dump

The logger can display the current stack contents (for VM debugging):

```cpp
std::string stack_dump(const std::vector<T>& stack);
```

**Example output:**
```
[10, 3.14, "Hello", true]
```

### Token String

Converts a token to a human‑readable string:

```cpp
std::string token_str(const std::string& type, const std::string& value);
```

**Examples:**
- `token_str("IDENTIFIER", "x")` → `"x (type=IDENTIFIER)"`
- `token_str("END_OF_FILE", "")` → `"EOF"`

---

## 🔌 Integration

### In the Compiler

```cpp
// Initialization
static void init_logger() {
    Logger::enable_logging(Compiler::log_out);
    Logger::enable_full_logging(Compiler::flog_out);
    Logger::enable_colors(true);
}

// Usage
Logger::debug("Parsing statement", 
              "current token: " + Logger::token_str(...));
Logger::info("Symbol added", "name=" + sym.name);
```

### In the Runtime

```cpp
// Initialization
static void init_logger() {
    Logger::enable_logging(Runtime::log_out);
    Logger::enable_full_logging(Runtime::flog_out);
    Logger::enable_colors(true);
}

// Usage
Logger::debug("read_byte", "0x" + std::to_string(val));
Logger::error("Unexpected end of bytecode", 
              "ip=" + std::to_string(ip));
```

---

## 🧪 Example Usage

### Compiler with Full Logging

```bash
askal-manager -compile program.akl -fl
```

**Output:**
```
INFO Compilation started → program.akl
INFO Tokenizing source → length=42
DEBUG Token → keyword=var
DEBUG Token → identifier=int
DEBUG Token → identifier=x
...
INFO Compilation successful → program.aklp (56 bytes)
```

### Runtime with Basic Logging

```bash
askal-manager -run program.aklp -l
```

**Output:**
```
Running: program.aklp
--- Program output ---
Hello, World!
--- End of output ---
Program executed successfully.
```

---

## ⚠️ Notes

- `DEBUG` messages are **never shown** unless `full_log_enabled` is set to `true`.
- Enabling `full_log_enabled` automatically implies `log_enabled = true`.
- The logger is **thread‑safe** only for sequential execution (single‑threaded).
- Color codes use ANSI escape sequences; they work on most terminals (Windows 10+, Linux, macOS).