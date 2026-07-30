# MMD — Memory Manager & Defragmenter

The MMD (Memory Manager & Defragmenter) is a dynamic memory management system designed for the Askal virtual machine. It provides a byte pool where variables are stored, supports allocation and deallocation, and includes compaction to reduce fragmentation.

---

## 🎯 Purpose

MMD is responsible for:
- Allocating memory blocks for variables
- Storing variable data in a contiguous byte pool
- Tracking variable metadata (id, type, name, size, offset)
- Freeing memory when variables are deleted
- Defragmenting the pool to reduce fragmentation

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    MMD (Memory Pool)                    │
│  ┌───────────────────────────────────────────────────┐  │
│  │  byte 0  │   ...   │  byte 15  │  byte 16  │ ...  │  │
│  └───────────────────────────────────────────────────┘  │
│         ↑                           ↑                   │
│         │                           │                   │
│    Variable A                 Variable B                │
│    (offset=0, size=4)        (offset=16, size=64)       │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Blocks Table                                     │  │
│  │  [offset=0, size=4, used=true]                    │  │
│  │  [offset=4, size=12, used=false]  ← free space    │  │
│  │  [offset=16, size=64, used=true]                  │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Variable Table (id → VarInfo)                    │  │
│  │  1 → {name="x", type=INT, offset=0, size=4}       │  │
│  │  2 → {name="s", type=STRING, offset=16, size=64}  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 📦 Components

### 1. Memory Pool

The pool is a `std::vector<unsigned char>` that stores raw byte data:

```cpp
std::vector<unsigned char> pool_;
size_t pool_size_;  // Current capacity
size_t used_;       // Amount of memory used
```

All variables are stored contiguously in this pool at their assigned offsets.

### 2. Blocks Table

The blocks table (`std::vector<Block>`) tracks all allocated and free memory regions:

```cpp
struct Block {
    size_t offset;   // Start position in the pool
    size_t size;     // Block size in bytes
    bool used;       // true if allocated, false if free
};
```

**Block states:**
- **Used** – occupied by a variable
- **Free** – available for future allocation

### 3. Variable Table

The variable table (`std::unordered_map<uint32_t, VarInfo>`) maps variable IDs to their metadata:

```cpp
struct VarInfo {
    uint32_t id;
    std::string name;
    DataType type;
    size_t offset;   // Position in the pool
    size_t size;     // Block size (weight)
    bool is_dynamic; // Can the variable grow?
    bool active;     // Is this variable still alive?
};
```

---

## ⚙️ Core Operations

### 1. Creating a Variable

```cpp
void create_variable(uint32_t id, DataType type, size_t weight,
                     const std::string& name = "", bool is_dynamic = false);
```

**Process:**
1. Validate that the id doesn't already exist
2. Allocate a block of `weight` bytes using `allocate_block()`
3. Store variable metadata in the variable table
4. Variable data is initially zeroed

**Example:**
```cpp
// Create an integer variable with id=1, weight=4
mmd.create_variable(1, DataType::INT, 4, "x", false);
```

### 2. Writing Data

```cpp
void set_variable(uint32_t id, const void* data, size_t size);
```

**Process:**
1. Find the variable by id
2. If the variable is dynamic and needs to grow, reallocate
3. Copy `size` bytes from `data` into the pool at the variable's offset
4. Update `used_` if necessary

**Example:**
```cpp
int value = 42;
mmd.set_variable(1, &value, sizeof(value));
```

### 3. Reading Data

```cpp
size_t get_variable(uint32_t id, void* buffer, size_t size) const;
```

**Process:**
1. Find the variable by id
2. Read up to `size` bytes from the pool at the variable's offset
3. Return the actual number of bytes read (min(size, variable.size))

**Example:**
```cpp
int value;
size_t read = mmd.get_variable(1, &value, sizeof(value));
```

### 4. Freeing a Variable

```cpp
void free_variable(uint32_t id);
```

**Process:**
1. Find the variable by id
2. Mark its block as free in the blocks table
3. Merge adjacent free blocks
4. Remove the variable from the variable table

---

## 🗜️ Compaction (Defragmentation)

Compaction is the process of moving all used blocks to the beginning of the pool, eliminating gaps of free space.

```cpp
size_t compact();
```

**Process:**
1. Sort blocks by offset
2. Iterate through used blocks
3. If a block is not at the expected position, move it
4. Update variable offsets in the variable table
5. Remove unused blocks
6. Return the number of bytes freed

**Before Compaction:**
```
[Used 4] [Free 12] [Used 64] [Free 8] [Used 16]
```

**After Compaction:**
```
[Used 4] [Used 64] [Used 16] [Free 84]
```

**Benefits:**
- Reduces fragmentation
- Allows larger allocations
- Improves cache locality

---

## 📈 Growth Strategies

When the pool is full, it can grow according to one of three strategies:

### 1. DOUBLE

Doubles the pool size when expansion is needed.

```cpp
// Example: 4096 → 8192 → 16384 → ...
```

**Use case:** When memory usage is unpredictable and performance is critical.

### 2. INCREMENTAL

Increases the pool size by a fixed increment.

```cpp
// Example: 4096 → 8192 → 12288 → 16384 → ...
```

**Use case:** When growth patterns are predictable.

### 3. EXACT

Increases the pool size exactly to the requested size.

```cpp
// Example: 4096 → 4100 → 4104 → ...
```

**Use case:** When memory is limited and exact control is needed.

### Configuration

```cpp
MMD mmd(
    4096,                           // initial_size
    MMD::GrowthStrategy::DOUBLE,    // strategy
    4096                            // increment (for INCREMENTAL)
);
```

---

## 🔍 Metadata Access

MMD provides several methods for inspecting variable metadata:

```cpp
// Get the type of a variable
DataType get_type(uint32_t id) const;

// Get the name of a variable
std::string get_name(uint32_t id) const;

// Check if a variable exists
bool has_variable(uint32_t id) const;

// Get the size (weight) of a variable
size_t get_size(uint32_t id) const;

// Get the offset of a variable in the pool
size_t get_offset(uint32_t id) const;

// Get the total number of variables
size_t get_variable_count() const;
```

---

## 🧪 Example Usage

```cpp
// Create MMD with 4KB initial pool
MMD mmd(4096, MMD::GrowthStrategy::DOUBLE);

// Create variables
mmd.create_variable(1, DataType::INT, 4, "x", false);
mmd.create_variable(2, DataType::STRING, 64, "message", false);

// Store values
int x = 42;
mmd.set_variable(1, &x, sizeof(x));

std::string msg = "Hello, World!";
mmd.set_variable(2, msg.c_str(), msg.size() + 1);

// Read values
int read_x;
mmd.get_variable(1, &read_x, sizeof(read_x));

char buffer[256];
size_t read = mmd.get_variable(2, buffer, sizeof(buffer) - 1);
buffer[read] = '\0';

// Compaction
size_t freed = mmd.compact();
std::cout << "Freed " << freed << " bytes\n";

// Statistics
std::cout << mmd.get_stats() << std::endl;
```

---

## 🛡️ Error Handling

| Error                         | Cause                                   |
|-------------------------------|-----------------------------------------|
| `Variable with id X not found` | Accessing a non‑existent variable       |
| `Variable with id X already exists` | Creating a variable with duplicate id |
| `Data size exceeds variable capacity` | Writing more data than the variable's weight |
| `Offset exceeds pool size`     | Corrupted memory or incorrect offset    |
| `Block not found at offset X`  | Double‑free or corrupted blocks table   |

---

## 📊 Debugging Tools

### Dump Memory

```cpp
std::string dump_memory(size_t bytes_count = 128) const;
```

Displays the pool contents in hex and ASCII format:

```
Memory dump (128 bytes):
  Pool size: 4096 bytes
  Used: 84 bytes

0000: 2A 00 00 00 48 65 6C 6C 6F 00 00 00 00 00 00 00  *...Hello.......
0010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
...
```

### Statistics

```cpp
std::string get_stats() const;
```

Displays detailed memory statistics:

```
=== Memory Statistics ===
  Pool size:     4096 bytes
  Used:          84 bytes
  Free:          4012 bytes
  Usage:         2%
  Live blocks:   2
  Total blocks:  3
  Variables:     2
  Fragmentation: 0 bytes
```

### List Variables

```cpp
std::string list_variables() const;
```

Lists all active variables:

```
Variables (id, name, type, offset, size, dynamic):
  1 : x : 209 : 0 : 4 : fixed
  2 : message : 208 : 16 : 64 : fixed
```