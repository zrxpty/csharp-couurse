---
[← Предыдущий: M01-L02](lesson-M01-L02-clr-il-jit.md) | [⬆ К модулю M01](../README.md) | [Следующий: M01-L04 →](lesson-M01-L04-dotnet-new-toplevel.md)
---

### Урок M01-L03: Установка SDK, IDE, dotnet CLI / Installing SDK, IDE, dotnet CLI

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Чтобы писать и запускать программы на C#, вам нужны три вещи: инструмент для создания кода (IDE/редактор), компилятор и среда выполнения (.NET SDK), и командный интерфейс для управления проектами (dotnet CLI). Давайте разберёмся с каждым элементом.

**.NET SDK vs Runtime — в чём разница?** Представьте кухню ресторана. *Runtime* — это обеденный зал и официанты: он позволяет уже готовым блюдам (скомпилированным приложениям) подаваться клиентам (запускаться на компьютере пользователя). Runtime содержит Common Language Runtime (CLR) и базовые библиотеки, но **не умеет** собирать новые программы. *SDK (Software Development Kit)* — это сама кухня с поварами, рецептами и оборудованием: он включает Runtime **плюс** компилятор Roslyn, инструменты сборки, шаблоны проектов и dotnet CLI. Правило простое: на машине разработчика ставим SDK, на сервере/у клиента — только Runtime.

**Установка .NET SDK 8.** Самый надёжный путь — официальный установщик с https://dotnet.microsoft.com/download. Для Windows скачайте `SDK x64` и запустите `.exe`. Для macOS на Apple Silicon берите `Arm64`, на Intel — `x64`. Для Linux используйте пакетный менеджер (Ubuntu: `apt install dotnet-sdk-8.0`) или скрипт `dotnet-install.sh` от Microsoft. На Windows удобна установка через `winget install Microsoft.DotNet.SDK.8`. Перезапустите терминал после установки, чтобы переменная PATH обновилась.

**Проверка установки.** Откройте терминал и выполните три ключевые команды. `dotnet --version` покажет версию CLI (например, `8.0.100`) — если команда не найдена, SDK не установлен или PATH не настроен. `dotnet --info` выдаёт полный отчёт: версии всех установленных SDK и Runtime, операционную систему, архитектуру. `dotnet --list-sdks` перечисляет только SDK — полезно, когда у вас их несколько (например, 8.0 и 9.0 рядом).

**SDK vs Runtime — как CLI их различает.** Команда `dotnet --list-runtimes` отдельно покажет установленные среды выполнения (Microsoft.NETCore.App, Microsoft.AspNetCore.App, Microsoft.WindowsDesktop.App). Это помогает диагностировать ошибку «You must install .NET to run this application», когда приложение собрано под Runtime, которого нет в системе.

**global.json — фиксация версии SDK.** Когда в команде работают несколько разработчиков на разных машинах, важно, чтобы все собирали проект одной и той же версией SDK. Файл `global.json`, помещённый в корень решения, указывает нужную версию. CLI ищет его от текущей директории вверх по дереву и использует указанный SDK. Создать его можно командой `dotnet new globaljson --sdk-version 8.0.100`. Поле `rollForward` определяет поведение, если точной версии нет: `disable`, `patch`, `feature`, `minor`, `latestPatch`, `latestFeature`, `latestMinor`.

**Выбор IDE.** *Visual Studio 2022* (только Windows) — полноценная интегрированная среда с мощным отладчиком, профайлером и дизайнерами; для .NET нужна редакция Community (бесплатна для обучения и небольших команд) или выше с компонентом «.NET desktop development». *Visual Studio Code + расширение C# Dev Kit* — лёгкий кроссплатформенный вариант, дающий IntelliSense, навигацию по коду, рефакторинг и отладку; отлично подходит для macOS/Linux и быстрого редактирования. *JetBrains Rider* — коммерческая кроссплатформенная IDE на базе платформы IntelliJ, любима многими профессионалами за глубокий анализ кода и удобные рефакторинги. Для старта хватит VS Code с C# Dev Kit — это бесплатно и работает везде.

**dotnet CLI — ваш швейцарский нож.** Команды группируются по существительным: `dotnet new` создаёт проекты и файлы из шаблонов (`dotnet new console`, `dotnet new gitignore`), `dotnet build` компилирует, `dotnet run` собирает и запускает, `dotnet test` прогоняет тесты, `dotnet add` добавляет пакеты и ссылки (`dotnet add package Newtonsoft.Json`), `dotnet restore` восстанавливает зависимости из NuGet. CLI един для всех ОС и всех IDE — знание команд пригодится в CI/CD и при работе с контейнерами.

#### Theory (EN)

To write and run C# programs you need three things: a tool to author code (an IDE/editor), a compiler and execution environment (the .NET SDK), and a command-line interface to manage projects (the dotnet CLI). Let us look at each one.

**.NET SDK vs Runtime — what is the difference?** Think of a restaurant. The *Runtime* is the dining hall and the waiters: it lets finished dishes (compiled applications) be served to customers (run on the user's machine). The Runtime contains the Common Language Runtime (CLR) and base libraries, but it **cannot** build new programs. The *SDK (Software Development Kit)* is the kitchen itself, with chefs, recipes, and equipment: it includes the Runtime **plus** the Roslyn compiler, build tools, project templates, and the dotnet CLI. The rule is simple: install the SDK on the developer's machine, only the Runtime on the server or end-user machine.

**Installing .NET SDK 8.** The most reliable path is the official installer from https://dotnet.microsoft.com/download. On Windows, download `SDK x64` and run the `.exe`. On macOS choose `Arm64` for Apple Silicon, `x64` for Intel. On Linux use a package manager (Ubuntu: `apt install dotnet-sdk-8.0`) or the `dotnet-install.sh` script from Microsoft. On Windows, `winget install Microsoft.DotNet.SDK.8` is convenient. Restart your terminal after installation so the PATH variable is refreshed.

**Verifying the installation.** Open a terminal and run three key commands. `dotnet --version` prints the CLI version (for example `8.0.100`); if the command is not found, the SDK is missing or PATH is not set. `dotnet --info` gives a full report: versions of every installed SDK and Runtime, the operating system, and the architecture. `dotnet --list-sdks` lists only the SDKs — useful when several are installed side by side (say 8.0 and 9.0).

**SDK vs Runtime — how the CLI tells them apart.** `dotnet --list-runtimes` separately lists installed runtimes (Microsoft.NETCore.App, Microsoft.AspNetCore.App, Microsoft.WindowsDesktop.App). This helps diagnose the "You must install .NET to run this application" error when an app was built for a Runtime that is not present.

**global.json — pinning the SDK version.** When a team works across several machines, it is important that everyone builds the project with the same SDK version. A `global.json` file placed at the root of the solution specifies the required version. The CLI searches for it from the current directory up the tree and uses the indicated SDK. Create it with `dotnet new globaljson --sdk-version 8.0.100`. The `rollForward` field controls behavior when the exact version is missing: `disable`, `patch`, `feature`, `minor`, `latestPatch`, `latestFeature`, `latestMinor`.

**Choosing an IDE.** *Visual Studio 2022* (Windows only) is a full integrated environment with a powerful debugger, profiler, and designers; for .NET you need the Community edition (free for learning and small teams) or higher with the ".NET desktop development" workload. *Visual Studio Code + the C# Dev Kit extension* is a lightweight cross-platform option that provides IntelliSense, code navigation, refactoring, and debugging; it is ideal for macOS/Linux and quick edits. *JetBrains Rider* is a commercial cross-platform IDE built on the IntelliJ platform, favored by many professionals for deep code analysis and convenient refactorings. To start, VS Code with C# Dev Kit is enough — it is free and works everywhere.

**dotnet CLI — your Swiss army knife.** Commands are grouped by nouns: `dotnet new` creates projects and files from templates (`dotnet new console`, `dotnet new gitignore`), `dotnet build` compiles, `dotnet run` builds and runs, `dotnet test` runs tests, `dotnet add` adds packages and references (`dotnet add package Newtonsoft.Json`), `dotnet restore` restores NuGet dependencies. The CLI is identical across all operating systems and IDEs — knowing the commands pays off in CI/CD and when working with containers.

#### Пример кода / Code Example
```csharp
// M01-L03: Проверка установки SDK и окружения
// Verifying SDK installation and environment
// C# 12 / .NET 8, top-level statements

using System.Runtime.InteropServices;

// Выводим версию .NET Runtime, на котором работает программа
// Print the .NET Runtime version the program runs on
Console.WriteLine($"Environment: .NET {Environment.Version}");
Console.WriteLine($"OS:          {RuntimeInformation.OSDescription}");
Console.WriteLine($"Architecture:{RuntimeInformation.OSArchitecture}");

// Читаем global.json (если он рядом с исполняемым файлом)
// Read global.json if it sits next to the executable
string globalJsonPath = Path.Combine(AppContext.BaseDirectory, "global.json");
if (File.Exists(globalJsonPath))
{
    // raw string literal (C# 11+) для многострочного шаблона
    // raw string literal (C# 11+) for a multi-line template
    string banner = """
                    ── global.json найден / found ──
                    """;
    Console.WriteLine(banner);
    Console.WriteLine(File.ReadAllText(globalJsonPath));
}
else
{
    Console.WriteLine("global.json не найден / not found — будет выбран последний SDK.");
}

// Pattern matching: различаем тип сборки по директиве
// Pattern matching: distinguish build configuration via directive
#if DEBUG
    const string config = "Debug";
#else
    const string config = "Release";
#endif

string message = config switch
{
    "Debug"   => "Запущена отладочная сборка — удобна для разработки.",
    "Release" => "Запущена релизная сборка — оптимизирована для продакшена.",
    _         => "Неизвестная конфигурация сборки."
};
Console.WriteLine(message);
```

#### Best Practices
- Держите на машине одну LTS-версию SDK (например, 8.0) как основную; дополнительные версии ставьте только при необходимости — это упрощает поддержку.
- Keep one LTS SDK release (e.g. 8.0) as the primary on your machine; add extra versions only when needed — it simplifies maintenance.
- Фиксируйте версию SDK в `global.json` в корне решения, чтобы все члены команды и CI собирали код одинаково.
- Pin the SDK version in a `global.json` at the solution root so every teammate and CI builds the code identically.
- Устанавливайте SDK через официальный установщик или пакетный менеджер ОС, а не копированием папок — так корректно обновится PATH и сертификаты.
- Install the SDK via the official installer or the OS package manager, not by copying folders — this correctly updates PATH and certificates.
- Используйте `dotnet --info` для диагностики окружения перед тем, как сообщать об ошибках сборки.
- Use `dotnet --info` to diagnose the environment before reporting build issues.

#### Частые ошибки / Common Mistakes
- Команда `dotnet` не найдена после установки → перезапустите терминал/IDE, чтобы обновился PATH; на Windows проверьте переменную среды в «Свойствах системы».
- `dotnet` command not found after install → restart the terminal/IDE so PATH refreshes; on Windows check the environment variable in System Properties.
- Сборка падает с ошибкой о неподходящей версии SDK → проверьте `dotnet --list-sdks` и создайте `global.json` с нужной версией или настройте `rollForward`.
- Build fails with a wrong SDK version error → check `dotnet --list-sdks` and create a `global.json` with the required version or set `rollForward`.
- На macOS поставили `x64` SDK на Apple Silicon → удалите и установите `Arm64`-сборку для лучшей производительности.
- Installed the `x64` SDK on Apple Silicon macOS → uninstall and install the `Arm64` build for better performance.
- Путают SDK и Runtime → запомните: SDK = разработка (компиляция + запуск), Runtime = только запуск готового приложения.
- Confusing SDK and Runtime → remember: SDK = development (compile + run), Runtime = only running a finished app.
- Ставят Runtime вместо SDK и удивляются, что `dotnet build` не работает → для разработки всегда устанавливайте именно SDK.
- Installing Runtime instead of SDK and wondering why `dotnet build` fails → for development always install the SDK itself.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я установил .NET SDK 8 и `dotnet --version` выводит корректную версию.
- [ ] I installed .NET SDK 8 and `dotnet --version` prints the correct version.
- [ ] Я умею отличать SDK от Runtime и знаю, что показывает `dotnet --info`.
- [ ] I can tell SDK from Runtime and know what `dotnet --info` reports.
- [ ] Я выбрал и установил IDE (VS / VS Code + C# Dev Kit / Rider).
- [ ] I chose and installed an IDE (VS / VS Code + C# Dev Kit / Rider).
- [ ] Я создал проект `dotnet new console` и запустил его командой `dotnet run`.
- [ ] I created a `dotnet new console` project and ran it with `dotnet run`.
- [ ] Я могу объяснить назначение `global.json` и поля `rollForward`.
- [ ] I can explain the purpose of `global.json` and the `rollForward` field.
- [ ] Я знаю команды `dotnet new`, `build`, `run`, `test`, `add package`.
- [ ] I know the commands `dotnet new`, `build`, `run`, `test`, `add package`.

#### Ресурсы / Resources
- Microsoft Learn — https://dotnet.microsoft.com/download — официальная страница загрузки .NET SDK и Runtime / official .NET SDK and Runtime download page.
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/tools/ — справочник по командам dotnet CLI / dotnet CLI command reference.
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/versions/selection — управление версиями SDK через global.json / controlling SDK versions via global.json.
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/tools/dotnet-install-script — скрипт `dotnet-install.{sh,ps1}` для автоматизации установки / `dotnet-install.{sh,ps1}` script for automated installation.
- JetBrains — https://www.jetbrains.com/rider/ — официальная страница Rider / official Rider page.
- Microsoft Learn — https://learn.microsoft.com/visualstudio/code/ — C# Dev Kit для VS Code / C# Dev Kit for VS Code.

---
[← Предыдущий: M01-L02](lesson-M01-L02-clr-il-jit.md) | [⬆ К модулю M01](../README.md) | [Следующий: M01-L04 →](lesson-M01-L04-dotnet-new-toplevel.md)
---
