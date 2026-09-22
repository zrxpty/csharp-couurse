---
[← К уроку M01-L02](lesson-M01-L02-clr-il-jit.md) | [⬆ К модулю M01](../README.md) | [Следующее ДЗ →](homework-M01-L03-sdk-ide-cli.md)
---

### Домашнее задание M01-L02: CLR, IL, JIT, сборки / Homework M01-L02: CLR, IL, JIT, Assemblies

**Урок / Lesson:** M01-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) На практике закрепить двухэтапную модель компиляции C# → IL → native, разобраться, что реально лежит внутри сборки (IL + метаданные + манифест), увидеть работу JIT и GC своими глазами и научиться извлекать метаданные через рефлексию. (EN) Get hands-on experience with the two-stage compilation model C# → IL → native, understand what really lives inside an assembly (IL + metadata + manifest), observe the JIT and the GC at work with your own eyes, and learn to extract metadata through reflection.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую опирается на ключевые тезисы урока: IL как «эсперанто» .NET, сборку как контейнер IL + метаданных + манифеста, JIT-компиляцию целых методов с кэшированием, обязанности CLR (GC, type safety, JIT) и контраст между классическим JIT и Native AOT. Вы не просто читаете теорию — вы создаёте сборку, вызываете метод, который JIT компилирует при первом вызове, и проверяете метаданные тем же способом (`Assembly.GetExecutingAssembly()`), что и в примере урока.
(EN) The homework rests directly on the lesson's key points: IL as the .NET "Esperanto", the assembly as a container of IL + metadata + manifest, JIT compilation of whole methods with caching, CLR responsibilities (GC, type safety, JIT), and the contrast between classic JIT and Native AOT. You do not just read theory — you create an assembly, invoke a method that the JIT compiles on first call, and inspect metadata using the same technique (`Assembly.GetExecutingAssembly()`) shown in the lesson example.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы только что прошли урок о том, что C# не превращается сразу в машинный код. Между исходником и процессором стоят три важных слоя: промежуточный язык IL, сборка как самодостаточный контейнер и среда CLR, которая на лету превращает IL в нативный код через JIT-компилятор. Это знание легко接受 как абстракцию и трудно — как инженерную реальность: пока вы сами не «потрогаете» IL, не увидите метаданные и не вызовете метод, который JIT компилирует при первом обращении, картинка остаётся теоретической.

В этом задании вы построите небольшое решение из двух проектов — библиотеки `Calculator.Lib` и консольного приложения `ClrLab.App`, которое её использует. Библиотека даст вам «чужую» сборку: вы сможете исследовать её метаданные и манифест через рефлексию, точно как в примере урока, но уже со своими типами и методами. Консольное приложение познакомит вас с обязанностями CLR: вы вызовете метод и попросите JIT показать разобранный нативный код, выделите и освободите память, чтобы увидеть GC в действии, и примените pattern matching из C# 12, чтобы прочувствовать type safety на уровне исполнения. Наконец, вы сравните сборку Debug и Release и опционально попробуете Native AOT — чтобы увидеть, как одна и та же кодовая база превращается в разные артефакты. По итогам вы должны не просто «знать», а уметь показать любому коллеге: вот IL, вот метаданные, вот JIT, вот GC — и объяснить, почему это устроено именно так.

#### Что нужно сделать (пошагово)

1. Создайте решение и проекты. Из пустой папки выполните:
   - `dotnet new sln -n ClrLab`
   - `dotnet new classlib -n Calculator.Lib -o src/Calculator.Lib -f net8.0`
   - `dotnet new console -n ClrLab.App -o src/ClrLab.App -f net8.0`
   - `dotnet sln add src/Calculator.Lib/Calculator.Lib.csproj src/ClrLab.App/ClrLab.App.csproj`
   - `dotnet add src/ClrLab.App/ClrLab.App.csproj reference src/Calculator.Lib/Calculator.Lib.csproj`
   В файле `src/Calculator.Lib/Calculator.Lib.csproj` явно укажите `<Version>1.2.0</Version>` и `<AssemblyVersion>1.2.0.0</AssemblyVersion>`, чтобы в манифесте появилась версия (урок предупреждает: игнор версии ведёт к `FileLoadException` и конфликтам зависимостей).

2. Напишите библиотеку `Calculator.Lib`. В `src/Calculator.Lib/Calculator.cs` создайте `public sealed class Calculator` с методами `Add(int,int)`, `Subtract(int,int)` и `SumAll(int[] values)`. Метод `SumAll` намеренно сделайте циклом `foreach` по массиву — это метод, который JIT скомпилирует при первом вызове, и его интересно дизассемблировать.

3. Напишите консольное приложение `ClrLab.App` с top-level statements. Используйте C# 12: collection expressions (`[]`), raw string literals (`"""..."""`), pattern matching `switch`. Программа должна: вызвать методы калькулятора; через `Assembly.GetExecutingAssembly()` и `typeof(Calculator).Assembly` распечатать имя, версию и публичные типы с их методами (метаданные); выделить 1000 буферов по 1 КБ, очистить список, вызвать `GC.Collect()` и напечатать `GC.GetTotalMemory` до и после (как в примере урока); продемонстрировать type safety через pattern matching над `object`.

4. Соберите и запустите: `dotnet build` затем `dotnet run --project src/ClrLab.App`. Зафиксируйте вывод. Убедитесь, что в `src/ClrLab.App/bin/Debug/net8.0/` появились `ClrLab.App.dll` и `Calculator.Lib.dll` — это и есть сборки с IL и метаданными. Откройте одну из них в текстовом редакторе и найдите строки с именами ваших типов и методов — это проявление метаданных.

5. Посмотрите JIT в действии. Запустите с переменной окружения: в PowerShell `$env:DOTNET_JitDisasm="Calculator.SumAll"; dotnet run --project src/ClrLab.App`. В консоли появится дизассемблированный нативный код метода `SumAll` — наглядное доказательство, что JIT переводит IL в инструкции процессора. Сравните вывод для Debug и Release (`dotnet run -c Release`): в Release будет меньше проверок границ и больше оптимизаций.

6. Сравните сборки. Выполните `dotnet build -c Release` и сравните размеры `Calculator.Lib.dll` в Debug и Release. Опционально: `dotnet publish src/ClrLab.App -c Release -r win-x64 --self-contained` — посмотрите, сколько сборок попадает в выходной каталог (это зависимости из манифеста).

7. Бонус (Native AOT): попробуйте `dotnet publish src/ClrLab.App -c Release -r win-x64 /p:PublishAot=true`. Если сборка упадёт из-за рефлексии — это и есть урок: Native AOT жертвует частью динамических возможностей ради холодного старта. Зафиксируйте наблюдение.

#### Требования к решению

Решение должно представлять собой работоспособное решение `ClrLab.sln` под .NET 8 (TFM `net8.0`), собираемое командой `dotnet build` без ошибок и предупреждений. Используйте современные возможности C# 12: top-level statements в `Program.cs`, collection expressions для инициализации `List<byte[]>`, raw string literals для многострочного вывода, pattern matching `switch` для разбора `object`. Код библиотеки должен быть в отдельном проекте с явно заданной версией в `.csproj` — это требование не косметическое: оно соответствует best practice урока о версионировании сборок и позволяет увидеть версию в манифесте.

Программа обязана демонстрировать все четыре обязанности CLR, упомянутые в уроке: JIT-компиляцию (метод, вызываемый впервые), управление памятью через GC (выделение + очистка + `GC.Collect()`), type safety (pattern matching гарантирует тип) и работу с метаданными (рефлексия над сборкой). Вывод должен быть человекочитаемым, двуязычные подписи допустимы, но не обязательны. Запрещено вызывать `GC.Collect()` в «обычном» коде как практику — он используется здесь только в учебных целях, что надо явно отметить комментарием со ссылкой на best practice урока. Все команды и наблюдения зафиксируйте в коротком `README.md` или в комментариях в коде.

#### Тонкости и подводные камни

- Не путайте два этапа компиляции. `dotnet build` превращает C# в IL и пакует его в `.dll`/`.exe`; нативный код создаёт JIT уже при запуске. Поэтому `.exe` под .NET — это не готовый машинный код, а сборка с точкой входа.
- JIT компилирует метод целиком при первом вызове и кэширует результат — повторные вызовы идут как нативный код. Не считайте CLR интерпретатором: это частая ошибка из урока.
- `Assembly.GetExecutingAssembly()` возвращает сборку, где выполняется текущий код; для «чужой» сборки используйте `typeof(Calculator).Assembly` — это надёжнее, чем загрузка по пути файла.
- `GC.Collect()` в учебном коде допустим, но в продакшене мешает эвристикам сборщика. После `buffers.Clear()` объекты становятся недостижимыми, и GC освободит их сам; ручной вызов нужен только чтобы увидеть эффект в демонстрации.
- Версия сборки берётся из манифеста. Если не задать `<Version>` явно, получите `1.0.0.0`, и при подключении нескольких библиотек могут возникнуть `FileLoadException` и конфликты — урок прямо об этом предупреждает.
- `DOTNET_JitDisasm` чувствителен к регистру и имени метода: указывайте полное имя вроде `Calculator.SumAll` или `Calculator.Lib.Calculator.SumAll`, иначе вывод будет пуст.
- Финализаторы недетерминированы: не полагайтесь на их порядок. Для предсказуемого освобождения используйте `IDisposable` — это из раздела частых ошибок урока.
- В Release проверок границ меньше, но они не исчезают полностью для публичных API; JIT умеет их устранять только внутри метода, где докажет безопасность.
- Native AOT не любит рефлексию: `asm.GetTypes()` может выкинуть или вернуть усечённый набор. Это наглядная демонстрация компромисса «холодный старт против динамизма» из теории урока.

#### Критерии приёмки

- [ ] Создано решение `ClrLab.sln` с проектами `Calculator.Lib` и `ClrLab.App` под `net8.0`.
- [ ] `Calculator.App` ссылается на `Calculator.Lib` (есть `ProjectReference`).
- [ ] В `.csproj` библиотеки явно заданы `<Version>` и `<AssemblyVersion>`.
- [ ] Код использует top-level statements, collection expressions, raw string literals, pattern matching (C# 12).
- [ ] Вызывается метод библиотеки (`Add`/`Subtract`/`SumAll`), который JIT компилирует при первом вызове.
- [ ] Через рефлексию распечатаны имя, версия и список типов с методами сборки `Calculator.Lib`.
- [ ] Демонстрируется работа GC: выделение памяти, очистка ссылок, `GC.Collect()`, вывод памяти до/после.
- [ ] Pattern matching над `object` иллюстрирует type safety CLR.
- [ ] `dotnet build` проходит без ошибок и предупреждений (warnings as errors по желанию).
- [ ] `DOTNET_JitDisasm` показывает дизассемблированный код `SumAll`.
- [ ] Сравнены Debug и Release: зафиксированы различия в размере и/или JIT-выводе.
- [ ] В выводе подтверждено: `.dll` содержит метаданные (видны имена типов/методов).
- [ ] `README.md` или комментарии фиксируют команды и наблюдения.
- [ ] Указано, что `GC.Collect()` использован только в учебных целях (ссылка на best practice).
- [ ] Бонус Native AOT хотя бы упомянут с наблюдением о рефлексии.

#### Подсказки (без прямого ответа)

- Для метаданных методов используйте `BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly`, чтобы не вытащить унаследованные от `object`.
- В raw string literals минимальный отступ определяется по закрывающему `"""` — выровняйте его аккуратно, иначе строки «уедут».
- Чтобы увидеть память, используйте `GC.GetTotalMemory(forceFullCollection: true)` после `GC.Collect()`.
- `DOTNET_JitDisasm` принимает несколько имён через запятую; если метод не нашёлся — проверьте имя через рефлексию `typeof(Calculator).GetMethod("SumAll")`.

#### Эталонное решение (разбор)

`src/Calculator.Lib/Calculator.cs`:
```csharp
// (RU) Библиотека-калькулятор: её IL и метаданные мы будем исследовать.
// (EN) Calculator library: we will inspect its IL and metadata.
namespace Calculator.Lib;

public sealed class Calculator
{
    // (RU) Простые методы — JIT скомпилирует их при первом вызове.
    // (EN) Simple methods — the JIT compiles them on first call.
    public int Add(int a, int b) => a + b;
    public int Subtract(int a, int b) => a - b;

    // (RU) Сумма массива: цикл foreach — удобная цель для дизассемблирования JIT.
    // (EN) Array sum: a foreach loop is a convenient target for JIT disassembly.
    public int SumAll(int[] values)
    {
        int sum = 0;
        foreach (int v in values) sum += v;
        return sum;
    }
}
```

`src/Calculator.App/Program.cs`:
```csharp
// (RU) Top-level statements, C# 12 / .NET 8.
// (EN) Top-level statements, C# 12 / .NET 8.
using System.Reflection;
using Calculator.Lib;

Calculator calc = new();

int r1 = calc.Add(3, 4);
int r2 = calc.Subtract(10, 6);
int r3 = calc.SumAll([1, 2, 3, 4, 5]); // collection expression

Console.WriteLine($"""
    (RU) Add(3,4)       = {r1}
    (RU) Subtract(10,6) = {r2}
    (RU) SumAll         = {r3}
    """);

// (RU) Метаданные «чужой» сборки через typeof — надёжнее пути к файлу.
// (EN) Metadata of the referenced assembly via typeof — safer than file paths.
Assembly libAsm = typeof(Calculator).Assembly;
AssemblyName libName = libAsm.GetName();
Console.WriteLine($"""
    (RU) Сборка библиотеки: {libName.Name}
    (EN) Library assembly:  {libName.Name}
    (RU) Версия:            {libName.Version}
    (EN) Version:           {libName.Version}
    """);

foreach (TypeInfo t in libAsm.GetTypes())
{
    Console.WriteLine($"  • Type: {t.FullName}");
    foreach (MethodInfo m in t.GetMethods(
                 BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly))
    {
        var ps = string.Join(", ", m.GetParameters().Select(p => p.ParameterType.Name));
        Console.WriteLine($"      - method: {m.Name}({ps})");
    }
}

// (RU) GC в учебном режиме. Внимание: GC.Collect() здесь только для демонстрации.
// (EN) GC in demo mode. Note: GC.Collect() is here for demonstration only.
List<byte[]> buffers = [];
for (int i = 0; i < 1000; i++) buffers.Add(new byte[1024]);

long before = GC.GetTotalMemory(forceFullCollection: false);
buffers.Clear();          // (RU) ссылки удалены → объекты недостижимы
GC.Collect();             // (RU) просим GC освободить память
long after = GC.GetTotalMemory(forceFullCollection: true);
Console.WriteLine($"(RU) Память до: {before:N0}, после GC: {after:N0}");

// (RU) Pattern matching — type safety CLR на уровне исполнения.
// (EN) Pattern matching — CLR type safety at execution time.
object maybe = Random.Shared.Next(2) == 0 ? (object)"hello" : 42;
string description = maybe switch
{
    string s => $"(RU) Строка: {s} / (EN) String: {s}",
    int i    => $"(RU) Число: {i} / (EN) Number: {i}",
    _        => "(RU) Неизвестно / (EN) Unknown"
};
Console.WriteLine(description);
```

Разбор по строкам. Класс `Calculator` объявлен `sealed` — это даёт JIT подсказку для девиртуализации (одна из оптимизаций из теории урока). Методы `Add`/`Subtract` — выражения, которые компилятор превратит в короткий IL (`ldarg`, `add`/`sub`, `ret`). `SumAll` намеренно написан `foreach`: это разворачивается в enumerator-цикл, и в дизассемблированном JIT-коде вы увидите, как Release-сборка устраняет проверки границ, когда JIT доказывает безопасность внутри метода.

В `Program.cs` сначала идут вызовы — именно здесь JIT впервые компилирует `Add`, `Subtract` и `SumAll`. Получение сборки через `typeof(Calculator).Assembly` иллюстрирует тезис урока «метаданные делают .NET самодостаточным»: вы не открываете файл на диске, а читаете манифест уже загруженной сборки. `libName.Version` вернёт `1.2.0.0`, потому что мы задали `<AssemblyVersion>` — это прямой ответ на частую ошибку «игнорировать версию в манифесте». Перечисление типов и методов через `GetTypes()`/`GetMethods()` показывает, что метаданные хранят полную сигнатуру, включая параметры.

Блок с `List<byte[]>` и `GC.Collect()` повторяет пример урока: выделение памяти, очистка ссылок (`Clear()`), сборка мусора. Комментарий явно отмечает, что в продакшене так делать нельзя — это best practice из урока. `GC.GetTotalMemory` до и после даёт числовое доказательство работы GC. Наконец, pattern matching `switch` над `object` демонстрирует type safety: CLR гарантирует, что в `maybe` либо `string`, либо `int`, и ветки `string s`/`int i` безопасны — никакого приведения указателей и риска повреждения памяти. Это и есть те самые обязанности CLR (JIT, GC, type safety, метаданные), которые урок просит уметь называть, только теперь показанные кодом.

#### Задания на углубление (бонус)

1. Добавьте в библиотеку generic-метод `T Max<T>(T a, T b) where T : IComparable<T>`. Посмотрите через `DOTNET_JitDisasm`, как JIT специализирует метод для конкретного `T` (например, `int`) — это демонстрация того, что JIT работает с конкретными типами, а не с обобщёнными.
2. Сравните размер `ClrLab.App.dll` иpublish-каталога для обычной публикации, self-contained и Native AOT. Объясните разницу в терминах IL, метаданных и нативного кода.
3. Подпишите сборку строгим именем (strong naming) и через `sn -v` проверьте подпись. Сравните манифест до и после — найдите公开ный ключ.
4. Напишите метод с `IDisposable` (например, класс-обёртку над `FileStream`), покажите детерминированное освобождение через `using`, и объясните, почему это надёжнее финализатора (недетерминированность из урока).

---

## Statement in English / Постановка на английском

#### Context & motivation

You have just finished a lesson explaining that C# does not turn into machine code directly. Between your source and the CPU sit three important layers: the Intermediate Language (IL), the assembly as a self-describing container, and the Common Language Runtime (CLR), which turns IL into native code on the fly through the Just-In-Time (JIT) compiler. This knowledge is easy to accept as an abstraction and hard to accept as an engineering reality: until you touch IL yourself, look at metadata, and call a method that the JIT compiles on first invocation, the picture stays theoretical.

In this assignment you will build a small solution of two projects — a library `Calculator.Lib` and a console application `ClrLab.App` that consumes it. The library gives you a "foreign" assembly: you will be able to inspect its metadata and manifest through reflection, exactly as in the lesson example, but now with your own types and methods. The console application will introduce you to the responsibilities of the CLR: you will call a method and ask the JIT to show the disassembled native code, allocate and release memory to watch the GC in action, and apply pattern matching from C# 12 to feel type safety at execution time. Finally, you will compare a Debug and a Release build and, optionally, try Native AOT — to see how the same code base becomes different artifacts. By the end you should not just "know" but be able to show any colleague: here is the IL, here is the metadata, here is the JIT, here is the GC — and explain why it is designed this way.

#### What to do step by step

1. Create the solution and the projects. From an empty folder run:
   - `dotnet new sln -n ClrLab`
   - `dotnet new classlib -n Calculator.Lib -o src/Calculator.Lib -f net8.0`
   - `dotnet new console -n ClrLab.App -o src/ClrLab.App -f net8.0`
   - `dotnet sln add src/Calculator.Lib/Calculator.Lib.csproj src/ClrLab.App/ClrLab.App.csproj`
   - `dotnet add src/ClrLab.App/ClrLab.App.csproj reference src/Calculator.Lib/Calculator.Lib.csproj`
   In `src/Calculator.Lib/Calculator.Lib.csproj` set `<Version>1.2.0</Version>` and `<AssemblyVersion>1.2.0.0</AssemblyVersion>` explicitly so that the manifest carries a real version (the lesson warns that ignoring versions leads to `FileLoadException` and dependency conflicts).

2. Write the library `Calculator.Lib`. In `src/Calculator.Lib/Calculator.cs` create a `public sealed class Calculator` with methods `Add(int,int)`, `Subtract(int,int)`, and `SumAll(int[] values)`. Implement `SumAll` with a `foreach` loop over the array on purpose — this is the method the JIT compiles on first call, and it is interesting to disassemble.

3. Write the console application `ClrLab.App` with top-level statements. Use C# 12: collection expressions (`[]`), raw string literals (`"""..."""`), and a `switch` pattern matching expression. The program must: call the calculator methods; print the name, version, and public types with their methods via `Assembly.GetExecutingAssembly()` and `typeof(Calculator).Assembly` (metadata); allocate 1000 buffers of 1 KB each, clear the list, call `GC.Collect()`, and print `GC.GetTotalMemory` before and after (as in the lesson example); demonstrate type safety through pattern matching over `object`.

4. Build and run: `dotnet build` then `dotnet run --project src/ClrLab.App`. Record the output. Verify that `src/ClrLab.App/bin/Debug/net8.0/` now contains `ClrLab.App.dll` and `Calculator.Lib.dll` — these are the assemblies holding IL and metadata. Open one of them in a text editor and search for the names of your types and methods — that is metadata showing through.

5. Watch the JIT at work. Run with an environment variable: in PowerShell `$env:DOTNET_JitDisasm="Calculator.SumAll"; dotnet run --project src/ClrLab.App`. The console will print the disassembled native code of `SumAll` — a tangible proof that the JIT turns IL into CPU instructions. Compare the output for Debug and Release (`dotnet run -c Release`): Release will show fewer bounds checks and more optimizations.

6. Compare the assemblies. Run `dotnet build -c Release` and compare the sizes of `Calculator.Lib.dll` in Debug and Release. Optionally: `dotnet publish src/ClrLab.App -c Release -r win-x64 --self-contained` — observe how many assemblies land in the output directory (these are the dependencies declared in the manifest).

7. Bonus (Native AOT): try `dotnet publish src/ClrLab.App -c Release -r win-x64 /p:PublishAot=true`. If the build fails because of reflection — that is the lesson itself: Native AOT trades some dynamic capabilities for a faster cold start. Record the observation.

#### Requirements

The solution must be a working `ClrLab.sln` targeting .NET 8 (TFM `net8.0`), building with `dotnet build` without errors and warnings. Use modern C# 12 features: top-level statements in `Program.cs`, collection expressions to initialize `List<byte[]>`, raw string literals for multi-line output, and a pattern matching `switch` to deconstruct an `object`. The library code must live in a separate project with an explicitly set version in the `.csproj` — this requirement is not cosmetic: it matches the lesson's best practice about versioning assemblies and lets you see the version in the manifest.

The program must demonstrate all four CLR responsibilities mentioned in the lesson: JIT compilation (a method invoked for the first time), memory management through GC (allocation + cleanup + `GC.Collect()`), type safety (pattern matching guarantees the type), and metadata handling (reflection over an assembly). The output must be human-readable; bilingual labels are allowed but not required. It is forbidden to call `GC.Collect()` in "ordinary" code as a habit — here it is used only for educational purposes, which you must note explicitly in a comment referencing the lesson's best practice. Record every command and observation in a short `README.md` or in code comments.

#### Pitfalls

- Do not confuse the two compilation stages. `dotnet build` turns C# into IL and packs it into `.dll`/`.exe`; the JIT creates native code at run time. Therefore a .NET `.exe` is not ready machine code — it is an assembly with an entry point.
- The JIT compiles a whole method on first call and caches the result — repeated calls run as native code. Do not think of the CLR as an interpreter: that is a common mistake from the lesson.
- `Assembly.GetExecutingAssembly()` returns the assembly where the current code runs; for a "foreign" assembly use `typeof(Calculator).Assembly` — it is more reliable than loading by file path.
- `GC.Collect()` is acceptable in teaching code but hurts the GC's heuristics in production. After `buffers.Clear()` the objects become unreachable and the GC reclaims them on its own; the manual call is only there to make the effect visible.
- The assembly version comes from the manifest. If you do not set `<Version>` explicitly you get `1.0.0.0`, and wiring several libraries together may cause `FileLoadException` and conflicts — the lesson warns about this directly.
- `DOTNET_JitDisasm` is case-sensitive and name-sensitive: use a full name like `Calculator.SumAll` or `Calculator.Lib.Calculator.SumAll`, otherwise the output will be empty.
- Finalizers are non-deterministic: do not rely on their order. For predictable cleanup use `IDisposable` — straight from the lesson's common mistakes.
- In Release there are fewer bounds checks, but they do not disappear entirely for public APIs; the JIT eliminates them only inside a method where it can prove safety.
- Native AOT dislikes reflection: `asm.GetTypes()` may throw or return a trimmed set. This is a vivid demonstration of the "cold start versus dynamism" trade-off from the theory.

#### Acceptance criteria

- [ ] A solution `ClrLab.sln` exists with projects `Calculator.Lib` and `ClrLab.App` on `net8.0`.
- [ ] `ClrLab.App` references `Calculator.Lib` (a `ProjectReference` is present).
- [ ] The library `.csproj` explicitly sets `<Version>` and `<AssemblyVersion>`.
- [ ] The code uses top-level statements, collection expressions, raw string literals, and pattern matching (C# 12).
- [ ] A library method (`Add`/`Subtract`/`SumAll`) is invoked, which the JIT compiles on first call.
- [ ] Reflection prints the name, version, and the list of types with methods of the `Calculator.Lib` assembly.
- [ ] The GC is demonstrated: memory allocation, reference cleanup, `GC.Collect()`, memory printed before/after.
- [ ] Pattern matching over `object` illustrates CLR type safety.
- [ ] `dotnet build` passes without errors and warnings (treat warnings as errors optionally).
- [ ] `DOTNET_JitDisasm` shows the disassembled code of `SumAll`.
- [ ] Debug and Release are compared: differences in size and/or JIT output are recorded.
- [ ] The output confirms that the `.dll` carries metadata (type/method names are visible).
- [ ] A `README.md` or comments record the commands and observations.
- [ ] It is noted that `GC.Collect()` is used only for educational purposes (a reference to the best practice).
- [ ] The Native AOT bonus is at least mentioned with an observation about reflection.

#### Hints (without giving the answer away)

- For method metadata use `BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly` so you do not pull in members inherited from `object`.
- In raw string literals the minimum indent is determined by the closing `"""` — align it carefully or the lines will drift.
- To observe memory, use `GC.GetTotalMemory(forceFullCollection: true)` after `GC.Collect()`.
- `DOTNET_JitDisasm` accepts several names separated by commas; if a method is not found, verify the name through reflection: `typeof(Calculator).GetMethod("SumAll")`.

#### Reference solution walk-through

`src/Calculator.Lib/Calculator.cs`:
```csharp
// (EN) Calculator library: we will inspect its IL and metadata.
namespace Calculator.Lib;

public sealed class Calculator
{
    // (EN) Simple methods — the JIT compiles them on first call.
    public int Add(int a, int b) => a + b;
    public int Subtract(int a, int b) => a - b;

    // (EN) Array sum: a foreach loop is a convenient target for JIT disassembly.
    public int SumAll(int[] values)
    {
        int sum = 0;
        foreach (int v in values) sum += v;
        return sum;
    }
}
```

`src/ClrLab.App/Program.cs`:
```csharp
// (EN) Top-level statements, C# 12 / .NET 8.
using System.Reflection;
using Calculator.Lib;

Calculator calc = new();

int r1 = calc.Add(3, 4);
int r2 = calc.Subtract(10, 6);
int r3 = calc.SumAll([1, 2, 3, 4, 5]); // collection expression

Console.WriteLine($"""
    (EN) Add(3,4)       = {r1}
    (EN) Subtract(10,6) = {r2}
    (EN) SumAll         = {r3}
    """);

// (EN) Metadata of the referenced assembly via typeof — safer than file paths.
Assembly libAsm = typeof(Calculator).Assembly;
AssemblyName libName = libAsm.GetName();
Console.WriteLine($"""
    (EN) Library assembly: {libName.Name}
    (EN) Version:          {libName.Version}
    """);

foreach (TypeInfo t in libAsm.GetTypes())
{
    Console.WriteLine($"  • Type: {t.FullName}");
    foreach (MethodInfo m in t.GetMethods(
                 BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly))
    {
        var ps = string.Join(", ", m.GetParameters().Select(p => p.ParameterType.Name));
        Console.WriteLine($"      - method: {m.Name}({ps})");
    }
}

// (EN) GC in demo mode. Note: GC.Collect() is here for demonstration only.
List<byte[]> buffers = [];
for (int i = 0; i < 1000; i++) buffers.Add(new byte[1024]);

long before = GC.GetTotalMemory(forceFullCollection: false);
buffers.Clear();          // (EN) references removed → objects unreachable
GC.Collect();             // (EN) ask the GC to reclaim memory
long after = GC.GetTotalMemory(forceFullCollection: true);
Console.WriteLine($"(EN) Memory before: {before:N0}, after GC: {after:N0}");

// (EN) Pattern matching — CLR type safety at execution time.
object maybe = Random.Shared.Next(2) == 0 ? (object)"hello" : 42;
string description = maybe switch
{
    string s => $"(EN) String: {s}",
    int i    => $"(EN) Number: {i}",
    _        => "(EN) Unknown"
};
Console.WriteLine(description);
```

Line-by-line walk-through. The `Calculator` class is declared `sealed` — this gives the JIT a hint for devirtualization, one of the optimizations named in the lesson theory. The `Add`/`Subtract` methods are expression-bodied; the compiler turns them into short IL (`ldarg`, `add`/`sub`, `ret`). `SumAll` is written with `foreach` on purpose: it expands into an enumerator-based loop, and in the disassembled JIT output you will see how a Release build eliminates bounds checks once the JIT proves safety inside the method.

In `Program.cs` the calls come first — this is exactly where the JIT compiles `Add`, `Subtract`, and `SumAll` for the first time. Obtaining the assembly through `typeof(Calculator).Assembly` illustrates the lesson's claim that "metadata makes .NET self-describing": you do not open a file on disk, you read the manifest of an already loaded assembly. `libName.Version` returns `1.2.0.0` because we set `<AssemblyVersion>` — a direct answer to the common mistake of "ignoring the version in the manifest". Enumerating types and methods through `GetTypes()`/`GetMethods()` shows that metadata stores full signatures, including parameters.

The block with `List<byte[]>` and `GC.Collect()` mirrors the lesson example: allocate memory, clear references (`Clear()`), collect garbage. The comment explicitly notes that this is forbidden in production — this is the best practice from the lesson. `GC.GetTotalMemory` before and after gives numeric proof of the GC at work. Finally, the `switch` pattern matching over `object` demonstrates type safety: the CLR guarantees that `maybe` holds either a `string` or an `int`, and the `string s`/`int i` branches are safe — no pointer casts, no risk of memory corruption. These are exactly the CLR responsibilities (JIT, GC, type safety, metadata) the lesson asks you to name, only now shown in code.

#### Going deeper (bonus)

1. Add a generic method `T Max<T>(T a, T b) where T : IComparable<T>` to the library. Watch with `DOTNET_JitDisasm` how the JIT specializes the method for a concrete `T` (for example `int`) — a demonstration that the JIT works with concrete types, not with open generics.
2. Compare the size of `ClrLab.App.dll` and the publish directory for a regular publish, a self-contained publish, and Native AOT. Explain the difference in terms of IL, metadata, and native code.
3. Sign the assembly with a strong name and verify it with `sn -v`. Compare the manifest before and after — find the public key.
4. Write a method with `IDisposable` (for example a wrapper over `FileStream`), show deterministic cleanup with `using`, and explain why this is more reliable than a finalizer (the non-determinism mentioned in the lesson).

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение `ClrLab.sln` собирается через `dotnet build`.
- [ ] (RU) В `.csproj` библиотеки задана версия.
- [ ] (RU) Вывод содержит метаданные сборки (имя, версия, типы, методы).
- [ ] (RU) Демонстрируется JIT (`DOTNET_JitDisasm`), GC и type safety.
- [ ] (RU) `README.md` фиксирует команды и наблюдения по Debug/Release.
- [ ] (EN) The `ClrLab.sln` solution builds with `dotnet build`.
- [ ] (EN) The library `.csproj` has an explicit version.
- [ ] (EN) The output shows assembly metadata (name, version, types, methods).
- [ ] (EN) JIT (`DOTNET_JitDisasm`), GC, and type safety are demonstrated.
- [ ] (EN) A `README.md` records the commands and Debug/Release observations.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/standard/clr — Common Language Runtime (CLR) / Среда CLR
- Microsoft Learn — https://learn.microsoft.com/dotnet/standard/assembly/ — Assemblies in .NET / Сборки в .NET
- Microsoft Learn — https://learn.microsoft.com/dotnet/standard/garbage-collection/ — Garbage Collection / Сборка мусора
- Microsoft Learn — https://learn.microsoft.com/dotnet/core/deploying/native-aot/ — Native AOT deployment / Развертывание Native AOT
- .NET JIT diagnostics — https://github.com/dotnet/runtime/blob/main/docs/workflow/debugging/libultask/jit-diagnostics.md — `DOTNET_JitDisasm` and friends / Диагностика JIT

---
[← К уроку M01-L02](lesson-M01-L02-clr-il-jit.md) | [⬆ К модулю M01](../README.md) | [Следующее ДЗ →](homework-M01-L03-sdk-ide-cli.md)
---
