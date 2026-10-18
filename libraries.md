
# Нативные библиотеки Askal

Библиотеки расширяют язык функциями, которых нет в ядре.
Каждая библиотека это отдельный файл: `.dll` на Windows, `.so` на Linux.

Подключение:

    use console;
    use file;
    use bmf;

Вызов:

    console >> println("hello");
    file >> load("data.txt");

## ABI

Каждая библиотека экспортирует одну C-функцию:

```c
int askal_db(const char* name,
             const uint8_t* in, size_t in_size,
             uint8_t* out, size_t* out_size);
```

Возвращаемое значение:

| Код | Значение |
|---|---|
| 0 (`ASKAL_OK`) | успех |
| 1 (`ASKAL_ERR_UNKNOWN`) | неизвестное имя функции |
| 2 (`ASKAL_ERR_BADARGS`) | неверные аргументы или ошибка выполнения |
| 3 (`ASKAL_ERR_INTERNAL`) | внутренняя ошибка |

При ошибке VM бросает исключение Askal с текстом из `out`
(если библиотека записала туда строку).

### Входной буфер

    [u32 argc][arg0][arg1]...

Каждый аргумент:

    [u8 type][payload]

| Тип | Константа | Payload |
|---|---|---|
| 0x10 | STR | `[u32 len][bytes]` |
| 0x11 | BOOL | 1 байт |
| 0x12 | FLOAT | 8 байт double |
| 0x13-0x17 | INT8..INT | 8 байт int64 |
| 0x18 | NULL | пусто |
| 0x19 | BIN | `[u32 len][bytes]` |
| 0x1A | HEX | `[u32 len][bytes]` |
| 0x1C | OBJECT | 8 байт handle |
| 0x1D | ARRAY | `[u32 count][item...]` |
| 0x1E | LIST | `[u32 count][item...]` |
| 0x1F | DICT | `[u32 count][key value ...]` |

Все целые передаются как 8 байт, тип используется для проверки.
`LIST`, `ARRAY`, `DICT` кодируются рекурсивно, элементы это
`[u8 type][payload]`. В `DICT` ключ всегда `STR`.

### Выходной буфер

    [u8 type][payload]

Одно значение. Для `LIST`:

    [0x1E][u32 count][item0][item1]...

Для `DICT`:

    [0x1F][u32 count][key0][value0]...

### `askal_plugin.h`

Общий заголовок. Содержит константы типов и кодов ошибок,
структуры `AkReader` и `AkWriter`, функции для чтения и записи
буфера (`ak_u8`, `ak_u32`, `ak_i64`, `ak_f64`, `ak_str`,
`ak_w_u8`, `ak_w_u32`, `ak_w_i64`, `ak_w_f64`, `ak_w_str`,
`ak_ret_int`, `ak_ret_bool`, `ak_ret_str`, `ak_ret_null`,
`ak_ret_list_begin`, `ak_ret_dict_begin`, `ak_w_elem_int`,
`ak_w_elem_str`, `ak_w_elem_bool`, `ak_w_elem_null`,
`ak_skip_value`).

Скелет:

```c
#include "askal_plugin.h"

extern "C" __declspec(dllexport)
int askal_db(const char* name,
             const uint8_t* in, size_t in_size,
             uint8_t* out, size_t* out_size)
{
    AkReader r;
    ak_reader_init(&r, in, in_size);

    AkWriter w;
    ak_writer_init(&w, out, *out_size);

    uint32_t argc = ak_u32(&r);

    if (strcmp(name, "my_func") == 0) {
        ak_ret_int(&w, 42);
        *out_size = (size_t)(w.p - out);
        return ASKAL_OK;
    }

    return ASKAL_ERR_UNKNOWN;
}
```

### Манифест

Рядом с каждой библиотекой лежит `<name>_manifest.json`:

```json
{
  "name": "console",
  "version": "1.0.0",
  "abi": 3,
  "functions": [
    {
      "name": "println",
      "args": [{"name":"value","type":"auto"}],
      "returns": "null"
    }
  ]
}
```

Используется IDE для автодополнения, hover и проверки типов.
На работу VM не влияет.

### Расположение

Библиотеки лежат рядом с `askal_vm.exe` или в `LD_LIBRARY_PATH`.
Имена: `<name>.dll` на Windows, `lib<name>.so` на Linux.

### Ограничения

Размер выходного буфера в текущей VM: 64 КБ. При превышении
библиотека возвращает `ASKAL_ERR_INTERNAL`.
Нет состояния между вызовами, кроме глобальных переменных внутри
самой библиотеки.

---

## console

Вывод, ввод, цвета, очистка экрана. Файл `console/dllmain.cpp`.

### Функции

| Функция | Аргументы | Возврат | Описание |
|---|---|---|---|
| `print` | value: auto | null | вывод без перевода строки |
| `println` | value: auto | null | вывод с переводом строки |
| `print_int` | value: int64 | null | целое число |
| `print_float` | value: float | null | float с 2 знаками |
| `print_bool` | value: bool | null | true / false |
| `print_hex` | value: int64 | null | `0x...` |
| `print_bin` | value: int64 | null | `0b...` |
| `newline` | нет | null | один перевод строки |
| `input` | нет | str | чтение строки из stdin |
| `read_char` | нет | str | один символ |
| `set_color` | r: int64, g: int64, b: int64 | null | цвет текста (RGB) |
| `reset_color` | нет | null | сброс цвета |
| `print_color` | r, g, b, value: auto | null | вывод значения в цвете |
| `clear` | нет | null | очистка экрана |

### Поведение

`print` и `println` поддерживают все типы, включая вложенные
`LIST` и `DICT`. Для `LIST` и `DICT` используется рекурсивная
печать, значения разделены `, `.

Цвета реализованы через ANSI escape-последовательности:
- `\x1b[38;2;R;G;Bm` для текста,
- `\x1b[0m` для сброса.

На Windows виртуальный терминал включается через
`SetConsoleMode(ENABLE_VIRTUAL_TERMINAL_PROCESSING)` при первом
вызове `set_color`. На Linux и macOS ничего делать не нужно.

---

## string

Работа со строками. Файл `string/dllmain.cpp`.

### Функции

| Функция | Аргументы | Возврат | Описание |
|---|---|---|---|
| `len` | s: auto | int64 | длина в байтах |
| `char_at` | s, i: int64 | int64 | ASCII-код символа, -1 если вне диапазона |
| `substr` | s, from: int64, to: int64 | str | подстрока `[from, to)` |
| `slice` | s, from, to | str | алиас `substr` |
| `to_upper` | s: auto | str | верхний регистр |
| `to_lower` | s: auto | str | нижний регистр |
| `trim` | s: auto | str | убрать пробелы по краям |
| `find` | s, sub: auto | int64 | первое вхождение или -1 |
| `replace` | s, from, to | str | заменить все вхождения |
| `split` | s, sep: auto | str | разделить и склеить через `\n` |
| `to_int` | s: auto | int64 | парсить как int (base 10) |
| `to_int_base` | s, base: int64 | int64 | парсить в указанной базе |
| `to_str` | v: auto | str | в строку |
| `to_base` | n: int64, base: int64 | str | целое в строку в базе 2..36 |
| `to_bin` | n: int64 | str | двоичная строка |
| `starts_with` | s, prefix: auto | bool | начинается с |
| `ends_with` | s, suffix: auto | bool | заканчивается на |
| `concat` | a, b: auto | str | склеить |
| `repeat` | s, n: int64 | str | повторить n раз |
| `pad_left` | s, width, ch | str | дополнить слева |
| `pad_right` | s, width, ch | str | дополнить справа |
| `cmp` | a, b | int64 | сравнение, -1 / 0 / 1 |
| `reverse` | s | str | обратный порядок |

### Поведение

`len` считает байты, не символы. Для UTF-8 кириллицы один символ
занимает 2 байта.
`char_at` возвращает код байта, не символа Unicode.
Отрицательные индексы в `substr` считаются с конца.
`split` возвращает строку, части разделены `\n`, не `list`
(ограничение текущей версии).

---

## file

Файлы, директории, пути, диски. Файл `file/dllmain.cpp`.

### Высокий уровень

| Функция | Аргументы | Возврат |
|---|---|---|
| `load` | path: str | str |
| `save` | path: str, data: str | bool |
| `append` | path: str, data: str | bool |
| `lines` | path: str | list |
| `save_lines` | path: str, lines: list | bool |
| `count_lines` | path: str | int |

### Низкий уровень

| Функция | Аргументы | Возврат |
|---|---|---|
| `open` | path: str, mode: str | int (handle) |
| `close` | handle: int | bool |
| `read` | handle: int, n: int | str |
| `read_all` | handle: int | str |
| `read_line` | handle: int | str |
| `write` | handle: int, data: str | int |
| `write_line` | handle: int, data: str | int |
| `size_h` | handle: int | int |
| `pos` | handle: int | int |
| `seek` | handle: int, offset: int | null |
| `eof` | handle: int | bool |
| `flush` | handle: int | null |

### Метаданные

| Функция | Аргументы | Возврат |
|---|---|---|
| `exists` | path: str | bool |
| `is_file` | path: str | bool |
| `is_dir` | path: str | bool |
| `size` | path: str | int |
| `modified` | path: str | int (unix) |

### Директории

| Функция | Аргументы | Возврат |
|---|---|---|
| `make_dir` | path: str | bool |
| `make_dirs` | path: str | bool |
| `delete_dir` | path: str | bool |
| `delete_dirs` | path: str | int |
| `list` | path: str | list |
| `list_files` | path: str | list |
| `list_dirs` | path: str | list |
| `list_paths` | path: str | list |
| `walk` | path: str | list |

### Копирование и удаление

| Функция | Аргументы | Возврат |
|---|---|---|
| `copy` | src: str, dst: str | bool |
| `copy_all` | src: str, dst: str | bool |
| `move` | src: str, dst: str | bool |
| `rename` | src: str, dst: str | bool |
| `delete` | path: str | bool |
| `delete_all` | path: str | int |
| `truncate` | path: str, size: int | bool |

### Пути

| Функция | Аргументы | Возврат |
|---|---|---|
| `name` | path: str | str |
| `parent` | path: str | str |
| `ext` | path: str | str |
| `stem` | path: str | str |
| `join` | parts: list | str |
| `absolute` | path: str | str |
| `real` | path: str | str |
| `is_absolute` | path: str | bool |

### Окружение и диски

| Функция | Аргументы | Возврат |
|---|---|---|
| `cwd` | нет | str |
| `cd` | path: str | bool |
| `temp` | нет | str |
| `home` | нет | str |
| `drives` | нет | list |
| `drive_info` | drive: str | dict |

### Поиск

| Функция | Аргументы | Возврат |
|---|---|---|
| `glob` | dir: str, pattern: str | list |
| `find` | dir: str, pattern: str | list |

### Поведение

Все пути в UTF-8. Внутри конвертируются в UTF-16 для Windows API
и в `std::filesystem` для переносимости.
`read` и `write` работают с байтами. Для чтения или записи по
смещению используется `seek`.
`glob` ищет в одной директории. `find` ищет рекурсивно.
`drive_info` возвращает dict с ключами `total` и `free`.

---

## system

Время, окружение, процессы. Файл `system/dllmain.cpp`.

### Функции

| Функция | Аргументы | Возврат |
|---|---|---|
| `time_ms` | нет | int64 |
| `time_unix` | нет | int64 |
| `sleep` | ms: int64 | null |
| `env` | name: str | str |
| `set_env` | name: str, value: str | bool |
| `args_count` | нет | int64 |
| `arg` | index: int64 | str |
| `cwd` | нет | str |
| `cd` | path: str | bool |
| `mkdir` | path: str | bool |
| `rmdir` | path: str | bool |
| `rename` | from: str, to: str | bool |
| `copy_file` | src: str, dst: str | bool |
| `list_dir` | path: str | str |
| `is_dir` | path: str | bool |
| `platform` | нет | str |
| `arch` | нет | str |
| `hostname` | нет | str |
| `username` | нет | str |
| `exit` | code: int64 | null |
| `exec` | cmd: str | int64 |
| `beep` | freq: int64, ms: int64 | null |

### Поведение

`platform` возвращает `windows`, `linux` или `macos`.
`arch` возвращает `x64`, `arm64`, `x86` или `unknown`.
`args_count` и `arg` пока заглушки, всегда 0 и пустая строка.
`list_dir` возвращает строку с именами через `\n`.
`exec` запускает shell-команду и возвращает код завершения.
`beep` на Linux и macOS выводит `\a`.

---

## tools

Утилиты. Файл `tools/dllmain.cpp`.

### Функции

| Функция | Аргументы | Возврат |
|---|---|---|
| `time_ms` | нет | int64 |
| `sleep` | ms: int64 | null |
| `rand_int` | min: int64, max: int64 | int64 |
| `rand_float` | нет | float |
| `clamp_int` | x, lo, hi: int64 | int64 |
| `abs_int` | x: int64 | int64 |
| `min_int` | a, b: int64 | int64 |
| `max_int` | a, b: int64 | int64 |
| `str_to_int` | s: str | int64 |
| `int_to_str` | x: int64 | str |
| `float_to_str` | x: float, prec: int64 | str |
| `str_to_float` | s: str | float |
| `bool_to_int` | b: bool | int64 |
| `int_to_bool` | x: int64 | bool |

### Поведение

`rand_int` возвращает значение в `[min, max]` включительно.
`rand_float` возвращает `[0.0, 1.0)`.
`str_to_int` и `str_to_float` при ошибке парсинга возвращают 0.
`float_to_str` форматирует с `prec` знаками после запятой.

---

## memory

Сырая память и числовые буферы. Файл `memory/dllmain.cpp`.

### Буферы

| Функция | Аргументы | Возврат |
|---|---|---|
| `new` | size: int | int (handle) |
| `delete` | handle: int | bool |
| `resize` | handle: int, new_size: int | int |
| `size` | handle: int | int |
| `copy` | dst, src, n: int | null |
| `move` | dst, src, n: int | null |
| `fill` | handle: int, byte: int | null |
| `compare` | a, b, n: int | int |
| `blocks` | нет | list |

### Типизированный доступ

| Функция | Аргументы | Возврат |
|---|---|---|
| `put_int8` | handle, off, value: int | null |
| `put_int16` | handle, off, value: int | null |
| `put_int32` | handle, off, value: int | null |
| `put_int64` | handle, off, value: int | null |
| `put_float` | handle, off: int, value: float | null |
| `put_str` | handle, off: int, value: str | int |
| `get_int8` | handle, off: int | int |
| `get_int16` | handle, off: int | int |
| `get_int32` | handle, off: int | int |
| `get_int64` | handle, off: int | int |
| `get_float` | handle, off: int | float |
| `get_str` | handle, off, n: int | str |

### Числовые списки

| Функция | Аргументы | Возврат |
|---|---|---|
| `new_list` | count: int | int (handle) |
| `list_len` | handle: int | int |
| `list_get` | handle: int, idx: int | int |
| `list_set` | handle, idx, value: int | null |
| `list_fill` | handle: int, value: int | null |

### Поведение

Буферы хранятся в глобальной таблице `handle -> vector<uint8_t>`.
Handle уникален, монотонно растёт.
`put_*` и `get_*` используют little-endian.
`put_str` записывает `[len][bytes]` и возвращает количество
записанных байт.
Числовые списки это `vector<int64_t>`, доступ по индексу без
проверки границ (проверка на стороне Askal).
`blocks` возвращает список активных handle.

---

## python

Встроенный CPython. Самая требовательная и хрупкая библиотека.
Файл `python_bridge/dllmain.cpp`.

### Назначение

Позволяет выполнять Python-код, вычислять выражения, вызывать
Python-функции и обмениваться значениями между Askal и Python.

### Реализация

Библиотека **не линкуется** с `python3XX.lib`. Она загружает
`python3XX.dll` через `LoadLibraryA` и получает все нужные
функции CPython через `GetProcAddress`.

Это значит: Askal не зависит от конкретной версии Python на
этапе сборки. Достаточно положить нужный `.dll` рядом с VM.

### Функции

| Функция | Аргументы | Возврат | Описание |
|---|---|---|---|
| `exec` | code: str | null | выполнить Python-код в `__main__` |
| `eval` | expr: str | auto | вычислить выражение, вернуть результат |
| `call` | path: str, args: list | auto | вызвать функцию по dotted-path |
| `get` | path: str | auto | получить глобал или атрибут |
| `set` | name: str, value: auto | null | установить глобал |

### Пайплайн вызова

**`python >> exec("code")`**:

1. Если интерпретатор ещё не инициализирован, вызвать
   `Py_Initialize`.
2. Получить `__main__` через `PyImport_AddModule("__main__")`
   и `PyModule_GetDict`.
3. Вызвать `PyRun_String(code, Py_file_input, dict, dict)`.
4. Если вернулось `NULL`, получить и сохранить текст ошибки
   через `PyErr_Fetch` и `PyErr_Print`.
5. `Py_DECREF` результата.

**`python >> eval("expr")`**:

1. То же, что `exec`, но `Py_eval_input`.
2. Результат это `PyObject*`. Преобразовать в `Value` Askal.
3. `Py_DECREF` результата.

**`python >> call("math.sqrt", [16.0])`**:

1. Разбить `path` по точкам. Первый сегмент это имя модуля.
2. `PyImport_ImportModule(name)`.
3. Для каждого следующего сегмента `PyObject_GetAttrString`.
4. Аргументы Askal преобразовать в Python-объекты, положить
   в `PyTuple`.
5. `PyObject_CallObject`.
6. Результат преобразовать в `Value`. `Py_DECREF` всего.

**`python >> get("math.pi")`**:

1. То же, что `call`, но без вызова, только получить атрибут.

**`python >> set("x", 42)`**:

1. `__main__` получить.
2. `PyDict_SetItemString(dict, "x", PyLong_FromLong(42))`.
3. `Py_DECREF` значения.

### Преобразование типов

**Askal → Python**:

| Askal | Python |
|---|---|
| `int8`..`int64`, `int` | `PyLong` |
| `float` | `PyFloat` |
| `bool` | `PyBool` |
| `str` | `PyUnicode` (UTF-8) |
| `null` | `Py_None` |
| `list` | `PyList` (рекурсивно) |
| `dict` | `PyDict` (ключи str) |

**Python → Askal**:

| Python | Askal |
|---|---|
| `PyLong` | `int64` |
| `PyFloat` | `float` |
| `PyBool` | `bool` |
| `PyUnicode` | `str` |
| `Py_None` | `null` |
| `PyList`, `PyTuple` | `list` |
| `PyDict` | `dict` (только str-ключи) |
| остальное | `null` |

### Требования (Windows)

Рядом с `askal_vm.exe` должны лежать:

| Файл | Откуда |
|---|---|
| `python3XX.dll` | установка Python (например `python314.dll`) |
| `python3.dll` | установка Python |
| `vcruntime140.dll` | установка Python |
| `vcruntime140_1.dll` | установка Python |
| `python3XX._pth` | создать вручную |

`python3XX._pth` нужен, чтобы убрать warning
`Could not find platform independent libraries`.
Содержимое файла `python314._pth`:

```
python314.zip
.
```

Первая строка это путь к stdlib или архив. Вторая это текущая
папка. Без этого файла Python ищет stdlib в системных путях и
падает при отсутствии.

### Хрупкие моменты

**`Py_None`, `Py_True`, `Py_False` не экспортируются** в
CPython 3.12 и новее. Попытка `GetProcAddress("Py_None")`
возвращает `NULL`, дальнейшее использование приводит к крашу.

Решение: получить через `builtins`:

```c
PyObject* builtins = PyImport_ImportModule("builtins");
PyObject* none  = PyObject_GetAttrString(builtins, "None");
PyObject* true_ = PyObject_GetAttrString(builtins, "True");
PyObject* false = PyObject_GetAttrString(builtins, "False");
```

Эти указатели нужно сохранить при инициализации и использовать
везде вместо прямого обращения к символам `Py_None` и т.п.

**Версия `.dll` должна совпадать с `._pth`**. `python314.dll`
требует `python314._pth`. Иначе интерпретатор не инициализируется
и выдаёт warning, после чего падает.

**`Py_Finalize` нельзя вызывать до освобождения всех объектов**.
Если остались живые `PyObject*` (строки, списки, функции),
`Py_Finalize` может упасть или утечь память.

Правило: все `PyObject*`, полученные из `python.dll`, должны быть
освобождены через `Py_DECREF` до `Py_Finalize`. Для объектов,
сохранённых в глобальных переменных библиотеки (например
`Py_None`), освобождать в конце, перед `Py_Finalize`.

**GIL**. CPython использует Global Interpreter Lock.
`python.dll` работает в одном потоке VM.
`PyGILState_Ensure` и `PyGILState_Release` не реализованы.
Использование `python` из `thread >> run` приведёт к гонке и
падению. Текущее ограничение: `python` только в главном потоке.

**Кодировка**. `PyUnicode_AsUTF8` возвращает `const char*`,
валидный пока жив `PyObject*`. Нельзя сохранять его в `std::string`
без копирования.

**`PyErr_Fetch` / `PyErr_Print`**. При ошибке в Python
`PyRun_String` возвращает `NULL`, а информация об ошибке лежит
во внутреннем состоянии интерпретатора. Если её не забрать,
следующий вызов может работать с устаревшей ошибкой. Всегда
вызывать `PyErr_Fetch` при `NULL` и очищать через `PyErr_Clear`.

**Модули**. `import` работает через `PyImport_ImportModule`.
Если модуль не установлен в stdlib и не лежит рядом,
`PyImport_ImportModule` вернёт `NULL`. Ошибка не бросается
автоматически, её нужно проверять.

### Отладка

```askal
use console;
use python;

// проверка инициализации
python >> exec("print('Python OK')");

// версия
var str ver = python >> eval("__import__('sys').version");
console >> println(ver);

// проверка None
var auto n = python >> get("None");
console >> println("None = " + n);
```

Если `python >> exec` падает, смотреть, что за `python3XX.dll`
рядом с VM. Если работает, но `get("None")` возвращает `null`,
значит `Py_None` не получен через `builtins`.

### Ограничения

- Только Windows в текущей реализации.
- Только главный поток (GIL не реализован).
- Нет `sys.path` из Askal, всё через `._pth`.
- Нет `import` как отдельной функции, только через `exec`.
- Нет обработки `__del__`, `atexit`, сигналов.

### Сборка

`python.dll` собирается как обычная библиотека Askal.
Линкуется только с `kernel32.lib` для `LoadLibraryA` и
`GetProcAddress`, с CPython не линкуется.

---

## bmf

Встраиваемая СУБД с фиксированной схемой. Файл `bmf/dllmain.cpp`.

### Модель

Запись имеет фиксированный размер, вычисляемый из схемы.
Поля упаковываются побитово без выравнивания. Размер записи
округляется вверх до целого байта.

Доступ к записи по id за O(1): `offset = data_offset + id * record_size`.

Файл `.bmfdb` состоит из областей:

| Область | Смещение | Размер |
|---|---|---|
| Заголовок | 0 | 512 байт |
| Схема | 512 | `field_count * 64`, выровнено на 512 |
| Bitmap | после схемы | `capacity / 8` байт |
| Данные | после bitmap | `capacity * record_size` |

Заголовок содержит: magic `BMFDB\1`, число полей, размер
записи в битах и байтах, число актуальных записей, максимальный
id, смещения областей, время создания и модификации.

Схема это массив записей по 64 байта на поле: тип, флаги,
размер в битах, смещение в битах внутри записи, имя.

Bitmap: один бит на одну запись. `1` занята, `0` свободна.

### Типы полей

| Тип | Размер | Примечание |
|---|---|---|
| `int8` | 8 бит | знаковое |
| `int16` | 16 бит | знаковое |
| `int32` | 32 бита | знаковое |
| `int64`, `int` | 64 бита | знаковое |
| `float32` | 32 бита | IEEE-754 single |
| `float64`, `float`, `double` | 64 бита | IEEE-754 double |
| `bool` | 1 бит | true / false |
| `str` | N бит | строка, N задаётся |
| `blob` | N бит | байты, N задаётся |

Для `str` и `blob` поле `bits` обязательно.
Для остальных `bits` игнорируется, размер берётся из типа.

### Схема

Это `list` из `dict`:

```askal
var list schema = [
    {"name": "id",   "type": "int64", "not_null": true},
    {"name": "age",  "type": "int8"},
    {"name": "name", "type": "str", "bits": 256},
    {"name": "data", "type": "blob", "bits": 1024}
];
```

Порядок элементов в `list` фиксирует порядок полей в записи.
`not_null` включает проверку при вставке.

### Функции

| Функция | Аргументы | Возврат |
|---|---|---|
| `create_db` | path: str, schema: list | int (handle) |
| `open_db` | path: str | int (handle) |
| `close_db` | handle: int | bool |
| `drop_db` | path: str | bool |
| `insert` | handle: int, values: dict | int (id) |
| `get` | handle: int, id: int | dict |
| `update` | handle: int, id: int, values: dict | bool |
| `delete` | handle: int, id: int | bool |
| `count` | handle: int | int |
| `find` | handle: int, field: str, value: auto | int |
| `find_all` | handle: int, field: str, value: auto | list |
| `restore_last` | handle: int | int |
| `truncate` | handle: int | bool |
| `version` | нет | str |
| `last_error` | нет | str |

### Поведение

`insert` находит первый свободный бит в bitmap, пишет запись,
ставит бит. Если свободных нет, пишет в конец.
`get` читает запись по offset и десериализует поля.
`update` читает запись, применяет переданные поля, пишет обратно.
`delete` сбрасывает бит в bitmap, увеличивает счётчик удалённых
в памяти, ставит id в стек удалённых.
`restore_last` берёт последний id из стека удалённых, ставит бит
обратно. Стек хранится только в памяти текущей сессии.
`truncate` обнуляет bitmap, счётчики, стек удалённых.
`find` делает полный скан: проходит по всем занятым id, читает
запись, сравнивает указанное поле со значением. Возвращает id
первого совпадения или 0, если не найдено.
`find_all` возвращает список всех совпавших id.

`create_db` возвращает handle или 0. `insert` возвращает id или
0 при ошибке. `get` при отсутствии id бросает исключение.

### Ограничения

Емкость базы фиксируется при создании (`BMF_DEFAULT_CAPACITY`,
1 млн записей). При превышении `insert` возвращает ошибку.
Схему нельзя изменить после создания.
`find` не использует индексы, работает за O(N).
Нет транзакций, нет concurrent access, нет шифрования
(планируется отдельной библиотекой).
