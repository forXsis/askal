# Компилятор Askal

Компилятор переводит исходник `.akl` в байткод `.aklp`.
Написан на C++20. Единственный исполняемый файл: `askal_compiler.exe`
(Windows) или `askal_compiler` (Linux).

## CLI

    askal_compiler --c file.akl
    askal_compiler --c file.akl -o out.aklp
    askal_compiler --c file.akl -v
    askal_compiler --c file.akl --dump
    askal_compiler --c file.akl -O0
    askal_compiler --c file.akl -O1
    askal_compiler --c file.akl -O2
    askal_compiler --h
    askal_compiler --v

| Флаг | Значение |
|---|---|
| `--c` | компилировать |
| `-o <path>` | путь к `.aklp` (по умолчанию рядом с `.akl`) |
| `-v` | verbose: сколько токенов, узлов, байт |
| `--dump` | hex-дамп байткода в stdout |
| `-O0` | без оптимизаций |
| `-O1` | constant folding (по умолчанию) |
| `-O2` | constant folding + peephole |

## Этапы

Пайплайн: лексер, парсер, три прохода кодогенератора, эмиссия, оптимизация.

### Лексер

Файл: `askal_lexer.cpp`, `askal_token.h`.

Вход: текст. Выход: `std::vector<Token>`.

Токены: литералы, идентификаторы, ключевые слова, типы, операторы.
Комментарии `//` и `/* */` пропускаются.

### Парсер

Файл: `askal_parser.cpp`, `askal_ast.h`.

Вход: токены. Выход: `Program` (AST).

Поддержка: `var`, `fun`, `class`, `if/else`, `while`, `for`,
`switch/case/default`, `try/catch/finally`, `throw`, `return`,
`break`, `continue`, `use`, `include`.

Выражения: precedence climbing по уровням C.

### Проход 1: `pass1_register_toplevel`

Регистрирует:
- глобальные функции (id, имя, сигнатура),
- классы (id, имя, поля, методы).

Идентификаторы выдаются монотонно. Дубликаты имён это ошибка.

### Проход 2: `pass2_register_and_resolve`

Регистрирует все переменные во всех областях видимости (функция,
блок, цикл, try/catch/finally). Резолвит ссылки на имена: локальные,
глобальные, поля классов, методы, функции, нативные библиотеки.
Определяет встроенные методы коллекций (`push`, `pop`, `len`, ...).

### `resolve_int_sizes`

Для переменных с типом `int` выбирает минимальный размер по значениям
литералов: `int8`, `int16`, `int32` или `int64`. Если инициализатор
не литерал, ставит `int64`.

### Проход 3: `pass3_typecheck`

Проверка типов:
- совместимость при присваивании,
- количество и типы аргументов функций,
- тип возврата,
- условия `if`, `while`, `for` (должны быть `bool`).

Ошибки бросаются как `std::runtime_error` с координатами.

### Эмиссия

Файл: `askal_codegen.cpp`.

Формирует секции байткода:
- `SEC_LIBS` (`use`),
- `SEC_VARS` (глобальные переменные),
- `SEC_FUNCS` (функции, включая тела),
- `SEC_CLASSES` (классы, списки полей и методов),
- `SEC_MAIN` (код верхнего уровня).

Порядок секций фиксирован и важен для загрузчика VM.

### Оптимизация

Файл: `askal_optimizer.cpp`.

`-O0`: без изменений.
`-O1`: constant folding выполняется в кодогенераторе.
`-O2`: плюс peephole:
- удаление `JMP` на следующую инструкцию,
- удаление `PUSH x; POP`,
- удаление мёртвого кода после `RET`, `HALT`, `JMP` до метки.

Пересчёт адресов переходов выполняется после удаления.

## `include`

Директива `include "file.akl"` обрабатывается до парсинга основного
файла: файл читается, парсится, его AST вставляется в начало
программы. Повторный include того же абсолютного пути игнорируется.

Защита от циклических include через `unordered_set<std::string>`
абсолютных путей.

## Ошибки

Формат:

    Compile error: <категория> at <строка>:<колонка> - <сообщение>

Категории: `Lexer error`, `Parser error`, `Codegen error`.
Все выбрасываются как `std::runtime_error`.

## Файлы

| Файл | Назначение |
|---|---|
| `main.cpp` | CLI, разбор флагов, вызов `compile_file` |
| `askal_compiler.cpp` | оркестрация: чтение, лексер, парсер, include, кодогенератор, запись |
| `askal_lexer.cpp` | лексер |
| `askal_parser.cpp` | парсер |
| `askal_codegen.cpp` | проходы 1-3, эмиссия |
| `askal_optimizer.cpp` | peephole |
| `askal_ast.h` | узлы AST |
| `askal_token.h` | токены |
| `askal_opcodes.h` | опкоды и `DataType` |

## Выходной файл

`.aklp` содержит секции `[u8 marker][payload]`:

| Маркер | Секция |
|---|---|
| `0x24` | `SEC_LIBS` |
| `0x20` | `SEC_VARS` |
| `0x21` | `SEC_FUNCS` |
| `0x22` | `SEC_CLASSES` |
| `0x23` | `SEC_MAIN` |

Формат каждой секции описан в `docs/vm.md`.
