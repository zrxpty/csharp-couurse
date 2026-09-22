---
[← К уроку M01-L06](lesson-M01-L06-hello-debug-nuget.md) | [⬆ К модулю M01](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M01-L06: Первый Hello World, отладка, NuGet-базис / Homework M01-L06: First Hello World, debugging, NuGet basics

**Урок / Lesson:** M01-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 2/5
**Цель / Goal:** (RU) Закрепить полный базовый цикл .NET-разработки — написание консольной программы на C# 12 с top-level statements и обработкой аргументов командной строки, целенаправленную отладку через breakpoints, Step Into/Over/Out и окно Watch, а также подключение и восстановление внешних зависимостей через NuGet (PackageReference, dotnet add package, dotnet restore), включая фиксацию версий и анализ транзитивного графа. (EN) Consolidate the full basic .NET development loop — writing a C# 12 console program with top-level statements and command-line argument handling, deliberate debugging through breakpoints, Step Into/Over/Out and the Watch window, and connecting and restoring external dependencies through NuGet (PackageReference, dotnet add package, dotnet restore), including version pinning and transitive graph analysis.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую опирается на три кита урока: top-level statements с массивом `args`, отладчик (breakpoints, Step Into/Over/Out, Watch) и NuGet через `PackageReference`. Вы повторите шаблонное сопоставление аргументов из примера урока, поставите брейкпойнты на тех же «подозрительных» местах и подключите Spectre.Console через `dotnet add package` с фиксированной версией, как показано в `.csproj` урока. (EN) The homework builds directly on the three pillars of the lesson: top-level statements with the `args` array, the debugger (breakpoints, Step Into/Over/Out, Watch), and NuGet through `PackageReference`. You will reuse the argument pattern matching from the lesson example, set breakpoints on the same "suspicious" spots, and add Spectre.Console via `dotnet add package` with a pinned version, exactly as the lesson `.csproj` shows.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы только что прошли три фундаментальных навыка, без которых ни один реальный .NET-проект не сдвинется с места: написание программы, её отладку и подключение внешних библиотек. В учебных примерах эти навыки кажутся тривиальными, но в инженерной практике они сплетаются в единый рабочий цикл: вы пишете утилиту, она ведёт себя не так, как ожидалось, вы ставите брейкпойнт, шагаете по коду, видите неверное значение в Watch, исправляете аргумент, проверяете — и параллельно тянете с NuGet библиотеку для красивого вывода, фиксируя версию, чтобы сборка в CI не сломалась через неделю.

В этом задании вы построите мини-утилиту `GreeterCli`, которая приветствует пользователя по имени, повторяет приветствие заданное число раз и формирует сводку. Утилита принимает аргументы `--name <имя>`, `--count N` и `--upper` (флаг перевода в верхний регистр), а в отсутствие аргументов выводит подсказку. Парсинг аргументов нужно реализовать через pattern matching C# 12 с `when`-фильтрами, как в примере урока. Для вывода сводки вы используете raw string literal с интерполяцией (`$$""" ... """`), а для красивого оформления — пакет Spectre.Console. Затем вы отладите программу: поставите брейкпойнт на строке с вычислением флага `shouldUpper`, проверите в Watch выражение `args.Contains("--upper")`, пройдёте Step Into в собственный метод `FormatName` и Step Over — по методам `Spectre.Console.AnsiConsole`.

Задание моделирует реальный сценарий: новые разработчики часто впервые сталкиваются с NuGet именно когда им нужен «красивый» вывод, парсинг JSON или логирование. Вы научитесь не просто ставить пакет, а понимать, что произошло: как изменился `.csproj`, что лежит в `obj/project.assets.json`, какие транзитивные пакеты подтянулись, как разрешается конфликт версий по правилу «ближайшего выигрыша». Эти знания — фундамент для следующих модулей, где вы будете работать с ASP.NET Core, тестами на xUnit и Entity Framework, каждый из которых тянет десятки транзитивных зависимостей.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект: `dotnet new console -n GreeterCli -o GreeterCli`. Откройте `GreeterCli.csproj` и убедитесь, что присутствуют `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`, `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>`. Если какого-то свойства нет — добавьте вручную.
2. Замените содержимое `Program.cs` на top-level программу. Реализуйте парсинг `args` через `foreach` и `switch` с pattern matching: ключи `--name` и `--count` должны ждать значение в следующем аргументе, флаг `--upper` устанавливается без значения, а любой позиционный аргумент без `--` становится именем, если `--name` не задан. Число `--count` валидируйте диапазоном 1–100 через шаблон `is > 0 and <= 100`; при невалидном значении печатайте ошибку и выходите с `return 1;`.
3. Реализуйте локальную функцию `FormatName(string name, bool upper)` внутри top-level кода (это допустимо в C# 12). При `upper == true` она возвращает `name.ToUpperInvariant()`, иначе `name`. Это будет ваша «коробка», в которую вы зайдёте через Step Into.
4. Для вывода приветствий используйте цикл `for` и интерполяцию с выравниванием: `Console.WriteLine($"[{i + 1,2}] Привет, {formatted}!");`. После цикла сформируйте сводку через raw string literal `$$""" ... """`, чтобы отразить значения `name`, `count`, `shouldUpper` и `args.Length`.
5. Добавьте пакет Spectre.Console: `dotnet add package Spectre.Console --version 0.48.0`. Откройте `.csproj` и проверьте, что появилась строка `<PackageReference Include="Spectre.Console" Version="0.48.0" />`. Замените обычный `Console.WriteLine` для сводки на `AnsiConsole.MarkupLine($"[bold green]Итог:[/] имя={Markup.Escape(formatted)}, повторов={count}");`.
6. Выполните `dotnet restore` и убедитесь, что пакет скачался в локальный кэш. Посмотрите файл `obj/project.assets.json` (через `read` или текстовый редактор) — найдите там упоминание `Spectre.Console` и хотя бы один транзитивный пакет (например, `Spectre.Console` зависит от внутренних сборок). Выполните `dotnet list package --include-transitive` и сохраните вывод для отчёта.
7. Запустите программу в нескольких режимах и зафиксируйте вывод: `dotnet run` (без аргументов → подсказка), `dotnet run -- --name Alice --count 3`, `dotnet run -- Alice --count 2 --upper`, `dotnet run -- --count 150` (ошибка валидации).
8. Перейдите в отладчик вашей IDE (VS Code с C# Dev Kit, Visual Studio или Rider). Поставьте breakpoint на строке `bool shouldUpper = args.Contains("--upper");`. Запустите отладку (F5) с аргументами `Alice --upper` (аргументы отладки настраиваются в `launch.json` / `launchSettings.json` / свойствах проекта). Программа должна остановиться на брейкпойнте.
9. В окне Watch добавьте три выражения: `args`, `args.Length`, `args.Contains("--upper")`. Выполните Step Over (F10) и убедитесь, что `shouldUpper` стало `true`. Затем дойдите до вызова `FormatName(name, shouldUpper)` и выполните Step Into (F11) — вы должны оказаться внутри функции. Внутри `FormatName` поставьте ещё один breakpoint на `return upper ? name.ToUpperInvariant() : name;` и проверьте в Watch значение `upper`.
10. Намеренно сломайте версию пакета: измените в `.csproj` `Version="0.48.0"` на `Version="0.99.99"` (несуществующую) и выполните `dotnet restore`. Зафиксируйте ошибку NU1102 (пакет не найден). Верните `0.48.0`. Затем добавьте второй пакет с конфликтом, например `dotnet add package Newtonsoft.Json --version 13.0.1`, и выполните `dotnet list package --include-transitive` — выпишите в отчёт полный транзитивный граф.
11. Создайте файл `Directory.Packages.props` в корне решения со строкой `<Project><PropertyGroup><ManageVersions>...</ManageVersions></PropertyGroup></Project>` — нет, правильнее: включите Central Package Management и перенесите версии туда. Зафиксируйте изменения в `.csproj` (версии из `PackageReference` убираются, остаются только `Include`).

Ожидаемый вывод для `dotnet run -- Alice --count 2 --upper`:
```
[ 1] Привет, ALICE!
[ 2] Привет, ALICE!
Итог: имя=ALICE, повторов=2, верхний регистр=True, аргументов=4
```

#### Требования к решению
Решение сдаётся в виде каталога `GreeterCli/` с файлами `Program.cs`, `GreeterCli.csproj`, `Directory.Packages.props` (если делали CPM), `launch.json` или `launchSettings.json` с профилем отладки и текстового отчёта `report.md`. Код должен компилироваться без предупреждений (тreat warnings as errors приветствуется, но не обязателен) под .NET 8 с C# 12 (LangVersion latest). Используйте только top-level statements; класс `Program` и явный `static void Main` запрещены — это противоречит теме урока. Парсинг аргументов обязан использовать pattern matching с `when`-фильтрами и операторами `or`/`and`; каскад `if/else if` не принимается. Все строковые значения, пришедшие от пользователя, перед передачей в `AnsiConsole.MarkupLine` должны экранироваться через `Markup.Escape`, иначе `[` в имени сломает разбор разметки. Версии всех пакетов зафиксированы явно — плавающие `*` и `-*` запрещены. Команда `dotnet restore` должна проходить чисто; `dotnet build` — без ошибок и предупреждений NU-семейства. В отчёте `report.md` должны быть: выводы всех четырёх запусков, скриншот или текстовая выдержка остановки на брейкпойнте со значением `shouldUpper`, вывод `dotnet list package --include-transitive`, описание ошибки NU1102 и объяснение, как Central Package Management меняет структуру `PackageReference`.

#### Тонкости и подводные камни
- `args` никогда не равен `null` в top-level коде, но индексация `args[0]` без проверки `args.Length` бросает `IndexOutOfRangeException`. Когда парсите ключи вида `--count`, помните, что значение находится в `args[i + 1]`, и проверяйте, что `i + 1 < args.Length`, иначе `--count` без числа молча «съест» конец массива или бросит исключение.
- Разделитель `--` в `dotnet run -- arg` обязателен: без него `dotnet` попытается интерпретировать `--name` как свой собственный флаг и выдаст `NETSDK1093` или «неизвестная опция». Запомните: всё после `--` достаётся вашей программе.
- Pattern matching `case var n when int.TryParse(n, out var parsed) && parsed is > 0 and <= 100` — идиоматичный способ одновременно распарсить и провалидировать число. Но `int.TryParse` принимает строку, поэтому сначала проверяйте, что `n` не начинается с `--`, иначе ключ `--count` попадёт в эту ветку как «не число» и просто игнорируется.
- Raw string literal `$$""" ... """` требует нечётного числа `$` (здесь два) и нечётного числа кавычек; внутри можно свободно использовать `{{...}}` для интерполяции, а обычные `{` и `}` не нужно экранировать. Если поставить один `$`, то `{{` станет литералом `{`, а не интерполяцией — частая ошибка.
- Spectre.Console использует собственный мини-язык разметки в квадратных скобках: `[bold green]...[/]`. Если имя пользователя содержит `[`, `AnsiConsole.MarkupLine` бросит исключение. Всегда оборачивайте пользовательский ввод в `Markup.Escape`.
- `dotnet add package` меняет только `.csproj`; сам пакет скачивается только при `restore`/`build`/`run`. Удаление строки из `.csproj` не чистит кэш `~/.nuget/packages` — для полного удаления используйте `dotnet nuget locals all --clear` с осторожностью.
- NU1102 (пакет не найден) отличается от NU1605 (конфликт версий, downgrade). При NU1605 смотрите на `dotnet list package --include-transitive` — правило «ближайшего выигрыша» означает, что прямая ссылка с меньшей версией не перекрывает транзитивную с большей, и нужно явно поднять версию.
- Central Package Management требует, чтобы в `Directory.Packages.props` был `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>`, а в `.csproj` элементы выглядели как `<PackageReference Include="X" />` без `Version`. Если забыть флаг — версии просто «потеряются» и restore упадёт с NU1002.
- В VS Code брейкпойнты на top-level коде иногда не срабатывают, если в `launch.json` не задан `"console": "internalConsole"` или если включена оптимизация. Убедитесь, что конфигурация запускает `dotnet run` с нужными аргументами через `"args"` и что `justMyCode` не фильтрует ваши методы.
- Step Into (F11) на вызове `AnsiConsole.MarkupLine` может увести в недра Spectre.Console и запутать — для библиотечных методов используйте Step Over (F10), как требует best practice урока. Step Into — только для собственных методов вроде `FormatName`.

#### Критерии приёмки
- [ ] Проект `GreeterCli` создан под .NET 8, в `.csproj` включены `Nullable`, `ImplicitUsings`, `LangVersion=latest`.
- [ ] `Program.cs` использует исключительно top-level statements; явного `class Program` / `static void Main` нет.
- [ ] Парсинг `args` реализован через `foreach` + `switch` с pattern matching (`when`, `or`, `and`).
- [ ] Ключ `--count` валидируется диапазоном 1–100 через шаблон `is > 0 and <= 100`; невалидное значение вызывает `return 1` с сообщением.
- [ ] Флаг `--upper` распознаётся и применяется через локальную функцию `FormatName`.
- [ ] Сводка выводится через raw string literal `$$""" ... """` с интерполяцией.
- [ ] Пакет `Spectre.Console` добавлен через `dotnet add package --version 0.48.0`; версия зафиксирована в `.csproj`.
- [ ] Пользовательский ввод передаётся в `AnsiConsole.MarkupLine` только через `Markup.Escape`.
- [ ] `dotnet restore` проходит без ошибок; в `obj/project.assets.json` присутствует `Spectre.Console`.
- [ ] `dotnet list package --include-transitive` выводит транзитивный граф; граф сохранён в отчёте.
- [ ] Намеренная подстановка `Version="0.99.99"` воспроизводит ошибку NU1102 (зафиксировано в отчёте).
- [ ] Запуск `dotnet run` без аргументов печатает подсказку и завершается с кодом 0.
- [ ] Запуск `dotnet run -- Alice --count 2 --upper` печатает две строки в верхнем регистре и сводку.
- [ ] В отладчике поставлен breakpoint на `shouldUpper`, программа останавливается, в Watch видно `args.Contains("--upper") == true`.
- [ ] Выполнен Step Into (F11) в `FormatName` и Step Over (F10) по библиотечному вызову; оба шага описаны в отчёте.
- [ ] (Бонус) Включён Central Package Management через `Directory.Packages.props`.

#### Подсказки (без прямого ответа)
- Вспомните пример урока: `case var s when s.StartsWith("--") is false:` выделяет позиционный аргумент-имя. Как совместить это с проверкой, что `--name` уже задан?
- Для значения после `--count` используйте индекс текущего аргумента плюс один; не забудьте проверить границу массива до обращения.
- `bool shouldUpper = args.Contains("--upper");` — простейший способ узнать про флаг, но `args.Contains` — линейный поиск; для небольшого CLI это допустимо.
- В `launch.json` поле `"args": ["Alice", "--count", "2", "--upper"]` передаёт аргументы в отладочную сессию; кавычки не нужны, если в значении нет пробелов.
- Чтобы найти транзитивные пакеты, выполните `dotnet list package --include-transitive` из каталога проекта, а не решения.
- При включении CPM версии из `PackageReference` переносятся в `<PackageVersion Include="..." Version="..." />` внутри `Directory.Packages.props`.

#### Эталонное решение (разбор)
```csharp
// GreeterCli / Program.cs — C# 12 / .NET 8, top-level statements
// Запуск / Run: dotnet run -- Alice --count 2 --upper
using Spectre.Console;

// Локальная функция — цель для Step Into (F11) / Local function — Step Into target
string FormatName(string name, bool upper) =>
    upper ? name.ToUpperInvariant() : name;

// Разбор аргументов через pattern matching / Parse args via pattern matching
string name = "Мир / World";
int count = 1;

for (int i = 0; i < args.Length; i++)
{
    switch (args[i])
    {
        // Ключи с ожидаемым значением / Keys expecting a value
        case "--name" or "-n":
            if (i + 1 < args.Length && !args[i + 1].StartsWith("--"))
            {
                name = args[++i]; // поглощаем следующий аргумент / consume next
            }
            break;

        case "--count" or "-c":
            if (i + 1 < args.Length &&
                int.TryParse(args[i + 1], out var parsed) &&
                parsed is > 0 and <= 100)
            {
                count = parsed;
                ++i;
            }
            else
            {
                AnsiConsole.MarkupLine("[red]Ошибка: --count ожидает целое 1..100 / Error: --count expects int 1..100[/]");
                return 1;
            }
            break;

        // Флаг без значения / Flag without a value
        case "--upper" or "-u":
            // обрабатывается ниже через args.Contains / handled below via args.Contains
            break;

        // Позиционный аргумент — имя, если ещё не задано / Positional — name if unset
        case var s when !s.StartsWith("--") && name == "Мир / World":
            name = s;
            break;
    }
}

// Удобная точка для breakpoint: поставьте сюда F9 / Convenient breakpoint spot
bool shouldUpper = args.Contains("--upper");

if (args.Length == 0)
{
    AnsiConsole.MarkupLine("[yellow]Привет, мир! / Hello, world![/]");
    AnsiConsole.MarkupLine("[dim]Подсказка / Hint: dotnet run -- <name> [--count N] [--upper][/]");
    return 0;
}

string formatted = FormatName(name, shouldUpper);

// Вывод повторов / Print repetitions
for (int i = 0; i < count; i++)
{
    AnsiConsole.MarkupLine($"[{i + 1,2}] [bold]Привет, {Markup.Escape(formatted)}![/]");
}

// Сводка через raw string literal с интерполяцией / Summary via interpolated raw string
var summary = $$"""
Итог / Summary:
  - Имя / Name         : {{Markup.Escape(formatted)}}
  - Повторов / Count   : {{count}}
  - Верхний / Upper    : {{shouldUpper}}
  - Аргументов / Args  : {{args.Length}}
""";

AnsiConsole.WriteLine(summary);
return 0;
```

Разбор по строкам. Первая строка `using Spectre.Console;` открывает доступ к `AnsiConsole` и `Markup.Escape` — без неё компилятор не найдёт эти типы, потому что `ImplicitUsings` не включает сторонние пакеты. Локальная функция `FormatName` объявлена прямо в top-level коде: C# 12 разрешает локальные функции в неявном `Main`, и это удобная цель для Step Into — вы зайдёте в неё, но не уйдёте в недра Spectre.Console. Парсинг аргументов построен на `switch` с pattern matching: `case "--name" or "-n":` использует оператор `or` для длинной и короткой форм ключа; `case var s when !s.StartsWith("--") && name == "Мир / World":` — это `when`-фильтр из примера урока, объединяющий два условия. Проверка `i + 1 < args.Length` перед `args[i + 1]` защищает от `IndexOutOfRangeException`, о которой предупреждает раздел «Частые ошибки» урока. Валидация `parsed is > 0 and <= 100` — ровно тот шаблон, что в примере с `--count`. Строка `bool shouldUpper = args.Contains("--upper");` специально вынесена в отдельную переменную: это идеальная точка для breakpoint и для наблюдения в Watch за `args.Contains("--upper")` — вы увидите, как значение вычисляется один раз, а не при каждой проверке. `Markup.Escape(formatted)` применяется ко всему пользовательскому вводу перед подстановкой в `MarkupLine`: без этого имя `Alice [admin]` выбросит исключение парсера разметки Spectre.Console. Raw string literal `$$""" ... """` с двумя `$` позволяет свободно писать `{` и `}` в тексте сводки и интерполировать через `{{...}}` — это тот же приём, что в примере урока с итогом. Завершающий `return 0;` (и `return 1;` в ветке ошибки) задаёт код выхода процесса, который потом можно проверить в CI через `$LASTEXITCODE` в PowerShell или `$?` в bash.

Для `.csproj` используйте шаблон из урока: `OutputType=Exe`, `TargetFramework=net8.0`, `Nullable=enable`, `ImplicitUsings=enable`, `LangVersion=latest` и `<PackageReference Include="Spectre.Console" Version="0.48.0" />`. Для Central Package Management создайте `Directory.Packages.props` с `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` и `<PackageVersion Include="Spectre.Console" Version="0.48.0" />`, а в `.csproj` оставьте `<PackageReference Include="Spectre.Console" />` без `Version`.

#### Задания на углубление (бонус)
1. Добавьте ключ `--lang ru|en`, который переключает язык приветствия («Привет» / «Hello»). Реализуйте выбор через `switch` выражение (switch expression), а не оператор.
2. Подключите пакет `McMaster.Extensions.CommandLineUtils` и перепарсите аргументы через его атрибуты `[Option]`. Сравните объём кода и удобство с ручным pattern matching.
3. Намеренно создайте конфликт версий: добавьте два пакета, один из которых транзитивно требует `System.Text.Json` 8.0.0, а другой — 7.0.0. Добейтесь предупреждения NU1605, затем разрешите его явной `<PackageReference Include="System.Text.Json" Version="8.0.0" />` и объясните, почему сработало правило «ближайшего выигрыша».
4. Настройте `Directory.Build.props` так, чтобы все проекты в решении автоматически включали `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` и `<Nullable>enable</Nullable>`, и убедитесь, что `GreeterCli` собирается без единого предупреждения.

---

## Statement in English / Постановка на английском

#### Context & motivation
You have just covered the three fundamental skills without which no real .NET project can move: writing a program, debugging it, and connecting external libraries. In toy examples these skills look trivial, but in engineering practice they weave into a single working loop: you write a utility, it misbehaves, you set a breakpoint, you step through the code, you spot a wrong value in Watch, you fix an argument, you verify — and in parallel you pull a NuGet library for pretty output, pinning the version so that the CI build does not break a week later.

In this assignment you will build a mini-utility `GreeterCli` that greets a user by name, repeats the greeting a given number of times, and prints a summary. The utility accepts `--name <name>`, `--count N`, and `--upper` (an upper-case flag), and prints a hint when no arguments are supplied. Argument parsing must be implemented with C# 12 pattern matching and `when` guards, exactly as in the lesson example. For the summary you will use a raw string literal with interpolation (`$$""" ... """`), and for pretty formatting you will use the Spectre.Console package. Then you will debug the program: set a breakpoint on the line that computes the `shouldUpper` flag, inspect the `args.Contains("--upper")` expression in Watch, Step Into your own `FormatName` method, and Step Over the `Spectre.Console.AnsiConsole` methods.

The assignment models a real scenario: junior developers often meet NuGet for the first time exactly when they need pretty output, JSON parsing, or logging. You will learn not just to add a package but to understand what happened: how the `.csproj` changed, what sits in `obj/project.assets.json`, which transitive packages were pulled in, how a version conflict is resolved by the "nearest wins" rule. This knowledge is the foundation for the next modules, where you will work with ASP.NET Core, xUnit tests, and Entity Framework, each of which drags in dozens of transitive dependencies.

#### What to do step by step
1. Create a new console project: `dotnet new console -n GreeterCli -o GreeterCli`. Open `GreeterCli.csproj` and verify that `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`, `<TargetFramework>net8.0</TargetFramework>`, and `<LangVersion>latest</LangVersion>` are present. Add any missing property by hand.
2. Replace the contents of `Program.cs` with a top-level program. Parse `args` with a `foreach` and a `switch` using pattern matching: the `--name` and `--count` keys must expect a value in the next argument, the `--upper` flag is set without a value, and any positional argument without `--` becomes the name if `--name` was not supplied. Validate `--count` against the 1–100 range with the `is > 0 and <= 100` pattern; on an invalid value print an error and exit with `return 1;`.
3. Implement a local function `FormatName(string name, bool upper)` inside the top-level code (this is allowed in C# 12). When `upper == true` it returns `name.ToUpperInvariant()`, otherwise `name`. This is the "box" you will enter via Step Into.
4. For the greetings use a `for` loop and interpolation with alignment: `Console.WriteLine($"[{i + 1,2}] Hello, {formatted}!");`. After the loop, build the summary through a raw string literal `$$""" ... """` capturing `name`, `count`, `shouldUpper`, and `args.Length`.
5. Add the Spectre.Console package: `dotnet add package Spectre.Console --version 0.48.0`. Open `.csproj` and confirm the line `<PackageReference Include="Spectre.Console" Version="0.48.0" />` appeared. Replace the plain `Console.WriteLine` for the summary with `AnsiConsole.MarkupLine($"[bold green]Summary:[/] name={Markup.Escape(formatted)}, repeats={count}");`.
6. Run `dotnet restore` and confirm the package was downloaded to the local cache. Inspect `obj/project.assets.json` (via `read` or a text editor) — find the `Spectre.Console` entry and at least one transitive package (Spectre.Console depends on internal assemblies). Run `dotnet list package --include-transitive` and save the output for the report.
7. Launch the program in several modes and record the output: `dotnet run` (no args → hint), `dotnet run -- --name Alice --count 3`, `dotnet run -- Alice --count 2 --upper`, `dotnet run -- --count 150` (validation error).
8. Switch to the debugger of your IDE (VS Code with C# Dev Kit, Visual Studio, or Rider). Set a breakpoint on the line `bool shouldUpper = args.Contains("--upper");`. Start debugging (F5) with the arguments `Alice --upper` (debug arguments are configured in `launch.json` / `launchSettings.json` / project properties). The program must stop at the breakpoint.
9. In the Watch window add three expressions: `args`, `args.Length`, `args.Contains("--upper")`. Step Over (F10) and confirm `shouldUpper` became `true`. Then walk to the `FormatName(name, shouldUpper)` call and perform Step Into (F11) — you must land inside the function. Inside `FormatName`, set another breakpoint on `return upper ? name.ToUpperInvariant() : name;` and verify the value of `upper` in Watch.
10. Deliberately break the package version: change `Version="0.48.0"` to `Version="0.99.99"` (non-existent) in `.csproj` and run `dotnet restore`. Capture the NU1102 error (package not found). Restore `0.48.0`. Then add a second package with a conflict, e.g. `dotnet add package Newtonsoft.Json --version 13.0.1`, and run `dotnet list package --include-transitive` — write the full transitive graph into the report.
11. Create a `Directory.Packages.props` file at the solution root with `<Project><PropertyGroup><ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally></PropertyGroup>...</Project>` and move the versions there. Fix the `.csproj` so that versions are removed from `PackageReference` and only `Include` remains.

Expected output for `dotnet run -- Alice --count 2 --upper`:
```
[ 1] Hello, ALICE!
[ 2] Hello, ALICE!
Summary: name=ALICE, repeats=2, upper=True, args=4
```

#### Requirements
The solution is submitted as a `GreeterCli/` directory with files `Program.cs`, `GreeterCli.csproj`, `Directory.Packages.props` (if you did CPM), `launch.json` or `launchSettings.json` with a debug profile, and a text report `report.md`. The code must compile without warnings under .NET 8 with C# 12 (LangVersion latest); treating warnings as errors is welcome but not mandatory. Use only top-level statements; an explicit `Program` class and `static void Main` are forbidden — they contradict the lesson topic. Argument parsing must use pattern matching with `when` guards and the `or`/`and` operators; a cascade of `if/else if` is not accepted. All string values coming from the user must be escaped with `Markup.Escape` before being passed to `AnsiConsole.MarkupLine`, otherwise a `[` in the name will break the markup parser. All package versions are pinned explicitly — floating `*` and `-*` are forbidden. `dotnet restore` must succeed cleanly; `dotnet build` must produce no errors and no NU-family warnings. The `report.md` must contain: the output of all four runs, a screenshot or textual excerpt of the breakpoint stop showing the value of `shouldUpper`, the output of `dotnet list package --include-transitive`, a description of the NU1102 error, and an explanation of how Central Package Management changes the structure of `PackageReference`.

#### Pitfalls
- `args` is never `null` in top-level code, but indexing `args[0]` without checking `args.Length` throws `IndexOutOfRangeException`. When you parse keys like `--count`, remember the value sits at `args[i + 1]`, and verify that `i + 1 < args.Length` — otherwise a trailing `--count` silently "eats" the end of the array or throws.
- The `--` separator in `dotnet run -- arg` is mandatory: without it `dotnet` will try to interpret `--name` as its own flag and emit `NETSDK1093` or "unknown option". Everything after `--` reaches your program.
- The pattern `case var n when int.TryParse(n, out var parsed) && parsed is > 0 and <= 100` is the idiomatic way to parse and validate a number at once. But `int.TryParse` accepts a string, so first ensure `n` does not start with `--`, otherwise the `--count` key falls into this branch as "not a number" and is silently ignored.
- A raw string literal `$$""" ... """` needs an odd number of `$` (here two) and an odd number of quotes; inside you can freely use `{{...}}` for interpolation, while plain `{` and `}` need no escaping. With a single `$`, `{{` becomes a literal `{` instead of an interpolation — a frequent mistake.
- Spectre.Console uses its own mini-markup language in square brackets: `[bold green]...[/]`. If a user name contains `[`, `AnsiConsole.MarkupLine` throws. Always wrap user input in `Markup.Escape`.
- `dotnet add package` only edits `.csproj`; the package itself is downloaded on `restore`/`build`/`run`. Removing the line from `.csproj` does not clean the `~/.nuget/packages` cache — for a full removal use `dotnet nuget locals all --clear` with care.
- NU1102 (package not found) is different from NU1605 (version conflict, downgrade). On NU1605 inspect `dotnet list package --include-transitive` — the "nearest wins" rule means a direct reference with a lower version does not override a transitive one with a higher version, and you must raise the version explicitly.
- Central Package Management requires `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` in `Directory.Packages.props`, and the `.csproj` items must look like `<PackageReference Include="X" />` without `Version`. Forgetting the flag simply "loses" the versions and restore fails with NU1002.
- In VS Code, breakpoints on top-level code sometimes do not hit if `launch.json` does not set `"console": "internalConsole"` or if optimization is on. Make sure the configuration launches with the right `"args"` and that `justMyCode` does not filter your methods.
- Step Into (F11) on `AnsiConsole.MarkupLine` can drag you into the bowels of Spectre.Console and confuse you — for library methods use Step Over (F10), as the lesson's best practice demands. Step Into is only for your own methods like `FormatName`.

#### Acceptance criteria
- [ ] The `GreeterCli` project is created under .NET 8; `.csproj` enables `Nullable`, `ImplicitUsings`, `LangVersion=latest`.
- [ ] `Program.cs` uses only top-level statements; no explicit `class Program` / `static void Main`.
- [ ] Argument parsing uses `foreach` + `switch` with pattern matching (`when`, `or`, `and`).
- [ ] The `--count` key is validated against 1–100 via the `is > 0 and <= 100` pattern; an invalid value triggers `return 1` with a message.
- [ ] The `--upper` flag is recognized and applied through the `FormatName` local function.
- [ ] The summary is printed through a raw string literal `$$""" ... """` with interpolation.
- [ ] The `Spectre.Console` package is added via `dotnet add package --version 0.48.0`; the version is pinned in `.csproj`.
- [ ] User input reaches `AnsiConsole.MarkupLine` only through `Markup.Escape`.
- [ ] `dotnet restore` completes without errors; `obj/project.assets.json` contains `Spectre.Console`.
- [ ] `dotnet list package --include-transitive` prints the transitive graph; the graph is saved in the report.
- [ ] Substituting `Version="0.99.99"` reproduces the NU1102 error (recorded in the report).
- [ ] Running `dotnet run` with no arguments prints the hint and exits with code 0.
- [ ] Running `dotnet run -- Alice --count 2 --upper` prints two upper-case lines and the summary.
- [ ] In the debugger a breakpoint is set on `shouldUpper`; the program stops; Watch shows `args.Contains("--upper") == true`.
- [ ] Step Into (F11) into `FormatName` and Step Over (F10) over the library call are performed; both steps are described in the report.
- [ ] (Bonus) Central Package Management is enabled via `Directory.Packages.props`.

#### Hints (no direct answer)
- Recall the lesson example: `case var s when s.StartsWith("--") is false:` isolates a positional name argument. How do you combine this with a check that `--name` is still unset?
- For the value after `--count`, use the index of the current argument plus one; remember to check the array bound before indexing.
- `bool shouldUpper = args.Contains("--upper");` is the simplest way to detect the flag; `args.Contains` is a linear scan, acceptable for a small CLI.
- In `launch.json` the field `"args": ["Alice", "--count", "2", "--upper"]` passes arguments to the debug session; quotes are unnecessary unless a value contains spaces.
- To find transitive packages, run `dotnet list package --include-transitive` from the project directory, not the solution.
- When enabling CPM, versions move from `PackageReference` into `<PackageVersion Include="..." Version="..." />` inside `Directory.Packages.props`.

#### Reference solution walk-through
```csharp
// GreeterCli / Program.cs — C# 12 / .NET 8, top-level statements
// Run: dotnet run -- Alice --count 2 --upper
using Spectre.Console;

// Local function — Step Into (F11) target
string FormatName(string name, bool upper) =>
    upper ? name.ToUpperInvariant() : name;

// Parse args via pattern matching
string name = "World";
int count = 1;

for (int i = 0; i < args.Length; i++)
{
    switch (args[i])
    {
        // Keys expecting a value
        case "--name" or "-n":
            if (i + 1 < args.Length && !args[i + 1].StartsWith("--"))
            {
                name = args[++i]; // consume next
            }
            break;

        case "--count" or "-c":
            if (i + 1 < args.Length &&
                int.TryParse(args[i + 1], out var parsed) &&
                parsed is > 0 and <= 100)
            {
                count = parsed;
                ++i;
            }
            else
            {
                AnsiConsole.MarkupLine("[red]Error: --count expects int 1..100[/]");
                return 1;
            }
            break;

        // Flag without a value
        case "--upper" or "-u":
            // handled below via args.Contains
            break;

        // Positional — name if unset
        case var s when !s.StartsWith("--") && name == "World":
            name = s;
            break;
    }
}

// Convenient breakpoint spot: set F9 here
bool shouldUpper = args.Contains("--upper");

if (args.Length == 0)
{
    AnsiConsole.MarkupLine("[yellow]Hello, world![/]");
    AnsiConsole.MarkupLine("[dim]Hint: dotnet run -- <name> [--count N] [--upper][/]");
    return 0;
}

string formatted = FormatName(name, shouldUpper);

// Print repetitions
for (int i = 0; i < count; i++)
{
    AnsiConsole.MarkupLine($"[{i + 1,2}] [bold]Hello, {Markup.Escape(formatted)}![/]");
}

// Summary via interpolated raw string
var summary = $$"""
Summary:
  - Name   : {{Markup.Escape(formatted)}}
  - Count  : {{count}}
  - Upper  : {{shouldUpper}}
  - Args   : {{args.Length}}
""";

AnsiConsole.WriteLine(summary);
return 0;
```

Walk-through line by line. The first line `using Spectre.Console;` opens access to `AnsiConsole` and `Markup.Escape` — without it the compiler will not find these types, because `ImplicitUsings` does not include third-party packages. The local function `FormatName` is declared directly in the top-level code: C# 12 allows local functions in the implicit `Main`, and it is a convenient Step Into target — you enter it but do not sink into the bowels of Spectre.Console. Argument parsing is built on a `switch` with pattern matching: `case "--name" or "-n":` uses the `or` operator for the long and short forms of the key; `case var s when !s.StartsWith("--") && name == "World":` is the `when` guard from the lesson example, combining two conditions. The `i + 1 < args.Length` check before `args[i + 1]` guards against the `IndexOutOfRangeException` that the lesson's "Common Mistakes" section warns about. The `parsed is > 0 and <= 100` validation is exactly the pattern from the `--count` example. The line `bool shouldUpper = args.Contains("--upper");` is deliberately extracted into a separate variable: it is an ideal breakpoint spot and a Watch target for `args.Contains("--upper")` — you see the value computed once instead of on every check. `Markup.Escape(formatted)` is applied to all user input before substitution into `MarkupLine`: without it the name `Alice [admin]` would throw a Spectre.Console markup parser exception. The raw string literal `$$""" ... """` with two `$` lets you freely write `{` and `}` in the summary text and interpolate via `{{...}}` — the same technique as the lesson's summary example. The final `return 0;` (and `return 1;` on the error branch) sets the process exit code, which can later be checked in CI via `$LASTEXITCODE` in PowerShell or `$?` in bash.

For `.csproj` use the template from the lesson: `OutputType=Exe`, `TargetFramework=net8.0`, `Nullable=enable`, `ImplicitUsings=enable`, `LangVersion=latest`, and `<PackageReference Include="Spectre.Console" Version="0.48.0" />`. For Central Package Management create `Directory.Packages.props` with `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` and `<PackageVersion Include="Spectre.Console" Version="0.48.0" />`, and in `.csproj` leave `<PackageReference Include="Spectre.Console" />` without `Version`.

#### Going deeper (bonus)
1. Add a `--lang ru|en` key that switches the greeting language ("Привет" / "Hello"). Implement the choice with a switch expression, not a switch statement.
2. Add the `McMaster.Extensions.CommandLineUtils` package and re-parse the arguments through its `[Option]` attributes. Compare the code volume and ergonomics against hand-written pattern matching.
3. Deliberately create a version conflict: add two packages, one transitively requiring `System.Text.Json` 8.0.0 and the other 7.0.0. Reproduce the NU1605 warning, then resolve it with an explicit `<PackageReference Include="System.Text.Json" Version="8.0.0" />` and explain why the "nearest wins" rule applied.
4. Configure `Directory.Build.props` so that every project in the solution automatically enables `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` and `<Nullable>enable</Nullable>`, and verify that `GreeterCli` builds without a single warning.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Каталог `GreeterCli/` содержит `Program.cs`, `GreeterCli.csproj`, `report.md`, профиль отладки.
- [ ] Код компилируется под .NET 8 / C# 12 без предупреждений.
- [ ] Все четыре запуска (`dotnet run`, `--name Alice --count 3`, `Alice --count 2 --upper`, `--count 150`) описаны в отчёте.
- [ ] В отчёте есть выдержка остановки на брейкпойнте со значением `shouldUpper`.
- [ ] В отчёте приведён вывод `dotnet list package --include-transitive`.
- [ ] В отчёте описана воспроизведённая ошибка NU1102 и её исправление.
- [ ] (Бонус) Включён Central Package Management через `Directory.Packages.props`.
- [ ] The `GreeterCli/` directory contains `Program.cs`, `GreeterCli.csproj`, `report.md`, and a debug profile.
- [ ] The code compiles under .NET 8 / C# 12 with no warnings.
- [ ] All four runs (`dotnet run`, `--name Alice --count 3`, `Alice --count 2 --upper`, `--count 150`) are described in the report.
- [ ] The report includes a breakpoint-stop excerpt showing the value of `shouldUpper`.
- [ ] The report includes the output of `dotnet list package --include-transitive`.
- [ ] The report describes the reproduced NU1102 error and its fix.
- [ ] (Bonus) Central Package Management is enabled via `Directory.Packages.props`.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/visualstudio/debugger/ — Visual Studio debugger navigator: breakpoints, stepping, Watch window.
- Microsoft Learn — https://learn.microsoft.com/nuget/ — NuGet documentation: PackageReference, dotnet CLI, package restore and versioning.
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/tutorials/top-level-statements — Top-level statements in C#.
- Microsoft Learn — https://learn.microsoft.com/nuget/consume-packages/central-package-management — Central Package Management (Directory.Packages.props).
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns — Pattern matching in C# 12.
- NuGet.org — https://www.nuget.org/ — Central package repository.
- Spectre.Console — https://spectreconsole.net/ — Documentation for the Spectre.Console markup and `Markup.Escape`.
---
[← К уроку M01-L06](lesson-M01-L06-hello-debug-nuget.md) | [⬆ К модулю M01](../README.md) | [Следующее ДЗ →]()
---
