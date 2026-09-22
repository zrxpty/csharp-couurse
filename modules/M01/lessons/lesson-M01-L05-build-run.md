---
[← Предыдущий: M01-L04](lesson-M01-L04-dotnet-new-toplevel.md) | [⬆ К модулю M01](../README.md) | [Следующий: M01-L06 →](lesson-M01-L06-hello-debug-nuget.md)
---

### Урок M01-L05: Компиляция, запуск, dotnet run/build / Compilation, running, dotnet run/build

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Когда вы пишете код на C#, вы работаете с текстовыми файлами `.cs`. Чтобы программа запустилась, этот текст нужно превратить во что-то, что понимает компьютер. Этот процесс называется **компиляцией**. В мире .NET он состоит из нескольких шагов, и важно понимать каждый, чтобы не путаться в ошибках и артефактах сборки.

**Жизненный цикл простой сборки.** Команда `dotnet build` берёт ваш проект (`.csproj`), считывает его настройки, передаёт исходники компилятору C# (`csc`), который превращает `.cs` в промежуточный язык IL (Intermediate Language) и упаковывает IL в сборку `.dll` (или `.exe`). IL — это не машинный код, а «полуготовый» код, который JIT-компилятор .NET runtime превратит в машинный код прямо во время выполнения. Аналогия: `.cs` — это рецепт повара на русском, IL — это универсальный рецепт на эсперанто, который любой повар поймёт, а машинный код — это уже готовое блюдо на тарелке.

**dotnet restore.** Прежде чем собрать проект, нужно восстановить зависимости — пакеты NuGet и сам SDK-функционал. Команда `dotnet restore` читает файл `.csproj`, список пакетов, скачивает их в глобальный кэш (`~/.nuget/packages`) и записывает файл `project.assets.json` в каталог `obj/`. Начиная с .NET Core, `restore` запускается неявно при `build`, `run` и `publish`, поэтому вручную его зовут редко — только когда нужно явно подтянуть пакеты в CI или после изменения `*.csproj`.

**dotnet build** компилирует проект и кладёт результат в `bin/`. Для проекта `MyApp.csproj` с целевой платформой `net8.0` путь выглядит так: `bin/Debug/net8.0/MyApp.dll`. Каталог `Debug` соответствует конфигурации. `dotnet run` — это удобная обёртка: она сама вызывает `build`, а затем запускает результат. Используйте `run` для быстрых итераций разработки и `build`, когда нужно проверить только успешность компиляции или посмотреть на артефакты.

**Конфигурации Debug и Release.** `Debug` — режим разработки: отключены оптимизации, включены отладочные символы, проверка переполнения и assertions. `Release` — режим для конечного пользователя: JIT применяет оптимизации, код работает быстрее, но отлаживать его тяжелее. Выбор делается флагом `-c Release` или переменной `DOTNET_CONFIGURATION`. По умолчанию берётся `Debug`. Хорошая практика — собирать и тестировать в обоих режимах перед релизом, потому что оптимизации иногда меняют поведение (например, порядок вычислений).

**Артефакты в `bin/Debug/net8.0/`.** После сборки там появится несколько файлов, и стоит знать назначение каждого:
- `MyApp.dll` — основная сборка с вашим кодом в IL.
- `MyApp.pdb` — Program Database, отладочные символы; без него стек вызовов будет без номеров строк.
- `MyApp.runtimeconfig.json` — описывает, какой runtime и какую версию .NET нужен для запуска, а также настройки garbage collector и сборки.
- `MyApp.deps.json` — список всех зависимостей, прямых и транзитивных; используется загрузчиком для разрешения сборок.
- `MyApp.dll.config` (если есть) — конфигурация `app.config` для старых сценариев.

**dotnet publish** готовит приложение к развёртыванию. В отличие от `build`, `publish` копирует все нужные файлы в один каталог, который можно перенести на сервер. Без флагов это будет framework-dependent deployment (FDD): на целевой машине должен стоять .NET runtime. С флагами `--self-contained` и `-r <RID>` (например, `win-x64`, `linux-x64`) в вывод попадёт сам runtime — приложение станет автономным, запустится на машине без .NET, но размер вырастет до ~60–80 МБ. Флаг `-p:PublishSingleFile=true` упаковывает всё в один `.exe`/`dll`, что удобно для распространения.

**Self-contained и RID.** Runtime Identifier (RID) описывает целевую ОС и архитектуру: `win-x64`, `linux-musl-x64`, `osx-arm64`. Self-contained publish привязан к RID, потому что runtime платформо-зависим. Framework-dependent приложения часто не требуют RID и запустятся на любой платформе с подходящим runtime. Выбор зависит от инфраструктуры: внутри контейнера с фиксированным .NET выгоднее FDD, для утилиты, раздаваемой пользователям, — self-contained.

#### Theory (EN)

When you write C# code, you work with `.cs` text files. To run a program, that text must be transformed into something the machine can execute. That transformation is **compilation**, and in .NET it has several stages you should understand to avoid confusion when build artifacts pile up or errors appear.

**The build lifecycle.** The `dotnet build` command takes your project (`.csproj`), reads its settings, hands source files to the C# compiler (`csc`), which turns `.cs` into Intermediate Language (IL) and packs that IL into a `.dll` (or `.exe`). IL is not machine code — it is a "half-finished" representation that the .NET runtime's JIT compiler turns into native code during execution. Analogy: the `.cs` file is a recipe in Russian; IL is a universal recipe in Esperanto that any chef can read; native machine code is the actual dish on the plate.

**dotnet restore.** Before building, dependencies — NuGet packages and SDK machinery — must be restored. `dotnet restore` reads `.csproj`, downloads packages into the global cache (`~/.nuget/packages`), and writes `project.assets.json` into `obj/`. Since .NET Core, restore runs implicitly during `build`, `run`, and `publish`, so you rarely invoke it manually — only when you must explicitly pull packages in CI or after editing `*.csproj`.

**dotnet build** compiles the project and places output in `bin/`. For a `MyApp.csproj` targeting `net8.0`, the path looks like `bin/Debug/net8.0/MyApp.dll`. The `Debug` segment corresponds to the configuration. `dotnet run` is a convenience wrapper: it calls `build` and then executes the result. Use `run` for fast development iterations and `build` when you only need to verify compilation or inspect artifacts.

**Debug vs Release.** `Debug` is the development mode: optimizations off, debug symbols on, overflow checks and assertions active. `Release` is the shipping mode: the JIT applies optimizations, the program runs faster, but debugging becomes harder. Choose with `-c Release` or the `DOTNET_CONFIGURATION` variable; the default is `Debug`. A sound practice is to build and test in both configurations before release, because optimizations can occasionally change observable behavior (for example, evaluation order or inlining).

**Artifacts in `bin/Debug/net8.0/`.** After a build several files appear, and each deserves a name:
- `MyApp.dll` — the main assembly holding your IL code.
- `MyApp.pdb` — Program Database, debug symbols; without it stack traces have no line numbers.
- `MyApp.runtimeconfig.json` — declares which runtime and .NET version the app needs, plus GC and rollout settings.
- `MyApp.deps.json` — the full dependency graph (direct and transitive); the loader uses it to resolve assemblies.
- `MyApp.dll.config` (if present) — legacy `app.config` settings.

**dotnet publish** prepares an application for deployment. Unlike `build`, `publish` copies every file needed to run into a single output folder you can ship to a server. Without extra flags this is a framework-dependent deployment (FDD): the target machine must have the .NET runtime. Add `--self-contained` together with `-r <RID>` (for example `win-x64`, `linux-x64`) and the runtime itself is bundled — the app becomes self-contained and runs on a machine without .NET installed, at the cost of ~60–80 MB. The `-p:PublishSingleFile=true` flag packs everything into a single `.exe`/`dll`, convenient for distribution.

**Self-contained and RID.** The Runtime Identifier (RID) describes the target OS and architecture: `win-x64`, `linux-musl-x64`, `osx-arm64`. Self-contained publishing requires a RID because the runtime is platform-specific. Framework-dependent apps often need no RID and run on any platform with a compatible runtime. The choice depends on infrastructure: inside a container with a fixed .NET, FDD is cheaper; for a utility handed to users, self-contained is safer.

#### Пример кода / Code Example

```csharp
// M01-L05: Демонстрация артефактов сборки и запуска
// Demonstrates build artifacts and runtime info via reflection.
// Скомпилируйте / Build:
//   dotnet build -c Release
// Запустите / Run:
//   dotnet run -c Release
// Опубликуйте / Publish (self-contained single file):
//   dotnet publish -c Release -r win-x64 --self-contained -p:PublishSingleFile=true

using System.Reflection;
using System.Runtime.InteropServices;

// Top-level statements — точка входа без boilerplate Main.
// Top-level statements: no Main boilerplate needed.
var assembly = Assembly.GetExecutingAssembly();
var name = assembly.GetName();

Console.WriteLine($"""
    === Информация о сборке / Build info ===
    Имя / Name:                {name.Name}
    Версия / Version:          {name.Version}
    Конфигурация / Config:     {GetConfiguration()}
    Каталог / Location:        {assembly.Location}
    .NET Runtime:              {RuntimeInformation.FrameworkDescription}
    ОС / OS:                   {RuntimeInformation.OSDescription}
    Архитектура / Arch:        {RuntimeInformation.OSArchitecture}
    """);

Console.WriteLine();
Console.WriteLine("=== Зависимости / Dependencies (deps.json на стадии загрузки) ===");

// Перебираем загруженные сборки, чтобы показать транзитивные зависимости runtime.
foreach (var referenced in assembly.GetReferencedAssemblies()
                                   .OrderBy(a => a.Name, StringComparer.Ordinal))
{
    Console.WriteLine($"  - {referenced.Name} {referenced.Version}");
}

// Лёгкая «бизнес-логика», чтобы продемонстрировать работу Release-оптимизаций.
int Factorial(int n) => n switch
{
    < 0   => throw new ArgumentOutOfRangeException(nameof(n)),
    0 or 1 => 1,
    _     => n * Factorial(n - 1)
};

Console.WriteLine();
Console.WriteLine($"5! = {Factorial(5)}  // 120");

// Определяем конфигурацию по наличию отладочных атрибутов сборки.
// Detect configuration via assembly-level debug attribute.
static string GetConfiguration()
{
#if DEBUG
    return "Debug (compiled)";
#else
    return "Release (compiled)";
#endif
}
```

#### Best Practices
- Перед коммитом запускайте `dotnet build` в обеих конфигурациях (`Debug` и `Release`) — оптимизации иногда вскрывают скрытые баги.
- Run `dotnet build` in both `Debug` and `Release` before committing — optimizations can reveal hidden bugs.
- Каталоги `bin/` и `obj/` добавляйте в `.gitignore`: они пересобираются и засоряют репозиторий.
- Add `bin/` and `obj/` to `.gitignore`: they are reproducible and only add noise to the repo.
- Для распространения конечным пользователям используйте `--self-contained -p:PublishSingleFile=true`, чтобы не зависеть от установочного .NET на машине клиента.
- For end-user distribution prefer `--self-contained -p:PublishSingleFile=true` so you do not depend on a preinstalled .NET on the client machine.
- Закрепляйте версию SDK и целевую платформу в `global.json` и `<TargetFramework>` для воспроизводимых сборок.
- Pin the SDK version in `global.json` and the target framework in `<TargetFramework>` for reproducible builds.

#### Частые ошибки / Common Mistakes
- Запуск `dotnet run MyProgram.dll` вместо `dotnet MyProgram.dll` или `dotnet run` (из каталога проекта) → `dotnet run` ожидает проект, а не собранный `.dll`; запускайте из папки проекта без аргумента-файла. (RU)
- Mixing `dotnet run MyProgram.dll` (wrong) instead of `dotnet MyProgram.dll` or plain `dotnet run` from the project folder → `dotnet run` expects a project, not a built `.dll`; run it from the project folder with no file argument. (EN)
- Редактирование файлов в `bin/Debug/net8.0/` в надежде, что изменения сохранятся → эти файлы перезаписываются при каждой сборке; меняйте исходники в `.cs`. (RU)
- Editing files in `bin/Debug/net8.0/` expecting changes to persist → these files are overwritten on every build; edit the `.cs` sources instead. (EN)
- Публикация self-contained без указания `-r <RID>` → `--self-contained` требует Runtime Identifier; добавьте `-r win-x64` или другой RID. (RU)
- Publishing `--self-contained` without a `-r <RID>` → self-contained publishing requires a Runtime Identifier; add `-r win-x64` or another RID. (EN)
- Запуск Release-сборки с отладчиком и удивление отсутствием номеров строк → включайте символы через `<DebugType>embedded</DebugType>` или используйте Debug для отладки. (RU)
- Debugging a Release build and being surprised by missing line numbers → enable symbols with `<DebugType>embedded</DebugType>` or debug in Debug mode. (EN)
- Путаница `MyApp.dll` и `MyApp.exe` → на Windows `MyApp.exe` — лишь загрузчик-аппелятор; реальный код живёт в `MyApp.dll`; на других ОС `.exe` вообще нет. (RU)
- Confusing `MyApp.dll` and `MyApp.exe` → on Windows `MyApp.exe` is just an apphost stub; the real code lives in `MyApp.dll`; on other OSes there is no `.exe` at all. (EN)

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я могу объяснить, чем отличаются `dotnet build`, `run` и `publish`.
- [ ] I can explain the difference between `dotnet build`, `run`, and `publish`.
- [ ] Я знаю назначение файлов `.dll`, `.pdb`, `.runtimeconfig.json`, `.deps.json` в каталоге `bin/Debug/net8.0/`.
- [ ] I know the purpose of `.dll`, `.pdb`, `.runtimeconfig.json`, `.deps.json` in `bin/Debug/net8.0/`.
- [ ] Я понимаю разницу между Debug и Release и когда использовать каждый.
- [ ] I understand the difference between Debug and Release and when to use each.
- [ ] Я могу опубликовать self-contained приложение с правильным RID.
- [ ] I can publish a self-contained application with the correct RID.
- [ ] Я знаю, что `dotnet restore` обычно запускается неявно.
- [ ] I know that `dotnet restore` usually runs implicitly.
- [ ] Я не коммичу `bin/` и `obj/` в репозиторий.
- [ ] I do not commit `bin/` and `obj/` into the repository.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/tools/dotnet-build — `dotnet build` / команда сборки проекта.
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/tools/dotnet-run — `dotnet run` / запуск исходного кода без явной сборки.
- Microsoft Learn — dotnet publish / самодостаточные развёртывания (self-contained deployments).
- Microsoft Learn — .NET runtime configuration files (`.runtimeconfig.json`, `.deps.json`).

---
[← Предыдущий: M01-L04](lesson-M01-L04-dotnet-new-toplevel.md) | [⬆ К модулю M01](../README.md) | [Следующий: M01-L06 →](lesson-M01-L06-hello-debug-nuget.md)
---
