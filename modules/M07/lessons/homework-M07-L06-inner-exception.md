---
[← К уроку M07-L06](lesson-M07-L06-inner-exception.md) | [⬆ К модулю M07](../README.md) | [Следующее ДЗ →](homework-M07-L07-result-pattern.md)
---

### Домашнее задание M07-L06: InnerException, цепочки / Homework M07-L06: InnerException, chains

**Урок / Lesson:** M07-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осмысленно оборачивать исключения на границах слоёв абстракции, сохраняя первопричину через `InnerException`, корректно обходить цепочку до корня, собирать и фильтровать множественные ошибки параллельных задач через `AggregateException`, а также перебрасывать исключения между потоками с сохранением оригинального стека через `ExceptionDispatchInfo`. (EN) Learn to deliberately wrap exceptions at abstraction-layer boundaries while preserving the root cause via `InnerException`, walk the chain to its origin, collect and filter multiple errors from parallel tasks through `AggregateException`, and rethrow exceptions across threads while keeping the original stack via `ExceptionDispatchInfo`.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит концепцию цепочки исключений: каждое звено добавляет контекст, а корень остаётся доступным через `InnerException`. ДЗ закрепляет все ключевые практики урока — передачу оригинала в конструктор, обход цепочки циклом, `AggregateException.Flatten()`/`Handle()`, `ExceptionDispatchInfo.Capture(ex).Throw()` и логирование через `ex.ToString()`, — а также явно отрабатывает частые ошибки `throw ex` и «глотание» исключений.
(EN) The lesson introduces the exception chain: each link adds context, while the root stays reachable through `InnerException`. This homework cements every key practice from the lesson — passing the original to the constructor, walking the chain in a loop, `AggregateException.Flatten()`/`Handle()`, `ExceptionDispatchInfo.Capture(ex).Throw()`, and logging via `ex.ToString()` — and explicitly drills the common mistakes of `throw ex` and swallowing exceptions.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы разрабатываете модуль импорта конфигурации для корпоративного сервиса «ReportsHub». Конфигурация сервиса разбита на несколько JSON-файлов: `database.json`, `cache.json`, `endpoints.json`. Каждый файл читается параллельно, разбирается и валидируется. Архитектура строго слоистая: слой `FileSource` отвечает только за чтение байтов с диска; слой `ConfigRepository` превращает сырые ошибки ввода-вывода в доменное `ConfigRepositoryException` с указанием имени сущности; слой `ConfigService` координирует загрузку всех файлов и при множественных сбоях собирает их в `AggregateException`; наконец, `ImportOrchestrator` перебрасывает агрегированную ошибку в поток диспетчера, сохраняя оригинальные стеки.

Бизнесу важно, чтобы при падении импорта оператор видел не «что-то сломалось», а полную картину: какой файл, какой слой, какая первопричина — и чтобы в лог попадал весь стек до строки, где реально упал `FileStream`. Одновременно с этим нельзя терять ни одну ошибку: если из трёх файлов упали два, в отчёт должны попасть оба сбоя, а не только первый. И наконец, поскольку чтение идёт в параллельных задачах, а обработка — в потоке диспетчера, нужно корректно маршалить исключение через границу потока, не обнуляя стек. Это классический сценарий, где механизм `InnerException` и `AggregateException` из урока M07-L06 не просто полезны, а являются единственным правильным способом сохранить диагностическую ценность ошибок.

#### Что нужно сделать (пошагово)

1. Создайте консольный проект на .NET 8: выполните команду `dotnet new console -n ReportsHub.Importer -o ReportsHub.Importer` в каталоге `modules/M07/homework`. Убедитесь, что в `ReportsHub.Importer.csproj` свойство `<TargetFramework>net8.0</TargetFramework>` присутствует, и добавьте `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`, `<LangVersion>latest</LangVersion>`.
2. В файле `Program.cs` оставьте top-level statements. Создайте каталог `Source/` с файлами `FileSource.cs`, `ConfigRepositoryException.cs`, `ConfigRepository.cs`, `ConfigService.cs`, `ImportOrchestrator.cs`, `ChainWalker.cs`, `Program.cs` (точка входа остаётся в корне).
3. Реализуйте `ConfigRepositoryException` — запечатанное (`sealed`) пользовательское исключение, производное от `Exception`, с контекстными свойствами `EntityName` (имя файла-сущности) и `Operation` (например, `"Read"`, `"Parse"`). Конструктор обязан принимать `string message`, `string? entityName`, `string? operation` и обязательно `Exception inner`, вызывая `base(message, inner)`. Добавьте статический фабричный метод `Wrap(string entity, string operation, Exception inner)` для удобства.
4. Реализуйте `FileSource.ReadAllTextAsync(string path)`: открывает файл через `File.OpenRead`, читает до конца. Если файл не существует, естественно возникает `FileNotFoundException` — НЕ перехватывайте его здесь, позвольте всплывать.
5. Реализуйте `ConfigRepository.LoadAsync(string entityName, string path)`: вызывает `FileSource.ReadAllTextAsync` внутри `try/catch (IOException ex)`. В `catch` выбросьте `ConfigRepositoryException.Wrap(entityName, "Read", ex)` — обязательно передавая `ex` как `innerException`. Также перехватите `FileNotFoundException` отдельно и оберните с `Operation = "Read"` и сообщением вида `"Файл конфигурации не найден: {path}"`. Никакого `throw ex` — только `throw new ..., ex`.
6. Реализуйте `ConfigService.LoadAllAsync(IReadOnlyList<(string entity, string path)> specs)`: запустите все загрузки через `Task.WhenAll`, но НЕ позволяйте выбросу только первой ошибки. Используйте приём из урока: `await Task.WhenAll(tasks).ContinueWith(_ => { }, TaskContinuationOptions.ExecuteSynchronously);`, затем обойдите все задачи, соберите из каждой `t.IsFaulted` её `t.Exception` (это `AggregateException`), вызовите `Flatten()` и добавьте все `InnerExceptions` в общий список. Если список непустой — выбросьте новый `AggregateException("Не удалось загрузить один или несколько файлов конфигурации", errors)`. Важно: оригинальные `ConfigRepositoryException` должны остаться внутри как внутренние.
7. Реализуйте `ChainWalker.EnumerateChain(Exception? ex)`: итератор (`yield return`), который проходит по `InnerException`, пока не `null`. Добавьте метод `GetRoot(Exception ex)`, возвращающий последний ненулевой элемент цепочки.
8. Реализуйте `ImportOrchestrator.RunOnDispatcher(Func<Task> work)`: имитирует запуск на потоке диспетчера через `Task.Run`. Внутри `try { await work(); } catch (Exception ex) { ExceptionDispatchInfo.Capture(ex).Throw(); throw; }` — это переброс с сохранением стека через границу потока. Подключите `using System.Runtime.ExceptionServices;`.
9. В `Program.cs` создайте три тестовых файла в каталоге `config/`: `database.json` (валидный, `{"connection":"localhost"}`), `cache.json` (отсутствует — намеренный сбой), `endpoints.json` (отсутствует — второй одновременный сбой). Запустите загрузку через `ImportOrchestrator.RunOnDispatcher(() => ConfigService.LoadAllAsync(specs))`.
10. В блоке `catch (AggregateException agg)` в точке входа: вызовите `agg.Handle(ex => ex is ConfigRepositoryException)` — это обработает доменные ошибки, а всё прочее перебросится. Для оставшихся доменных ошибок распечатайте каждую через `ChainWalker.EnumerateChain` и через `ex.ToString()` (чтобы увидеть полную трассировку со всеми `InnerException`).
11. Добавьте отдельный демонстрационный метод `ShowThrowExDanger()`, который сознательно вызывает `throw ex` в одном случае и `throw` в другом, и сравните стеки в выводе — это закрепляет антипаттерн из урока.
12. Запустите проект командой `dotnet run --project ReportsHub.Importer`. Ожидаемый вывод: два сообщения о не найденных `cache.json` и `endpoints.json`, для каждого — цепочка из `ConfigRepositoryException → FileNotFoundException`, и полный дамп через `ToString()`. Убедитесь, что в стеке видна строка реального места сбоя (где `File.OpenRead`), а не строка `throw ex`.

#### Требования к решению

Решение должно быть полностью рабочим на C# 12 / .NET 8: используйте top-level statements, pattern matching (`is`, switch expressions где уместно), nullable-аннотации, `sealed` для пользовательских исключений. Все пользовательские исключения обязаны передавать оригинал в конструктор через `innerException` — ни одного выброса нового исключения без `inner` быть не должно (кроме совсем уж тривиальных `ArgumentNullException(nameof(...))` для предусловий). Категорически запрещено использовать `throw ex` где-либо, кроме демонстрационного метода `ShowThrowExDanger`, где он показан как антипаттерн.

Параллельная загрузка обязана собирать ВСЕ ошибки, а не только первую: проверьте, что при двух упавших файлах в финальный `AggregateException` попадают оба `ConfigRepositoryException`. Используйте `Flatten()` — даже если в данном сценарии вложенности нет, это правильная защитная практика. Логирование должно идти через `ex.ToString()`, а не через `ex.Message` — чтобы видеть стеки и всю цепочку. Для переброса между потоком диспетчера и основным потоком применяйте `ExceptionDispatchInfo.Capture(ex).Throw()`. Имена файлов, сущностей и операций должны фигурировать в контекстных свойствах исключения, а не только в сообщении. Код должен компилироваться без предупреждений и проходить `dotnet build -warnaserror`.

#### Тонкости и подводные камни

- **`throw ex` против `throw` против `throw new ..., ex`.** Это центральная тонкость урока. `throw ex` обнуляет стек вызовов в точке перехвата — вы теряете информацию о том, где реально возник сбой. `throw;` сохраняет оригинальный стек, но не добавляет контекст. `throw new ...(msg, ex)` создаёт новое звено цепочки, сохраняя корень через `InnerException`, и добавляет контекст текущего слоя. В репозитории нужен именно третий вариант. Демонстрационный метод должен наглядно показать разницу в стеках.
- **Не оборачивайте ради оборачивания.** Если новое исключение не добавляет информации — не создавайте его. `try { ... } catch (Exception ex) { throw new Exception("error", ex); }` затирает тип и создаёт шум. Оборачивайте только осмысленным типом и только на границе слоёв. В `FileSource` ничего оборачивать не нужно — пусть `FileNotFoundException` всплывает.
- **`InnerException` (единственное) против `InnerExceptions` (множественное).** У `AggregateException` свойство называется `InnerExceptions` и возвращает `ReadOnlyCollection<Exception>`. У обычного `Exception` — `InnerException` (одно). Не перепутайте при обходе.
- **`AggregateException.Flatten()`.** В параллельных сценариях агрегаты могут быть вложенными (агрегат внутри агрегата). `Flatten()` схлопывает их в один плоский список. Применяйте всегда при сборе ошибок из `Task.Exception`.
- **`AggregateException.Handle(func)`.** Возвращает `void`, но перебрасывает новым `AggregateException` всё, для чего функция вернула `false`. Это декларативная фильтрация. Учтите: если хотя бы один внутренний элемент не обработан, будет выброс — поэтому вызывайте внутри `try/catch(AggregateException)`.
- **`Task.WhenAll` выбрасывает только первое.** Чтобы собрать все, нельзя просто `await Task.WhenAll(tasks)` в `try/catch` — вы получите лишь одну ошибку. Используйте приём с `ContinueWith(_ => {}, ExecuteSynchronously)` и обходом `t.IsFaulted`.
- **`ExceptionDispatchInfo.Capture(ex).Throw()`.** Не выбрасывает `ex` заново, а «восстанавливает» его бросание с оригинальным стеком. Незаменим при маршалинге между потоками. Учтите, что после `.Throw()` инструкция `throw;` после неё формально недостижима, но компилятор требует либо её, либо `return` — добавьте `throw;` для удовлетворения анализатора.
- **`ex.ToString()` вместо `ex.Message`.** `ToString()` автоматически разворачивает всю цепочку `InnerException` вместе со стеками. Логирование только `Message` теряет 90% диагностической информации.
- **Контекстные свойства.** Не ограничивайтесь сообщением. Включайте `EntityName`, `Operation`, идентификаторы — они видны в логах и удобны для фильтрации. Сделайте исключение `sealed` и свойства — `init`/`get`-only.

#### Критерии приёмки

- [ ] Проект компилируется `dotnet build -warnaserror` без ошибок и предупреждений на .NET 8 / C# 12.
- [ ] `ConfigRepositoryException` — `sealed`, наследует `Exception`, имеет `EntityName` и `Operation` и конструктор с обязательным `Exception inner`.
- [ ] При оборачивании в `ConfigRepository` оригинал передаётся как `innerException` (видно в `InnerException` корня).
- [ ] Нигде, кроме `ShowThrowExDanger`, не используется `throw ex` — только `throw;` или `throw new ..., ex`.
- [ ] `ChainWalker.EnumerateChain` корректно обходит цепочку до `null` и возвращает все звенья, включая корень.
- [ ] `ChainWalker.GetRoot` возвращает первопричину (последний ненулевой `InnerException`).
- [ ] `ConfigService.LoadAllAsync` собирает ВСЕ ошибки упавших задач через `Flatten().InnerExceptions`, а не только первую.
- [ ] При двух упавших файлах финальный `AggregateException` содержит ровно два внутренних `ConfigRepositoryException`.
- [ ] `ImportOrchestrator` использует `ExceptionDispatchInfo.Capture(ex).Throw()` для переброса через границу потока.
- [ ] В выводе виден оригинальный стек до строки реального сбоя (`File.OpenRead`), а не до `throw ex`.
- [ ] Логирование идёт через `ex.ToString()` — в выводе присутствует строка `" ---> "` (разделитель `InnerException`).
- [ ] `AggregateException.Handle` применяется для фильтрации доменных ошибок от прочих.
- [ ] `ShowThrowExDanger` наглядно демонстрирует разницу стеков `throw ex` и `throw`.
- [ ] Все nullable-аннотации корректны, нет `null!` заглушек и подавлений `!`.
- [ ] Код запускается `dotnet run` и выводит ожидаемую диагностику для двух отсутствующих файлов.

#### Подсказки (без прямого ответа)

- Вспомните аналогию из урока про病历 медицинскую карту: каждое звено цепочки — заключение специалиста определённого уровня. Подумайте, какой «уровень» представляет каждый ваш класс.
- Для сбора всех ошибок `Task.WhenAll` не пытайтесь ловить исключение в `try/catch` вокруг `await` — перехватите сам `Task`, дождитесь его завершения через `ContinueWith` и проверьте `IsFaulted`.
- Метод `ExceptionDispatchInfo.Capture(ex).Throw()` формально не возвращает управление нормальным путём, но анализатор потока требует `throw;` после него — это нормально.
- Чтобы увидеть разницу `throw ex` и `throw`, посмотрите на первую строку стека: при `throw ex` она указывает на строку `catch`, при `throw` — на реальное место возникновения.
- `ex.ToString()` содержит подстроку `" ---> "` между звеньями цепочки — это удобный маркер для проверки, что цепочка действительно формируется.
- Для `AggregateException.Handle` помните: функция-обработчик получает `Exception` и возвращает `bool`. `true` = «я обработал, не перебрасывай», `false` = «перебрось в новом агрегате».

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — ReportsHub.Importer: InnerException chains + AggregateException.
// Двуязычные комментарии: RU + EN.

using System.Runtime.ExceptionServices;

namespace ReportsHub.Importer;

// Пользовательское исключение слоя репозитория: несёт контекст, сохраняет корень.
// Repository-layer custom exception: carries context, preserves the root.
public sealed class ConfigRepositoryException : Exception
{
    public string? EntityName { get; }      // Имя сущности / Entity name
    public string? Operation  { get; }      // Операция / Operation

    public ConfigRepositoryException(string message, string? entityName, string? operation, Exception inner)
        : base(message, inner)              // inner → InnerException / inner → InnerException
    {
        EntityName = entityName;
        Operation  = operation;
    }

    // Удобная фабрика обёртки. / Convenience wrap factory.
    public static ConfigRepositoryException Wrap(string entity, string operation, Exception inner)
        => new($"Сбой загрузки конфигурации '{entity}' ({operation}) / Config load failure '{entity}' ({operation})",
               entity, operation, inner);
}

// Низкоуровневый слой: только чтение байтов. Ничего не оборачивает.
// Low-level layer: raw byte reads only. Wraps nothing.
public static class FileSource
{
    public static async Task<string> ReadAllTextAsync(string path)
    {
        await using var stream = File.OpenRead(path);   // FileNotFoundException всплывёт естественным путём
        using var reader = new StreamReader(stream);    // / propagates naturally
        return await reader.ReadToEndAsync();
    }
}

// Слой репозитория: оборачивает IOException и FileNotFoundException в доменное исключение.
// Repository layer: wraps IOException and FileNotFoundException into a domain exception.
public static class ConfigRepository
{
    public static async Task<string> LoadAsync(string entityName, string path)
    {
        try
        {
            return await FileSource.ReadAllTextAsync(path);
        }
        catch (FileNotFoundException ex)
        {
            // Ключевой момент урока: передаём ex как innerException — корень не теряется.
            // Key lesson point: pass ex as innerException — root is preserved.
            throw ConfigRepositoryException.Wrap(entityName, "Read",
                new FileNotFoundException($"Файл конфигурации не найден: {path} / Config file not found: {path}", ex));
        }
        catch (IOException ex)
        {
            throw ConfigRepositoryException.Wrap(entityName, "Read", ex);
        }
    }
}

// Обход цепочки до первопричины. / Walk the chain to the root cause.
public static class ChainWalker
{
    public static IEnumerable<Exception> EnumerateChain(Exception? ex)
    {
        while (ex is not null)
        {
            yield return ex;
            ex = ex.InnerException;          // следующее звено / next link
        }
    }

    public static Exception GetRoot(Exception ex)
    {
        var root = ex;
        while (root.InnerException is not null)
            root = root.InnerException;
        return root;
    }
}

// Сервис: параллельная загрузка, сбор ВСЕХ ошибок в AggregateException.
// Service: parallel load, collecting ALL errors into AggregateException.
public static class ConfigService
{
    public static async Task LoadAllAsync(IReadOnlyList<(string Entity, string Path)> specs)
    {
        var tasks = specs.Select(s => ConfigRepository.LoadAsync(s.Entity, s.Path)).ToList();

        // Не выбрасываем сразу: ждём все и собираем. / Don't throw at once: wait and collect.
        await Task.WhenAll(tasks)
                  .ContinueWith(_ => { }, TaskContinuationOptions.ExecuteSynchronously);

        var errors = new List<Exception>();
        foreach (var t in tasks)
        {
            if (t.IsFaulted && t.Exception is AggregateException agg)
            {
                // Flatten схлопывает вложенные агрегаты в плоский список.
                // Flatten collapses nested aggregates into a flat list.
                errors.AddRange(agg.Flatten().InnerExceptions);
            }
        }

        if (errors.Count > 0)
            throw new AggregateException(
                "Не удалось загрузить один или несколько файлов конфигурации / One or more config files failed to load",
                errors);
    }
}

// Переброс через границу потока с сохранением стека. / Rethrow across thread boundary, preserving stack.
public static class ImportOrchestrator
{
    public static async Task RunOnDispatcher(Func<Task> work)
    {
        try
        {
            await Task.Run(work);                 // имитация потока диспетчера / dispatcher-thread simulation
        }
        catch (Exception ex)
        {
            ExceptionDispatchInfo.Capture(ex).Throw(); // стек сохранён / stack preserved
            throw;                                // для анализатора потока / for flow analysis
        }
    }
}

// Демонстрация антипаттерна throw ex. / Anti-pattern throw ex demo.
public static class ThrowDanger
{
    public static void BadThrowEx()
    {
        try { ThrowInner(); }
        catch (Exception ex) { throw ex; }       // СТЕК ОБНУЛЁН тут / STACK RESET here
    }

    public static void GoodThrow()
    {
        try { ThrowInner(); }
        catch { throw; }                          // стек сохранён / stack preserved
    }

    private static void ThrowInner() => throw new InvalidOperationException("Внутренний сбой / Inner failure");
}
```

**Разбор по строкам.** `ConfigRepositoryException` — `sealed` с контекстными свойствами `EntityName` и `Operation`; конструктор вызывает `base(message, inner)`, что и устанавливает `InnerException` — это центральный механизм урока. Статическая фабрика `Wrap` инкапсулирует типовой сценарий оборачивания на границе слоёв и гарантирует, что `inner` всегда передаётся. `FileSource.ReadAllTextAsync` намеренно ничего не ловит: `FileNotFoundException` всплывает «голым», что соответствует принципу «не оборачивай, если не добавляешь контекста». В `ConfigRepository.LoadAsync` два `catch` — для `FileNotFoundException` и `IOException` — оба выбрасывают `ConfigRepositoryException.Wrap(..., ex)`, передавая оригинал как `innerException`; в случае отсутствующего файла мы создаём новое `FileNotFoundException` с уточняющим сообщением, но всё равно вкладываем оригинал `ex`, чтобы не потерять его стек.

`ChainWalker.EnumerateChain` — итератор с `yield return`, проходящий по `InnerException` до `null`; `GetRoot` возвращает последнее звено. Это прямая реализация «обхода цепочки до первопричины» из урока. `ConfigService.LoadAllAsync` использует приём `ContinueWith(_ => {}, ExecuteSynchronously)`, чтобы дождаться всех задач без немедленного выброса, затем обходит `t.IsFaulted` и через `agg.Flatten().InnerExceptions` собирает все ошибки. Это решает тонкость «`Task.WhenAll` выбрасывает только первое». Если ошибки есть — выбрасывается новый `AggregateException` с доменными исключениями внутри.

`ImportOrchestrator.RunOnDispatcher` применяет `ExceptionDispatchInfo.Capture(ex).Throw()` — это переброс с сохранением оригинального стека через границу потока, ровно как рекомендует урок. Завершающий `throw;` нужен анализатору потока, поскольку `.Throw()` технически не помечен как не возвращающий управление. `ThrowDanger` контрастно показывает `throw ex` (стек обнулён в точке `catch`) и `throw` (стек сохранён) — это наглядная демонстрация главного антипаттерна. Во всём решении нет ни одного выброса нового исключения без `innerException` (кроме тривиальных предусловий), что соответствует лучшим практикам урока.

#### Задания на углубление (бонус)

1. **Дерево причин.** Расширьте `ConfigRepositoryException` поддержкой нескольких одновременных причин: добавьте конструктор, принимающий `IEnumerable<Exception>` и формирующий `AggregateException` как `InnerException`. Реализуйте метод `EnumerateDeep`, который рекурсивно обходит и `InnerException`, и `InnerExceptions` у `AggregateException`, строя плоское дерево.
2. **Декларативная фильтрация политик.** Реализуйте класс `ExceptionPolicy` с методом `Handle(AggregateException agg)`, который через `Handle(func)` обрабатывает `ConfigRepositoryException` (логирует и глотает), перебрасывает `IOException` как есть, а всё прочее оборачивает в `FatalImportException`. Покройте юнит-тестами.
3. **Сериализация цепочки.** Сделайте `ConfigRepositoryException` сериализуемым (`[Serializable]`, конструктор `protected(SerializationInfo, StreamingContext)`), запишите в лог через `JsonSerializer` структуру цепочки (тип, сообщение, стек каждого звена), и проверьте round-trip.
4. **Бенчмарк `throw` vs `throw ex` vs `EDI`.** С помощью `BenchmarkDotNet` измерьте накладные расходы трёх способов переброса на 100 000 итераций. Объясните, почему `ExceptionDispatchInfo` медленнее `throw`, но почему его всё равно используют.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are developing the configuration-import module for an enterprise service called "ReportsHub". The service configuration is split across several JSON files: `database.json`, `cache.json`, `endpoints.json`. Each file is read in parallel, parsed, and validated. The architecture is strictly layered: the `FileSource` layer is responsible only for reading bytes from disk; the `ConfigRepository` layer turns raw I/O errors into the domain `ConfigRepositoryException` carrying the entity name; the `ConfigService` layer coordinates loading all files and, when multiple failures occur, aggregates them into an `AggregateException`; finally, the `ImportOrchestrator` rethrows the aggregated error onto the dispatcher thread while preserving the original stacks.

The business requires that, when import fails, the operator sees not "something broke" but the full picture: which file, which layer, what root cause — and that the log captures the entire stack down to the line where `FileStream` actually failed. At the same time, no error may be lost: if two of three files failed, both failures must reach the report, not just the first one. And because reading happens in parallel tasks while handling happens on the dispatcher thread, you must correctly marshal the exception across the thread boundary without resetting the stack. This is the canonical scenario where the `InnerException` and `AggregateException` mechanisms from lesson M07-L06 are not merely helpful but are the only correct way to preserve the diagnostic value of failures.

#### What to do step by step

1. Create a .NET 8 console project: run `dotnet new console -n ReportsHub.Importer -o ReportsHub.Importer` inside the `modules/M07/homework` directory. Verify that `ReportsHub.Importer.csproj` contains `<TargetFramework>net8.0</TargetFramework>`, and add `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`, `<LangVersion>latest</LangVersion>`.
2. Keep top-level statements in `Program.cs`. Create a `Source/` folder with files `FileSource.cs`, `ConfigRepositoryException.cs`, `ConfigRepository.cs`, `ConfigService.cs`, `ImportOrchestrator.cs`, `ChainWalker.cs`; the entry point stays at the project root.
3. Implement `ConfigRepositoryException` — a `sealed` custom exception derived from `Exception` with contextual properties `EntityName` (the file/entity name) and `Operation` (e.g. `"Read"`, `"Parse"`). The constructor must accept `string message`, `string? entityName`, `string? operation`, and a mandatory `Exception inner`, calling `base(message, inner)`. Add a static factory `Wrap(string entity, string operation, Exception inner)` for convenience.
4. Implement `FileSource.ReadAllTextAsync(string path)`: opens the file with `File.OpenRead`, reads to the end. If the file does not exist, `FileNotFoundException` is raised naturally — do NOT catch it here, let it propagate.
5. Implement `ConfigRepository.LoadAsync(string entityName, string path)`: calls `FileSource.ReadAllTextAsync` inside `try/catch (IOException ex)`. In the `catch`, throw `ConfigRepositoryException.Wrap(entityName, "Read", ex)` — always passing `ex` as `innerException`. Also catch `FileNotFoundException` separately and wrap it with `Operation = "Read"` and a message like `"Config file not found: {path}"`. No `throw ex` anywhere — only `throw new ..., ex`.
6. Implement `ConfigService.LoadAllAsync(IReadOnlyList<(string entity, string path)> specs)`: launch all loads through `Task.WhenAll`, but do NOT allow only the first error to be thrown. Use the lesson's technique: `await Task.WhenAll(tasks).ContinueWith(_ => { }, TaskContinuationOptions.ExecuteSynchronously);`, then iterate every task, collect from each `t.IsFaulted` its `t.Exception` (an `AggregateException`), call `Flatten()`, and add all `InnerExceptions` to a shared list. If the list is non-empty, throw a new `AggregateException("One or more config files failed to load", errors)`. Important: the original `ConfigRepositoryException` instances must remain inside as inner exceptions.
7. Implement `ChainWalker.EnumerateChain(Exception? ex)`: an iterator (`yield return`) that walks `InnerException` until `null`. Add a `GetRoot(Exception ex)` method returning the last non-null element of the chain.
8. Implement `ImportOrchestrator.RunOnDispatcher(Func<Task> work)`: simulate running on a dispatcher thread via `Task.Run`. Inside, `try { await work(); } catch (Exception ex) { ExceptionDispatchInfo.Capture(ex).Throw(); throw; }` — this rethrows with the original stack across the thread boundary. Add `using System.Runtime.ExceptionServices;`.
9. In `Program.cs`, create three test files under `config/`: `database.json` (valid, `{"connection":"localhost"}`), `cache.json` (missing — intentional failure), `endpoints.json` (missing — a second simultaneous failure). Run the load through `ImportOrchestrator.RunOnDispatcher(() => ConfigService.LoadAllAsync(specs))`.
10. In the `catch (AggregateException agg)` at the entry point: call `agg.Handle(ex => ex is ConfigRepositoryException)` — this handles domain errors, while anything else is rethrown. For the remaining domain errors, print each via `ChainWalker.EnumerateChain` and via `ex.ToString()` (to see the full trace with all `InnerException` links).
11. Add a separate demo method `ShowThrowExDanger()` that deliberately uses `throw ex` in one branch and `throw` in another, and compares the stacks in the output — this cements the lesson's anti-pattern.
12. Run the project with `dotnet run --project ReportsHub.Importer`. Expected output: two messages about missing `cache.json` and `endpoints.json`, each with a chain of `ConfigRepositoryException → FileNotFoundException`, plus a full dump via `ToString()`. Verify that the stack shows the real failure line (where `File.OpenRead` is) and not the `throw ex` line.

#### Requirements

The solution must be fully working on C# 12 / .NET 8: use top-level statements, pattern matching (`is`, switch expressions where appropriate), nullable annotations, and `sealed` for custom exceptions. Every custom exception must pass the original to the constructor via `innerException` — there must be no throw of a new exception without `inner`, except for trivial `ArgumentNullException(nameof(...))` precondition checks. Using `throw ex` anywhere is strictly forbidden except inside the demo method `ShowThrowExDanger`, where it is shown as an anti-pattern.

The parallel load must collect ALL errors, not only the first: verify that when two files fail, both `ConfigRepositoryException` instances land in the final `AggregateException`. Use `Flatten()` — even if there is no nesting in this scenario, it is the correct defensive practice. Logging must go through `ex.ToString()`, not `ex.Message`, so stacks and the whole chain are visible. For rethrowing between the dispatcher thread and the main thread, use `ExceptionDispatchInfo.Capture(ex).Throw()`. File, entity, and operation names must appear in the exception's contextual properties, not just in the message. The code must compile without warnings and pass `dotnet build -warnaserror`.

#### Pitfalls

- **`throw ex` vs `throw` vs `throw new ..., ex`.** This is the lesson's central pitfall. `throw ex` resets the call stack at the catch point — you lose where the failure actually originated. `throw;` keeps the original stack but adds no context. `throw new ...(msg, ex)` creates a new chain link, preserving the root through `InnerException` and adding the current layer's context. The repository needs the third form. The demo method should make the stack difference visible.
- **Don't wrap for the sake of wrapping.** If the new exception adds no information, do not create it. `try { ... } catch (Exception ex) { throw new Exception("error", ex); }` erases the type and adds noise. Wrap only with a meaningful type and only at layer boundaries. In `FileSource`, wrap nothing — let `FileNotFoundException` propagate.
- **`InnerException` (singular) vs `InnerExceptions` (plural).** `AggregateException` exposes `InnerExceptions` returning `ReadOnlyCollection<Exception>`. A plain `Exception` exposes `InnerException` (single). Do not confuse them when iterating.
- **`AggregateException.Flatten()`.** In parallel scenarios aggregates can nest (an aggregate inside an aggregate). `Flatten()` collapses them into one flat list. Always apply it when collecting errors from `Task.Exception`.
- **`AggregateException.Handle(func)`.** Returns `void`, but rethrows a new `AggregateException` containing everything for which the function returned `false`. This is declarative filtering. Note: if at least one inner element is unhandled, it throws — so call it inside `try/catch(AggregateException)`.
- **`Task.WhenAll` throws only the first.** To collect all, you cannot simply `await Task.WhenAll(tasks)` in `try/catch` — you get only one error. Use the `ContinueWith(_ => {}, ExecuteSynchronously)` trick and iterate `t.IsFaulted`.
- **`ExceptionDispatchInfo.Capture(ex).Throw()`.** It does not throw `ex` anew — it "restores" its throw with the original stack. Indispensable when marshaling across threads. Note that after `.Throw()` the following `throw;` is formally unreachable, but the analyzer requires either it or a `return` — add `throw;` to satisfy it.
- **`ex.ToString()` over `ex.Message`.** `ToString()` automatically expands the whole `InnerException` chain with stack traces. Logging only `Message` loses 90% of the diagnostic information.
- **Contextual properties.** Do not limit yourself to the message. Include `EntityName`, `Operation`, identifiers — they are visible in logs and convenient for filtering. Make the exception `sealed` and the properties `init`/get-only.

#### Acceptance criteria

- [ ] The project compiles with `dotnet build -warnaserror` with no errors or warnings on .NET 8 / C# 12.
- [ ] `ConfigRepositoryException` is `sealed`, inherits `Exception`, has `EntityName` and `Operation`, and a constructor with a mandatory `Exception inner`.
- [ ] When wrapping in `ConfigRepository`, the original is passed as `innerException` (visible in the root's `InnerException`).
- [ ] Nowhere except `ShowThrowExDanger` is `throw ex` used — only `throw;` or `throw new ..., ex`.
- [ ] `ChainWalker.EnumerateChain` correctly walks the chain to `null` and returns all links including the root.
- [ ] `ChainWalker.GetRoot` returns the root cause (the last non-null `InnerException`).
- [ ] `ConfigService.LoadAllAsync` collects ALL errors from faulted tasks via `Flatten().InnerExceptions`, not only the first.
- [ ] With two failed files, the final `AggregateException` contains exactly two inner `ConfigRepositoryException` instances.
- [ ] `ImportOrchestrator` uses `ExceptionDispatchInfo.Capture(ex).Throw()` to rethrow across the thread boundary.
- [ ] The output shows the original stack down to the real failure line (`File.OpenRead`), not the `throw ex` line.
- [ ] Logging goes through `ex.ToString()` — the output contains the `" ---> "` separator (the `InnerException` delimiter).
- [ ] `AggregateException.Handle` is applied to filter domain errors from the rest.
- [ ] `ShowThrowExDanger` clearly demonstrates the stack difference between `throw ex` and `throw`.
- [ ] All nullable annotations are correct, with no `null!` stubs and no `!` suppressions.
- [ ] The code runs with `dotnet run` and prints the expected diagnostics for the two missing files.

#### Hints

- Recall the lesson's medical-record analogy: each chain link is a specialist's conclusion at a given level. Think about which "level" each of your classes represents.
- To collect all `Task.WhenAll` errors, do not try to catch the exception in a `try/catch` around `await` — capture the `Task` itself, await its completion via `ContinueWith`, and inspect `IsFaulted`.
- `ExceptionDispatchInfo.Capture(ex).Throw()` formally does not return normally, but the flow analyzer requires `throw;` after it — that is expected.
- To see the `throw ex` vs `throw` difference, look at the first stack line: with `throw ex` it points to the `catch` line; with `throw` it points to the real origin.
- `ex.ToString()` contains the substring `" ---> "` between chain links — a handy marker to verify the chain is actually built.
- For `AggregateException.Handle`, remember: the handler takes an `Exception` and returns `bool`. `true` means "I handled it, don't rethrow"; `false` means "rethrow in a new aggregate".

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — ReportsHub.Importer: InnerException chains + AggregateException.
// Bilingual comments: RU + EN.

using System.Runtime.ExceptionServices;

namespace ReportsHub.Importer;

// Repository-layer custom exception: carries context, preserves the root.
// Пользовательское исключение слоя репозитория: несёт контекст, сохраняет корень.
public sealed class ConfigRepositoryException : Exception
{
    public string? EntityName { get; }      // Entity name / Имя сущности
    public string? Operation  { get; }      // Operation / Операция

    public ConfigRepositoryException(string message, string? entityName, string? operation, Exception inner)
        : base(message, inner)              // inner → InnerException / inner → InnerException
    {
        EntityName = entityName;
        Operation  = operation;
    }

    // Convenience wrap factory. / Удобная фабрика обёртки.
    public static ConfigRepositoryException Wrap(string entity, string operation, Exception inner)
        => new($"Config load failure '{entity}' ({operation}) / Сбой загрузки конфигурации '{entity}' ({operation})",
               entity, operation, inner);
}

// Low-level layer: raw byte reads only. Wraps nothing.
// Низкоуровневый слой: только чтение байтов. Ничего не оборачивает.
public static class FileSource
{
    public static async Task<string> ReadAllTextAsync(string path)
    {
        await using var stream = File.OpenRead(path);   // FileNotFoundException propagates naturally
        using var reader = new StreamReader(stream);    // / всплывает естественным путём
        return await reader.ReadToEndAsync();
    }
}

// Repository layer: wraps IOException and FileNotFoundException into a domain exception.
// Слой репозитория: оборачивает IOException и FileNotFoundException в доменное исключение.
public static class ConfigRepository
{
    public static async Task<string> LoadAsync(string entityName, string path)
    {
        try
        {
            return await FileSource.ReadAllTextAsync(path);
        }
        catch (FileNotFoundException ex)
        {
            // Key lesson point: pass ex as innerException — root is preserved.
            // Ключевой момент урока: передаём ex как innerException — корень не теряется.
            throw ConfigRepositoryException.Wrap(entityName, "Read",
                new FileNotFoundException($"Config file not found: {path} / Файл конфигурации не найден: {path}", ex));
        }
        catch (IOException ex)
        {
            throw ConfigRepositoryException.Wrap(entityName, "Read", ex);
        }
    }
}

// Walk the chain to the root cause. / Обход цепочки до первопричины.
public static class ChainWalker
{
    public static IEnumerable<Exception> EnumerateChain(Exception? ex)
    {
        while (ex is not null)
        {
            yield return ex;
            ex = ex.InnerException;          // next link / следующее звено
        }
    }

    public static Exception GetRoot(Exception ex)
    {
        var root = ex;
        while (root.InnerException is not null)
            root = root.InnerException;
        return root;
    }
}

// Service: parallel load, collecting ALL errors into AggregateException.
// Сервис: параллельная загрузка, сбор ВСЕХ ошибок в AggregateException.
public static class ConfigService
{
    public static async Task LoadAllAsync(IReadOnlyList<(string Entity, string Path)> specs)
    {
        var tasks = specs.Select(s => ConfigRepository.LoadAsync(s.Entity, s.Path)).ToList();

        // Don't throw at once: wait and collect. / Не выбрасываем сразу: ждём все и собираем.
        await Task.WhenAll(tasks)
                  .ContinueWith(_ => { }, TaskContinuationOptions.ExecuteSynchronously);

        var errors = new List<Exception>();
        foreach (var t in tasks)
        {
            if (t.IsFaulted && t.Exception is AggregateException agg)
            {
                // Flatten collapses nested aggregates into a flat list.
                // Flatten схлопывает вложенные агрегаты в плоский список.
                errors.AddRange(agg.Flatten().InnerExceptions);
            }
        }

        if (errors.Count > 0)
            throw new AggregateException(
                "One or more config files failed to load / Не удалось загрузить один или несколько файлов конфигурации",
                errors);
    }
}

// Rethrow across thread boundary, preserving stack. / Переброс через границу потока с сохранением стека.
public static class ImportOrchestrator
{
    public static async Task RunOnDispatcher(Func<Task> work)
    {
        try
        {
            await Task.Run(work);                 // dispatcher-thread simulation / имитация потока диспетчера
        }
        catch (Exception ex)
        {
            ExceptionDispatchInfo.Capture(ex).Throw(); // stack preserved / стек сохранён
            throw;                                // for flow analysis / для анализатора потока
        }
    }
}

// Anti-pattern throw ex demo. / Демонстрация антипаттерна throw ex.
public static class ThrowDanger
{
    public static void BadThrowEx()
    {
        try { ThrowInner(); }
        catch (Exception ex) { throw ex; }       // STACK RESET here / СТЕК ОБНУЛЁН тут
    }

    public static void GoodThrow()
    {
        try { ThrowInner(); }
        catch { throw; }                          // stack preserved / стек сохранён
    }

    private static void ThrowInner() => throw new InvalidOperationException("Inner failure / Внутренний сбой");
}
```

**Line-by-line walk-through.** `ConfigRepositoryException` is `sealed` with contextual `EntityName` and `Operation` properties; its constructor calls `base(message, inner)`, which is exactly what sets `InnerException` — the lesson's central mechanism. The static `Wrap` factory encapsulates the typical wrap-at-boundary scenario and guarantees `inner` is always passed. `FileSource.ReadAllTextAsync` deliberately catches nothing: `FileNotFoundException` propagates "bare", matching the "don't wrap when you add no context" principle. In `ConfigRepository.LoadAsync`, two `catch` blocks — for `FileNotFoundException` and `IOException` — both throw `ConfigRepositoryException.Wrap(..., ex)`, passing the original as `innerException`; for the missing-file case we create a new `FileNotFoundException` with a clearer message but still embed the original `ex` so its stack is not lost.

`ChainWalker.EnumerateChain` is a `yield return` iterator walking `InnerException` until `null`; `GetRoot` returns the last link. This is a direct implementation of the lesson's "walk the chain to the root cause". `ConfigService.LoadAllAsync` uses the `ContinueWith(_ => {}, ExecuteSynchronously)` trick to await all tasks without throwing immediately, then iterates `t.IsFaulted` and collects via `agg.Flatten().InnerExceptions`. This solves the "`Task.WhenAll` throws only the first" pitfall. If there are errors, a new `AggregateException` is thrown with the domain exceptions inside.

`ImportOrchestrator.RunOnDispatcher` applies `ExceptionDispatchInfo.Capture(ex).Throw()` — a rethrow that preserves the original stack across the thread boundary, exactly as the lesson recommends. The trailing `throw;` is needed by the flow analyzer because `.Throw()` is not technically marked as non-returning. `ThrowDanger` contrasts `throw ex` (stack reset at the catch) and `throw` (stack preserved) — a vivid demonstration of the main anti-pattern. Across the whole solution, there is no throw of a new exception without `innerException` (except trivial preconditions), matching the lesson's best practices.

#### Going deeper (bonus)

1. **Causality tree.** Extend `ConfigRepositoryException` to support multiple simultaneous causes: add a constructor taking `IEnumerable<Exception>` and forming an `AggregateException` as the `InnerException`. Implement `EnumerateDeep` that recursively walks both `InnerException` and the `InnerExceptions` of any `AggregateException`, building a flat tree.
2. **Declarative policy filtering.** Implement an `ExceptionPolicy` class with a `Handle(AggregateException agg)` method that, via `Handle(func)`, handles `ConfigRepositoryException` (logs and swallows), rethrows `IOException` as-is, and wraps anything else in a `FatalImportException`. Cover it with unit tests.
3. **Chain serialization.** Make `ConfigRepositoryException` serializable (`[Serializable]`, a `protected(SerializationInfo, StreamingContext)` constructor), log the chain structure (type, message, stack of each link) through `JsonSerializer`, and verify round-trip.
4. **`throw` vs `throw ex` vs `EDI` benchmark.** With `BenchmarkDotNet`, measure the overhead of the three rethrow forms over 100,000 iterations. Explain why `ExceptionDispatchInfo` is slower than `throw` but is still used.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект собирается `dotnet build -warnaserror` без ошибок и предупреждений.
- [ ] `ConfigRepositoryException` — `sealed`, с контекстными свойствами и обязательным `inner`.
- [ ] Нигде кроме `ShowThrowExDanger` нет `throw ex`.
- [ ] `ConfigService.LoadAllAsync` собирает все ошибки через `Flatten().InnerExceptions`.
- [ ] `ImportOrchestrator` использует `ExceptionDispatchInfo.Capture(ex).Throw()`.
- [ ] Логирование через `ex.ToString()` (виден `" ---> "`).
- [ ] `dotnet run` выводит диагностику для двух отсутствующих файлов.
- [ ] The project builds with `dotnet build -warnaserror` cleanly.
- [ ] `ConfigRepositoryException` is `sealed` with contextual props and a mandatory `inner`.
- [ ] No `throw ex` anywhere outside `ShowThrowExDanger`.
- [ ] `ConfigService.LoadAllAsync` collects all errors via `Flatten().InnerExceptions`.
- [ ] `ImportOrchestrator` uses `ExceptionDispatchInfo.Capture(ex).Throw()`.
- [ ] Logging via `ex.ToString()` (the `" ---> "` marker is visible).
- [ ] `dotnet run` prints the diagnostics for the two missing files.

#### Ресурсы / Resources
- [Microsoft Learn — Exception.InnerException](https://learn.microsoft.com/dotnet/api/system.exception.innerexception)
- [Microsoft Learn — AggregateException](https://learn.microsoft.com/dotnet/api/system.aggregateexception)
- [Microsoft Learn — AggregateException.Flatten](https://learn.microsoft.com/dotnet/api/system.aggregateexception.flatten)
- [Microsoft Learn — AggregateException.Handle](https://learn.microsoft.com/dotnet/api/system.aggregateexception.handle)
- [Microsoft Learn — ExceptionDispatchInfo](https://learn.microsoft.com/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo)
- [Microsoft Learn — Best practices for exceptions](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions)
- [Урок M07-L06 / Lesson M07-L06](lesson-M07-L06-inner-exception.md)
