---
[← К уроку M01-L05](lesson-M01-L05-build-run.md) | [⬆ К модулю M01](../README.md) | [Следующее ДЗ →](homework-M01-L06-hello-debug-nuget.md)
---

### Домашнее задание M01-L05: Компиляция, запуск, dotnet run/build / Homework M01-L05: Compilation, running, dotnet run/build

**Урок / Lesson:** M01-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) На практике пройти весь жизненный цикл .NET-сборки: от исходников `.cs` до IL-сборки в `bin/`, затем до запуска через `dotnet run` и до публикации через `dotnet publish` в режимах framework-dependent и self-contained. Научиться различать артефакты `bin/Debug/net8.0/`, понимать разницу между `Debug` и `Release`, корректно выбирать RID и не попадать в типичные ловушки вроде `dotnet run MyApp.dll` или редактирования файлов в `bin/`. (EN) Walk the full .NET build lifecycle hands-on: from `.cs` sources to IL assemblies in `bin/`, then to running via `dotnet run`, and finally to publishing through `dotnet publish` in framework-dependent and self-contained modes. Learn to distinguish the artifacts in `bin/Debug/net8.0/`, understand the difference between `Debug` and `Release`, pick the right RID, and avoid classic traps such as `dotnet run MyApp.dll` or editing files inside `bin/`.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет ключевые темы урока M01-L05: жизненный цикл компиляции `.cs → IL → .dll`, неявный `dotnet restore`, команды `build`/`run`/`publish`, конфигурации `Debug`/`Release`, назначение артефактов `.dll/.pdb/.runtimeconfig.json/.deps.json`, а также best practices из урока (сборка в двух конфигурациях, `.gitignore` для `bin/obj`, закрепление SDK в `global.json`). Решение опирается на пример кода урока (top-level statements, raw strings, pattern matching) и осознанно обходит каждую частую ошибку из раздела «Частые ошибки».
(EN) The homework directly reinforces the core topics of lesson M01-L05: the `.cs → IL → .dll` compilation lifecycle, implicit `dotnet restore`, the `build`/`run`/`publish` commands, the `Debug`/`Release` configurations, the purpose of the `.dll/.pdb/.runtimeconfig.json/.deps.json` artifacts, and the lesson's best practices (building in both configurations, `.gitignore` for `bin/obj`, pinning the SDK via `global.json`). The solution reuses the lesson's code patterns (top-level statements, raw strings, pattern matching) and deliberately avoids every pitfall listed in the "Common Mistakes" section.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы только что прошли урок о том, как текст на C# превращается в работающую программу. На теоретическом уровне всё ясно: `.cs` компилируется в IL, IL упаковывается в `.dll`, а JIT превращает IL в машинный код во время запуска. Но между «понимаю теорию» и «умею собрать и опубликовать реальный проект» — пропасть, которую можно перейти только руками. Именно поэтому данное домашнее задание построено как мини-проект: вы создадите консольное приложение, пройдёте весь путь от `dotnet new` до `dotnet publish --self-contained`, и на каждом шаге будете фиксировать, какие файлы появились, какой у них размер и почему.

Мотивация двойная. Во-первых, вы должны перестать бояться каталога `bin/`: научиться читать его содержимое и понимать назначение каждого файла — это базовый навык любого .NET-разработчика, без которого невозможно отлаживать проблемы с зависимостями, конфигурациями и развёртыванием. Во-вторых, вы должны на себе почувствовать разницу между `Debug` и `Release`, между FDD и self-contained, между `dotnet run` и прямым запуском `.dll`. Эти отличия часто объясняют на словах, но только собственный замер размеров и времён сборки делает знание интуитивным.

Дополнительно задание тренирует культуру воспроизводимых сборок: вы закрепите версию SDK через `global.json`, добавите `bin/` и `obj/` в `.gitignore` и убедитесь, что проект собирается на чистой машине после `git clone`. Это именно те best practices, которые урок выделяет как обязательные для командной работы. В результате у вас появится маленький, но полностью «боевой» репозиторий, демонстрирующий зрелое обращение с инструментарием .NET 8.

#### Что нужно сделать (пошагово)

1. **Создайте рабочую папку и проект.** В отдельном каталоге выполните `dotnet new console -n BuildInsights -o BuildInsights --framework net8.0`. Убедитесь, что создались `BuildInsights.csproj`, `Program.cs` и `obj/`. Откройте `Program.cs` — там будет стандартный `Console.WriteLine("Hello, World!");`. Замените его целиком на эталонный код из раздела «Эталонное решение» ниже (он использует top-level statements, raw string literal, pattern matching и `#if DEBUG`).

2. **Закрепите SDK через `global.json`.** В корне (на уровень выше `BuildInsights/`, в папке задания) выполните `dotnet new globaljson --sdk-version 8.0.x` (подставьте актуальный установленный SDK, узнать который можно через `dotnet --version`). Проверьте, что `dotnet --info` показывает именно этот SDK как активный для каталога. Это обеспечит воспроизводимость сборок.

3. **Соберите проект в Debug.** Из папки `BuildInsights/` запустите `dotnet build -c Debug`. Зафиксируйте в отчётной строке: время сборки (выводится в конце), путь к артефактам (должен быть `bin/Debug/net8.0/`) и список файлов в этом каталоге. Убедитесь, что присутствуют `BuildInsights.dll`, `BuildInsights.pdb`, `BuildInsights.runtimeconfig.json`, `BuildInsights.deps.json`. На Windows также появится `BuildInsights.exe` — это apphost-заглушка, а не «настоящий» исполняемый файл.

4. **Запустите двумя способами.** Сначала — `dotnet run -c Debug` из папки проекта: обратите внимание, что `run` сам вызывает `build` и лишь потом стартует сборку. Затем — прямой запуск IL-сборки: `dotnet bin/Debug/net8.0/BuildInsights.dll`. Вывод должен быть одинаковым. Запишите, какой из способов быстрее на повторном запуске и почему (подсказка: `run` каждый раз проверяет актуальность сборки).

5. **Соберите в Release и сравните.** Выполните `dotnet build -c Release`. Сравните размеры `BuildInsights.dll` в `bin/Debug/net8.0/` и `bin/Release/net8.0/` — Release-сборка обычно чуть меньше за счёт оптимизаций и отсутствия части отладочной информации. Запустите `dotnet run -c Release` и сверьте поле `Конфигурация / Config` в выводе программы: оно должно показать `Release (compiled)` благодаря `#if DEBUG`.

6. **Опубликуйте как framework-dependent.** Выполните `dotnet publish -c Release -o ./publish-fdd`. Загляните в `publish-fdd/`: там должны быть те же типы файлов, что и в `bin/Release/net8.0/`, но без промежуточных артефактов. Обратите внимание, что никакого встроенного runtime нет — приложение рассчитывает на установленный .NET 8 на целевой машине. Запишите суммарный размер каталога.

7. **Опубликуйте как self-contained single-file.** Определите свой RID: `win-x64`, `linux-x64` или `osx-arm64` (узнайте через `dotnet --info`, поле `OSArchitecture`/`RuntimeIdentifier`). Выполните `dotnet publish -c Release -r <RID> --self-contained -p:PublishSingleFile=true -o ./publish-sc`. Сравните размер получившегося одиночного файла с размером FDD-каталога — он должен быть заметно больше (десятки мегабайт), потому что внутрь упакован runtime. Запустите этот файл напрямую (двойной клик или `./publish-sc/BuildInsights` в терминале) и убедитесь, что программа работает без установленного .NET.

8. **Настройте `.gitignore`.** В корне репозитория создайте `.gitignore` с минимумом: `bin/`, `obj/`, `publish-fdd/`, `publish-sc/`. Убедитесь через `git status` (если инициализирован репозиторий), что ни один из этих каталогов не попадает в индекс. Если репозитория нет — просто покажите, что команда `git check-ignore bin/BuildInsights/obj` возвращает совпадение.

9. **Соберите финальный отчёт.** В файле `report.md` (в корне задания) соберите: таблицу размеров артефактов по шагам 3, 5, 6, 7; вывод `dotnet --info`; список файлов из `bin/Debug/net8.0/` с пояснением назначения каждого (опираясь на урок). Это и есть artefact, который вы сдаёте вместе с кодом.

#### Требования к решению

- Решение должно работать на .NET 8 ( TargetFramework `net8.0`) и C# 12: используйте top-level statements, raw string literals (`"""..."""`) для многострочного вывода, pattern matching с `or` в switch-выражениях, как в примере урока.
- Код `Program.cs` должен выводить информацию о сборке через рефлексию (`Assembly.GetExecutingAssembly()`), конфигурацию через `#if DEBUG`, информацию о runtime и ОС через `RuntimeInformation`, а также вычислять факториал через рекурсивную локальную функцию со switch-выражением и guard `or` — это напрямую перекликается с примером кода урока.
- Команды нужно выполнять ровно те, что описаны: `dotnet build`, `dotnet run`, `dotnet publish` с указанными флагами. Запуск `dotnet run MyApp.dll` считается ошибкой и должен быть осознанно избегнут — в отчёте явно укажите, почему эта команда неверна.
- Все артефакты `bin/`, `obj/`, `publish-*` должны быть проигнорированы git. В репозиторий сдаются только `BuildInsights.csproj`, `Program.cs`, `global.json`, `.gitignore` и `report.md`.
- В отчёте `report.md` должна быть таблица сравнения Debug vs Release (размер `.dll`) и FDD vs self-contained (размер развёртывания) — без чисел задание не принимается.
- Конфигурация должна определяться именно `#if DEBUG`, а не чтением переменных окружения, потому что это compile-time признак, который честно отражает, в каком режиме собран IL.

#### Тонкости и подводные камни

- **`dotnet run` ожидает проект, а не `.dll`.** Самая частая ошибка — `dotnet run bin/Debug/net8.0/BuildInsights.dll`. Эта команда пытается интерпретировать аргумент как файл проекта для сборки и падает. Правильно: либо `dotnet run` из папки проекта (без аргумента), либо `dotnet bin/Debug/net8.0/BuildInsights.dll` для прямого запуска уже собранной сборки.
- **Не редактируйте файлы в `bin/`.** Любые правки `.dll` или `.runtimeconfig.json` исчезнут при следующем `build`. Меняйте только `.cs` и `.csproj`. Это особенно соблазнительно для `.runtimeconfig.json`, но он полностью перегенерируется из `.csproj` и свойств SDK.
- **`--self-contained` требует RID.** Если выполнить `dotnet publish --self-contained` без `-r win-x64` (или другого RID), публикация завершится ошибкой или молча сделает framework-dependent. Всегда комбинируйте `--self-contained` с `-r <RID>`.
- **`MyApp.exe` на Windows — не основной код.** Это apphost-заглушка, которая находит и запускает `MyApp.dll` через hostpolicy. Реальная логика — в `.dll`. На Linux/macOS `.exe` вообще нет, есть только `MyApp.dll` (или один файл без расширения при `PublishSingleFile`).
- **Release может «съесть» номера строк в стеке.** Если в Release-сборке падает исключение, стек может быть без номеров строк, потому что оптимизации инлайнят и переупорядочивают код. Для отладки используйте Debug; для прод-сборок включайте `<DebugType>embedded</DebugType>`, чтобы pdb «жил» внутри dll.
- **`dotnet restore` почти всегда неявный.** Вызывать его вручную нужно редко: только в CI-пайплайнах с явным шагом restore или после правки `.csproj` в проектах со сложными зависимостями. В этом ДЗ он не требуется отдельно — `build` сам его вызовет.
- **Размеры публикуемых артефактов обманчивы.** FDD-каталог кажется маленьким, но требует .NET на машине. Self-contained single-file большой, но самодостаточен. Выбор — это компромисс, который вы должны проговорить в отчёте, а не просто перечислить числа.

#### Критерии приёмки

- [ ] Создан проект `BuildInsights` через `dotnet new console --framework net8.0`.
- [ ] `Program.cs` заменён на эталонный код (top-level statements, raw string, pattern matching, `#if DEBUG`).
- [ ] В корне задания есть `global.json` с закреплённой версией SDK 8.x.
- [ ] Выполнен `dotnet build -c Debug`, в отчёте зафиксированы путь и список файлов `bin/Debug/net8.0/`.
- [ ] Программа запущена и через `dotnet run -c Debug`, и через `dotnet bin/Debug/net8.0/BuildInsights.dll` — выводы совпадают.
- [ ] Выполнен `dotnet build -c Release`, размер `.dll` сравнён с Debug в таблице.
- [ ] `dotnet run -c Release` выводит `Release (compiled)` в поле Config.
- [ ] Выполнен `dotnet publish -c Release -o ./publish-fdd`, размер каталога зафиксирован.
- [ ] Выполнен `dotnet publish -c Release -r <RID> --self-contained -p:PublishSingleFile=true -o ./publish-sc`, размер одного файла зафиксирован.
- [ ] Self-contained файл запущен напрямую и работает без отдельного вызова `dotnet`.
- [ ] В `.gitignore` добавлены `bin/`, `obj/`, `publish-fdd/`, `publish-sc/`.
- [ ] В отчёте `report.md` есть таблицы сравнения Debug/Release и FDD/self-contained с числами.
- [ ] В отчёте перечислены и объяснены файлы `bin/Debug/net8.0/` (`.dll`, `.pdb`, `.runtimeconfig.json`, `.deps.json`, опционально `.exe`).
- [ ] В отчёте явно указано, почему `dotnet run MyApp.dll` — ошибка.
- [ ] В репозиторий не попадают `bin/` и `obj/`.

#### Подсказки (без прямого ответа)

- Если `dotnet --version` показывает версию, отличную от ожидаемой, проверьте `global.json` — он может «пинать» SDK, которого нет, и тогда dotnet подскажет, какой установить.
- Размеры артефактов удобно собирать одной командой: на PowerShell `Get-ChildItem -Recurse publish-sc | Measure-Object -Sum Length`, на Linux/macOS `du -sh publish-sc`.
- Чтобы понять, почему `dotnet run` каждый раз «тратит время» даже без изменений, прочитайте про incremental build и про то, что `run` всё равно проверяет timestamps.
- Если self-contained файл не запускается на Linux/macOS, проверьте битовое разрешение (`chmod +x`) и что RID совпал с архитектурой машины.
- В `#if DEBUG` нет магии: этот символ определяет сам компилятор по конфигурации, и его можно подсмотреть через `<DefineConstants>` в `obj/Debug/.../BuildInsights.csproj.CoreCompileInputs.cache` или через `dotnet build --verbosity:detailed`.

#### Эталонное решение (разбор)

```csharp
// M01-L05. Домашнее задание: BuildInsights.
// Программа показывает, как разные этапы сборки отражаются на артефактах и выводе.
// Build:   dotnet build -c Debug      (или -c Release)
// Run:     dotnet run -c Debug        (или dotnet bin/Debug/net8.0/BuildInsights.dll)
// Publish: dotnet publish -c Release -r win-x64 --self-contained -p:PublishSingleFile=true

using System.Reflection;
using System.Runtime.InteropServices;

// Точка входа через top-level statements — Main не пишем вручную.
// Entry point via top-level statements; no explicit Main required.
var assembly = Assembly.GetExecutingAssembly();
var name = assembly.GetName();

// Raw string literal (""" ... """) позволяет вставлять многострочный текст
// без экранирования кавычек и без интерполяции фигурных скобок.
// Raw string literal enables multi-line output without escaping.
Console.WriteLine($"""
    === Информация о сборке / Build info ===
    Имя / Name:            {name.Name}
    Версия / Version:      {name.Version}
    Конфигурация / Config: {GetConfiguration()}
    Каталог / Location:    {assembly.Location}
    .NET Runtime:          {RuntimeInformation.FrameworkDescription}
    ОС / OS:               {RuntimeInformation.OSDescription}
    Архитектура / Arch:    {RuntimeInformation.OSArchitecture}
    """);

Console.WriteLine();
Console.WriteLine("=== Зависимости / Dependencies ===");

// Перебор референсных сборок показывает, что .deps.json и загрузчик
// разрешают не только прямые, но и транзитивные зависимости.
// Iterating referenced assemblies mirrors what .deps.json resolves at load time.
foreach (var referenced in assembly.GetReferencedAssemblies()
                                    .OrderBy(a => a.Name, StringComparer.Ordinal))
{
    Console.WriteLine($"  - {referenced.Name} {referenced.Version}");
}

Console.WriteLine();
Console.WriteLine($"5! = {Factorial(5)}  // 120");

// Рекурсивная локальная функция со switch-выражением и паттерном `or`.
// Recursive local function with switch expression and `or` pattern.
int Factorial(int n) => n switch
{
    < 0     => throw new ArgumentOutOfRangeException(nameof(n)),
    0 or 1  => 1,
    _       => n * Factorial(n - 1)
};

// Compile-time определение конфигурации через символ DEBUG.
// Compile-time configuration detection via the DEBUG symbol.
static string GetConfiguration()
{
#if DEBUG
    return "Debug (compiled)";
#else
    return "Release (compiled)";
#endif
}
```

Разбор по строкам. Первая содержательная строка `Assembly.GetExecutingAssembly()` возвращает сборку, в которой находится текущий код — это именно тот `BuildInsights.dll`, который лежит в `bin/.../net8.0/`. Через `GetName()` мы достаём имя и версию, которые совпадают с `<AssemblyName>` и `<Version>` из `.csproj`. Raw string literal `"""..."""` — это новинка C# 12, которая позволяет писать многострочный текст без `@` и без экранирования `{}`; интерполяция работает, потому что перед `"""` стоит `$`. Это прямо соответствует примеру кода урока.

Цикл по `GetReferencedAssemblies()` перебирает сборки, на которые ссылается наша — фактически это содержимое `deps.json` в виде объектов. Сортировка через `OrderBy(..., StringComparer.Ordinal)` даёт стабильный вывод, удобный для сравнения Debug и Release сборок. Локальная функция `Factorial` использует switch-выражение с паттернами: `< 0` — relational pattern, `0 or 1` — pattern `or`, `_` — discard. Это тренирует pattern matching из C# 12, как в примере урока.

Самое важное для темы урока — функция `GetConfiguration()` с `#if DEBUG`. Директива `#if DEBUG` вычисляется на этапе компиляции: в Debug-конфигурации компилятор определяет символ `DEBUG` (через `<DefineConstants>`), и в IL попадает строка `"Debug (compiled)"`. В Release символ не определён, и в IL попадает `"Release (compiled)"`. Это честный compile-time признак: даже если запустить Debug-сборку через `dotnet bin/Debug/.../BuildInsights.dll` без слова Debug в команде, вывод всё равно покажет `Debug (compiled)`, потому что символ уже «вшит» в IL. Именно этот механизм урок подчёркивает, говоря, что конфигурация — это свойство сборки, а не команды запуска. Запуск программы в обеих конфигурациях — это наглядная демонстрация того, как `dotnet build -c Release` меняет выходной IL, и почему «собирать в двух конфигурациях перед коммитом» — не пустая рекомендация.

#### Задания на углубление (бонус)

1. **Сравните JIT-оптимизации.** Добавьте цикл, вычисляющий `Factorial(10)` миллион раз, замерьте время в Debug и Release через `Stopwatch`. Объясните в отчёте, почему Release быстрее, и какие оптимизации (inlining, tail-call) могли сработать.
2. **PublishReadyToRun.** Опубликуйте self-contained с дополнительным флагом `-p:PublishReadyToRun=true` и сравните размер и время старта. ReadyToRun — это AOT-подобная предкомпиляция IL в native-код; опишите, как она соотносится с концепцией IL/JIT из урока.
3. **`.runtimeconfig.json` под лупой.** Откройте `BuildInsights.runtimeconfig.json` и найдите секцию `runtimeOptions`. Добавьте в `.csproj` `<ServerGarbageCollection>true</ServerGarbageCollection>` и пересоберите — покажите, как это отразилось в json. Это демонстрирует, что `.csproj` → `.runtimeconfig.json` — это трансформация конфигурации.
4. **Cross-compile для другого RID.** На Windows попробуйте `dotnet publish -c Release -r linux-x64 --self-contained` и объясните, почему полученный файл нельзя запустить локально, но можно — в Linux-контейнере. Это углубляет понимание, что RID определяет целевую платформу, а не платформу сборки.

---

## Statement in English / Постановка на английском

#### Context & motivation

You have just finished the lesson on how C# text becomes a running program. In theory it all looks simple: `.cs` is compiled into IL, IL is packed into a `.dll`, and the JIT turns IL into native machine code at run time. But there is a wide gap between "I understand the theory" and "I can actually build and publish a real project", and the only way across that gap is hands-on practice. That is why this homework is structured as a mini-project: you will create a console application, walk the entire path from `dotnet new` to `dotnet publish --self-contained`, and at every step record which files appear, how large they are, and why.

The motivation is twofold. First, you must stop being afraid of the `bin/` folder. Learning to read its contents and to understand the purpose of every file is a foundational skill for any .NET developer — without it you cannot debug dependency issues, configuration mismatches, or deployment failures. Second, you need to feel the difference between `Debug` and `Release`, between framework-dependent (FDD) and self-contained deployments, between `dotnet run` and launching a `.dll` directly. These distinctions are often described in words, but only measuring sizes and build times yourself makes the knowledge intuitive.

Additionally, the assignment trains a culture of reproducible builds: you will pin the SDK version through `global.json`, add `bin/` and `obj/` to `.gitignore`, and verify that the project still builds on a clean checkout after `git clone`. These are exactly the best practices the lesson flags as mandatory for teamwork. By the end you will have a small but genuinely production-shaped repository that demonstrates a mature command of the .NET 8 tooling.

#### What to do step by step

1. **Create the working folder and the project.** In a dedicated directory run `dotnet new console -n BuildInsights -o BuildInsights --framework net8.0`. Confirm that `BuildInsights.csproj`, `Program.cs`, and `obj/` were created. Open `Program.cs`; it will contain the default `Console.WriteLine("Hello, World!");`. Replace it entirely with the reference code from the "Reference solution" section below (it uses top-level statements, a raw string literal, pattern matching, and `#if DEBUG`).

2. **Pin the SDK with `global.json`.** In the parent directory of the project (the assignment root) run `dotnet new globaljson --sdk-version 8.0.x`, substituting your actual installed SDK (check with `dotnet --version`). Verify that `dotnet --info` reports exactly this SDK as active for the folder. This guarantees reproducible builds across machines.

3. **Build in Debug.** From the `BuildInsights/` folder run `dotnet build -c Debug`. Record in your report: the build time (printed at the end), the output path (it must be `bin/Debug/net8.0/`), and the list of files in that folder. Confirm that `BuildInsights.dll`, `BuildInsights.pdb`, `BuildInsights.runtimeconfig.json`, and `BuildInsights.deps.json` are present. On Windows you will also see `BuildInsights.exe` — that is an apphost stub, not the "real" executable.

4. **Run it two ways.** First, run `dotnet run -c Debug` from the project folder: notice that `run` itself invokes `build` and only then launches the assembly. Second, launch the IL assembly directly: `dotnet bin/Debug/net8.0/BuildInsights.dll`. The output must be identical. Note which method is faster on a repeated invocation and why (hint: `run` always checks whether the build is up to date).

5. **Build in Release and compare.** Run `dotnet build -c Release`. Compare the sizes of `BuildInsights.dll` in `bin/Debug/net8.0/` and `bin/Release/net8.0/` — the Release build is usually slightly smaller thanks to optimizations and reduced debug information. Run `dotnet run -c Release` and check the `Config` field in the output: it must read `Release (compiled)` thanks to `#if DEBUG`.

6. **Publish as framework-dependent.** Run `dotnet publish -c Release -o ./publish-fdd`. Look inside `publish-fdd/`: you should see the same kinds of files as in `bin/Release/net8.0/`, but without intermediate artifacts. Note that there is no bundled runtime — the app expects .NET 8 to be installed on the target machine. Record the total size of the folder.

7. **Publish as self-contained single-file.** Determine your RID: `win-x64`, `linux-x64`, or `osx-arm64` (check `dotnet --info`, the `OSArchitecture`/`RuntimeIdentifier` fields). Run `dotnet publish -c Release -r <RID> --self-contained -p:PublishSingleFile=true -o ./publish-sc`. Compare the size of the resulting single file with the size of the FDD folder — it must be noticeably larger (tens of megabytes), because the runtime is bundled inside. Launch that file directly (double-click or `./publish-sc/BuildInsights` in a terminal) and confirm that the program runs without a separately installed .NET.

8. **Set up `.gitignore`.** At the repository root create a `.gitignore` containing at least: `bin/`, `obj/`, `publish-fdd/`, `publish-sc/`. Use `git status` (if a repository is initialized) to verify that none of these folders are staged. If there is no repository yet, demonstrate that `git check-ignore bin/BuildInsights/obj` returns a match.

9. **Assemble the final report.** In a file called `report.md` at the assignment root, collect: a table of artifact sizes for steps 3, 5, 6, and 7; the output of `dotnet --info`; and a list of files in `bin/Debug/net8.0/` with an explanation of each (based on the lesson). This is the artefact you submit alongside the code.

#### Requirements

- The solution must run on .NET 8 (`TargetFramework` `net8.0`) and C# 12: use top-level statements, raw string literals (`"""..."""`) for multi-line output, and pattern matching with `or` in switch expressions, mirroring the lesson example.
- `Program.cs` must print assembly information via reflection (`Assembly.GetExecutingAssembly()`), the configuration via `#if DEBUG`, runtime and OS information via `RuntimeInformation`, and compute a factorial through a recursive local function with a switch expression and an `or` pattern — this directly mirrors the lesson's code example.
- You must run exactly the commands described: `dotnet build`, `dotnet run`, `dotnet publish` with the specified flags. Running `dotnet run MyApp.dll` is considered an error and must be deliberately avoided; in the report explicitly explain why that command is wrong.
- All `bin/`, `obj/`, and `publish-*` artefacts must be ignored by git. The repository should contain only `BuildInsights.csproj`, `Program.cs`, `global.json`, `.gitignore`, and `report.md`.
- The `report.md` file must contain a table comparing Debug vs Release (`.dll` size) and FDD vs self-contained (deployment size) — without actual numbers the submission is not accepted.
- Configuration must be detected through `#if DEBUG`, not by reading environment variables, because it is a compile-time signal that honestly reflects the mode the IL was built in.

#### Pitfalls

- **`dotnet run` expects a project, not a `.dll`.** The most common mistake is `dotnet run bin/Debug/net8.0/BuildInsights.dll`. That command tries to interpret the argument as a project file to build and fails. The correct forms are either `dotnet run` from the project folder (no argument) or `dotnet bin/Debug/net8.0/BuildInsights.dll` to launch an already-built assembly directly.
- **Do not edit files under `bin/`.** Any modification to a `.dll` or `.runtimeconfig.json` will vanish on the next `build`. Edit only `.cs` and `.csproj`. This is particularly tempting for `.runtimeconfig.json`, but it is fully regenerated from `.csproj` and SDK properties.
- **`--self-contained` requires a RID.** If you run `dotnet publish --self-contained` without `-r win-x64` (or another RID), the publish either errors out or silently falls back to framework-dependent. Always pair `--self-contained` with `-r <RID>`.
- **`MyApp.exe` on Windows is not the main code.** It is an apphost stub that locates and launches `MyApp.dll` through hostpolicy. The real logic lives in the `.dll`. On Linux/macOS there is no `.exe` at all — only `MyApp.dll` (or a single extension-less file under `PublishSingleFile`).
- **Release may strip line numbers from stack traces.** If a Release build throws, the stack trace may lack line numbers, because optimizations inline and reorder code. Use Debug for debugging; for production builds enable `<DebugType>embedded</DebugType>` so the pdb lives inside the dll.
- **`dotnet restore` is almost always implicit.** You rarely invoke it manually — only in CI pipelines with an explicit restore step, or after editing `.csproj` in dependency-heavy projects. This homework does not require a separate restore call; `build` will trigger it for you.
- **Published artifact sizes are deceptive.** An FDD folder looks small, but requires .NET on the target machine. A self-contained single file is large, but fully self-sufficient. The choice is a trade-off that you must articulate in the report, not just list numbers.

#### Acceptance criteria

- [ ] The `BuildInsights` project was created via `dotnet new console --framework net8.0`.
- [ ] `Program.cs` was replaced with the reference code (top-level statements, raw string, pattern matching, `#if DEBUG`).
- [ ] A `global.json` pinning an 8.x SDK exists at the assignment root.
- [ ] `dotnet build -c Debug` was executed; the report records the path and the file list of `bin/Debug/net8.0/`.
- [ ] The program was run both via `dotnet run -c Debug` and via `dotnet bin/Debug/net8.0/BuildInsights.dll`, with identical output.
- [ ] `dotnet build -c Release` was executed; the `.dll` size was compared with Debug in a table.
- [ ] `dotnet run -c Release` prints `Release (compiled)` in the Config field.
- [ ] `dotnet publish -c Release -o ./publish-fdd` was executed; the folder size was recorded.
- [ ] `dotnet publish -c Release -r <RID> --self-contained -p:PublishSingleFile=true -o ./publish-sc` was executed; the single-file size was recorded.
- [ ] The self-contained file was launched directly and runs without a separate `dotnet` invocation.
- [ ] `.gitignore` contains `bin/`, `obj/`, `publish-fdd/`, `publish-sc/`.
- [ ] `report.md` contains comparison tables Debug/Release and FDD/self-contained with real numbers.
- [ ] The report lists and explains the files in `bin/Debug/net8.0/` (`.dll`, `.pdb`, `.runtimeconfig.json`, `.deps.json`, optionally `.exe`).
- [ ] The report explicitly states why `dotnet run MyApp.dll` is an error.
- [ ] `bin/` and `obj/` are not committed to the repository.

#### Hints (no direct answer)

- If `dotnet --version` reports an unexpected version, check `global.json` — it may pin an SDK that is not installed, in which case dotnet tells you which one to install.
- A convenient way to measure artifact sizes: on PowerShell `Get-ChildItem -Recurse publish-sc | Measure-Object -Sum Length`, on Linux/macOS `du -sh publish-sc`.
- To understand why `dotnet run` "wastes time" even without changes, read about incremental builds and the fact that `run` still checks timestamps.
- If the self-contained file does not launch on Linux/macOS, check the executable bit (`chmod +x`) and make sure the RID matches the machine architecture.
- There is no magic in `#if DEBUG`: the compiler defines that symbol based on configuration, and you can peek at it through `<DefineConstants>` in `obj/Debug/.../BuildInsights.csproj.CoreCompileInputs.cache` or via `dotnet build --verbosity:detailed`.

#### Reference solution walk-through

```csharp
// M01-L05. Homework: BuildInsights.
// The program shows how different build stages are reflected in artifacts and output.
// Build:   dotnet build -c Debug      (or -c Release)
// Run:     dotnet run -c Debug        (or dotnet bin/Debug/net8.0/BuildInsights.dll)
// Publish: dotnet publish -c Release -r win-x64 --self-contained -p:PublishSingleFile=true

using System.Reflection;
using System.Runtime.InteropServices;

// Entry point via top-level statements; no explicit Main is needed.
var assembly = Assembly.GetExecutingAssembly();
var name = assembly.GetName();

// Raw string literal (""" ... """) enables multi-line text
// without escaping quotes or interpolating curly braces manually.
Console.WriteLine($"""
    === Build info ===
    Name:            {name.Name}
    Version:         {name.Version}
    Config:          {GetConfiguration()}
    Location:        {assembly.Location}
    .NET Runtime:    {RuntimeInformation.FrameworkDescription}
    OS:              {RuntimeInformation.OSDescription}
    Arch:            {RuntimeInformation.OSArchitecture}
    """);

Console.WriteLine();
Console.WriteLine("=== Dependencies ===");

// Iterating referenced assemblies mirrors what .deps.json resolves at load time,
// including transitive dependencies the loader must locate.
foreach (var referenced in assembly.GetReferencedAssemblies()
                                    .OrderBy(a => a.Name, StringComparer.Ordinal))
{
    Console.WriteLine($"  - {referenced.Name} {referenced.Version}");
}

Console.WriteLine();
Console.WriteLine($"5! = {Factorial(5)}  // 120");

// Recursive local function with a switch expression and an `or` pattern.
int Factorial(int n) => n switch
{
    < 0     => throw new ArgumentOutOfRangeException(nameof(n)),
    0 or 1  => 1,
    _       => n * Factorial(n - 1)
};

// Compile-time configuration detection via the DEBUG symbol.
static string GetConfiguration()
{
#if DEBUG
    return "Debug (compiled)";
#else
    return "Release (compiled)";
#endif
}
```

Line-by-line walk-through. The first meaningful line, `Assembly.GetExecutingAssembly()`, returns the assembly that contains the currently executing code — exactly the `BuildInsights.dll` that lives in `bin/.../net8.0/`. Through `GetName()` we obtain the name and version, which match `<AssemblyName>` and `<Version>` from the `.csproj`. The raw string literal `"""..."""` is a C# 12 feature that lets you write multi-line text without `@` and without escaping `{}`; interpolation works because a `$` precedes the `"""`. This mirrors the lesson's code example directly.

The loop over `GetReferencedAssemblies()` enumerates the assemblies our code references — effectively the contents of `deps.json` as objects. Sorting with `OrderBy(..., StringComparer.Ordinal)` produces a stable output that is easy to diff between Debug and Release builds. The local function `Factorial` uses a switch expression with patterns: `< 0` is a relational pattern, `0 or 1` is an `or` pattern, and `_` is a discard. This trains the C# 12 pattern-matching syntax shown in the lesson.

The most important part for the lesson's theme is `GetConfiguration()` with `#if DEBUG`. The `#if DEBUG` directive is evaluated at compile time: in the Debug configuration the compiler defines the `DEBUG` symbol (via `<DefineConstants>`), so the string `"Debug (compiled)"` is baked into the IL. In Release the symbol is undefined, so `"Release (compiled)"` is baked in instead. This is an honest compile-time signal: even if you launch the Debug assembly through `dotnet bin/Debug/.../BuildInsights.dll` without the word "Debug" in the command, the output still reads `Debug (compiled)`, because the symbol is already embedded in the IL. This is exactly the mechanism the lesson highlights when it says configuration is a property of the assembly, not of the launch command. Running the program in both configurations is a tangible demonstration of how `dotnet build -c Release` changes the emitted IL, and why "build in both configurations before committing" is not an empty recommendation.

#### Going deeper (bonus)

1. **Compare JIT optimizations.** Add a loop computing `Factorial(10)` a million times and measure the time in Debug and Release with `Stopwatch`. Explain in the report why Release is faster and which optimizations (inlining, tail calls) may have applied.
2. **PublishReadyToRun.** Publish self-contained with the extra flag `-p:PublishReadyToRun=true` and compare the file size and startup time. ReadyToRun is an AOT-like pre-compilation of IL into native code; describe how it relates to the IL/JIT concepts from the lesson.
3. **`.runtimeconfig.json` under a magnifier.** Open `BuildInsights.runtimeconfig.json` and find the `runtimeOptions` section. Add `<ServerGarbageCollection>true</ServerGarbageCollection>` to the `.csproj`, rebuild, and show how the json changed. This demonstrates that `.csproj` → `.runtimeconfig.json` is a configuration transformation.
4. **Cross-compile for another RID.** On Windows try `dotnet publish -c Release -r linux-x64 --self-contained` and explain why the resulting file cannot be run locally but can be run inside a Linux container. This deepens the understanding that the RID specifies the target platform, not the build platform.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `BuildInsights` создан, `Program.cs` соответствует эталону.
- [ ] `global.json` закрепляет SDK 8.x.
- [ ] Выполнены `build` (Debug+Release), `run`, `publish` (FDD + self-contained).
- [ ] `report.md` содержит таблицы сравнения и объяснения артефактов.
- [ ] `.gitignore` исключает `bin/`, `obj/`, `publish-*`.
- [ ] The `BuildInsights` project is created and `Program.cs` matches the reference.
- [ ] `global.json` pins an 8.x SDK.
- [ ] `build` (Debug+Release), `run`, `publish` (FDD + self-contained) are all executed.
- [ ] `report.md` contains comparison tables and artifact explanations.
- [ ] `.gitignore` excludes `bin/`, `obj/`, `publish-*`.

#### Ресурсы / Resources
- Microsoft Learn — `dotnet build` — https://learn.microsoft.com/dotnet/core/tools/dotnet-build
- Microsoft Learn — `dotnet run` — https://learn.microsoft.com/dotnet/core/tools/dotnet-run
- Microsoft Learn — `dotnet publish` — https://learn.microsoft.com/dotnet/core/tools/dotnet-publish
- Microsoft Learn — .NET application publishing models (FDD vs self-contained) — https://learn.microsoft.com/dotnet/core/deploying/
- Microsoft Learn — .NET runtime configuration files — https://learn.microsoft.com/dotnet/core/runtime-config/
- Microsoft Learn — `global.json` overview — https://learn.microsoft.com/dotnet/core/tools/global-json
- Microsoft Learn — RID catalog — https://learn.microsoft.com/dotnet/core/rid-catalog

---
[← К уроку M01-L05](lesson-M01-L05-build-run.md) | [⬆ К модулю M01](../README.md) | [Следующее ДЗ →](homework-M01-L06-hello-debug-nuget.md)
---
