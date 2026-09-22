---
[← К уроку M01-L01](lesson-M01-L01-csharp-dotnet-intro.md) | [⬆ К модулю M01](../README.md) | [Следующее ДЗ: M01-L02 →](homework-M01-L02-clr-il-jit.md)
---

### Домашнее задание M01-L01: Что такое C# и .NET / Homework M01-L01: What is C# and .NET

**Урок / Lesson:** M01-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 1/5
**Цель / Goal:** (RU) На практике различить C# как язык и .NET как платформу, освоить создание консольного проекта на .NET 8 с top-level statements, collection expressions и raw string literals, продемонстрировать кроссплатформенность через `Environment.OSVersion`/`Environment.Version` и работу BCL (`System.IO`, `System.Linq`, `System.Collections.Generic`), а также закрепить понимание managed code, CLR, IL, JIT и границ между автоматическим освобождением памяти (GC) и ручным `Dispose`. (EN) In practice, distinguish C# as a language from .NET as a platform, create a .NET 8 console project with top-level statements, collection expressions and raw string literals, demonstrate cross-platform behavior through `Environment.OSVersion`/`Environment.Version` and BCL usage (`System.IO`, `System.Linq`, `System.Collections.Generic`), and consolidate the understanding of managed code, CLR, IL, JIT and the boundary between automatic memory management (GC) and manual `Dispose`.

#### Связь с уроком / Connection to the lesson
(RU) Домашнее задание напрямую закрепляет все ключевые концепции урока: разделение C# (язык команд) и .NET (движок + BCL + CLR), исторические ветки платформы (Framework, Mono, Core, современный .NET 5–8 с LTS у 6 и 8), управляемый код и работу GC, кроссплатформенность через один IL на разные ОС, а также современные возможности C# 12 — top-level statements, collection expressions и raw string literals. Ученик не просто читает теорию, а собирает проект, запускает его, видит версию рантайма и сохраняет артефакт через BCL. (EN) The homework directly reinforces every key concept of the lesson: the separation of C# (the language of commands) from .NET (the engine + BCL + CLR), the historical branches of the platform (Framework, Mono, Core, modern .NET 5–8 with LTS at 6 and 8), managed code and the work of the GC, cross-platform behavior through one IL across OSes, and modern C# 12 features — top-level statements, collection expressions and raw string literals. The student does not just read theory; they build a project, run it, observe the runtime version, and persist an artifact through the BCL.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы пришли в команду, которая поддерживает Legacy-приложение на .NET Framework 4.8 и параллельно строит новый микросервис на современном .NET 8. Тимлид просит вас подготовить «шпаргалку-репорт» о текущем состоянии платформы .NET, которая должна работать как самостоятельная консольная утилита: она определяет, на какой ОС и на какой версии рантайма запущена, собирает справку об исторических версиях .NET (Framework, Core, современный .NET с отметкой LTS), фильтрует их через LINQ, формирует многострочный отчёт с помощью raw string literal и сохраняет его в файл через `System.IO`. Это реалистичная задача: подобные «самоописывающиеся» утилиты часто используются в DevOps-пайплайнах для диагностики окружения сборки и деплоя, где важно знать, попал ли в контейнер правильный образ .NET и какая ОС под ним. Утилита также служит наглядным доказательством ключевых тезисов урока: один и тот же IL запускается на разных ОС, потому что CLR и BCL реализованы для каждой платформы; при этом код не содержит ручного освобождения памяти, потому что GC забирает управляемые объекты сам, а `Dispose` нужен лишь там, где есть неуправляемые ресурсы (файлы, сокеты). Вы учитесь не путать язык C# с платформой .NET, не путать .NET Framework с современным .NET и опираться на BCL вместо «изобретения велосипеда». Дополнительно вы закрепляете современные синтаксические возможности C# 12, которые сокращают шаблонный код и делают программу более читаемой, что особенно ценно в коротких утилитах и скриптах.

#### Что нужно сделать (пошагово)
1. **Проверьте установленный SDK.** Откройте терминал и выполните команду `dotnet --version`. Ожидаемый вывод — `8.x.x` (например, `8.0.404`). Если установлена версия ниже 8, установите .NET 8 SDK с официального сайта Microsoft. Дополнительно выполните `dotnet --list-sdks`, чтобы убедиться, что SDK 8 присутствует в списке. Запомните: SDK и Runtime — разные сущности; SDK включает Runtime для разработки, а на проде обычно ставят только Runtime.
2. **Создайте проект.** В терминале перейдите в рабочую папку (например, `C:\projects\course\homework`) и выполните: `dotnet new console -n M01L01.Report -o M01L01.Report --framework net8.0`. Ключ `--framework net8.0` фиксирует TFM (Target Framework Moniker). Откройте файл `M01L01.Report.csproj` и убедитесь, что там есть `<TargetFramework>net8.0</TargetFramework>` и `<ImplicitUsings>enable</ImplicitUsings>`. Implicit usings подключают `System`, `System.IO`, `System.Linq` и `System.Collections.Generic` автоматически — это часть современного .NET.
3. **Изучите сгенерированный `Program.cs`.** В .NET 8 шаблон `dotnet new console` создаёт файл с top-level statements: нет ни класса `Program`, ни метода `Main`. Это и есть современный стиль точки входа. Запомните: компилятор сам генерирует синтетический класс `Program` с методом `Main` под капотом.
4. **Реализуйте логику.** Замените содержимое `Program.cs` кодом из раздела «Эталонное решение». Программа должна: определить ОС через `Environment.OSVersion.Platform` (switch expression), вывести версию рантайма через `Environment.Version`, объявить список версий .NET через collection expression (`[ ... ]`), отфильтровать его через LINQ (`Where`, `Select`, `First`), сформировать отчёт через raw string literal (`""" ... """`), сохранить отчёт в файл через `File.WriteAllText` в каталог `Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "M01-L01")`, и вывести путь в консоль.
5. **Запустите проект.** Выполните `dotnet run --project M01L01.Report`. Ожидаемый вывод содержит строку вида «Привет от .NET 8.0.x на Windows» (или `Unix` на Linux/macOS), затем сводку версий .NET, затем таблицу метаданных и путь к сохранённому файлу. Скопируйте путь к файлу и откройте его в текстовом редакторе — убедитесь, что содержимое совпадает с выведенной сводкой.
6. **Опубликуйте проект как single-file.** Выполните `dotnet publish M01L01.Report -c Release -r win-x64 --self-contained false -o ./publish`. Ключ `--self-contained false` создаёт framework-dependent сборку (зависит от установленного .NET 8 на целевой машине). Затем выполните `./publish/M01L01.Report.exe` и убедитесь, что утилита работает. Это упражнение иллюстрирует, что один IL запускается на разных ОС: тот же `dotnet publish -r linux-x64` дал бы бинарник для Linux.
7. **Проверьте кроссплатформенность (опционально, если есть доступ к другой ОС или Docker).** Если у вас установлен Docker, выполните `docker run --rm -v ${PWD}:/app -w /app mcr.microsoft.com/dotnet/sdk:8.0 dotnet run --project M01L01.Report`. Вывод покажет `PlatformID.Unix`, что доказывает тезис урока: один IL — разные ОС.

#### Требования к решению
- Проект должен таргетировать `net8.0` (TFM), без суффиксов `-windows`/`-ios`, поскольку приложение не использует платформо-специфичные API.
- Точка входа должна быть написана в стиле top-level statements: ни `class Program`, ни `static void Main` вручную не объявляются.
- Список версий .NET должен быть объявлен с помощью collection expression (`List<string> x = [ ... ];`), а не `new List<string> { ... }`.
- Многострочный отчёт должен формироваться через raw string literal (`""" ... """`) с интерполяцией (`$""" ... """`) там, где нужно подставить значение.
- Обязательно использование как минимум трёх пространств имён из BCL: `System` (`Console`, `Environment`), `System.IO` (`Path`, `Directory`, `File`), `System.Linq` (`Where`, `Select`, `First`), `System.Collections.Generic` (`List<T>`, `Dictionary<K,V>`).
- В коде должна быть продемонстрирована работа с управляемой памятью: создаётся `Dictionary<string, object>`, но ни одного `Dispose` или финализатора вручную не вызывается — GC освобождает его сам. Это иллюстрирует managed code.
- Файл записывается через `File.WriteAllText` — вызов `Dispose`/`using` не нужен, потому что этот статический метод сам открывает, пишет и закрывает поток внутри себя. Ученик должен понимать разницу: `File.WriteAllText` безопасен по умолчанию, а вот `FileStream` в долгоживущем поле требует `Dispose` или `using`.
- Код должен компилироваться без предупреждений (команда `dotnet build -warnaserror`), если это требование не нарушается учебной необходимостью.
- Вывод программы должен содержать: версию рантайма (`Environment.Version`), определённую ОС, отфильтрованную рекомендованную версию .NET и путь к сохранённому файлу.

#### Тонкости и подводные камни
- **.NET Framework ≠ современный .NET.** Многие новички путают «.NET 4.8» и «.NET 8». Это разные рантаймы: Framework — только Windows и в режиме поддержки (security fixes), современный .NET (5/6/7/8) — кроссплатформенный и активно развивается. Версии 6 и 8 — LTS, 7 — STS (уже вышел из поддержки). Не пытайтесь запустить код с collection expressions на .NET Framework 4.8 — там нет компилятора C# 12.
- **C# — не .NET.** На .NET можно писать на F# и VB.NET; C# — лишь один из языков. Не называйте SDK «C# SDK» — это .NET SDK.
- **IL и JIT — два этапа.** `dotnet build` компилирует C# в IL (Intermediate Language), а JIT во время выполнения превращает IL в машинный код под конкретную ОС и архитектуру. Поэтому один `dll`/`exe` работает везде, где есть соответствующий CLR.
- **GC против Dispose.** GC освобождает управляемую память автоматически; вручную `Dispose` нужен только для неуправляемых ресурсов (файлы, сокеты, `SafeHandle`). Не вызывайте `GC.Collect()` без обоснования профайлером — это антипаттерн.
- **Кроссплатформенность ограничена системными API.** Если вы вызовете Win32 API через P/Invoke, код перестанет быть переносимым. Для чисто BCL-кода (`System.IO.Path`, `Environment.GetFolderPath`) переносимость сохраняется.
- **Top-level statements — не для всего.** Они работают только в одном файле проекта и только для точки входа. В библиотеках классов их нет. Если добавить ещё один метод рядом, он должен быть локальной функцией или static-методом в отдельном файле.
- **Collection expressions и типы.** `List<string> x = ["a", "b"];` работает благодаря неявному преобразованию коллекции, но тип слева должен поддерживать это (`List<T>`, `T[]`, `Span<T>`, `IEnumerable<T>`). Для `IEnumerable<T>` создаётся скрытая реализация.
- **Raw string literals и отступы.** Закрывающий `"""` задаёт базовый отступ: всё содержимое сдвигается влево на эту величину. Несоблюдение этого правила ломает форматирование отчёта.
- **`Environment.Version` в .NET 8** возвращает `8.0.x`, а не `4.0.x` (как в Framework). Не путайте с `Environment.OSVersion.Version` — это версия ОС, а не рантайма.
- **Implicit usings.** В `net8.0` включены по умолчанию для шаблона console. Если выключить `<ImplicitUsings>enable</ImplicitUsings>`, придётся добавлять `using System;` и др. вручную.

#### Критерии приёмки
- [ ] Создан проект `M01L01.Report` с TFM `net8.0`.
- [ ] `dotnet --version` показывает версию 8.x.x.
- [ ] `Program.cs` использует top-level statements (нет `class Program`/`Main`).
- [ ] Список версий .NET объявлен через collection expression `[ ... ]`.
- [ ] В коде присутствует raw string literal для многострочного отчёта.
- [ ] Использованы минимум три пространства имён BCL: `System`, `System.IO`, `System.Linq` (или `System.Collections.Generic`).
- [ ] Программа выводит версию рантайма через `Environment.Version`.
- [ ] Программа определяет ОС через `Environment.OSVersion.Platform`.
- [ ] LINQ-фильтрация находит рекомендованную версию .NET 8.
- [ ] Отчёт сохраняется в файл через `File.WriteAllText` в `SpecialFolder.LocalApplicationData`.
- [ ] Выводится путь к сохранённому файлу.
- [ ] Код не содержит ручного `Dispose`/`GC.Collect()` — управляемые объекты отданы GC.
- [ ] `dotnet build` завершается без ошибок (предупреждения допустимы, но желательно без них).
- [ ] `dotnet run` выводит ожидаемый текст и путь к файлу.
- [ ] Ученик может устно объяснить разницу между .NET Framework, .NET Core и современным .NET 5–8.

#### Подсказки (без прямого ответа)
- Для switch по `PlatformID` используйте switch expression — он короче if-else и идиоматичен для C# 8+.
- Для объединения путей применяйте `Path.Combine`, а не конкатенацию строк со слешами — это убирает проблемы с `\` vs `/` между ОС.
- Чтобы получить каталог для пользовательских данных приложения, используйте `Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData)`.
- Помните, что `File.WriteAllText` сам закрывает поток — это и есть пример API BCL, который прячет неуправляемый ресурс за удобной обёрткой.
- Для raw string literal следите, чтобы закрывающий `"""` стоял в той же колонке, что и желаемый базовый отступ содержимого.

#### Эталонное решение (разбор)
```csharp
// ДЗ M01-L01: самоописывающийся репорт окружения .NET / self-describing .NET environment report
// Top-level statements — точка входа без boilerplate-класса Program.
// Top-level statements — entry point without a boilerplate Program class.

// using не нужны благодаря ImplicitUsings в net8.0, но показаны для ясности:
// using System; using System.IO; using System.Linq; using System.Collections.Generic;

// 1) Кроссплатформенность: определяем ОС через BCL (System.Environment).
//    PlatformID — enum из BCL; switch expression — современный синтаксис C# 8+.
string osLabel = Environment.OSVersion.Platform switch
{
    PlatformID.Win32NT => "Windows",
    PlatformID.Unix    => "Linux/macOS",
    _                  => "Unknown OS"
};

// 2) Версия рантайма (НЕ версия ОС). В .NET 8 Environment.Version == 8.0.x.
//    Это доказывает, что код исполняется под управляемым CLR, а не нативно.
Console.WriteLine($"Привет от .NET {Environment.Version} на {osLabel}!");

// 3) Collection expression (C# 12): список версий .NET с пометкой LTS.
//    Исторические ветки: Framework, Core, современный .NET (5/6/7/8).
List<string> frameworks =
[
    ".NET Framework 4.8 (Windows-only, в поддержке)",
    "Mono (Xamarin/Unity)",
    ".NET Core 3.1 (конец поддержки)",
    ".NET 5 (конец поддержки)",
    ".NET 6 (LTS)",
    ".NET 7 (конец поддержки)",
    ".NET 8 (LTS, рекомендован)",
];

// 4) LINQ (System.Linq): фильтруем рекомендованные LTS-версии современного .NET.
//    Where + Select + First — цепочка методов расширения поверх IEnumerable<T>.
string recommended = frameworks
    .Where(f => f.Contains("8") && f.Contains("LTS"))
    .Select(f => f.Split('(')[0].Trim())
    .First();

// 5) Raw string literal (C# 11+) с интерполяцией — многострочный отчёт без эскейпов.
string report = $"""
    === Отчёт об окружении .NET / .NET Environment Report ===
    Операционная система / OS          : {osLabel}
    Версия рантайма .NET / Runtime     : {Environment.Version}
    Рекомендованная версия / Recommended: {recommended}

    История платформы / Platform history:
    - .NET Framework 4.x: только Windows, режим поддержки (security fixes).
    - Mono: независимая реализация, основа Xamarin/Unity.
    - .NET Core 1.x–3.1: кроссплатформенный предшественник современного .NET.
    - .NET 5/6/7/8: единая платформа; LTS у версий 6 и 8.

    Managed code: памятью управляет CLR через сборщик мусора (GC).
    Dispose нужен только для неуправляемых ресурсов (файлы, сокеты).
    """;

Console.WriteLine(report);

// 6) BCL System.Collections.Generic: Dictionary — управляемый объект, GC освободит сам.
var meta = new Dictionary<string, object>
{
    ["Language"]       = "C#",
    ["LanguageVersion"]= 12,
    ["Platform"]       = ".NET 8",
    ["CrossPlatform"]  = true,
    ["Managed"]        = true,
    ["BCL_Examples"]   = "System.IO, System.Linq, System.Collections.Generic",
};

foreach (var (key, value) in meta)
{
    Console.WriteLine($"  {key,-16}: {value}");
}

// 7) System.IO: сохраняем отчёт в файл. File.WriteAllText сам открывает/закрывает поток.
string outputDir = Path.Combine(
    Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
    "M01-L01");
Directory.CreateDirectory(outputDir);
string filePath = Path.Combine(outputDir, "report.txt");
File.WriteAllText(filePath, report);

Console.WriteLine($"Файл сохранён / File saved: {filePath}");
```

**Разбор по строкам.** Шаг 1 — switch expression по `PlatformID` демонстрирует, что определение ОС идёт через BCL, а не через P/Invoke к Win32, поэтому код остаётся кроссплатформенным. Шаг 2 — `Environment.Version` возвращает версию рантайма (8.0.x), что подтверждает тезис урока: код исполняется под управляемым CLR. Шаг 3 — collection expression `[ ... ]` вместо `new List<string> { ... }` — это C# 12; компилятор сам создаёт `List<string>` и заполняет его. Шаг 4 — LINQ-цепочка `Where → Select → First` иллюстрирует BCL `System.Linq`: фильтрация по наличию «8» и «LTS», проекция (отрезаем текст в скобках), выбор первого. Шаг 5 — raw string literal `$""" ... """` формирует многострочный отчёт без `\\n` и эскейпов кавычек; интерполяция подставляет переменные. Шаг 6 — `Dictionary<string, object>` создаётся без `Dispose`, потому что это чисто управляемый объект: GC соберёт его, когда он выйдет из области видимости. Шаг 7 — `File.WriteAllText` прячет неуправляемый ресурс (файловый дескриптор) за удобным статическим API; именно поэтому ручной `Dispose` не нужен, в отличие от долгоживущего `FileStream`. `Path.Combine` и `Environment.GetFolderPath` гарантируют корректные пути на любой ОС, что и доказывает кроссплатформенность. Все семь шагов суммарно закрепляют: C# как язык (`switch`, `var`, интерполяция, raw strings, collection expressions), .NET как платформу (CLR через `Environment.Version`, BCL через `System.*`, кроссплатформенность через `Path`/`SpecialFolder`), и границу GC/Dispose.

#### Задания на углубление (бонус)
1. **Добавьте отдельный метод `IsLts(string label)`,** который возвращает `true` для строк, содержащих «LTS». Примените его в LINQ-фильтрации вместо инлайн-условия. Подумайте: можно ли объявить этот метод как локальную функцию рядом с top-level statements?
2. **Сериализуйте словарь `meta` в JSON** через `System.Text.Json.JsonSerializer.Serialize` и допишите его в конец файла `report.txt` через `File.AppendAllText`. Объясните, почему `JsonSerializer` — это часть BCL, а не внешняя библиотека.
3. **Добавьте таймер:** измерьте время выполнения блока LINQ+сохранения через `System.Diagnostics.Stopwatch` и выведите его в отчёт. Обсудите, насколько корректно сравнивать производительность на холодном старте JIT.
4. **Соберите проект под две платформы** (`win-x64` и `linux-x64`) и сравните размеры `dll`. Объясните, почему IL-часть остаётся одинаковой, а нативная shim может отличаться.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you have joined a team that maintains a legacy application on .NET Framework 4.8 while simultaneously building a new microservice on modern .NET 8. Your tech lead asks you to prepare a "cheat-sheet report" about the current state of the .NET platform that should work as a standalone console utility: it detects which OS and which runtime version it is running on, collects reference data about historical .NET versions (Framework, Core, modern .NET with LTS marks), filters them through LINQ, builds a multi-line report with a raw string literal, and persists it to a file through `System.IO`. This is a realistic task: such self-describing utilities are commonly used in DevOps pipelines to diagnose build and deployment environments, where it is important to know whether the correct .NET image landed in the container and which OS lies underneath. The utility also serves as a tangible proof of the lesson's key claims: the same IL runs on different OSes because the CLR and BCL are implemented for each platform; at the same time, the code contains no manual memory release, because the GC reclaims managed objects on its own, while `Dispose` is only needed where unmanaged resources (files, sockets) exist. You learn not to confuse the C# language with the .NET platform, not to confuse .NET Framework with modern .NET, and to rely on the BCL instead of reinventing the wheel. In addition, you consolidate modern C# 12 syntactic features that reduce boilerplate and make the program more readable, which is especially valuable in short utilities and scripts.

#### What to do step by step
1. **Check the installed SDK.** Open a terminal and run `dotnet --version`. The expected output is `8.x.x` (for example `8.0.404`). If a version below 8 is installed, install the .NET 8 SDK from the official Microsoft site. Also run `dotnet --list-sdks` to confirm that SDK 8 is present in the list. Remember: the SDK and the Runtime are different entities; the SDK includes the Runtime for development, while production usually installs only the Runtime.
2. **Create the project.** In the terminal, navigate to your working folder (for example, `C:\projects\course\homework`) and run: `dotnet new console -n M01L01.Report -o M01L01.Report --framework net8.0`. The `--framework net8.0` flag pins the TFM (Target Framework Moniker). Open the `M01L01.Report.csproj` file and confirm that it contains `<TargetFramework>net8.0</TargetFramework>` and `<ImplicitUsings>enable</ImplicitUsings>`. Implicit usings wire up `System`, `System.IO`, `System.Linq` and `System.Collections.Generic` automatically — this is part of modern .NET.
3. **Inspect the generated `Program.cs`.** In .NET 8 the `dotnet new console` template produces a file with top-level statements: there is no `Program` class and no `Main` method. This is the modern entry-point style. Remember: the compiler synthesizes a `Program` class with a `Main` method under the hood.
4. **Implement the logic.** Replace the contents of `Program.cs` with the code from the "Reference solution" section. The program must: detect the OS through `Environment.OSVersion.Platform` (switch expression), print the runtime version through `Environment.Version`, declare the list of .NET versions through a collection expression (`[ ... ]`), filter it through LINQ (`Where`, `Select`, `First`), build the report through a raw string literal (`""" ... """`), save the report to a file through `File.WriteAllText` into the directory `Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "M01-L01")`, and print the path to the console.
5. **Run the project.** Execute `dotnet run --project M01L01.Report`. The expected output contains a line like "Hello from .NET 8.0.x on Windows" (or `Unix` on Linux/macOS), followed by the .NET version summary, then the metadata table, and finally the path to the saved file. Copy the file path and open it in a text editor — confirm that the contents match the printed summary.
6. **Publish the project as single-file.** Run `dotnet publish M01L01.Report -c Release -r win-x64 --self-contained false -o ./publish`. The `--self-contained false` flag produces a framework-dependent build (it depends on .NET 8 being installed on the target machine). Then run `./publish/M01L01.Report.exe` and confirm the utility works. This exercise illustrates that a single IL runs on different OSes: the same `dotnet publish -r linux-x64` would produce a Linux binary.
7. **Verify cross-platform behavior (optional, if you have access to another OS or Docker).** If Docker is installed, run `docker run --rm -v ${PWD}:/app -w /app mcr.microsoft.com/dotnet/sdk:8.0 dotnet run --project M01L01.Report`. The output will show `PlatformID.Unix`, which proves the lesson's claim: one IL — different OSes.

#### Requirements
- The project must target `net8.0` (TFM), without `-windows`/`-ios` suffixes, because the application uses no platform-specific APIs.
- The entry point must be written in top-level statements style: no manual `class Program` and no `static void Main`.
- The list of .NET versions must be declared with a collection expression (`List<string> x = [ ... ];`), not `new List<string> { ... }`.
- The multi-line report must be built with a raw string literal (`""" ... """`) with interpolation (`$""" ... """`) where a value needs to be substituted.
- At least three BCL namespaces must be used: `System` (`Console`, `Environment`), `System.IO` (`Path`, `Directory`, `File`), `System.Linq` (`Where`, `Select`, `First`), `System.Collections.Generic` (`List<T>`, `Dictionary<K,V>`).
- The code must demonstrate managed-memory behavior: a `Dictionary<string, object>` is created, but no `Dispose` or finalizer is called manually — the GC reclaims it on its own. This illustrates managed code.
- The file is written through `File.WriteAllText` — no `Dispose`/`using` is needed, because this static method opens, writes, and closes the stream internally. The student must understand the difference: `File.WriteAllText` is safe by default, whereas a long-lived `FileStream` field requires `Dispose` or `using`.
- The code must compile without warnings (`dotnet build -warnaserror`), unless an educational need overrides this requirement.
- The program output must contain: the runtime version (`Environment.Version`), the detected OS, the filtered recommended .NET version, and the path to the saved file.

#### Pitfalls
- **.NET Framework ≠ modern .NET.** Many beginners confuse ".NET 4.8" with ".NET 8". These are different runtimes: Framework is Windows-only and in maintenance (security fixes), while modern .NET (5/6/7/8) is cross-platform and actively developed. Versions 6 and 8 are LTS, 7 is STS (already out of support). Do not try to run code with collection expressions on .NET Framework 4.8 — there is no C# 12 compiler there.
- **C# is not .NET.** You can write F# and VB.NET on .NET; C# is just one of the languages. Do not call the SDK the "C# SDK" — it is the .NET SDK.
- **IL and JIT are two stages.** `dotnet build` compiles C# into IL (Intermediate Language), and the JIT turns IL into machine code for the specific OS and architecture at run time. This is why a single `dll`/`exe` works wherever a matching CLR exists.
- **GC vs Dispose.** The GC frees managed memory automatically; manual `Dispose` is only for unmanaged resources (files, sockets, `SafeHandle`). Do not call `GC.Collect()` without profiler evidence — it is an anti-pattern.
- **Cross-platform is limited by system APIs.** If you invoke a Win32 API via P/Invoke, the code stops being portable. For pure BCL code (`System.IO.Path`, `Environment.GetFolderPath`) portability is preserved.
- **Top-level statements are not for everything.** They work only in a single project file and only for the entry point. Class libraries do not have them. If you add another method next to them, it must be a local function or a static method in a separate file.
- **Collection expressions and types.** `List<string> x = ["a", "b"];` works thanks to collection-expression conversion, but the type on the left must support it (`List<T>`, `T[]`, `Span<T>`, `IEnumerable<T>`). For `IEnumerable<T>` a hidden implementation is created.
- **Raw string literals and indentation.** The closing `"""` defines the base indent: the entire content is shifted left by that amount. Violating this rule breaks the report formatting.
- **`Environment.Version` on .NET 8** returns `8.0.x`, not `4.0.x` (as on Framework). Do not confuse it with `Environment.OSVersion.Version` — that is the OS version, not the runtime version.
- **Implicit usings.** They are enabled by default for the console template in `net8.0`. If you turn off `<ImplicitUsings>enable</ImplicitUsings>`, you will have to add `using System;` and others manually.

#### Acceptance criteria
- [ ] A project `M01L01.Report` was created with the TFM `net8.0`.
- [ ] `dotnet --version` shows version 8.x.x.
- [ ] `Program.cs` uses top-level statements (no `class Program`/`Main`).
- [ ] The list of .NET versions is declared through a collection expression `[ ... ]`.
- [ ] The code contains a raw string literal for the multi-line report.
- [ ] At least three BCL namespaces are used: `System`, `System.IO`, `System.Linq` (or `System.Collections.Generic`).
- [ ] The program prints the runtime version through `Environment.Version`.
- [ ] The program detects the OS through `Environment.OSVersion.Platform`.
- [ ] LINQ filtering finds the recommended .NET 8 version.
- [ ] The report is saved to a file through `File.WriteAllText` into `SpecialFolder.LocalApplicationData`.
- [ ] The path to the saved file is printed.
- [ ] The code contains no manual `Dispose`/`GC.Collect()` — managed objects are left to the GC.
- [ ] `dotnet build` completes without errors (warnings are acceptable, but ideally none).
- [ ] `dotnet run` prints the expected text and the file path.
- [ ] The student can verbally explain the difference between .NET Framework, .NET Core, and modern .NET 5–8.

#### Hints (no direct answer)
- For the switch over `PlatformID`, use a switch expression — it is shorter than if-else and idiomatic for C# 8+.
- To combine paths, use `Path.Combine` instead of string concatenation with slashes — this removes `\` vs `/` issues across OSes.
- To get the application data directory for the current user, use `Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData)`.
- Remember that `File.WriteAllText` closes the stream by itself — this is an example of a BCL API that hides an unmanaged resource behind a convenient wrapper.
- For raw string literals, make sure the closing `"""` sits in the same column as the desired base indent of the content.

#### Reference solution walk-through
```csharp
// Homework M01-L01: self-describing .NET environment report
// Top-level statements — entry point without a boilerplate Program class.

// usings are implicit in net8.0, shown for clarity:
// using System; using System.IO; using System.Linq; using System.Collections.Generic;

// 1) Cross-platform: detect the OS via the BCL (System.Environment).
//    PlatformID is a BCL enum; switch expression is modern C# 8+ syntax.
string osLabel = Environment.OSVersion.Platform switch
{
    PlatformID.Win32NT => "Windows",
    PlatformID.Unix    => "Linux/macOS",
    _                  => "Unknown OS"
};

// 2) Runtime version (NOT the OS version). On .NET 8 Environment.Version == 8.0.x.
//    This proves the code runs under a managed CLR, not natively.
Console.WriteLine($"Hello from .NET {Environment.Version} on {osLabel}!");

// 3) Collection expression (C# 12): list of .NET versions with LTS marks.
//    Historical branches: Framework, Core, modern .NET (5/6/7/8).
List<string> frameworks =
[
    ".NET Framework 4.8 (Windows-only, in maintenance)",
    "Mono (Xamarin/Unity)",
    ".NET Core 3.1 (end of support)",
    ".NET 5 (end of support)",
    ".NET 6 (LTS)",
    ".NET 7 (end of support)",
    ".NET 8 (LTS, recommended)",
];

// 4) LINQ (System.Linq): filter the recommended LTS versions of modern .NET.
//    Where + Select + First — a chain of extension methods over IEnumerable<T>.
string recommended = frameworks
    .Where(f => f.Contains("8") && f.Contains("LTS"))
    .Select(f => f.Split('(')[0].Trim())
    .First();

// 5) Raw string literal (C# 11+) with interpolation — multi-line report without escapes.
string report = $"""
    === .NET Environment Report ===
    Operating system / OS           : {osLabel}
    .NET runtime version / Runtime  : {Environment.Version}
    Recommended version / Recommended: {recommended}

    Platform history:
    - .NET Framework 4.x: Windows-only, maintenance mode (security fixes).
    - Mono: independent implementation, basis of Xamarin/Unity.
    - .NET Core 1.x–3.1: cross-platform predecessor of modern .NET.
    - .NET 5/6/7/8: unified platform; LTS at versions 6 and 8.

    Managed code: memory is managed by the CLR via the garbage collector (GC).
    Dispose is only needed for unmanaged resources (files, sockets).
    """;

Console.WriteLine(report);

// 6) BCL System.Collections.Generic: Dictionary — a managed object, the GC frees it.
var meta = new Dictionary<string, object>
{
    ["Language"]       = "C#",
    ["LanguageVersion"]= 12,
    ["Platform"]       = ".NET 8",
    ["CrossPlatform"]  = true,
    ["Managed"]        = true,
    ["BCL_Examples"]   = "System.IO, System.Linq, System.Collections.Generic",
};

foreach (var (key, value) in meta)
{
    Console.WriteLine($"  {key,-16}: {value}");
}

// 7) System.IO: save the report to a file. File.WriteAllText opens/closes the stream itself.
string outputDir = Path.Combine(
    Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
    "M01-L01");
Directory.CreateDirectory(outputDir);
string filePath = Path.Combine(outputDir, "report.txt");
File.WriteAllText(filePath, report);

Console.WriteLine($"File saved: {filePath}");
```

**Line-by-line walk-through.** Step 1 — the switch expression over `PlatformID` shows that OS detection goes through the BCL, not through P/Invoke to Win32, so the code stays cross-platform. Step 2 — `Environment.Version` returns the runtime version (8.0.x), which confirms the lesson's claim: the code runs under a managed CLR. Step 3 — the collection expression `[ ... ]` instead of `new List<string> { ... }` is C# 12; the compiler builds the `List<string>` and fills it for you. Step 4 — the LINQ chain `Where → Select → First` illustrates the BCL `System.Linq`: filtering by the presence of "8" and "LTS", projection (stripping the parenthesized text), and selecting the first match. Step 5 — the raw string literal `$""" ... """` builds a multi-line report without `\\n` and without escaping quotes; interpolation substitutes the variables. Step 6 — the `Dictionary<string, object>` is created without `Dispose`, because it is a purely managed object: the GC collects it when it goes out of scope. Step 7 — `File.WriteAllText` hides the unmanaged resource (the file descriptor) behind a convenient static API; this is why a manual `Dispose` is not needed, unlike for a long-lived `FileStream`. `Path.Combine` and `Environment.GetFolderPath` guarantee correct paths on any OS, which proves cross-platform behavior. All seven steps together reinforce: C# as a language (`switch`, `var`, interpolation, raw strings, collection expressions), .NET as a platform (CLR via `Environment.Version`, BCL via `System.*`, cross-platform via `Path`/`SpecialFolder`), and the GC/Dispose boundary.

#### Going deeper (bonus)
1. **Add a separate method `IsLts(string label)`** that returns `true` for strings containing "LTS". Use it in the LINQ filter instead of the inline condition. Think: can you declare this method as a local function next to the top-level statements?
2. **Serialize the `meta` dictionary to JSON** through `System.Text.Json.JsonSerializer.Serialize` and append it to the end of `report.txt` via `File.AppendAllText`. Explain why `JsonSerializer` is part of the BCL and not an external library.
3. **Add a timer:** measure the duration of the LINQ+save block with `System.Diagnostics.Stopwatch` and include it in the report. Discuss how fair it is to compare performance on a cold JIT start.
4. **Build the project for two platforms** (`win-x64` and `linux-x64`) and compare the sizes of the `dll`. Explain why the IL part stays the same while the native shim may differ.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `M01L01.Report` создан, TFM `net8.0`.
- [ ] (RU) `Program.cs` использует top-level statements.
- [ ] (RU) Применены collection expressions и raw string literal.
- [ ] (RU) Использованы минимум три пространства имён BCL.
- [ ] (RU) Программа выводит версию рантайма и определяет ОС.
- [ ] (RU) Отчёт сохраняется в файл через `System.IO`.
- [ ] (RU) Нет ручного `Dispose`/`GC.Collect()` для управляемых объектов.
- [ ] (RU) Ученик объясняет разницу Framework/Core/современный .NET.
- [ ] (EN) Project `M01L01.Report` created, TFM `net8.0`.
- [ ] (EN) `Program.cs` uses top-level statements.
- [ ] (EN) Collection expressions and raw string literal are used.
- [ ] (EN) At least three BCL namespaces are used.
- [ ] (EN) The program prints the runtime version and detects the OS.
- [ ] (EN) The report is saved to a file through `System.IO`.
- [ ] (EN) No manual `Dispose`/`GC.Collect()` for managed objects.
- [ ] (EN) The student explains Framework/Core/modern .NET differences.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/tour-of-csharp/ — Tour of C# / Обзор языка C#
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/introduction — Introduction to .NET / Введение в .NET
- Microsoft Learn — https://learn.microsoft.com/dotnet/standard/components — .NET architectural components / Архитектурные компоненты .NET
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12 — What's new in C# 12 / Что нового в C# 12
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/strings/raw-string-literals — Raw string literals
- .NET GitHub — https://github.com/dotnet — platform source code / исходный код платформы

---
[← К уроку M01-L01](lesson-M01-L01-csharp-dotnet-intro.md) | [⬆ К модулю M01](../README.md) | [Следующее ДЗ: M01-L02 →](homework-M01-L02-clr-il-jit.md)
---
