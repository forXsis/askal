# Askal
<img width="233" height="211" alt="icon" src="https://github.com/user-attachments/assets/dbcb699e-ec88-4898-b8d9-4b400100595e" />


**Askal** - статически типизированный компилируемый язык программирования,
написанный с нуля на C++17. Собственный компилятор, собственный байткод
(`.aklp`), собственная лёгкая виртуальная машина, нативные библиотеки
через DLL, обработка исключений, классы, коллекции и автоматическое
управление памятью.

Проект создан для изучения устройства компиляторов и виртуальных машин,
но при этом пригоден для написания реальных программ.

---

## Содержание

- [Возможности](#возможности)
- [Пример](#пример)
- [Быстрый старт](#быстрый-старт)
- [Структура проекта](#структура-проекта)
- [Компоненты](#компоненты)
- [Сборка](#сборка)
- [Документация](#документация)
- [Примеры](#примеры)
- [Библиотеки](#библиотеки)
- [Лицензия](#лицензия)
- [Благодарности](#благодарности)

---

## Возможности

### Язык

- **Типы:** `int8`, `int16`, `int32`, `int64`, `int`, `float`, `bool`,
  `str`, `bin`, `hex`, `auto`, `null`, `object`.
- **Арифметика и логика:** `+ - * / %`, `== != < > <= >=`, `&& || !`,
  унарные `-` и `!`, тернарный оператор, `++`/`--`, составные
  присваивания (`+=`, `-=`, ...).
- **Управляющие конструкции:** `if` / `else if` / `else`, `while`, `for`,
  `switch` / `case` / `default`, `break`, `continue`.
- **Функции:** объявление, вызов, рекурсия, forward reference,
  множественные аргументы, возвращаемое значение.
- **ООП:** классы, поля, методы, `init` (конструктор), `self`, объекты
  как аргументы и возвращаемые значения, `free()`.
- **Коллекции:** `list` (динамический список), `array` (фиксированный
  массив), `dict` (словарь `str → значение`). Литералы `[...]` и `{...}`,
  доступ по индексу `[i]`, встроенные методы (`push`, `pop`, `insert`,
  `remove`, `clear`, `has`, `keys`, `values`, `fill`, `len`).
- **Исключения:** `try` / `catch` / `finally` / `throw`, вложенные
  `try`, броски значения любого типа.
- **Модули:** `use` для нативных библиотек, `include` для `.akl`-файлов.
- **Память:** автоматический ref-counting GC. `free()` необязателен,
  двойной вызов безопасен.

### Компилятор

- Три прохода: регистрация имён → резолв ссылок → проверка типов.
- Constant folding и peephole-оптимизации.
- Флаги `-O0`, `-O1`, `-O2` (уровни оптимизации).
- `--dump` для hex-дампа байткода.

### Виртуальная машина

- Стековая машина с одним байтом на опкод.
- 100+ опкодов.
- `--trace`, `--dump` для отладки.
- Runtime-ошибки вместо крешей.

### Нативные библиотеки

- 9 DLL из коробки: `console`, `string`, `file`, `system`, `tools`,
  `memory`, `array`, `list`, `dict`.
- Простой ABI через `askal_db`.
- Манифесты (`*_manifest.json`) для IDE и автодополнения.

### Упаковка в EXE

- `askal_pack` собирает автономный `.exe`: VM + байткод + нужные DLL
  в одном файле.

---

## Пример

```askal
use console;
use string;

class Vec2 {
    var float x = 0.0;
    var float y = 0.0;

    fun init(float x0, float y0) {
        x = x0;
        y = y0;
        return null;
    }

    fun length_sq() -> float {
        return x * x + y * y;
    }

    fun print() {
        console >> print("Vec2(");
        console >> print_float(x);
        console >> print(", ");
        console >> print_float(y);
        console >> println(")");
        return null;
    }
}

fun main() {
    var Vec2 v = Vec2(3.0, 4.0);
    v >> print();
    console >> print("len_sq = ");
    console >> print_float(v >> length_sq());
    console >> println("");
    v >> free();

    var list xs = [1, 2, 3, 4, 5];
    var int64 total = 0;
    for (var int64 i = 0; i < len(xs); i = i + 1) {
        total = total + xs[i];
    }
    console >> print("sum = ");
    console >> print_int(total);
    console >> println("");

    var dict person = {"name": "Alice", "age": 30};
    console >> println(person["name"]);

    try {
        throw "test error";
    } catch (e) {
        console >> println("caught");
    } finally {
        console >> println("done");
    }

    return null;
}

main();
```

---

## Быстрый старт

### Требования

- **Windows** 10 / 11 (x64)

### Установка

1. Скачай релизный архив `askal-vX.Y.Z.zip` со страницы
   [Releases](https://github.com/forXsis/askal/Releases).
2. Распакуй куда угодно. Внутри — папка `tools/` со всеми
   бинарниками, DLL и манифестами.
3. Добавь `tools/` в `PATH` (опционально).
4. Проверь установку:

   ```
   askal_compiler --version
   askal_vm --version
   ```

### Первая программа

Создай `hello.akl`:

```askal
use console;

console >> println("Hello, World!");
```

Компиляция и запуск:

```
askal_compiler --c hello.akl
askal_vm --r hello.aklp
```

Вывод:

```
Hello, World!
```

### Упаковка в `.exe`

```
askal_pack --input hello.aklp --output hello.exe --all-libs --libs-path tools
hello.exe
```

---

## Структура проекта

```
askal/
├── askal_compiler/           # Исходники компилятора (.cpp, .h, .vcxproj)
├── askal_vm/                 # Исходники виртуальной машины
├── askal_pack/               # Упаковщик в .exe
├── libraryes/                # Нативные библиотеки (.dll)
├── examples/                 # .akl примеры
├── docs/
│   ├── LANGUAGE.md           # Полное руководство по языку
│   └── syntax.html           # Шпаргалка (HTML)
├── tools/                    # Собранные бинарники, DLL, манифесты
│   ├── askal_compiler.exe
│   ├── askal_vm.exe
│   ├── askal_pack.exe
│   ├── pack_vm.exe
│   ├── *.dll
│   ├── *_manifest.json
│   └── builtins.json
├── Release/                  # Релизные архивы
├── CHANGELOG.md
├── LICENSE
└── README.md
```

---

## Компоненты

| Компонент            | Назначение                                     |
|----------------------|-------------------------------------------------|
| `askal_compiler.exe` | Компилирует `.akl` → `.aklp`                    |
| `askal_vm.exe`       | Запускает `.aklp`                                |
| `askal_pack.exe`     | Упаковывает `.aklp` + `.dll` → автономный `.exe` |
| `pack_vm.exe`        | VM, встраиваемая в `.exe`                        |
| `*.dll`              | Нативные библиотеки                              |

---

## Сборка

### Из исходников

1. Открой `askal.sln` (или отдельные `.vcxproj`) в **Visual Studio 2022**.
2. Выбери **Configuration: Release**, **Platform: x64**.
3. **Build → Build Solution** (Ctrl+Shift+B).
4. Собери также все `.dll`-проекты.
5. Скопируй результаты в `tools/` (или запусти `build-prep.bat`).

### Требования к компилятору

- MSVC 19.30+ (Visual Studio 2022, v143 toolset или новее)
- C++17 (используется также C++20 в некоторых местах)

### Оптимизация

```bash
askal_compiler --c file.akl -O0    # без оптимизации
askal_compiler --c file.akl -O1    # constant folding (по умолчанию)
askal_compiler --c file.akl -O2    # + peephole
```

---

## Документация

- **[docs/LANGUAGE.md](docs/LANGUAGE.md)** - полное руководство по языку
  (типы, операторы, классы, коллекции, исключения, библиотеки).
- **[docs/syntax.html](docs/syntax.html)** - шпаргалка по синтаксису
  (открывается в браузере).

---

## Примеры

В папке `examples/`:

- `hello.akl` - минимальная программа.
- `oop_point.akl` - классы, методы, `self`.
- `collections.akl` - list, dict, array.
- `exceptions.akl` - try/catch/finally.
- `algorithms.akl` - сортировки, рекурсия, числа Фибоначчи.

---

Запуск:

```
askal_compiler --c tests/test_all.akl -v
askal_vm --r tests/test_all.aklp
```

---

## Библиотеки

### console

Вывод, ввод, цвета, очистка экрана.

```askal
console >> println("Hello");
console >> print_int(42);
console >> set_color(255, 100, 0);
console >> print_color(0, 255, 0, "Green text");
console >> reset_color();
```

### string

Работа со строками: длина, подстроки, регистр, поиск, замена,
конвертации.

```askal
var str s = "Hello, World";
console >> println(string >> to_upper(s));
console >> println(string >> substr(s, 0, 5));
console >> println(string >> to_bin(42));
```

### file

Файловые операции.

```askal
var str content = file >> read_file("data.txt");
file >> write_file("out.txt", "Hello");
```

### system

Системные функции: время, sleep, env, работа с директориями,
информация о платформе.

```askal
system >> sleep(1000);
console >> println(system >> platform());
console >> println(system >> cwd());
```

### tools

Утилиты: random, clamp, min/max, конвертации.

```askal
var int64 r = tools >> rand_int(1, 100);
var int64 x = tools >> clamp_int(200, 0, 100);  // 100
```

---

## Лицензия

Лицензия **Apache 2.0**. См. [LICENSE](LICENSE).

---

## Благодарности

Askal создан как учебный проект по компиляторам и виртуальным
машинам. Вдохновлён языками C, C#, Python, Go и Lua.
