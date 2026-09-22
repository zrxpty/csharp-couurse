# Задания модуля M01 / Exercises: M01
# Введение в C# и .NET / Introduction to C# and .NET

## Задания по урокам / Per-lesson exercises

### Задание M01-L01
**Задача / Task (RU):** Опросить окружение выполнения и вывести в консоль версию .NET, ОС, имя машины и текущий пользователь.
**Task (EN):** Inspect the runtime environment and print to the console the .NET version, OS, machine name, and current user.

**Требования / Requirements:**
- Использовать свойства класса `Environment`: `Environment.Version`, `OSVersion`, `MachineName`, `UserName`.
- Вывести также `RuntimeInformation.FrameworkDescription` из `System.Runtime.InteropServices`.
- Форматировать вывод с подписями полей на русском и английском (одна строка — RU, рядом — EN).
- Использовать интерполяцию строк `$"..."`.
- Use `Environment` class properties and `RuntimeInformation.FrameworkDescription`.
- Use string interpolation; label each line bilingually.

**Критерии приёмки / Acceptance criteria:**
- [ ] Программа компилируется и запускается через `dotnet run`.
- [ ] В выводе присутствуют все четыре значения среды.
- [ ] Все значения выведены одной программой без исключений.
- [ ] Program compiles and runs via `dotnet run` with no exceptions.
- [ ] All four environment values are present in output.

**Время / Time:** 30–45 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8 — Program.cs (top-level statements)
using System.Runtime.InteropServices;

Console.WriteLine($".NET версия / .NET version  : {Environment.Version}");
Console.WriteLine($".NET описание / Description : {RuntimeInformation.FrameworkDescription}");
Console.WriteLine($"ОС / OS                     : {Environment.OSVersion}");
Console.WriteLine($"Машина / Machine            : {Environment.MachineName}");
Console.WriteLine($"Пользователь / User         : {Environment.UserName}");
```

---

### Задание M01-L02
**Задача / Task (RU):** Исследовать собственную сборку через рефлексию: вывести имя сборки, версию, список всех публичных типов и их методов.
**Task (EN):** Inspect the own assembly via reflection: print assembly name, version, list of all public types and their methods.

**Требования / Requirements:**
- Получить текущую сборку через `Assembly.GetEntryAssembly()` или `typeof(Program).Assembly`.
- Вывести `GetName().Name` и `GetName().Version`.
- Перечислить все типы через `GetTypes()`, отфильтровать `IsPublic` или не вложенные публичные.
- Для каждого типа вывести его имя и сигнатуры публичных методов (`GetMethods()` с `BindingFlags.Public | BindingFlags.Instance | BindingFlags.Static | BindingFlags.DeclaredOnly`).
- Получить сборку через `Assembly.GetEntryAssembly()`.
- Отфильтровать публичные типы; перечислить их методы с объявленными сигнатурами.

**Критерии приёмки / Acceptance criteria:**
- [ ] В выводе присутствует имя и версия сборки.
- [ ] Для каждого публичного типа выведен список его методов.
- [ ] Системные типы (`System.*`) не дублируются — выводятся только типы вашей сборки.
- [ ] Assembly name and version printed.
- [ ] Methods of each public type listed; no `System.*` types leaked.

**Время / Time:** 45–60 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8 — Program.cs
using System.Reflection;

var asm = Assembly.GetEntryAssembly() ?? typeof(Program).Assembly;
var name = asm.GetName();

Console.WriteLine($"Сборка / Assembly : {name.Name}");
Console.WriteLine($"Версия / Version  : {name.Version}");

foreach (var t in asm.GetTypes().Where(t => t.IsPublic))
{
    Console.WriteLine($"\nТип / Type: {t.FullName}");
    var flags = BindingFlags.Public | BindingFlags.Instance | BindingFlags.Static | BindingFlags.DeclaredOnly;
    foreach (var m in t.GetMethods(flags))
    {
        var parms = string.Join(", ", m.GetParameters().Select(p => $"{p.ParameterType.Name} {p.Name}"));
        Console.WriteLine($"  - {m.ReturnType.Name} {m.Name}({parms})");
    }
}

// Публичный класс для демонстрации рефлексии
public class Greeter
{
    public string Name { get; }
    public Greeter(string name) => Name = name;
    public string SayHello() => $"Привет, {Name}!";
    public static string Banner() => "=== M01-L02 ===";
}
```

---

### Задание M01-L03
**Задача / Task (RU):** Установить/проверить .NET SDK, выполнить `dotnet --info`, создать файл `global.json`, закрепляющий версию SDK, и убедиться, что команда honourит его.
**Task (EN):** Install/verify the .NET SDK, run `dotnet --info`, create a `global.json` pinning the SDK version, and verify the CLI honours it.

**Требования / Requirements:**
- Выполнить `dotnet --info` и зафиксировать установленную версию SDK.
- Создать `global.json` через `dotnet new globaljson --sdk-version <X.Y.Z>` (или вручную).
- Запустить `dotnet --version` в папке с `global.json` — выведенная версия должна совпадать с закреплённой.
- Если закреплённой версии нет — команда должна сообщить ошибку `No usable SDK found` (проверить поведение).
- Run `dotnet --info`; record the SDK version.
- Create `global.json` pinning that version; verify `dotnet --version` matches in that folder.
- Verify the error behaviour when pinning to a non-installed version.

**Критерии приёмки / Acceptance criteria:**
- [ ] Вывод `dotnet --info` приложен или зафиксирован в отчёте.
- [ ] Файл `global.json` существует и содержит поле `"version"`.
- [ ] `dotnet --version` выводит закреплённую версию.
- [ ] Демонстрируется поведение при отсутствующей версии SDK.
- [ ] `dotnet --info` output recorded.
- [ ] `global.json` with `"version"` field present.
- [ ] `dotnet --version` honours the pinned version.

**Время / Time:** 30–45 мин

**Пример решения / Sample solution (для менторов):**
```bash
# 1. Проверка SDK
dotnet --info

# 2. Закрепление версии
dotnet new globaljson --sdk-version 8.0.100

# 3. Проверка
dotnet --version   # должно вывести 8.0.100
```

Пример `global.json` / Sample `global.json`:
```json
{
  "sdk": {
    "version": "8.0.100",
    "rollForward": "latestFeature"
  }
}
```

---

### Задание M01-L04
**Задача / Task (RU):** Создать консольный проект через `dotnet new console`, использовать top-level statements и обработать аргументы командной строки `args`.
**Task (EN):** Create a console project via `dotnet new console`, use top-level statements, and process command-line arguments `args`.

**Требования / Requirements:**
- Создать проект: `dotnet new console -n ArgsDemo`.
- Использовать top-level statements (без явного `class Program` / `static Main`).
- Если `args.Length == 0` — вывести подсказку по использованию.
- Если передан аргумент `--name <имя>` — вывести приветствие с именем.
- Если передан `--count N` — вывести приветствие N раз.
- Использовать цикл `for` и интерполяцию строк.
- Use top-level statements.
- Handle empty `args` with a usage hint.
- Support `--name <name>` and `--count N` arguments.
- Use a `for` loop and string interpolation.

**Критерии приёмки / Acceptance criteria:**
- [ ] Проект создан одной командой `dotnet new console`.
- [ ] В `Program.cs` нет явного объявления `class Program`.
- [ ] `dotnet run` без аргументов выводит подсказку.
- [ ] `dotnet run -- --name Anna --count 3` печатает приветствие 3 раза.
- [ ] Project created with a single `dotnet new console` command.
- [ ] No explicit `class Program` in `Program.cs`.
- [ ] Empty args → usage hint; `--name`/`--count` honoured.

**Время / Time:** 30–60 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8 — Program.cs (top-level statements)
if (args.Length == 0)
{
    Console.WriteLine("Использование / Usage: dotnet run -- --name <имя> [--count N]");
    Console.WriteLine("Example: dotnet run -- --name Anna --count 3");
    return;
}

string name = "Мир";
int count = 1;

for (int i = 0; i < args.Length; i++)
{
    switch (args[i])
    {
        case "--name" when i + 1 < args.Length:
            name = args[++i];
            break;
        case "--count" when i + 1 < args.Length && int.TryParse(args[++i], out var n):
            count = n > 0 ? n : 1;
            break;
    }
}

for (int i = 0; i < count; i++)
{
    Console.WriteLine($"[{i + 1}] Привет, {name}! / Hello, {name}!");
}
```

---

### Задание M01-L05
**Задача / Task (RU):** Собрать и запустить проект в конфигурациях Debug и Release, сравнить размеры выходных файлов и время старта; объяснить роль JIT и оптимизаций.
**Task (EN):** Build and run the project in Debug and Release, compare output file sizes and startup time, and explain the role of JIT and optimizations.

**Требования / Requirements:**
- Выполнить `dotnet build -c Debug` и `dotnet build -c Release`.
- Запустить обе версии (`dotnet run -c Debug` / `-c Release`).
- Измерить размер `.dll` в `bin/Debug/...` и `bin/Release/...`.
- Замерить время старта (например, через `Measure-Command` в PowerShell или `time`).
- Кратко описать (2–4 предложения), почему Release-сборка меньше/быстрее.
- Build in Debug and Release; run both.
- Measure `.dll` size and startup time for each.
- Explain in 2–4 sentences why Release is smaller/faster.

**Критерии приёмки / Acceptance criteria:**
- [ ] Команды `dotnet build -c Debug` и `-c Release` выполнены успешно.
- [ ] Зафиксированы размеры `.dll` для обеих конфигураций.
- [ ] Зафиксировано время старта для обеих конфигураций.
- [ ] В отчёте — краткое объяснение роли JIT и оптимизаций Release.
- [ ] Both builds succeed.
- [ ] DLL sizes and startup times recorded.
- [ ] JIT/optimization explanation present.

**Время / Time:** 30–45 мин

**Пример решения / Sample solution (для менторов):**
```bash
dotnet build -c Debug
dotnet build -c Release

# Размеры DLL
ls bin/Debug/net8.0/ArgsDemo.dll
ls bin/Release/net8.0/ArgsDemo.dll

# Время старта (PowerShell)
Measure-Command { dotnet run -c Debug --no-build --name Test }
Measure-Command { dotnet run -c Release --no-build --name Test }
```

Краткое объяснение / Brief explanation:
- **Debug**: без оптимизаций, с отладочной информацией (PDB), JIT генерирует неоптимизированный код для удобства stepping.
- **Release**: оптимизации компилятора C# + JIT (inlining, удаление мёртвого кода), меньше PDB-нагрузки → меньше `.dll`, быстрее старт и выполнение.

---

### Задание M01-L06
**Задача / Task (RU):** Расширить «Hello World» для приёма `args`, добавить отладочную точку останова и подключить сторонний пакет через `dotnet add package`.
**Task (EN):** Extend "Hello World" to accept `args`, add a debug breakpoint, and add a third-party package via `dotnet add package`.

**Требования / Requirements:**
- В качестве пакета добавить `Spectre.Console` (или `Newtonsoft.Json`): `dotnet add package Spectre.Console`.
- Использовать добавленную библиотеку (например, вывести цветной текст или сериализовать объект в JSON).
- Принимать `args[0]` как имя и выводить приветствие.
- Поставить точку останова (`Debugger.Break()` или через IDE) и продемонстрировать остановку.
- Вывести список установленных пакетов через `dotnet list package`.
- Add `Spectre.Console` (or `Newtonsoft.Json`) via `dotnet add package`.
- Use the added library (colored markup or JSON serialization).
- Read `args[0]` as the name and greet.
- Demonstrate a breakpoint via `Debugger.Break()` or IDE.
- Run `dotnet list package` to show installed packages.

**Критерии приёмки / Acceptance criteria:**
- [ ] Пакет добавлен и виден в `dotnet list package`.
- [ ] Программа использует API добавленного пакета.
- [ ] Аргумент `args[0]` обрабатывается.
- [ ] Демонстрируется срабатывание точки останова.
- [ ] Package visible in `dotnet list package`.
- [ ] Program uses the added package's API.
- [ ] `args[0]` processed; breakpoint demonstrated.

**Время / Time:** 45–60 мин

**Пример решения / Sample solution (для менторов):**
```csharp
// C# 12 / .NET 8 — Program.cs
// Пакет: dotnet add package Spectre.Console
using Spectre.Console;

string name = args.Length > 0 ? args[0] : "Мир";

// Точка останова для отладки
System.Diagnostics.Debugger.Break();

AnsiConsole.MarkupLine($"[green]Привет, [yellow]{name}[/]![/] / Hello, [yellow]{name}[/]!");

var table = new Table()
    .AddColumn("Параметр / Param")
    .AddColumn("Значение / Value")
    .AddRow("Имя / Name", name)
    .AddRow(".NET", Environment.Version.ToString())
    .AddRow("ОС / OS", Environment.OSVersion.VersionString);

AnsiConsole.Write(table);
```

```bash
dotnet add package Spectre.Console
dotnet list package
dotnet run -- Anna
```

---

## Мини-проект модуля / Module mini-project

### Базовая версия / Base version
**Задача / Task (RU):** Консольная программа «О себе»: принимает имя и возраст через `args`, определяет ОС и версию .NET через `Environment` и выводит карточку-профиль.
**Task (EN):** Console program "About me": accepts name and age via `args`, detects OS and .NET version via `Environment`, and prints a profile card.

**Требования / Requirements:**
- Принимать `args`: `--name <имя>` и `--age <возраст>` (age как целое).
- Если `args` пусты — вывести подсказку и завершиться с кодом `1`.
- Получить ОС через `Environment.OSVersion` и версию .NET через `Environment.Version`.
- Вывести карточку профиля в виде выровненных строк с подписями RU+EN.
- Использовать top-level statements и интерполяцию строк.
- Parse `--name` and `--age` (integer) from `args`.
- Empty `args` → usage hint and exit code `1`.
- OS via `Environment.OSVersion`; .NET via `Environment.Version`.
- Print a profile card with aligned bilingual labels.
- Top-level statements + string interpolation.

**Критерии приёмки / Acceptance criteria:**
- [ ] `dotnet run -- --name Anna --age 30` выводит карточку со всеми полями.
- [ ] Без аргументов — подсказка и код возврата `1`.
- [ ] Возраст — целое число (некорректный ввод обрабатывается).
- [ ] Карточка содержит: имя, возраст, ОС, версию .NET.
- [ ] `--name`/`--age` produce the profile card.
- [ ] Empty args → usage + exit code 1.
- [ ] Card contains: name, age, OS, .NET version.

**Время / Time:** 60–90 мин

**Пример решения / Sample solution:**
```csharp
// C# 12 / .NET 8 — Program.cs (top-level statements)
if (args.Length == 0)
{
    Console.Error.WriteLine("Использование / Usage: dotnet run -- --name <имя> --age <возраст>");
    return 1;
}

string name = "Не указано / N/A";
int age = 0;

for (int i = 0; i < args.Length; i++)
{
    switch (args[i])
    {
        case "--name" when i + 1 < args.Length:
            name = args[++i];
            break;
        case "--age" when i + 1 < args.Length && int.TryParse(args[++i], out var a):
            age = a;
            break;
    }
}

var os = Environment.OSVersion;
var net = Environment.Version;

Console.WriteLine("==============================");
Console.WriteLine(" Карточка профиля / Profile card ");
Console.WriteLine("==============================");
Console.WriteLine($"{"Имя / Name",-20}: {name}");
Console.WriteLine($"{"Возраст / Age",-20}: {age}");
Console.WriteLine($"{"ОС / OS",-20}: {os}");
Console.WriteLine($"{".NET версия",-20}: {net}");
Console.WriteLine("==============================");

return 0;
```

---

### Pro-версия / Pro version
**Задача / Task (RU):** Расширить базовую версию: валидация `args` с понятными ошибками, switch-expression для ролей (`--role student|mentor|admin`), raw-string баннер, метрики GC.
**Task (EN):** Extend the base version: `args` validation with clear errors, switch-expression for roles (`--role student|mentor|admin`), raw-string banner, GC metrics.

**Доп. требования / Extra requirements:**
- Валидация `args`: обязательные `--name` и `--age`; `--age` в диапазоне 0–120; некорректный ввод → сообщение в `Console.Error` + код `2`.
- `--role` обрабатывается через switch-expression: каждой роли соответствует свой префикс приветствия и набор прав (строка).
- Raw-string литерал (`"""..."""`) для многострочного ASCII-баннера.
- Метрики GC: вывести `GC.CollectionCount(0/1/2)`, `GC.TotalMemory(true)`, `GC.MaxGeneration`.
- Код возврата: `0` — успех, `1` — нет аргументов, `2` — ошибка валидации.
- Validate `args`: `--name` and `--age` required; age 0–120; invalid → `Console.Error` + exit `2`.
- `--role` handled via switch-expression: each role gets a greeting prefix and permissions string.
- Raw-string literal (`"""..."""`) for a multi-line ASCII banner.
- GC metrics: `GC.CollectionCount(0/1/2)`, `GC.TotalMemory(true)`, `GC.MaxGeneration`.
- Exit codes: `0` success, `1` no args, `2` validation error.

**Критерии приёмки / Acceptance criteria:**
- [ ] Некорректный `--age` (например `abc` или `200`) → ошибка и код `2`.
- [ ] `--role mentor` меняет приветствие и права.
- [ ] Баннер выведен через raw-string литерал без экранирования.
- [ ] Выведены все три счётчика GC поколений и `TotalMemory`.
- [ ] Коды возврата: 0/1/2 соответствуют спецификации.
- [ ] Invalid `--age` → error + exit `2`.
- [ ] `--role mentor` changes greeting and permissions.
- [ ] Banner uses a raw-string literal (no escape sequences).
- [ ] All GC generation counters and `TotalMemory` printed.
- [ ] Exit codes 0/1/2 match the spec.

**Время / Time:** 90–120 мин

**Пример решения / Sample solution:**
```csharp
// C# 12 / .NET 8 — Program.cs (top-level statements)
using System.Runtime.InteropServices;

const int ExitOk = 0;
const int ExitNoArgs = 1;
const int ExitInvalid = 2;

if (args.Length == 0)
{
    Console.Error.WriteLine("Использование / Usage: dotnet run -- --name <имя> --age <0..120> --role student|mentor|admin");
    return ExitNoArgs;
}

string? name = null;
int? age = null;
string role = "student";

for (int i = 0; i < args.Length; i++)
{
    switch (args[i])
    {
        case "--name" when i + 1 < args.Length:
            name = args[++i];
            break;
        case "--age" when i + 1 < args.Length:
            if (!int.TryParse(args[++i], out var a) || a < 0 || a > 120)
            {
                Console.Error.WriteLine($"Ошибка / Error: неверный возраст / invalid age: '{args[i]}'");
                return ExitInvalid;
            }
            age = a;
            break;
        case "--role" when i + 1 < args.Length:
            role = args[++i];
            break;
    }
}

if (string.IsNullOrWhiteSpace(name) || age is null)
{
    Console.Error.WriteLine("Ошибка / Error: обязательны --name и --age");
    return ExitInvalid;
}

// Switch-expression для ролей
var (greeting, perms) = role switch
{
    "student" => ("Студент", "read"),
    "mentor"  => ("Ментор",  "read;review"),
    "admin"   => ("Админ",   "read;write;admin"),
    _         => ("Гость",   "read")
};

// Raw-string баннер
var banner = """
   ____  __  ______  ____  ____  ___
  / __ \/ / / / __ \/ __ \/ __ \/ _ \
 / / / / /_/ / / / / / / / / / / , _/
/_/ /_/\____/_/ /_/_/ /_/_/ /_/_/|_|
""";

Console.WriteLine(banner);
Console.WriteLine();
Console.WriteLine("==============================");
Console.WriteLine(" Карточка профиля / Profile card ");
Console.WriteLine("==============================");
Console.WriteLine($"{"Имя / Name",-22}: {name}");
Console.WriteLine($"{"Возраст / Age",-22}: {age}");
Console.WriteLine($"{"Роль / Role",-22}: {greeting} ({perms})");
Console.WriteLine($"{"ОС / OS",-22}: {Environment.OSVersion}");
Console.WriteLine($".NET {"",-17}: {RuntimeInformation.FrameworkDescription}");
Console.WriteLine("==============================");

// Метрики GC
Console.WriteLine();
Console.WriteLine("Метрики GC / GC metrics:");
Console.WriteLine($"  MaxGeneration      : {GC.MaxGeneration}");
Console.WriteLine($"  Gen0 collections   : {GC.CollectionCount(0)}");
Console.WriteLine($"  Gen1 collections   : {GC.CollectionCount(1)}");
Console.WriteLine($"  Gen2 collections   : {GC.CollectionCount(2)}");
Console.WriteLine($"  Total memory (KB)  : {GC.GetTotalMemory(forceFullCollection: true) / 1024.0:F2}");

return ExitOk;
```
