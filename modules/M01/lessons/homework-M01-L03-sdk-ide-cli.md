---
[← К уроку M01-L03](lesson-M01-L03-sdk-ide-cli.md) | [⬪ К модулю M01](../README.md) | [Следующее ДЗ →](homework-M01-L04-dotnet-new-toplevel.md)
---

### Домашнее задание M01-L03: Установка SDK, IDE, dotnet CLI / Homework M01-L03: Installing SDK, IDE, dotnet CLI

**Урок / Lesson:** M01-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 2/5
**Цель / Goal:** (RU) Самостоятельно установить .NET SDK 8, настроить IDE, освоить ключевые команды `dotnet CLI` (`--version`, `--info`, `--list-sdks`, `--list-runtimes`, `new`, `build`, `run`, `add package`), научиться фиксировать версию SDK через `global.json` с полем `rollForward` и написать консольное приложение на C# 12, которое диагностирует окружение разработки, используя top-level statements, raw string literals и pattern matching. (EN) Independently install .NET SDK 8, configure an IDE, master the key `dotnet CLI` commands (`--version`, `--info`, `--list-sdks`, `--list-runtimes`, `new`, `build`, `run`, `add package`), learn to pin the SDK version with `global.json` and its `rollForward` field, and write a C# 12 console application that diagnoses the development environment using top-level statements, raw string literals, and pattern matching.

#### Связь с уроком / Connection to the lesson
(RU) Домашнее задание напрямую закрепляет ключевые концепции урока M01-L03: различие между SDK и Runtime, диагностику окружения через `dotnet --info`/`--list-sdks`/`--list-runtimes`, фиксацию версии SDK через `global.json` с политикой `rollForward`, а также базовые команды CLI `dotnet new`, `build`, `run`. Повторяются примеры кода урока — top-level statements, `Environment.Version`, `RuntimeInformation`, raw string literals и pattern matching через `switch`. (EN) This homework directly reinforces the core concepts of lesson M01-L03: the distinction between SDK and Runtime, environment diagnostics through `dotnet --info`/`--list-sdks`/`--list-runtimes`, pinning the SDK version with `global.json` and its `rollForward` policy, and the basic CLI commands `dotnet new`, `build`, `run`. The lesson's code patterns are reused — top-level statements, `Environment.Version`, `RuntimeInformation`, raw string literals, and pattern matching via `switch`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы только что закончили изучать урок об установке .NET SDK 8, выборе IDE и работе с `dotnet CLI`. Теория понятна: SDK — это «кухня» с компилятором Roslyn и шаблонами, а Runtime — «обеденный зал», где уже готовые приложения подаются пользователю. Однако теория без практики быстро забывается, а окружение разработчика — это первое, что ломается у новичка: то `dotnet` не найден в терминале, то сборка падает с непонятной ошибкой версии SDK, то на macOS случайно ставят `x64` вместо `Arm64`. Чтобы не наступать на эти грабли в реальном проекте, вы должны один раз пройти весь путь «от нуля до работающего диагностического приложения» собственными руками.

В этом задании вы扮演аете роль онбординг-инженера, которому нужно подготовить чистую машину к работе над командным проектом на C# 12 / .NET 8. Вы установите SDK, проверите окружение, зафиксируете версию SDK в `global.json`, создадите консольный проект «Environment Probe», который собирает и выводит всю диагностическую информацию об окружении, добавите в него NuGet-пакет и убедитесь, что приложение работает в обеих конфигурациях — Debug и Release. Заодно вы потренируете современные возможности C# 12: top-level statements, raw string literals и pattern matching, которые были показаны в примере кода урока. По итогам у вас будет воспроизводимый чек-лист онбординга, который можно использовать на каждой новой машине.

#### Что нужно сделать (пошагово)
1. **Установите .NET SDK 8.** На Windows выполните `winget install Microsoft.DotNet.SDK.8`; на macOS Apple Silicon скачайте `Arm64`-установщик с https://dotnet.microsoft.com/download; на Linux используйте `apt install dotnet-sdk-8.0` или скрипт `dotnet-install.sh`. После установки **обязательно перезапустите терминал**, иначе переменная PATH не обновится и команда `dotnet` будет «не найдена» — это самая частая ошибка из урока.

2. **Проверьте установку тремя командами.** Выполните `dotnet --version` (ожидаемый вывод — что-то вроде `8.0.1xx`), затем `dotnet --info` (полный отчёт: версии всех SDK и Runtime, ОС, архитектура) и отдельно `dotnet --list-sdks` и `dotnet --list-runtimes`. Сравните вывод `--list-runtimes` с `--list-sdks`: вы должны увидеть, что SDK включает несколько Runtime (Microsoft.NETCore.App, Microsoft.AspNetCore.App, возможно Microsoft.WindowsDesktop.App), тогда как машина без SDK показала бы только Runtime. Скопируйте точную версию SDK из вывода — она понадобится для `global.json`.

3. **Зафиксируйте версию SDK в `global.json`.** Создайте пустую папку `m01-l03-homework` и внутри неё выполните `dotnet new globaljson --sdk-version <ваша-версия>`, подставив версию из шага 2. Откройте полученный `global.json` и добавьте поле `"rollForward": "patch"` (или `"latestMinor"`), чтобы при отсутствии точной версии CLI брал ближайший патч. Объясните в комментарии файла, почему вы выбрали именно это значение.

4. **Создайте консольный проект.** В той же папке выполните `dotnet new console -n EnvironmentProbe -f net8.0`. Убедитесь, что создался файл `EnvironmentProbe.csproj` с `<TargetFramework>net8.0</TargetFramework>` и `Program.cs` с top-level statements, выводящими `Hello, World!`. Запустите его: `dotnet run --project EnvironmentProbe`. Ожидаемый вывод — `Hello, World!`.

5. **Замените `Program.cs` на диагностическое приложение.** Напишите код (см. эталонное решение ниже), который: выводит версию .NET Runtime через `Environment.Version`, описание ОС и архитектуру через `RuntimeInformation`, определяет конфигурацию сборки (`Debug`/`Release`) через директивы `#if DEBUG` и сопоставляет её pattern matching-выражением `switch`, а также ищет и читает `global.json` рядом с исполняемым файлом и выводит его содержимое в красивом баннере через raw string literal. Используйте только top-level statements — без явного `class Program` и `static void Main`.

6. **Добавьте NuGet-пакет.** Выполните `dotnet add EnvironmentProbe package Spectre.Console` (или `Newtonsoft.Json`, если Spectre недоступен). Убедитесь через `dotnet restore`, что пакет восстановился, и что в `EnvironmentProbe.csproj` появилась строка `<PackageReference Include="..." />`. В коде использовать пакет не обязательно — цель шага в том, чтобы вы один раз прошли полный цикл `add package` + `restore`.

7. **Соберите и запустите в двух конфигурациях.** Выполните `dotnet build EnvironmentProbe` (Debug по умолчанию), затем `dotnet run --project EnvironmentProbe -c Release`. Сравните вывод: в Debug pattern matching должен сообщить «отладочная сборка», в Release — «релизная». Это закрепляет связь директив `#if DEBUG` с конфигурацией.

8. **Создайте `.gitignore`.** Выполните `dotnet new gitignore` в корне `m01-l03-homework`. Проверьте, что папки `bin/` и `obj/` теперь игнорируются. Этот шаг нужен, чтобы с первого дня приучить себя не коммитить артефакты сборки.

#### Требования к решению
- На машине установлен именно .NET **SDK 8**, а не только Runtime — это проверяется командой `dotnet --list-sdks`, в выводе которой должна присутствовать строка вида `8.0.1xx`. Если список пуст, задание считается невыполненным.
- В корне папки `m01-l03-homework` лежит файл `global.json` с полями `sdk.version` и `sdk.rollForward`; версия совпадает с одной из установленных, а значение `rollForward` — одно из допустимых (`disable`, `patch`, `feature`, `minor`, `latestPatch`, `latestFeature`, `latestMinor`).
- Проект `EnvironmentProbe` создан командой `dotnet new console`, целевой фреймворк — `net8.0`, язык — C# 12 (подразумевается SDK 8).
- `Program.cs` использует **только** top-level statements: никаких явных `class Program` / `static void Main` / `namespace`. В файле есть raw string literal (`"""..."""`) для баннера и pattern matching через `switch` для конфигурации сборки.
- В коде используется `Environment.Version`, `RuntimeInformation.OSDescription`, `RuntimeInformation.OSArchitecture`, `AppContext.BaseDirectory` и `File.Exists`/`File.ReadAllText` для чтения `global.json`.
- В `EnvironmentProbe.csproj` есть как минимум один `<PackageReference>`, добавленный через `dotnet add package`, и `dotnet restore` завершается без ошибок.
- Приложение запускается и в Debug, и в Release через `dotnet run -c`, и вывод корректно меняется в зависимости от конфигурации благодаря `#if DEBUG`.
- В репозитории есть `.gitignore` из шаблона `dotnet new gitignore`, папки `bin/` и `obj/` в него попадают.

#### Тонкости и подводные камни
- **PATH не обновился.** Самая частая ошибка из урока: после установки SDK команда `dotnet` «не найдена». Решение — перезапустить терминал, а в крайнем случае на Windows проверить переменную среды в «Свойствах системы → Переменные среды». IDE тоже нужно перезапустить, иначе она «видит» старый PATH.
- **SDK и Runtime — разные вещи.** Если поставить только Runtime, то `dotnet build` не сработает, потому что компилятор Roslyn живёт именно в SDK. Проверяйте через `dotnet --list-sdks` — список не должен быть пустым.
- **Архитектура на macOS.** На Apple Silicon нужно ставить `Arm64`-сборку; `x64` будет работать через трансляцию Rosetta 2, но медленнее. Если случайно поставили `x64` — удалите и переустановите `Arm64`.
- **`global.json` и `rollForward`.** Если указать точную версию SDK, которой нет на машине, и `rollForward: "disable"`, сборка упадёт. Поэтому для командной работы обычно выбирают `patch` или `latestMinor`. Внимательно проверьте, что версия в `global.json` действительно установлена (`dotnet --list-sdks`).
- **Где лежит `global.json`.** CLI ищет его от текущей дектории вверх по дереву. Если вы создали его не в корне решения, а глубоко внутри проекта, он всё равно подействует — но это запутает коллег. Держите его в корне.
- **`#if DEBUG` и `dotnet run -c Release`.** Директива `DEBUG` определяется только в конфигурации Debug. Если вы запускаете `dotnet run` без `-c`, по умолчанию используется Debug — и ветка pattern matching «отладочная сборка» сработает. Чтобы увидеть «релизную», обязательно передайте `-c Release`.
- **raw string literal и отступы.** Содержимое `"""..."""` выравнивается по минимальному отступу закрывающих кавычек. Если отступы сбиты, в выводе появятся лишние пробелы. Следите, чтобы закрывающие `"""` стояли на отдельной строке с тем же отступом, что и крайний левый контент.
- **`AppContext.BaseDirectory` vs `Directory.GetCurrentDirectory`.** В первом случае — папка, где лежит скомпилированный бинарник (`bin/.../`), во втором — папка, откуда запущен `dotnet run`. Для поиска `global.json`, который вы положили в корень решения, надёжнее всего подниматься вверх от `BaseDirectory` до первого совпадения — именно так сделано в эталонном решении.
- **`dotnet restore` может быть не нужен.** Начиная с .NET Core 2.0, `dotnet run` и `dotnet build` неявно делают restore. Но явный `dotnet restore` полезен для диагностики NuGet-проблем и обязателен в CI — поэтому в задании он есть как отдельный шаг.

#### Критерии приёмки
- [ ] `dotnet --version` выводит версию `8.0.1xx` без ошибки «команда не найдена».
- [ ] Вывод `dotnet --list-sdks` содержит SDK версии 8.0; вывод `--list-runtimes` содержит `Microsoft.NETCore.App`.
- [ ] В корне `m01-l03-homework` лежит `global.json` с `sdk.version` и `sdk.rollForward`.
- [ ] Значение `rollForward` — одно из допустимых и обосновано комментарием в файле.
- [ ] Проект `EnvironmentProbe` создан через `dotnet new console`, целевой фреймворк `net8.0`.
- [ ] `Program.cs` использует только top-level statements, без `class Program` и `Main`.
- [ ] В коде есть raw string literal `"""..."""` для баннера.
- [ ] В коде есть pattern matching `switch` по конфигурации сборки.
- [ ] Выводятся `Environment.Version`, `RuntimeInformation.OSDescription`, `RuntimeInformation.OSArchitecture`.
- [ ] Приложение ищет `global.json` и выводит его содержимое, если найден, или сообщение об отсутствии.
- [ ] В `EnvironmentProbe.csproj` есть `<PackageReference>` через `dotnet add package`.
- [ ] `dotnet restore` завершается успешно, без ошибок NuGet.
- [ ] `dotnet run --project EnvironmentProbe` (Debug) выводит ветку «отладочная сборка».
- [ ] `dotnet run --project EnvironmentProbe -c Release` выводит ветку «релизная сборка».
- [ ] В корне лежит `.gitignore` из шаблона `dotnet new gitignore`, `bin/` и `obj/` игнорируются.

#### Подсказки (без прямого ответа)
- Если `dotnet` не находится после установки — проблема не в установщике, а в сессии терминала. Закройте все окна терминала и откройте заново.
- Для `global.json` версия SDK берётся из первой колонки вывода `dotnet --list-sdks` (формат `8.0.1xx [C:\...]`).
- Чтобы найти `global.json` из кода, поднимайтесь от `AppContext.BaseDirectory` вверх через `Directory.GetParent`, пока не дойдёте до корня диска — где-то по пути файл должен найтись.
- В pattern matching по строке конфигурации не забудьте ветку `_` для неизвестных значений — это best practice даже когда вариантов всего два.
- Raw string literal с тремя кавычками позволяет вставлять внутри обычные кавычки и `{}` без экранирования — используйте это для баннера с разделителями.

#### Эталонное решение (разбор)
```csharp
// EnvironmentProbe/Program.cs
// M01-L03: диагностика окружения разработки
// M01-L03: development environment diagnostics
// C# 12 / .NET 8, top-level statements

using System.Runtime.InteropServices;

// --- 1. Базовая информация о Runtime и ОС / Basic Runtime & OS info ---
Console.WriteLine($"Runtime:      .NET {Environment.Version}");
Console.WriteLine($"OS:           {RuntimeInformation.OSDescription}");
Console.WriteLine($"Architecture: {RuntimeInformation.OSArchitecture}");
Console.WriteLine($"Base dir:     {AppContext.BaseDirectory}");

// --- 2. Поиск global.json вверх по дереву / Walk up the tree for global.json ---
string? globalJsonPath = FindGlobalJson(AppContext.BaseDirectory);

// raw string literal (C# 11+) для многострочного баннера
// raw string literal (C# 11+) for a multi-line banner
string banner = """
               ────────────────────────────────────────
                 global.json report / отчёт global.json
               ────────────────────────────────────────
               """;

if (globalJsonPath is not null)
{
    Console.WriteLine(banner);
    Console.WriteLine($"Path: {globalJsonPath}");
    Console.WriteLine(File.ReadAllText(globalJsonPath));
}
else
{
    Console.WriteLine("global.json not found / не найден — будет выбран последний SDK.");
}

// --- 3. Конфигурация сборки через #if DEBUG + pattern matching ---
// Build configuration via #if DEBUG + pattern matching
#if DEBUG
    const string config = "Debug";
#else
    const string config = "Release";
#endif

string message = config switch
{
    "Debug"   => "Отладочная сборка — удобна для разработки / Debug build — good for development.",
    "Release" => "Релизная сборка — оптимизирована для продакшена / Release build — optimized for production.",
    _         => "Неизвестная конфигурация / Unknown configuration."
};
Console.WriteLine(message);

// --- 4. Локальная функция поиска global.json / Local function to locate global.json ---
static string? FindGlobalJson(string startDir)
{
    DirectoryInfo? dir = new(startDir);
    while (dir is not null)
    {
        string candidate = Path.Combine(dir.FullName, "global.json");
        if (File.Exists(candidate))
        {
            return candidate;
        }
        dir = dir.Parent;
    }
    return null;
}
```

**Разбор по строкам.** Первые три `Console.WriteLine` повторяют пример кода из урока: `Environment.Version` даёт версию CLR, на которой реально работает программа (не путать с версией SDK!), а `RuntimeInformation` сообщает ОС и архитектуру — это та информация, которую вы только что видели в `dotnet --info`, но теперь получаете изнутри самого приложения. `AppContext.BaseDirectory` важен, потому что `dotnet run` запускает бинарник из `bin/Debug/net8.0/`, и именно оттуда нужно начинать поиск `global.json`, а не из текущей рабочей директории.

Далее идёт локальная функция `FindGlobalJson`, которая поднимается от `BaseDirectory` вверх через `DirectoryInfo.Parent`, пока не найдёт `global.json` или не упрётся в корень диска. Это решение «подводного камня» из урока: CLI ищет `global.json` от текущей директории вверх, а наше приложение должно повторить ту же логику относительно своего бинарника. Возвращаемый тип `string?` и проверка `is not null` — современный nullable-анализ C# 8+.

Raw string literal `"""..."""` позволяет вывести многострочный баннер без экранирования `\n` и без возни с `$"..."`. Отступы внутри выравниваются по закрывающим кавычкам — поэтому три строки баннера в выводе окажутся без ведущих пробелов. Если бы внутри нужно было вставить фигурные скобки или кавычки, их не пришлось бы экранировать.

Блок `#if DEBUG ... #else ... #endif` — единственное место, где смешаны препроцессор и runtime-код. Константа `config` получает значение на этапе компиляции, и именно поэтому при `dotnet run -c Release` ветка меняется. Pattern matching через `switch` превращает строку в человекочитаемое сообщение; ветка `_` защищает от будущих конфигураций (например, `PublishSingleFile`). Всё это — top-level statements: нет ни `class Program`, ни `Main`, компилятор сам генерирует точку входа. Именно так и должен выглядеть современный консольный код на .NET 8, что и подчёркивается в примере урока.

#### Задания на углубление (бонус)
1. **Эксперимент с `rollForward`.** Временно измените версию в `global.json` на несуществующую (например, `8.0.999`) и перебирайте значения `rollForward` от `disable` до `latestMinor`. Зафиксируйте, при каких значениях `dotnet build` проходит, а при каких падает. Сделайте вывод о том, какое значение безопасно для командной работы.
2. **Диагностика «You must install .NET».** Сымитируйте ситуацию урока: найдите машину (или контейнер) только с Runtime, без SDK, и попробуйте запустить ваш опубликованный `EnvironmentProbe.dll` через `dotnet EnvironmentProbe.dll`. Добейтесь ошибки и научитесь её читать.
3. **Скрипт онбординга.** Напишите PowerShell- или bash-скрипт `probe.sh`/`probe.ps1`, который автоматически выполняет `dotnet --version`, `--list-sdks`, `--list-runtimes` и формирует Markdown-отчёт о готовности машины к разработке. Это мост к CI/CD, о котором упоминается в уроке.
4. **PublishSingleFile.** Опубликуйте приложение как один файл: `dotnet publish -c Release -r win-x64 --self-contained false -p:PublishSingleFile=true`. Проверьте, что `FindGlobalJson` по-прежнему находит `global.json`, если положить его рядом с опубликованным `.exe`.

---

## Statement in English / Постановка на английском

### Homework M01-L03: Installing SDK, IDE, dotnet CLI

#### Context & motivation
You have just finished the lesson on installing the .NET SDK 8, choosing an IDE, and working with the `dotnet CLI`. The theory is clear: the SDK is the "kitchen" with the Roslyn compiler and templates, while the Runtime is the "dining hall" where finished applications are served to the user. Theory without practice fades quickly, however, and the developer environment is the very first thing that breaks for a newcomer: the `dotnet` command is not found in the terminal, the build crashes with a cryptic SDK version error, or on macOS the `x64` SDK is installed instead of `Arm64`. To avoid stepping on these rakes in a real project, you must walk the whole path "from zero to a working diagnostic application" with your own hands exactly once.

In this assignment you play the role of an onboarding engineer who needs to prepare a clean machine for work on a team C# 12 / .NET 8 project. You will install the SDK, verify the environment, pin the SDK version in `global.json`, create an "Environment Probe" console project that collects and prints all the diagnostic information about the environment, add a NuGet package to it, and make sure the application works in both Debug and Release configurations. Along the way you will practice modern C# 12 features: top-level statements, raw string literals, and pattern matching, all of which were shown in the lesson's code example. As a result you will have a reproducible onboarding checklist that you can reuse on every new machine.

#### What to do step by step
1. **Install .NET SDK 8.** On Windows run `winget install Microsoft.DotNet.SDK.8`; on macOS Apple Silicon download the `Arm64` installer from https://dotnet.microsoft.com/download; on Linux use `apt install dotnet-sdk-8.0` or the `dotnet-install.sh` script. After the install **make sure to restart the terminal**, otherwise the PATH variable will not refresh and the `dotnet` command will be "not found" — this is the most common mistake mentioned in the lesson.

2. **Verify the installation with three commands.** Run `dotnet --version` (expected output is something like `8.0.1xx`), then `dotnet --info` (a full report: versions of every SDK and Runtime, OS, architecture), and separately `dotnet --list-sdks` and `dotnet --list-runtimes`. Compare the output of `--list-runtimes` with `--list-sdks`: you should see that the SDK ships with several runtimes (Microsoft.NETCore.App, Microsoft.AspNetCore.App, possibly Microsoft.WindowsDesktop.App), whereas a machine without the SDK would show only runtimes. Copy the exact SDK version from the output — you will need it for `global.json`.

3. **Pin the SDK version in `global.json`.** Create an empty folder `m01-l03-homework` and inside it run `dotnet new globaljson --sdk-version <your-version>`, substituting the version from step 2. Open the resulting `global.json` and add the field `"rollForward": "patch"` (or `"latestMinor"`) so that, when the exact version is missing, the CLI picks the closest patch. Explain in a comment in the file why you chose this value.

4. **Create the console project.** In the same folder run `dotnet new console -n EnvironmentProbe -f net8.0`. Make sure that the file `EnvironmentProbe.csproj` was created with `<TargetFramework>net8.0</TargetFramework>` and that `Program.cs` contains top-level statements that print `Hello, World!`. Run it: `dotnet run --project EnvironmentProbe`. Expected output — `Hello, World!`.

5. **Replace `Program.cs` with the diagnostic application.** Write the code (see the reference solution below) that: prints the .NET Runtime version via `Environment.Version`, the OS description and architecture via `RuntimeInformation`, determines the build configuration (`Debug`/`Release`) via `#if DEBUG` directives and maps it with a pattern matching `switch` expression, and also searches for and reads `global.json` next to the executable and prints its content in a nice banner via a raw string literal. Use only top-level statements — no explicit `class Program` and `static void Main`.

6. **Add a NuGet package.** Run `dotnet add EnvironmentProbe package Spectre.Console` (or `Newtonsoft.Json` if Spectre is unavailable). Verify with `dotnet restore` that the package was restored and that a `<PackageReference Include="..." />` line appeared in `EnvironmentProbe.csproj`. You do not have to use the package in code — the goal of this step is for you to walk the full `add package` + `restore` cycle once.

7. **Build and run in two configurations.** Run `dotnet build EnvironmentProbe` (Debug by default), then `dotnet run --project EnvironmentProbe -c Release`. Compare the output: in Debug the pattern matching should say "debug build", in Release — "release build". This reinforces the link between `#if DEBUG` and the configuration.

8. **Create a `.gitignore`.** Run `dotnet new gitignore` in the root of `m01-l03-homework`. Check that the `bin/` and `obj/` folders are now ignored. This step is needed so that from day one you train yourself not to commit build artifacts.

#### Requirements
- The machine has .NET **SDK 8** installed, not just the Runtime — this is verified by `dotnet --list-sdks`, whose output must contain a line like `8.0.1xx`. If the list is empty, the assignment is considered failed.
- The root of `m01-l03-homework` contains a `global.json` with the `sdk.version` and `sdk.rollForward` fields; the version matches one of the installed ones, and the value of `rollForward` is one of the allowed (`disable`, `patch`, `feature`, `minor`, `latestPatch`, `latestFeature`, `latestMinor`).
- The `EnvironmentProbe` project was created with `dotnet new console`, the target framework is `net8.0`, the language is C# 12 (implied by SDK 8).
- `Program.cs` uses **only** top-level statements: no explicit `class Program` / `static void Main` / `namespace`. The file contains a raw string literal (`"""..."""`) for the banner and pattern matching via `switch` for the build configuration.
- The code uses `Environment.Version`, `RuntimeInformation.OSDescription`, `RuntimeInformation.OSArchitecture`, `AppContext.BaseDirectory`, and `File.Exists`/`File.ReadAllText` to read `global.json`.
- `EnvironmentProbe.csproj` contains at least one `<PackageReference>` added via `dotnet add package`, and `dotnet restore` completes without errors.
- The application runs in both Debug and Release via `dotnet run -c`, and the output changes correctly depending on the configuration thanks to `#if DEBUG`.
- The repository contains a `.gitignore` from the `dotnet new gitignore` template, and the `bin/` and `obj/` folders are covered by it.

#### Pitfalls
- **PATH did not refresh.** The most common mistake from the lesson: after installing the SDK the `dotnet` command is "not found". The fix is to restart the terminal, and in the worst case on Windows check the environment variable in "System Properties → Environment Variables". The IDE must also be restarted, otherwise it "sees" the old PATH.
- **SDK and Runtime are different things.** If you install only the Runtime, `dotnet build` will not work, because the Roslyn compiler lives in the SDK. Check with `dotnet --list-sdks` — the list must not be empty.
- **Architecture on macOS.** On Apple Silicon you must install the `Arm64` build; `x64` will work through the Rosetta 2 translation but slower. If you accidentally installed `x64` — uninstall it and reinstall `Arm64`.
- **`global.json` and `rollForward`.** If you specify an exact SDK version that is not on the machine and set `rollForward: "disable"`, the build will crash. That is why for team work `patch` or `latestMinor` is usually chosen. Carefully verify that the version in `global.json` is actually installed (`dotnet --list-sdks`).
- **Where `global.json` lives.** The CLI looks for it from the current directory up the tree. If you created it not at the solution root but deep inside a project, it will still take effect — but this will confuse colleagues. Keep it at the root.
- **`#if DEBUG` and `dotnet run -c Release`.** The `DEBUG` directive is only defined in the Debug configuration. If you run `dotnet run` without `-c`, Debug is used by default — and the "debug build" branch of the pattern matching will fire. To see the "release" one, you must pass `-c Release`.
- **Raw string literal and indentation.** The content of `"""..."""` is aligned by the minimum indentation of the closing quotes. If the indentation is off, extra spaces will appear in the output. Make sure the closing `"""` sits on its own line with the same indentation as the leftmost content.
- **`AppContext.BaseDirectory` vs `Directory.GetCurrentDirectory`.** The former is the folder where the compiled binary lives (`bin/.../`), the latter is the folder from which `dotnet run` was launched. To find `global.json`, which you placed at the solution root, it is most reliable to walk upward from `BaseDirectory` to the first match — exactly as done in the reference solution.
- **`dotnet restore` may be unnecessary.** Starting with .NET Core 2.0, `dotnet run` and `dotnet build` do an implicit restore. But an explicit `dotnet restore` is useful for NuGet diagnostics and is mandatory in CI — that is why it appears in the assignment as a separate step.

#### Acceptance criteria
- [ ] `dotnet --version` prints a version like `8.0.1xx` without a "command not found" error.
- [ ] The output of `dotnet --list-sdks` contains an SDK of version 8.0; the output of `--list-runtimes` contains `Microsoft.NETCore.App`.
- [ ] The root of `m01-l03-homework` contains a `global.json` with `sdk.version` and `sdk.rollForward`.
- [ ] The value of `rollForward` is one of the allowed ones and is justified by a comment in the file.
- [ ] The `EnvironmentProbe` project was created with `dotnet new console`, the target framework is `net8.0`.
- [ ] `Program.cs` uses only top-level statements, without `class Program` and `Main`.
- [ ] The code contains a raw string literal `"""..."""` for the banner.
- [ ] The code contains a pattern matching `switch` over the build configuration.
- [ ] `Environment.Version`, `RuntimeInformation.OSDescription`, `RuntimeInformation.OSArchitecture` are printed.
- [ ] The application searches for `global.json` and prints its content if found, or a message about its absence.
- [ ] `EnvironmentProbe.csproj` contains a `<PackageReference>` via `dotnet add package`.
- [ ] `dotnet restore` completes successfully, without NuGet errors.
- [ ] `dotnet run --project EnvironmentProbe` (Debug) prints the "debug build" branch.
- [ ] `dotnet run --project EnvironmentProbe -c Release` prints the "release build" branch.
- [ ] The root contains a `.gitignore` from the `dotnet new gitignore` template, `bin/` and `obj/` are ignored.

#### Hints (no direct answer)
- If `dotnet` is not found after installation — the problem is not the installer but the terminal session. Close all terminal windows and open them again.
- For `global.json` the SDK version is taken from the first column of the `dotnet --list-sdks` output (format `8.0.1xx [C:\...]`).
- To find `global.json` from code, walk upward from `AppContext.BaseDirectory` via `Directory.GetParent` until you reach the drive root — somewhere along the way the file should be found.
- In pattern matching over the configuration string, do not forget the `_` branch for unknown values — this is a best practice even when there are only two options.
- A raw string literal with three quotes lets you insert ordinary quotes and `{}` inside without escaping — use this for a banner with separators.

#### Reference solution walk-through
```csharp
// EnvironmentProbe/Program.cs
// M01-L03: development environment diagnostics
// C# 12 / .NET 8, top-level statements

using System.Runtime.InteropServices;

// --- 1. Basic Runtime & OS info ---
Console.WriteLine($"Runtime:      .NET {Environment.Version}");
Console.WriteLine($"OS:           {RuntimeInformation.OSDescription}");
Console.WriteLine($"Architecture: {RuntimeInformation.OSArchitecture}");
Console.WriteLine($"Base dir:     {AppContext.BaseDirectory}");

// --- 2. Walk up the tree for global.json ---
string? globalJsonPath = FindGlobalJson(AppContext.BaseDirectory);

// raw string literal (C# 11+) for a multi-line banner
string banner = """
               ────────────────────────────────────────
                 global.json report
               ────────────────────────────────────────
               """;

if (globalJsonPath is not null)
{
    Console.WriteLine(banner);
    Console.WriteLine($"Path: {globalJsonPath}");
    Console.WriteLine(File.ReadAllText(globalJsonPath));
}
else
{
    Console.WriteLine("global.json not found — the latest SDK will be selected.");
}

// --- 3. Build configuration via #if DEBUG + pattern matching ---
#if DEBUG
    const string config = "Debug";
#else
    const string config = "Release";
#endif

string message = config switch
{
    "Debug"   => "Debug build — good for development.",
    "Release" => "Release build — optimized for production.",
    _         => "Unknown configuration."
};
Console.WriteLine(message);

// --- 4. Local function to locate global.json ---
static string? FindGlobalJson(string startDir)
{
    DirectoryInfo? dir = new(startDir);
    while (dir is not null)
    {
        string candidate = Path.Combine(dir.FullName, "global.json");
        if (File.Exists(candidate))
        {
            return candidate;
        }
        dir = dir.Parent;
    }
    return null;
}
```

**Line-by-line walk-through.** The first three `Console.WriteLine` calls mirror the lesson's code example: `Environment.Version` gives the version of the CLR the program actually runs on (do not confuse it with the SDK version!), while `RuntimeInformation` reports the OS and architecture — the same information you just saw in `dotnet --info`, but now obtained from inside the application. `AppContext.BaseDirectory` matters because `dotnet run` launches the binary from `bin/Debug/net8.0/`, and that is where the search for `global.json` must start, not from the current working directory.

Next comes the local function `FindGlobalJson`, which walks upward from `BaseDirectory` through `DirectoryInfo.Parent` until it either finds `global.json` or hits the drive root. This solves the "pitfall" from the lesson: the CLI searches for `global.json` from the current directory upward, and our application must replicate the same logic relative to its own binary. The return type `string?` and the `is not null` check are the modern nullable analysis of C# 8+.

The raw string literal `"""..."""` lets us print a multi-line banner without escaping `\n` and without fiddling with `$"..."`. The indentation inside is aligned by the closing quotes — that is why the three banner lines end up in the output without leading spaces. If we needed curly braces or quotes inside, they would not have to be escaped.

The `#if DEBUG ... #else ... #endif` block is the only place where preprocessor and runtime code are mixed. The `config` constant gets its value at compile time, which is exactly why the branch changes when you run `dotnet run -c Release`. The pattern matching `switch` turns the string into a human-readable message; the `_` branch protects against future configurations (for example, `PublishSingleFile`). All of this is top-level statements: there is no `class Program` and no `Main`, the compiler generates the entry point itself. This is exactly how modern console code on .NET 8 should look, which is emphasized in the lesson's example.

#### Going deeper (bonus)
1. **Experiment with `rollForward`.** Temporarily change the version in `global.json` to a non-existent one (for example, `8.0.999`) and iterate the `rollForward` values from `disable` to `latestMinor`. Record at which values `dotnet build` succeeds and at which it fails. Draw a conclusion about which value is safe for team work.
2. **Diagnose "You must install .NET".** Reproduce the lesson's scenario: find a machine (or container) with only the Runtime, no SDK, and try to run your published `EnvironmentProbe.dll` via `dotnet EnvironmentProbe.dll`. Make it fail and learn to read the error.
3. **Onboarding script.** Write a PowerShell or bash script `probe.sh`/`probe.ps1` that automatically runs `dotnet --version`, `--list-sdks`, `--list-runtimes` and produces a Markdown report of the machine's readiness for development. This is a bridge to CI/CD mentioned in the lesson.
4. **PublishSingleFile.** Publish the application as a single file: `dotnet publish -c Release -r win-x64 --self-contained false -p:PublishSingleFile=true`. Check that `FindGlobalJson` still finds `global.json` if you place it next to the published `.exe`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Установлен .NET SDK 8, `dotnet --version` работает.
- [ ] (RU) Создан `global.json` с `sdk.version` и `sdk.rollForward`.
- [ ] (RU) Проект `EnvironmentProbe` создан через `dotnet new console`, фреймворк `net8.0`.
- [ ] (RU) `Program.cs` использует top-level statements, raw string literal и pattern matching.
- [ ] (RU) Добавлен NuGet-пакет через `dotnet add package`, `dotnet restore` успешен.
- [ ] (RU) Приложение запускается в Debug и Release с разным выводом.
- [ ] (RU) В корне есть `.gitignore` из шаблона `dotnet new gitignore`.
- [ ] (EN) .NET SDK 8 installed, `dotnet --version` works.
- [ ] (EN) `global.json` created with `sdk.version` and `sdk.rollForward`.
- [ ] (EN) `EnvironmentProbe` project created via `dotnet new console`, framework `net8.0`.
- [ ] (EN) `Program.cs` uses top-level statements, raw string literal, and pattern matching.
- [ ] (EN) NuGet package added via `dotnet add package`, `dotnet restore` succeeds.
- [ ] (EN) Application runs in Debug and Release with different output.
- [ ] (EN) Root contains a `.gitignore` from the `dotnet new gitignore` template.

#### Ресурсы / Resources
- Microsoft Learn — https://dotnet.microsoft.com/download — официальная страница загрузки .NET SDK и Runtime / official .NET SDK and Runtime download page.
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/tools/ — справочник по командам dotnet CLI / dotnet CLI command reference.
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/versions/selection — управление версиями SDK через global.json / controlling SDK versions via global.json.
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/tools/dotnet-install-script — скрипт `dotnet-install.{sh,ps1}` для автоматизации установки / `dotnet-install.{sh,ps1}` script for automated installation.
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12 — что нового в C# 12 / what is new in C# 12.
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/tokens/raw-string — raw string literals в C# 11+ / raw string literals in C# 11+.
