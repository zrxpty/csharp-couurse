---
[← Предыдущий: M01-L01](lesson-M01-L01-csharp-dotnet-intro.md) | [⬆ К модулю M01](../README.md) | [Следующий: M01-L03 →](lesson-M01-L03-sdk-ide-cli.md)
---

### Урок M01-L02: CLR, IL, JIT, сборки / CLR, IL, JIT, Assemblies

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Когда вы пишете код на C#, он не превращается сразу в инструкции процессора, как это происходит в C или C++. Вместо этого компилятор C# (`csc.exe`) переводит исходный текст в промежуточный язык — **IL (Intermediate Language)**, иногда называемый MSIL или CIL. IL — это байткод: набор инструкций, понятный виртуальной машине .NET, но не конкретному процессору. Можно представить IL как универсальный «эсперанто» для всех .NET-языков: код на C#, F# и VB.NET компилируется в один и тот же IL, а значит, может свободно взаимодействовать.

Полученный IL упаковывается в **сборку (assembly)** — файл с расширением `.dll` (библиотека) или `.exe` (исполняемый файл). Важно понимать: внутри сборки лежит не только IL, но и **метаданные (metadata)** — полное описание типов, методов, параметров, полей и их сигнатур. Метаданные делают .NET самодостаточной платформой: рефлексия, IntelliSense, сериализация и проверка типов на этапе выполнения опираются именно на них. Дополнительно каждая сборка содержит **манифест (manifest)** — «паспорт» сборки, где указаны её имя, версия, культура, открытый ключ (для строгого имени) и список зависимостей от других сборок.

Когда вы запускаете приложение, в дело вступает **CLR (Common Language Runtime)** — среда выполнения .NET. CLR загружает сборки, проверяет безопасность и корректность IL, а затем вызывает **JIT-компилятор (Just-In-Time)**. JIT работает «на лету»: когда метод впервые вызывается, JIT переводит IL этого метода в нативный машинный код для конкретного процессора и операционной системы. Результат кэшируется в памяти, поэтому повторные вызовы выполняются уже как нативный код — быстро. Аналогия: IL — это запись выступления на универсальном языке, а JIT — переводчик, который переводит каждый фрагмент прямо перед тем, как его произнести, и держит перевод под рукой для следующего раза. В отличие от классических интерпретаторов, JIT не переводит строку за строкой во время выполнения — он компилирует целые методы, что позволяет применять оптимизации (inline-подстановку, устранение проверок границ, девиртуализацию).

CLR отвечает не только за компиляцию. Среди его ключевых обязанностей — **управление памятью (Garbage Collection, GC)**. GC автоматически отслеживает объекты в управляемой куче, определяет достижимые ссылки и освобождает недостижимые, избавляя разработчика от ручного `free`. Это исключает целый класс ошибок: утечки памяти, двойное освобождение, использование после освобождения. Кроме того, CLR обеспечивает **безопасность типов (type safety)**: переменная, объявленная как `string`, гарантированно содержит `string` или `null`, а не произвольные байты. Это предотвращает повреждение памяти, переполнения буферов и подделку указателей. На этой базе строятся проверки безопасности доступа к коду (CAS в .NET Framework, в .NET Core/.NET 5+ — более лёгкая модель с AOT и trimming).

Современный .NET 8 добавляет ещё один путь: **AOT-компиляция (Ahead-Of-Time)**, например Native AOT, которая превращает IL в нативный код заранее, на этапе сборки, уменьшая время запуска и размер образа ценой части динамических возможностей рефлексии. Но классическая модель JIT остаётся основной для серверных и десктопных приложений, потому что она сочетает переносимость IL с производительностью нативного кода.

#### Theory (EN)

When you write C# code, it does not become raw CPU instructions immediately, as it would in C or C++. Instead, the C# compiler (`csc.exe`) translates your source into an intermediate language called **IL (Intermediate Language)**, also known as MSIL or CIL. IL is bytecode: a set of instructions meant for the .NET virtual machine, not for any particular CPU. Think of IL as a universal "Esperanto" for all .NET languages — C#, F#, and VB.NET all compile to the same IL, which is why they can interoperate seamlessly and share libraries.

The resulting IL is packaged into an **assembly** — a file with a `.dll` (library) or `.exe` (executable) extension. Importantly, an assembly contains not only IL but also **metadata** — a complete description of every type, method, parameter, field, and signature. Metadata makes .NET self-describing: reflection, IntelliSense, serialization, and runtime type verification all depend on it. In addition, every assembly carries a **manifest** — the assembly's "passport", listing its name, version, culture, public key (for strong naming), and the list of other assemblies it depends on.

When you run the application, the **CLR (Common Language Runtime)** takes over. The CLR loads assemblies, verifies security and IL correctness, and then invokes the **JIT compiler (Just-In-Time)**. The JIT works on the fly: the first time a method is called, the JIT translates that method's IL into native machine code for the specific CPU and operating system. The result is cached in memory, so subsequent calls execute as fast native code. An analogy: IL is a speech written in a universal language, and the JIT is an interpreter who translates each passage right before delivering it — and keeps the translation ready for the next time. Unlike a classic line-by-line interpreter, the JIT compiles whole methods, which enables optimizations such as inlining, bounds-check elimination, and devirtualization.

The CLR is responsible for more than compilation. One of its core duties is **memory management through Garbage Collection (GC)**. The GC automatically tracks objects in the managed heap, identifies reachable references, and reclaims unreachable ones, freeing developers from manual `free` calls. This eliminates an entire category of bugs: memory leaks, double-free, and use-after-free. Beyond that, the CLR enforces **type safety**: a variable declared as `string` is guaranteed to hold a `string` or `null`, never arbitrary bytes. This prevents memory corruption, buffer overruns, and pointer spoofing. On this foundation rest code-access security checks (CAS in .NET Framework; in .NET Core/.NET 5+ a leaner model with AOT and trimming support).

Modern .NET 8 adds another path: **AOT compilation (Ahead-Of-Time)**, such as Native AOT, which compiles IL to native code ahead of time, at build time, reducing startup time and image size at the cost of some dynamic reflection capabilities. However, the classic JIT model remains the default for server and desktop applications because it combines the portability of IL with the performance of native code.

#### Пример кода / Code Example

```csharp
// Демонстрация IL, метаданных, JIT и сборок / IL, metadata, JIT, assembly demo
// C# 12 / .NET 8 — top-level statements / top-level statements

using System.Reflection;

// Простой метод, который JIT скомпилирует при первом вызове
// A simple method the JIT compiles on first call
int Add(int a, int b) => a + b;

// Первый вызов → JIT переводит IL метода Add в нативный код
// First call → JIT compiles Add's IL into native code
int result = Add(3, 4);
Console.WriteLine($"3 + 4 = {result} (RU: результат посчитан нативным кодом / EN: computed by native code)");

// Метаданные сборки: получаем информацию о текущей сборке через рефлексию
// Assembly metadata: inspect the current assembly via reflection
Assembly asm = Assembly.GetExecutingAssembly();
Console.WriteLine($"""
    (RU) Сборка: {asm.GetName().Name}
    (EN) Assembly: {asm.GetName().Name}
    (RU) Версия:  {asm.GetName().Version}
    (EN) Version: {asm.GetName().Version}
    """);

// Перечисляем публичные типы — это и есть метаданные, извлечённые из сборки
// List public types — this is metadata extracted from the assembly
foreach (TypeInfo t in asm.GetTypes().Take(3))
{
    Console.WriteLine($"  • Type: {t.FullName}");
}

// Пример управления памятью: GC сам освободит недостижимые объекты
// Memory management example: GC reclaims unreachable objects automatically
List<byte[]> buffers = [];
for (int i = 0; i < 1000; i++)
{
    buffers.Add(new byte[1024]); // выделяем память / allocate memory
}

buffers.Clear(); // ссылки удалены → объекты стали недостижимыми / references removed → unreachable
GC.Collect();    // просим GC освободить память / ask GC to reclaim memory
Console.WriteLine($"""
    (RU) Память освобождена сборщиком мусора.
    (EN) Memory reclaimed by the garbage collector.
    """);

// Pattern matching (C# 12) — безопасная работа с типами благодаря type safety CLR
// Pattern matching (C# 12) — safe type handling thanks to CLR type safety
object maybe = Random.Shared.Next(2) == 0 ? "hello" : 42;
string description = maybe switch
{
    string s => $"(RU) Строка: {s} / (EN) String: {s}",
    int i    => $"(RU) Число: {i} / (EN) Number: {i}",
    _        => "(RU) Неизвестно / (EN) Unknown"
};
Console.WriteLine(description);
```

#### Best Practices
- (RU) Компилируйте в Release для продакшена — JIT применяет более агрессивные оптимизации (inline, устранение проверок).
- (EN) Build in Release for production — the JIT applies more aggressive optimizations (inlining, check elimination).
- (RU) Не вызывайте `GC.Collect()` в обычном коде — это мешает эвристикам сборщика и ухудшает производительность.
- (EN) Do not call `GC.Collect()` in normal code — it interferes with the GC's heuristics and hurts performance.
- (RU) Версонируйте сборки и используйте строгие имена (strong naming) для публичных библиотек.
- (EN) Version your assemblies and use strong naming for public libraries.
- (RU) Для холодного старта микросервисов рассмотрите Native AOT, но проверьте совместимость с рефлексией.
- (EN) For cold-start-sensitive microservices consider Native AOT, but verify reflection compatibility.
- (RU) Доверяйте метаданным вместо ручной сериализации конфигурации типов — это снижает количество ошибок.
- (EN) Rely on metadata rather than hand-rolled type configuration — it reduces bugs.

#### Частые ошибки / Common Mistakes
- (RU) Думать, что `.exe` содержит готовый машинный код → на самом деле внутри IL и метаданные, нативный код создаёт JIT. Помните о двух этапах: компиляция C#→IL, затем IL→native.
- (EN) Assuming `.exe` contains ready machine code → it actually holds IL and metadata; the JIT produces native code. Remember two stages: C#→IL, then IL→native.
- (RU) Считать CLR «интерпретатором» → JIT компилирует целые методы и кэширует результат, поэтому повторные вызовы быстрые.
- (EN) Believing the CLR is an "interpreter" → the JIT compiles whole methods and caches the result, so repeated calls are fast.
- (RU) Вызывать `GC.Collect()` после каждого освобождения ссылок → просто обнуляйте ссылки (`buffers.Clear()`), GC справится сам.
- (EN) Calling `GC.Collect()` after every reference cleanup → just drop the references (`buffers.Clear()`); the GC handles the rest.
- (RU) Игнорировать версию в манифесте → получаете `FileLoadException` и конфликты зависимостей. Указывайте версию явно.
- (EN) Ignoring the version in the manifest → you get `FileLoadException` and dependency conflicts. Set versions explicitly.
- (RU) Полагаться на порядок финализации → финализаторы не детерминированы. Используйте `IDisposable` для предсказуемого освобождения.
- (EN) Relying on finalization order → finalizers are non-deterministic. Use `IDisposable` for predictable cleanup.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] (RU) Я могу объяснить, чем IL отличается от нативного машинного кода.
- [ ] (EN) I can explain how IL differs from native machine code.
- [ ] (RU) Я понимаю, что JIT компилирует метод при первом вызове и кэширует результат.
- [ ] (EN) I understand the JIT compiles a method on first call and caches the result.
- [ ] (RU) Я знаю, что сборка содержит IL + метаданные + манифест.
- [ ] (EN) I know an assembly contains IL + metadata + manifest.
- [ ] (RU) Я могу назвать минимум три обязанности CLR: GC, type safety, JIT-компиляция.
- [ ] (EN) I can name at least three CLR responsibilities: GC, type safety, JIT compilation.
- [ ] (RU) Я отличаю `.dll` (библиотека) от `.exe` (точка входа) и понимаю, что оба содержат IL.
- [ ] (EN) I distinguish `.dll` (library) from `.exe` (entry point) and know both contain IL.
- [ ] (RU) Я понимаю разницу между JIT и Native AOT и когда выбирать каждый.
- [ ] (EN) I understand the difference between JIT and Native AOT and when to choose each.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/standard/clr — Common Language Runtime (CLR) / Среда CLR
- Microsoft Learn — https://learn.microsoft.com/dotnet/standard/assembly/ — Assemblies in .NET / Сборки в .NET

---
[← Предыдущий: M01-L01](lesson-M01-L01-csharp-dotnet-intro.md) | [⬆ К модулю M01](../README.md) | [Следующий: M01-L03 →](lesson-M01-L03-sdk-ide-cli.md)
---
