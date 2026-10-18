# Виртуальная машина Askal

Стековая виртуальная машина. Загружает `.aklp`, исполняет байткод.
Написана на C++20. Единственный исполняемый файл: `askal_vm.exe`
(Windows) или `askal_vm` (Linux).

## CLI

    askal_vm --r file.aklp
    askal_vm --r file.aklp -v
    askal_vm --r file.aklp --trace
    askal_vm --r file.aklp --dump
    askal_vm --call-func <id> --aklp <path>
    askal_vm --version

| Флаг | Значение |
|---|---|
| `--r` | запустить байткод |
| `-v` | verbose: globals, functions, classes, размер main |
| `--trace` | трассировка опкодов и операций стека |
| `--dump` | дамп глобальных переменных после исполнения |
| `--call-func <id>` | вызвать функцию по id (внутренний режим для thread) |
| `--aklp <path>` | путь к файлу при `--call-func` |
| `--version` | версия VM |

## Формат `.aklp`

Файл состоит из секций. Каждая секция начинается с маркера `u8`.

| Маркер | Секция | Содержимое |
|---|---|---|
| `0x24` | `SEC_LIBS` | `[u32 count]` затем для каждой: `[u32 len][name]` |
| `0x20` | `SEC_VARS` | последовательность `DECL_VAR` |
| `0x21` | `SEC_FUNCS` | последовательность `DECL_FUN` |
| `0x22` | `SEC_CLASSES` | последовательность `DECL_CLASS` |
| `0x23` | `SEC_MAIN` | байткод до конца файла |

Порядок секций важен. `SEC_LIBS` обязательна первой, если есть.

### `DECL_VAR`

    [u8 0x30 DECL_VAR]
    [u32 id]
    [u8  type]
    [u32 name_len][name]
    [default value]

Значение по умолчанию для числовых типов: нули нужного размера.
Для `str`, `bin`, `hex`, `auto`, `object`, `list`, `dict`, `array`:
`[u32 0]` (пустая строка).

### `DECL_FUN`

    [u8 0x31 DECL_FUN]
    [u32 id]
    [u32 name_len][name]
    [u8  return_type]
    [u8  is_method]
    [u8  argc]
    для каждого аргумента:
        [u32 arg_id]
        [u8  arg_type]
        [u32 arg_name_len][arg_name]
    [u32 local_count]
    для каждой локальной:
        [u32 local_id]
        [u8  local_type]
        [u32 local_name_len][local_name]
    [u8 0x02 START_DATA]
    [тело: байткод]
    [u8 0x03 END_DATA]

### `DECL_CLASS`

    [u8 0x32 DECL_CLASS]
    [u32 class_id]
    [u32 name_len][name]
    [u32 field_count]
    для каждого поля: [u32 var_id]
    [u32 method_count]
    для каждого метода: [u32 fun_id]

## Опкоды

Один опкод занимает один байт. Диапазоны:

| Диапазон | Назначение |
|---|---|
| `0x00-0x0F` | служебные: `HALT`, `NOP`, `START_DATA`, `END_DATA` |
| `0x10-0x1F` | типы данных |
| `0x20-0x2F` | секции файла |
| `0x30-0x3F` | объявления |
| `0x40-0x4F` | стек |
| `0x50-0x5F` | переменные |
| `0x60-0x6F` | функции |
| `0x70-0x7F` | классы |
| `0x80-0x8F` | арифметика |
| `0x90-0x9F` | сравнения |
| `0xA0-0xAF` | логика |
| `0xB0-0xBF` | управление |
| `0xC0-0xCF` | коллекции |
| `0xD0-0xD3` | thread |

Полный список в `vm_opcodes.h`.

### Форматы инструкций

| Опкод | Размер | Формат |
|---|---|---|
| `PUSH` | переменный | `[0x40][u8 type][payload]` |
| `LOAD_VAR` | 5 | `[0x50][u32 id]` |
| `STORE_VAR` | 5 | `[0x51][u32 id]` |
| `CALL` | 6 | `[0x60][u32 fun_id][u8 argc]` |
| `CALL_METHOD` | 6 | `[0x73][u32 fun_id][u8 argc]` |
| `CALL_NATIVE` | переменный | `[0x62][u32 lib_len][lib][u32 fn_len][fn][u8 argc]` |
| `JMP` / `JZ` / `JNZ` | 5 | `[u8 op][u32 target]` |
| `NEW_CLASS` | 5 | `[0x70][u32 class_id]` |
| `NEW_ARRAY` | 5 | `[0xC0][u32 n]` |
| `PUSH_HANDLER` | 5 | `[0xB3][u32 target]` |

Формат `PUSH` зависит от `type`:

| `type` | Payload |
|---|---|
| `INT8` | 1 байт |
| `INT16` | 2 байта |
| `INT32` | 4 байта |
| `INT64`, `INT`, `FLOAT` | 8 байт |
| `BOOL` | 1 байт |
| `STR`, `BIN`, `HEX` | `[u32 len][bytes]` |
| `NULL` | пусто |

Все числа little-endian.

## Архитектура

Стековая машина. Данные и фреймы хранятся в `std::vector`.

Значение:

```cpp
struct Value {
    DataType type = DataType::NULL_;
    int64_t  i = 0;
    double   f = 0.0;
    bool     b = false;
    std::string s;
    std::shared_ptr<VMObject> obj;
    std::shared_ptr<VMList>   list;
    std::shared_ptr<VMDict>   dict;
};
```

Память: ref-counting через `shared_ptr`. `free()` необязателен.
Двойной `free()` безопасен.

Фрейм вызова:

```cpp
struct Frame {
    uint32_t fun_id;
    size_t   saved_ip;
    const std::vector<uint8_t>* saved_code;
    std::unordered_map<uint32_t, Value> locals;
};
```

Для методов `self` хранится в `locals[0]`.

## Файлы

| Файл | Назначение |
|---|---|
| `main.cpp` | CLI, загрузка, запуск |
| `vm.cpp` | `load`, `execute`, `call_function`, `find_function_by_name`, `dump_globals` |
| `vm_exec.cpp` | главный цикл `run_impl`, `CALL_NATIVE`, обработка `THROW` |
| `vm_frame.cpp` | вход и выход из функции |
| `vm_loader.cpp` | разбор `.aklp` |
| `vm_memory.cpp` | `LOAD_VAR`, `STORE_VAR` |
| `vm_native.cpp` | загрузка `.dll`/`.so` и вызов `askal_db` |
| `vm_stack.cpp` | `push`, `pop` |
| `vm_thread.cpp` | `thread >> run/await/alive/kill` |
| `askal_plugin.h` | ABI нативных библиотек |
| `vm_opcodes.h` | опкоды и `DataType` |

## Нативные вызовы

`CALL_NATIVE` упаковывает аргументы по ABI из `askal_plugin.h`:

Вход:

    [u32 argc]
    [arg0]
    [arg1]
    ...

Каждый arg:

    [u8 type][payload]

Выход:

    [u8 type][payload]

VM загружает `.dll` (Windows, `LoadLibraryA`) или `.so` (Linux,
`dlopen`), ищет символ `askal_db`, вызывает его.

Если библиотека вернула `ASKAL_ERR_*`, VM бросает исключение Askal
(перехватывается `try/catch` в коде Askal). Если в момент ошибки
handler не зарегистрирован, программа завершается с `Runtime error`.

## Обработка исключений

Стек обработчиков: `handler_stack` в `VM`.

`PUSH_HANDLER target` регистрирует адрес `catch`.
`POP_HANDLER` снимает.
`THROW` снимает значение со стека, если есть handler, прыгает в
`catch` и кладёт значение на стек. Иначе завершение программы с
`Runtime error`.

Ошибки из `CALL_NATIVE` обрабатываются так же (фикс: `CALL_NATIVE`
перехватывает `std::exception` и ведёт себя как `THROW`).

## `thread`

`THREAD_RUN [func_id][args]`:

- Windows: `CreateProcessW` + `CreatePipe`. Дочерний процесс:
  `askal_vm --call-func <id> --aklp <path>`.
- Linux: `fork` + `execvp` + `pipe`.

Аргументы передаются в stdin дочернего процесса:

    [u32 len][blob]

Результат читается из stdout:

    [u32 len][blob]

Сериализация значений: `VM::pack_value` и `VM::unpack_value`.

`THREAD_AWAIT` ждёт завершения и возвращает результат.
`THREAD_ALIVE` проверяет, жив ли процесс.
`THREAD_KILL` завершает процесс.

## Ошибки

    Runtime error: <сообщение>

Примеры:
- `Unknown variable id=0`
- `Bad args for: bmf::insert - field 'age': value 999 out of range for int8`
- `Stack underflow`
- `Class id <id> not found`

При `--trace` каждая инструкция и операция стека печатаются в stderr.
