---
[← К уроку M01-L04](lesson-M01-L04-dotnet-new-toplevel.md) | [⬆ К модулю M01](../README.md) | [Следующее ДЗ →](homework-M01-L05-build-run.md)
---

### Домашнее задание M01-L04: dotnet new, структура проекта, top-level statements / Homework M01-L04: dotnet new, project structure, top-level statements

**Урок / Lesson:** M01-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться создавать консольный проект через `dotnet new`, читать и править `.csproj`, писать лаконичную точку входа на top-level statements с локальными функциями, интерполяцией, pattern matching, raw-строками и обработкой аргументов командной строки, а также осознанно работать со служебными каталогами `obj/` и `bin/`. (EN) Learn to scaffold a console project with `dotnet new`, read and edit `.csproj`, write a concise top-level-statements entry point with local functions, interpolation, pattern matching, raw strings and command-line argument handling, and reason about the `obj/` and `bin/` service directories.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет все ключевые темы урока: шаблон `dotnet new console`, устройство `.csproj` (`OutputType`, `TargetFramework`, `ImplicitUsings`, `Nullable`, `LangVersion`), top-level statements с ограничением «один файл на проект» и ошибкой `CS7022`, виды `using` (явный, `global using`, `using static`, implicit usings), file-scoped namespace, назначение `obj/` и `bin/`, а также best practices и частые ошибки из урока. Студент не просто повторяет команды, а сознательно воспроизводит каждую тонкость.
(EN) The homework directly reinforces every key topic of the lesson: the `dotnet new console` template, the anatomy of `.csproj` (`OutputType`, `TargetFramework`, `ImplicitUsings`, `Nullable`, `LangVersion`), top-level statements with the one-file-per-project rule and the `CS7022` error, the kinds of `using` (explicit, `global using`, `using static`, implicit usings), file-scoped namespaces, the role of `obj/` and `bin/`, plus the lesson's best practices and common mistakes. The student consciously reproduces each nuance rather than merely copying commands.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы присоединились к команде, которая автоматизирует сборку небольших внутренних утилит на .NET 8. Ваш первый таск — подготовить каркас консольного приложения `GreeterApp`, которое будет принимать имя пользователя и режим запуска через аргументы командной строки, формировать приветствие и сводку по переданным параметрам, а также возвращать осмысленный код завершения процесса. Команда придерживается современных соглашений: минимум церемониального кода, top-level statements в качестве точки входа, включённые `ImplicitUsings` и `Nullable`, file-scoped namespace во всех новых файлах, осознанный выбор `TargetFramework=net8.0`. Прежде чем писать бизнес-логику, вы обязаны пройти весь путь «от пустой папки до работающего приложения»: создать проект правильной командой, изучить сгенерированные файлы и каталоги, осмыслить каждую строку `.csproj`, а затем дописать код, демонстрирующий ключевые возможности C# 12 — коллекционные выражения, pattern matching, raw-строковые литералы, локальные функции и `using static`. Это задание не про «сделать, чтобы работало», а про понимание устройства проекта: почему файл проекта выглядит именно так, что именно генерирует компилятор за вас, где лежат артефакты сборки и почему их не коммитят. Попутно вы на практике столкнётесь с частыми ошибками — попыткой завести второй файл с top-level statements (и увидите `CS7022`), конфликтом при объявлении `args` и необходимостью чистить `obj/` и `bin/`. В результате у вас сформируется мышечная память по базовой структуре любого .NET-проекта, которая понадобится во всех последующих уроках модуля.

#### Что нужно сделать (пошагово)

1. **Подготовьте рабочее место.** Создайте чистую папку `M01-L04-work` вне репозитория курса (например, рядом с ним) и перейдите в неё. Проверьте текущий каталог командой `pwd` (или `cd` без аргументов на Windows), чтобы не создать проект в неожиданном месте — это одна из частых ошибок урока.

2. **Создайте проект правильной командой.** Выполните `dotnet new console -n GreeterApp`. Флаг `-n` одновременно задаёт имя проекта и имя папки. Если вы выполните `dotnet new console` без `-n`, проект создастся в текущей папке и получит её имя — это допустимо, но в задании требуется именно `GreeterApp`. После выполнения у вас должны появиться: `GreeterApp/GreeterApp.csproj`, `GreeterApp/Program.cs` и служебные `GreeterApp/obj/` (папка `bin/` появится после первой сборки).

3. **Изучите `GreeterApp.csproj`.** Откройте его в редакторе. Убедитесь, что внутри есть `<Project Sdk="Microsoft.NET.Sdk">`, `<PropertyGroup>`, `<OutputType>Exe</OutputType>`, `<TargetFramework>net8.0</TargetFramework>`, `<ImplicitUsings>enable</ImplicitUsings>` и `<Nullable>enable</Nullable>`. При необходимости добавьте `<LangVersion>latest</LangVersion>`, чтобы гарантированно использовать C# 12. Запомните: `.csproj` — это XML-«паспорт» проекта, код в нём не живёт, но без него компилятор не узнает, что собирать.

4. **Прочитайте `Program.cs`.** В шаблоне по умолчанию будет одна-две строки top-level statements. Убедитесь, что в файле нет ни `class Program`, ни `static void Main` — компилятор синтезирует `Main` за вас. Запомните правило: в проекте может быть максимум один файл с top-level statements.

5. **Напишите новую точку входа.** Замените содержимое `Program.cs` кодом из раздела «Эталонное решение» ниже. Код должен: объявлять локальную функцию `FormatGreeting`, использовать коллекционное выражение `string[] modes = ["short", "full"];`, применять pattern matching для разбора аргумента режима, формировать многострочный баннер через raw string literal `"""..."""`, выводить результат через `using static System.Console` (чтобы писать `WriteLine` без `Console.`), обрабатывать `args` без его объявления и возвращать код завершения через `return`.

6. **Добавьте второй исходный файл.** Создайте `GreeterApp/Metadata.cs` с file-scoped namespace `namespace GreeterApp;` и статическим классом `Metadata`, содержащим метод `string BuildSummary(string name, int argCount)`. Вызовите его из `Program.cs`. Убедитесь, что во втором файле **нет** top-level statements — иначе вы получите ошибку `CS7022`.

7. **Эксперимент с ошибкой `CS7022` (обязательно).** Временно добавьте во `Metadata.cs` строку `System.Console.WriteLine("hi");` на верхнем уровне (вне класса), соберите проект через `dotnet build` и зафиксируйте ошибку `CS7022: Program has more than one entry point`. После этого уберите строку и пересоберите. В отчёт запишите точный текст ошибки и причину: двух файлов с top-level statements в одном проекте быть не может.

8. **Соберите и запустите.** Выполните `dotnet build GreeterApp/GreeterApp.csproj`. Убедитесь, что появилась папка `bin/Debug/net8.0/` с `GreeterApp.dll` и (на Windows) `GreeterApp.exe`. Запустите приложение несколькими способами: `dotnet run --project GreeterApp` без аргументов; с аргументами `--project GreeterApp -- Alice full`; напрямую `dotnet GreeterApp/bin/Debug/net8.0/GreeterApp.dll Alice full`. Зафиксируйте выходной код через `echo $?` (bash) или `$LASTEXITCODE` (PowerShell) — он должен быть `0` при успехе и `2` при неизвестном режиме.

9. **Проверьте служебные каталоги.** Удалите `GreeterApp/bin` и `GreeterApp/obj` вручную (или через `dotnet clean GreeterApp`), затем выполните `dotnet build` снова — каталоги пересоздадутся. Убедитесь, что в `.gitignore` (если инициализируете git) попадают `bin/` и `obj/`.

10. **Ответьте в отчёте.** Коротко (по 1–2 предложения): зачем нужен `OutputType=Exe`; чем отличается явный `using` от `global using` и от implicit usings; почему file-scoped namespace предпочтительнее классического; что лежит в `obj/` и что в `bin/`.

#### Требования к решению

Решение должно представлять собой проект `GreeterApp`, созданный строго командой `dotnet new console -n GreeterApp`, с `TargetFramework=net8.0`, `ImplicitUsings=enable`, `Nullable=enable` и `LangVersion=latest`. Точка входа обязана использовать top-level statements (классический `Main` запрещён). В `Program.cs` должны быть продемонстрированы: локальная функция, коллекционное выражение (`[...]`), pattern matching (минимум `switch`-выражение или `is`-pattern), raw string literal (`"""..."""`), интерполяция строк, использование `args` без его объявления и `return` для кода завершения. Обязательно наличие второго файла `Metadata.cs` с file-scoped namespace и типом, вызываемым из точки входа; в этом файле top-level statements запрещены. Программа должна корректно обрабатывать три случая: нет аргументов (приветствие по умолчанию), один аргумент-имя, имя плюс режим `short|full`; при неизвестном режиме — возвращать код `2`. Сборка должна проходить без предупреждений, связанных с nullability. Служебные каталоги `bin/` и `obj/` не должны попасть в систему контроля версий; в отчёте требуется подтвердить знание команды `dotnet clean` и ручного удаления этих папок. Все лучшие практики урока должны быть соблюдены: `Program.cs` остаётся минимальным, явные `using` добавляются только для того, что не покрывает implicit usings, `TargetFramework` выбран осознанно.

#### Тонкости и подводные камни

- **Два файла с top-level statements → `CS7022`.** Это самая частая ошибка урока. Компилятор синтезирует `Main` для каждого такого файла и не может выбрать вход. Решение: top-level statements — только в `Program.cs`, во всех остальных файлах — классы/структуры внутри namespace.
- **Объявление `args` в top-level файле.** Если написать `string[] args = ...;`, компилятор сообщит о конфликте с неявным параметром синтетического `Main`. Просто используйте `args` как уже существующую переменную.
- **`using static System.Console`.** После него можно писать `WriteLine(...)` без `Console.`. Но будьте осторожны в больших файлах: читаемость может упасть. В задании это допустимо для демонстрации.
- **Raw string literal требует минимум три кавычки `"""` и аккуратных отступов.** Закрывающие `"""` должны быть выровнены так, чтобы компилятор корректно определил общий отступ; иначе в строку попадут лишние пробелы. Если внутри строки есть сами кавычки — используйте четыре или больше `"` в ограничителях.
- **`ImplicitUsings=enable` добавляет `System`, `System.IO`, `System.Linq` и др. через автогенерируемый `GlobalUsings.cs`.** Не дублируйте их явными `using System;` — это бессмысленный шум. Явный `using` нужен только для редких пространств (например, `System.Globalization`, `System.Text.RegularExpressions`).
- **`<Nullable>enable</Nullable>` с первого дня.** Компилятор будет выдавать предупреждения на потенциально-null значения. Не глушите их `!` бездумно — лучше обрабатывайте.
- **Правка `.csproj` вручную.** Любая опечатка в XML (незакрытый тег, лишний пробел в имени) ломает сборку с непонятным сообщением. Проверяйте закрытие тегов и используйте IDE с подсветкой XML.
- **Удаление `obj/` во время активной сборки.** Файлы заблокированы процессом; сначала остановите `dotnet build`/отладку, затем чистите.
- **`TargetFramework=net8.0` — это LTS.** Не используйте `net6.0`/`net7.0` без причины. Если в команде кто-то соберёт под старый фреймворк, новые возможности C# 12 могут быть недоступны или потребуют `LangVersion` явным образом.
- **`return` из top-level задаёт `int` exit code.** Если ничего не возвращать, процесс завершится с кодом `0`. Это полезно для скриптов и CI.

#### Критерии приёмки

- [ ] Проект создан командой `dotnet new console -n GreeterApp` (в отчёте — точная команда и вывод).
- [ ] `GreeterApp.csproj` содержит `OutputType=Exe`, `TargetFramework=net8.0`, `ImplicitUsings=enable`, `Nullable=enable`, `LangVersion=latest`.
- [ ] `Program.cs` использует top-level statements, без `class Program` и `Main`.
- [ ] В `Program.cs` есть локальная функция, коллекционное выражение, pattern matching, raw string literal, интерполяция.
- [ ] Использован `using static System.Console` и явный `using` для редкого namespace.
- [ ] `args` используется без объявления; `return` задаёт код завершения.
- [ ] Создан `Metadata.cs` с file-scoped namespace `namespace GreeterApp;` и типом, вызываемым из `Program.cs`.
- [ ] В `Metadata.cs` нет top-level statements.
- [ ] Воспроизведена и описана ошибка `CS7022` (добавление второго top-level файла, сборка, фиксация, откат).
- [ ] `dotnet build` проходит без ошибок; `bin/Debug/net8.0/GreeterApp.dll` существует.
- [ ] Программа корректно отрабатывает три сценария: без аргументов, с именем, с именем и режимом `short|full`.
- [ ] При неизвестном режиме процесс возвращает код `2`; при успехе — `0`.
- [ ] `bin/` и `obj/` удалены и пересозданы через `dotnet build`; описано поведение `dotnet clean`.
- [ ] В отчёте даны ответы на вопросы про `OutputType`, виды `using`, file-scoped namespace, назначение `obj/bin`.
- [ ] В `.gitignore` учтены `bin/` и `obj/` (если инициализирован git).
- [ ] Код содержит двуязычные комментарии RU+EN в ключевых местах.

#### Подсказки (без прямого ответа)

- Вспомните, что top-level файл — это «тело метода `Main`»: всё, что вы пишете на верхнем уровне, выполняется при запуске.
- Для pattern matching по режиму удобен `switch`-выражение, возвращающий строку-шаблон приветствия.
- Raw string literal начинается и заканчивается тремя кавычками; отступ закрывающей строки задаёт «срезаемый» отступ всего содержимого.
- Коллекционное выражение `["short", "full"]` создаёт массив — его можно передавать в `Contains` или использовать в pattern matching.
- Чтобы не дублировать implicit usings, загляните в автогенерируемый `obj/Debug/net8.0/GreeterApp.GlobalUsings.g.cs` — там видно, что уже подключено.
- Если `args.Length == 0`, используйте значение по умолчанию для имени (например, `"World"`), но это не освобождает от возврата корректного exit code.
- Помните: локальная функция в top-level файле компилируется как обычный метод синтетического класса.

#### Эталонное решение (разбор)

```csharp
// Program.cs — точка входа GreeterApp на C# 12 / .NET 8
// Entry point of GreeterApp using top-level statements (C# 12 / .NET 8)

using System.Globalization;        // явный using для CultureInfo / explicit using
using static System.Console;       // статичный using: WriteLine без Console. / static using

// Top-level statements: инструкции выполняются напрямую, без Main.
// Top-level statements: statements run directly, no Main needed.

string appName = "GreeterApp";
DateTime builtAt = DateTime.UtcNow;

// Коллекционное выражение для списка режимов (C# 12).
// Collection expression for the list of modes (C# 12).
string[] knownModes = ["short", "full"];

// args доступен неявно — объявлять его нельзя.
// `args` is available implicitly; do not declare it.
string name = args.Length > 0 ? args[0] : "World";
string mode = args.Length > 1 ? args[1] : "short";

// Локальная функция прямо в top-level файле.
// A local function is allowed directly in a top-level file.
string FormatGreeting(string who, string how) => how switch
{
    "short" => $"Hi, {who}!",
    "full"  => $"Hello, {who}! Welcome to {appName}.",
    _       => $"Hello, {who}."
};

// Pattern matching: проверка режима.
// Pattern matching: validate the mode.
if (!knownModes.Contains(mode))
{
    // Явный using System.Globalization + invariant culture для стабильного вывода.
    // Explicit System.Globalization using + invariant culture for stable output.
    WriteLine(CultureInfo.InvariantCulture, $"Unknown mode: '{mode}'. Expected one of {string.Join(", ", knownModes)}.");
    return 2; // код завершения процесса / process exit code
}

string greeting = FormatGreeting(name, mode);

// Raw string literal для многострочного баннера (C# 11+, работает в C# 12).
// Raw string literal for a multi-line banner.
string banner = """
    ┌──────────────────────────────┐
    │  GreeterApp · dotnet new     │
    └──────────────────────────────┘
    """;

WriteLine(banner);
WriteLine(CultureInfo.InvariantCulture, $"{greeting} (built at {builtAt:O})");

// Вызов типа из второго файла с file-scoped namespace.
// Call a type from a second file with a file-scoped namespace.
string summary = GreeterApp.Metadata.BuildSummary(name, args.Length);
WriteLine(summary);

return 0; // успех / success
```

```csharp
// Metadata.cs — второй файл проекта, file-scoped namespace, без top-level statements.
// Second file of the project, file-scoped namespace, no top-level statements.

namespace GreeterApp; // file-scoped namespace (C# 10+) — одна строка, меньше отступов
                     // file-scoped namespace — single line, fewer indents

internal static class Metadata
{
    // Метод вызывается из точки входа в Program.cs.
    // This method is called from the entry point in Program.cs.
    internal static string BuildSummary(string name, int argCount) =>
        $"Caller: {name}, args received: {argCount}.";
}
```

```xml
<!-- GreeterApp.csproj — паспорт проекта / project passport -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>              <!-- исполняемый файл / executable -->
    <TargetFramework>net8.0</TargetFramework> <!-- целевой фреймворк LTS / target LTS -->
    <ImplicitUsings>enable</ImplicitUsings>   <!-- неявные using / implicit usings -->
    <Nullable>enable</Nullable>               <!-- nullable-аннотации / nullable annotations -->
    <LangVersion>latest</LangVersion>         <!-- C# 12+ -->
  </PropertyGroup>

</Project>
```

**Разбор по строкам.** Команда `dotnet new console -n GreeterApp` создаёт скелет, в котором `OutputType=Exe` указывает компилятору, что результат — исполняемый файл (а не библиотека), а `TargetFramework=net8.0` фиксирует актуальный LTS. `ImplicitUsings=enable` добавляет частые пространства имён через автогенерируемый `GlobalUsings.g.cs`, поэтому в `Program.cs` нет `using System;` — это одна из лучших практик урока: явные `using` только для редкого, в нашем случае `System.Globalization` для `CultureInfo.InvariantCulture`. `using static System.Console` импортирует статические члены класса `Console`, позволяя писать `WriteLine` без префикса. Top-level statements означают, что компилятор сам оборачивает инструкции в синтетический `Main(string[] args)`, поэтому переменная `args` уже доступна, и объявлять её нельзя — частая ошибка урока. Локальная функция `FormatGreeting` демонстрирует, что в top-level файле можно объявлять методы; она использует `switch`-выражение (pattern matching) для выбора шаблона. Коллекционное выражение `["short", "full"]` — синтаксис C# 12, создающий массив. Raw string literal `"""..."""` позволяет включать многострочный текст с символами рамки без экранирования; выравнивание закрывающих кавычек определяет срезаемый отступ. Проверка `knownModes.Contains(mode)` защищает от неизвестного режима и возвращает `return 2` — это задаёт exit code процесса, что полезно для скриптов и CI. Второй файл `Metadata.cs` иллюстрирует file-scoped namespace `namespace GreeterApp;` (современная форма с точкой с запятой, экономит отступ) и класс `Metadata`, вызываемый из точки входа; здесь top-level statements запрещены — иначе компилятор выдаёт `CS7022` (две точки входа). Сборка через `dotnet build` кладёт вывод в `bin/Debug/net8.0/`, промежуточные артефакты — в `obj/`; эти папки не коммитятся и легко пересоздаются. Таким образом, решение покрывает все темы урока: шаблон `dotnet new`, структуру `.csproj`, top-level statements, виды `using`, file-scoped namespace и служебные каталоги.

#### Задания на углубление (бонус)

1. **Global using вручную.** Добавьте в проект файл `Usings.cs` со строкой `global using System.Text.RegularExpressions;` и используйте `Regex` в `Program.cs` для проверки, что имя состоит только из букв. Объясните, чем `global using` отличается от обычного и от implicit usings.
2. **Второй проект-библиотека.** Создайте рядом `dotnet new classlib -n GreeterLib`, поменяйте в нём `OutputType` (по умолчанию `Library`) и добавьте ссылку из `GreeterApp` через `dotnet add GreeterApp reference GreeterLib`. Вызовите из библиотеки метод в `Program.cs`. Сравните `OutputType=Exe` и `Library`.
3. **Классический Main для сравнения.** В отдельной папке создайте проект с классическим `class Program { static void Main() { ... } }` (без top-level) и сравните объём церемониального кода. Опишите, какие преимущества даёт top-level форма и когда всё-таки стоит вернуться к классической (например, для перегрузок `Main` или асинхронной точки входа с `async Task Main`).
4. **Управление exit code.** Расширьте программу: код `1` — если имя пустое, `2` — неизвестный режим, `3` — слишком много аргументов. Напишите скрипт (bash или PowerShell), который запускает приложение с разными аргументами и проверяет `$?`/`$LASTEXITCODE`.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you have just joined a team that automates the build of small internal .NET 8 utilities. Your first task is to prepare the skeleton of a console application called `GreeterApp` that will accept a user name and a run mode through command-line arguments, produce a greeting and a summary of the parameters passed in, and return a meaningful process exit code. The team follows modern conventions: minimal ceremony, top-level statements as the entry point, `ImplicitUsings` and `Nullable` enabled, file-scoped namespaces in every new file, and a deliberate choice of `TargetFramework=net8.0`. Before writing any business logic you must walk the whole path "from an empty folder to a working application": create the project with the correct command, study the generated files and directories, reason about every line of `.csproj`, and then write code that demonstrates the key features of C# 12 — collection expressions, pattern matching, raw string literals, local functions and `using static`. This assignment is not about "making it work"; it is about understanding the anatomy of a project: why the project file looks the way it does, what the compiler synthesizes on your behalf, where build artifacts live and why they are not committed. Along the way you will reproduce, on purpose, the common mistakes described in the lesson — attempting to introduce a second top-level-statements file (and observing `CS7022`), clashing with `args` by declaring it, and cleaning `obj/` and `bin/`. By the end you will have built muscle memory for the basic structure of any .NET project, which you will rely on in every subsequent lesson of the module.

#### What to do step by step

1. **Prepare the workspace.** Create a clean folder `M01-L04-work` outside the course repository (for example next to it) and navigate into it. Verify the current directory with `pwd` (or `cd` without arguments on Windows) so the project does not land somewhere unexpected — this is one of the common mistakes from the lesson.

2. **Create the project with the correct command.** Run `dotnet new console -n GreeterApp`. The `-n` flag sets both the project name and the folder name. If you run `dotnet new console` without `-n`, the project is created in the current folder and takes the folder's name — that is acceptable in general, but this assignment specifically requires `GreeterApp`. After execution you should have: `GreeterApp/GreeterApp.csproj`, `GreeterApp/Program.cs`, and the service folder `GreeterApp/obj/` (the `bin/` folder appears after the first build).

3. **Study `GreeterApp.csproj`.** Open it in an editor. Confirm it contains `<Project Sdk="Microsoft.NET.Sdk">`, `<PropertyGroup>`, `<OutputType>Exe</OutputType>`, `<TargetFramework>net8.0</TargetFramework>`, `<ImplicitUsings>enable</ImplicitUsings>`, and `<Nullable>enable</Nullable>`. Add `<LangVersion>latest</LangVersion>` if needed to guarantee C# 12. Remember: `.csproj` is the XML "passport" of the project; no code lives in it, yet without it the compiler would not know what to build.

4. **Read `Program.cs`.** The default template contains one or two lines of top-level statements. Confirm there is neither a `class Program` nor a `static void Main` — the compiler synthesizes `Main` for you. Remember the rule: a project may have at most one file with top-level statements.

5. **Write the new entry point.** Replace the contents of `Program.cs` with the code from the "Reference solution" section below. The code must: declare a local function `FormatGreeting`, use a collection expression `string[] modes = ["short", "full"];`, apply pattern matching to parse the mode argument, build a multi-line banner with a raw string literal `"""..."""`, output the result through `using static System.Console` (so you can call `WriteLine` without the `Console.` prefix), handle `args` without declaring it, and return an exit code via `return`.

6. **Add a second source file.** Create `GreeterApp/Metadata.cs` with a file-scoped namespace `namespace GreeterApp;` and a static class `Metadata` containing a method `string BuildSummary(string name, int argCount)`. Call it from `Program.cs`. Make sure the second file does **not** contain top-level statements — otherwise you will hit `CS7022`.

7. **The `CS7022` experiment (mandatory).** Temporarily add the line `System.Console.WriteLine("hi");` at the top level of `Metadata.cs` (outside any class), build the project with `dotnet build`, and record the error `CS7022: Program has more than one entry point`. Then remove the line and rebuild. In your report write down the exact error text and the reason: there cannot be two files with top-level statements in one project.

8. **Build and run.** Execute `dotnet build GreeterApp/GreeterApp.csproj`. Confirm that the folder `bin/Debug/net8.0/` was created with `GreeterApp.dll` and (on Windows) `GreeterApp.exe`. Run the application in several ways: `dotnet run --project GreeterApp` with no arguments; with arguments `--project GreeterApp -- Alice full`; directly `dotnet GreeterApp/bin/Debug/net8.0/GreeterApp.dll Alice full`. Capture the exit code via `echo $?` (bash) or `$LASTEXITCODE` (PowerShell) — it must be `0` on success and `2` on an unknown mode.

9. **Inspect the service directories.** Delete `GreeterApp/bin` and `GreeterApp/obj` by hand (or run `dotnet clean GreeterApp`), then run `dotnet build` again — the folders will be recreated. Confirm that `bin/` and `obj/` are covered by `.gitignore` if you initialize git.

10. **Answer in the report.** Briefly (one or two sentences each): why `OutputType=Exe` is needed; the difference between an explicit `using`, a `global using`, and implicit usings; why a file-scoped namespace is preferable to the classic form; what lives in `obj/` and what lives in `bin/`.

#### Requirements

The solution must be a project `GreeterApp` created strictly with `dotnet new console -n GreeterApp`, with `TargetFramework=net8.0`, `ImplicitUsings=enable`, `Nullable=enable`, and `LangVersion=latest`. The entry point must use top-level statements (a classic `Main` is forbidden). `Program.cs` must demonstrate: a local function, a collection expression (`[...]`), pattern matching (at least a `switch` expression or an `is` pattern), a raw string literal (`"""..."""`), string interpolation, use of `args` without declaring it, and a `return` for the exit code. A second file `Metadata.cs` with a file-scoped namespace and a type called from the entry point is mandatory; top-level statements are forbidden in that file. The program must correctly handle three cases: no arguments (default greeting), a single name argument, a name plus a mode `short|full`; on an unknown mode it must return exit code `2`. The build must complete without nullability-related warnings. The service folders `bin/` and `obj/` must not be committed to version control; the report must confirm knowledge of `dotnet clean` and of manual deletion of these folders. All best practices from the lesson must be observed: `Program.cs` stays minimal, explicit `using` lines are added only for namespaces that implicit usings do not already cover, and `TargetFramework` is chosen deliberately.

#### Pitfalls

- **Two files with top-level statements → `CS7022`.** The most common mistake of the lesson. The compiler synthesizes a `Main` for each such file and cannot pick the entry point. Fix: top-level statements live only in `Program.cs`; every other file contains types inside a namespace.
- **Declaring `args` in a top-level file.** If you write `string[] args = ...;` the compiler reports a conflict with the implicit parameter of the synthetic `Main`. Just use `args` as an already-existing variable.
- **`using static System.Console`.** After it you can write `WriteLine(...)` without `Console.`. Be careful in large files, though — readability may suffer. In this assignment it is acceptable as a demonstration.
- **A raw string literal requires at least three quotes `"""` and careful indentation.** The closing `"""` must be aligned so the compiler can determine the common indentation; otherwise extra spaces leak into the string. If the content itself contains quotes, use four or more `"` as the delimiter.
- **`ImplicitUsings=enable` adds `System`, `System.IO`, `System.Linq` and others through an auto-generated `GlobalUsings.cs`.** Do not duplicate them with explicit `using System;` — that is pointless noise. An explicit `using` is only needed for rare namespaces (for example `System.Globalization` or `System.Text.RegularExpressions`).
- **`<Nullable>enable</Nullable>` from day one.** The compiler will warn about potentially null values. Do not suppress them with `!` blindly — handle them properly.
- **Hand-editing `.csproj`.** Any XML typo (an unclosed tag, a stray space in a name) breaks the build with a cryptic message. Verify tag closure and use an IDE with XML highlighting.
- **Deleting `obj/` during an active build.** Files are locked by the process; stop `dotnet build`/debugging first, then clean.
- **`TargetFramework=net8.0` is LTS.** Do not use `net6.0`/`net7.0` without a reason. If someone builds for an older framework, C# 12 features may be unavailable or may require `LangVersion` explicitly.
- **`return` from top-level sets the `int` exit code.** If you return nothing, the process exits with code `0`. This is useful for scripts and CI.

#### Acceptance criteria

- [ ] The project is created with `dotnet new console -n GreeterApp` (the report shows the exact command and output).
- [ ] `GreeterApp.csproj` contains `OutputType=Exe`, `TargetFramework=net8.0`, `ImplicitUsings=enable`, `Nullable=enable`, `LangVersion=latest`.
- [ ] `Program.cs` uses top-level statements, with no `class Program` and no `Main`.
- [ ] `Program.cs` contains a local function, a collection expression, pattern matching, a raw string literal, and interpolation.
- [ ] `using static System.Console` is used, plus an explicit `using` for a rare namespace.
- [ ] `args` is used without being declared; `return` sets the exit code.
- [ ] `Metadata.cs` is created with a file-scoped namespace `namespace GreeterApp;` and a type called from `Program.cs`.
- [ ] `Metadata.cs` has no top-level statements.
- [ ] The `CS7022` error is reproduced and described (adding a second top-level file, building, recording, reverting).
- [ ] `dotnet build` succeeds; `bin/Debug/net8.0/GreeterApp.dll` exists.
- [ ] The program handles three scenarios correctly: no arguments, a name, a name plus a mode `short|full`.
- [ ] On an unknown mode the process returns code `2`; on success — `0`.
- [ ] `bin/` and `obj/` are deleted and recreated via `dotnet build`; the behavior of `dotnet clean` is described.
- [ ] The report answers the questions about `OutputType`, the kinds of `using`, file-scoped namespaces, and the purpose of `obj/bin`.
- [ ] `.gitignore` covers `bin/` and `obj/` (if git is initialized).
- [ ] The code has bilingual RU+EN comments in key places.

#### Hints (no direct answer)

- Recall that a top-level file is effectively "the body of `Main`": everything you write at the top level runs when the program starts.
- For matching the mode, a `switch` expression returning a greeting template is convenient.
- A raw string literal starts and ends with three quotes; the indentation of the closing line defines the indentation stripped from the content.
- The collection expression `["short", "full"]` produces an array you can pass to `Contains` or use in pattern matching.
- To avoid duplicating implicit usings, look at the auto-generated `obj/Debug/net8.0/GreeterApp.GlobalUsings.g.cs` to see what is already included.
- If `args.Length == 0`, use a default for the name (for example `"World"`), but still return a correct exit code.
- Remember: a local function in a top-level file is compiled as a regular method of the synthetic class.

#### Reference solution walk-through

```csharp
// Program.cs — entry point of GreeterApp (C# 12 / .NET 8)
// Entry point of GreeterApp using top-level statements (C# 12 / .NET 8)

using System.Globalization;        // explicit using for CultureInfo
using static System.Console;       // static using: WriteLine without the Console. prefix

// Top-level statements: statements run directly, no Main needed.
// Top-level statements: statements run directly, no Main needed.

string appName = "GreeterApp";
DateTime builtAt = DateTime.UtcNow;

// Collection expression for the list of modes (C# 12).
// Collection expression for the list of modes (C# 12).
string[] knownModes = ["short", "full"];

// `args` is available implicitly; do not declare it.
// `args` is available implicitly; do not declare it.
string name = args.Length > 0 ? args[0] : "World";
string mode = args.Length > 1 ? args[1] : "short";

// A local function is allowed directly in a top-level file.
// A local function is allowed directly in a top-level file.
string FormatGreeting(string who, string how) => how switch
{
    "short" => $"Hi, {who}!",
    "full"  => $"Hello, {who}! Welcome to {appName}.",
    _       => $"Hello, {who}."
};

// Pattern matching: validate the mode.
// Pattern matching: validate the mode.
if (!knownModes.Contains(mode))
{
    // Explicit System.Globalization using + invariant culture for stable output.
    // Explicit System.Globalization using + invariant culture for stable output.
    WriteLine(CultureInfo.InvariantCulture, $"Unknown mode: '{mode}'. Expected one of {string.Join(", ", knownModes)}.");
    return 2; // process exit code
}

string greeting = FormatGreeting(name, mode);

// Raw string literal for a multi-line banner (C# 11+, works in C# 12).
// Raw string literal for a multi-line banner.
string banner = """
    ┌──────────────────────────────┐
    │  GreeterApp · dotnet new     │
    └──────────────────────────────┘
    """;

WriteLine(banner);
WriteLine(CultureInfo.InvariantCulture, $"{greeting} (built at {builtAt:O})");

// Call a type from a second file with a file-scoped namespace.
// Call a type from a second file with a file-scoped namespace.
string summary = GreeterApp.Metadata.BuildSummary(name, args.Length);
WriteLine(summary);

return 0; // success
```

```csharp
// Metadata.cs — second file of the project, file-scoped namespace, no top-level statements.
// Second file of the project, file-scoped namespace, no top-level statements.

namespace GreeterApp; // file-scoped namespace (C# 10+) — single line, fewer indents
                     // file-scoped namespace — single line, fewer indents

internal static class Metadata
{
    // This method is called from the entry point in Program.cs.
    // This method is called from the entry point in Program.cs.
    internal static string BuildSummary(string name, int argCount) =>
        $"Caller: {name}, args received: {argCount}.";
}
```

```xml
<!-- GreeterApp.csproj — project passport -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>              <!-- executable -->
    <TargetFramework>net8.0</TargetFramework> <!-- target LTS framework -->
    <ImplicitUsings>enable</ImplicitUsings>   <!-- implicit usings -->
    <Nullable>enable</Nullable>               <!-- nullable annotations -->
    <LangVersion>latest</LangVersion>         <!-- C# 12+ -->
  </PropertyGroup>

</Project>
```

**Line-by-line walk-through.** The command `dotnet new console -n GreeterApp` produces a skeleton in which `OutputType=Exe` tells the compiler the output is an executable (not a library), and `TargetFramework=net8.0` pins the current LTS. `ImplicitUsings=enable` pulls in common namespaces through the auto-generated `GlobalUsings.g.cs`, which is why `Program.cs` has no `using System;` — this is one of the lesson's best practices: explicit `using` lines only for the rare stuff, in our case `System.Globalization` for `CultureInfo.InvariantCulture`. The `using static System.Console` import brings the static members of `Console` into scope, allowing `WriteLine` without the prefix. Top-level statements mean the compiler wraps the instructions in a synthetic `Main(string[] args)`, so the `args` variable is already available and must not be declared — a common mistake from the lesson. The local function `FormatGreeting` shows that methods can be declared in a top-level file; it uses a `switch` expression (pattern matching) to pick a template. The collection expression `["short", "full"]` is C# 12 syntax that creates an array. The raw string literal `"""..."""` lets you embed multi-line text with box-drawing characters without escaping; the alignment of the closing quotes defines the indentation stripped from the content. The `knownModes.Contains(mode)` check guards against an unknown mode and returns `return 2`, which sets the process exit code — useful for scripts and CI. The second file `Metadata.cs` illustrates a file-scoped namespace `namespace GreeterApp;` (the modern semicolon form, saving an indent) and a `Metadata` class called from the entry point; top-level statements are forbidden here — otherwise the compiler raises `CS7022` (two entry points). Building with `dotnet build` places the output in `bin/Debug/net8.0/` and intermediate artifacts in `obj/`; these folders are not committed and are easily recreated. Thus the solution covers every topic of the lesson: the `dotnet new` template, the structure of `.csproj`, top-level statements, the kinds of `using`, file-scoped namespaces, and the service directories.

#### Going deeper (bonus)

1. **A manual global using.** Add a file `Usings.cs` with the line `global using System.Text.RegularExpressions;` and use `Regex` in `Program.cs` to validate that the name contains only letters. Explain how a `global using` differs from a plain `using` and from implicit usings.
2. **A second library project.** Create a sibling project `dotnet new classlib -n GreeterLib`, note that its `OutputType` defaults to `Library`, and add a reference from `GreeterApp` with `dotnet add GreeterApp reference GreeterLib`. Call a method from the library inside `Program.cs`. Compare `OutputType=Exe` and `Library`.
3. **A classic Main for comparison.** In a separate folder create a project with a classic `class Program { static void Main() { ... } }` (no top-level statements) and compare the amount of ceremony. Describe the advantages of the top-level form and when you might still prefer the classic one (for example, `Main` overloads or an `async Task Main` entry point).
4. **Exit-code management.** Extend the program: code `1` if the name is empty, `2` for an unknown mode, `3` for too many arguments. Write a script (bash or PowerShell) that runs the app with different arguments and checks `$?`/`$LASTEXITCODE`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `GreeterApp` создан через `dotnet new console -n GreeterApp`.
- [ ] (RU) `.csproj` содержит корректные `OutputType`, `TargetFramework=net8.0`, `ImplicitUsings`, `Nullable`, `LangVersion`.
- [ ] (RU) `Program.cs` — top-level statements с локальной функцией, коллекционным выражением, pattern matching, raw string, `using static`, `args`, `return`.
- [ ] (RU) `Metadata.cs` — file-scoped namespace, тип вызывается из точки входа, без top-level.
- [ ] (RU) Воспроизведена и описана ошибка `CS7022`.
- [ ] (RU) Сборка без ошибок, `bin/Debug/net8.0/GreeterApp.dll` существует.
- [ ] (RU) Три сценария запуска отрабатывают корректно; exit code `0`/`2`.
- [ ] (RU) `bin/` и `obj/` пересозданы; `dotnet clean` описан; `.gitignore` учтён.
- [ ] (RU) Отчёт с ответами на вопросы урока.
- [ ] (EN) Project `GreeterApp` created via `dotnet new console -n GreeterApp`.
- [ ] (EN) `.csproj` has correct `OutputType`, `TargetFramework=net8.0`, `ImplicitUsings`, `Nullable`, `LangVersion`.
- [ ] (EN) `Program.cs` uses top-level statements with a local function, a collection expression, pattern matching, a raw string, `using static`, `args`, `return`.
- [ ] (EN) `Metadata.cs` has a file-scoped namespace, a type called from the entry point, no top-level statements.
- [ ] (EN) The `CS7022` error is reproduced and described.
- [ ] (EN) Build succeeds; `bin/Debug/net8.0/GreeterApp.dll` exists.
- [ ] (EN) Three run scenarios work correctly; exit code `0`/`2`.
- [ ] (EN) `bin/` and `obj/` are recreated; `dotnet clean` is described; `.gitignore` is respected.
- [ ] (EN) Report answers the lesson's questions.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/tools/dotnet-new — `dotnet new` and project templates.
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/program-structure/top-level-statements — Top-level statements.
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/program-structure/namespaces — Namespaces (file-scoped form).
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/proposals/csharp-12.0/collection-expressions — Collection expressions (C# 12).
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/tokens/raw-string — Raw string literals.
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/tools/dotnet-build — `dotnet build` and `obj/bin` artifacts.
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/tools/dotnet-clean — `dotnet clean`.
