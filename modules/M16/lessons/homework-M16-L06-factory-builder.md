---
[← К уроку M16-L06](lesson-M16-L06-factory-builder.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →](homework-M16-L07-strategy-decorator-adapter.md)
---

### Домашнее задание M16-L06: Factory, Abstract Factory, Builder / Homework M16-L06: Factory, Abstract Factory, Builder

**Урок / Lesson:** M16-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять три порождающих паттерна — Factory Method, Abstract Factory и Fluent Builder — в реальном проекте на C# 12 / .NET 8, используя первичные конструкторы, рекорды и init-свойства, и закрепить умение выбирать правильный паттерн по числу продуктов и сложности объекта. (EN) Learn to apply three creational patterns — Factory Method, Abstract Factory, and Fluent Builder — in a real C# 12 / .NET 8 project, using primary constructors, records, and init-only properties, and reinforce the ability to choose the right pattern based on the number of products and object complexity.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую опирается на код и best practices урока M16-L06: первичные конструкторы для фабрик и билдеров, возвращение интерфейсов из фабрик, неизменяемые результаты сборки через `record` + `init`, валидация в `Build()`. Вы будете строить мини-систему отчётов, где каждый из трёх паттернов решает свою задачу — так же, как в демонстрации урока, но с большим объёмом и реальными ограничениями.
(EN) The homework builds directly on the code and best practices of lesson M16-L06: primary constructors for factories and builders, returning interfaces from factories, immutable build results via `record` + `init`, validation in `Build()`. You will build a mini reporting system where each of the three patterns solves its own task — just like the lesson demo, but with more volume and real constraints.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы работаете в команде, которая разрабатывает внутренний сервис аналитики «AcmeReports». Сервис должен принимать сырые события из разных источников (база данных, REST API, локальный CSV-файл), преобразовывать их в единый доменный объект `ReportRequest`, а затем формировать итоговый отчёт в одном из нескольких форматов: PDF, HTML, Markdown или Excel-совместимый CSV. Каждый формат требует своего «движка» рендеринга со своим набором согласованных компонентов: заголовок, таблица данных, нижний колонтитул. При этом источник данных выбирается по конфигурации окружения (dev/prod), а формат отчёта — по запросу пользователя.

Эта задача идеально ложится на три порождающих паттерна из урока. Источник данных — это полиморфное создание одного продукта (`IReportSource`), которое удобно реализовать через Factory Method: каждый подкласс-создатель решает, какой конкретный источник вернуть. Формат отчёта — это целое семейство согласованных компонентов (`IReportHeader`, `IReportTable`, `IReportFooter`), и здесь нужен Abstract Factory, чтобы клиент получил гарантированно совместимый набор для выбранного формата. Наконец, сам `ReportRequest` имеет много необязательных полей (подзаголовок, автор, теги, период, фильтры, параметры пагинации), и его удобнее собирать через Fluent Builder, чем передавать через бог-конструктор.

Мотивация не академическая: на практике прямой `new` для каждого формата и источника быстро приводит к разбросанным `switch` и `if`, которые растут при добавлении новых вариантов. Порождающие паттерны изолируют клиента от конкретных типов, делают расширение безопасным (новый формат = новая фабрика, старый код не трогается) и упрощают тестирование через подмену фабрик в DI. В этом задании вы построите именно такую расширяемую систему и убедитесь, что добавление нового формата или источника требует только нового класса, а не правок в клиентском коде.

#### Что нужно сделать (пошагово)
1. Создайте новый проект консольного приложения на .NET 8 через `dotnet new console -n AcmeReports -o AcmeReports --framework net8.0`. Перейдите в каталог `cd AcmeReports` и убедитесь, что `dotnet build` проходит без ошибок. Откройте `Program.cs` и удалите шаблонный код — вы будете писать top-level statements.
2. В каталоге проекта создайте подпапки `Sources`, `Formats`, `Builders` и `Abstractions`. Разделение по папкам не случайно: оно отражает слои домена и упрощает навигацию, когда типов станет много.
3. В `Abstractions` опишите интерфейсы продуктов. Сначала контракт источника данных: `IReportSource` с методом `Task<IReadOnlyList<ReportRow>> ReadAsync(CancellationToken ct)`, где `ReportRow` — рекорд с полями `Category`, `Metric`, `Value`. Затем контракт компонентов отчёта: `IReportHeader` с `string Render(ReportRequest req)`, `IReportTable` с `string Render(IReadOnlyList<ReportRow> rows)`, `IReportFooter` с `string Render(DateTime generatedAt)`.
4. В `Builders` реализуйте `ReportRequest` как неизменяемый `record` с `init`-свойствами и обязательным `Title`, и `ReportBuilder` — изменяемый класс с fluent-методами `WithTitle`, `WithSubtitle`, `WithAuthor`, `WithTags`, `WithPeriod`, `WithFilters`, `WithPageSize`, `WithFormat`. Каждый метод возвращает `this`. Метод `Build()` возвращает `ReportRequest` и валидирует обязательные поля (Title, Format), выбрасывая `InvalidOperationException` с понятным сообщением на русском и английском.
5. В `Sources` реализуйте три конкретных источника: `DatabaseSource` (имитирует чтение из БД, возвращает тестовые строки), `RestApiSource` (имитирует HTTP-вызов с `Task.Delay`), `CsvFileSource` (читает встроенную строку CSV). Затем реализуйте фабричный метод: абстрактный класс `ReportSourceFactory` с primary ctor `ReportSourceFactory(SourceConfig config)` и абстрактным методом `IReportSource Create()`, плюс конкретные `DatabaseSourceFactory`, `RestApiSourceFactory`, `CsvFileSourceFactory`.
6. В `Formats` реализуйте Abstract Factory. Опишите интерфейс `IReportFactory` с методами `CreateHeader()`, `CreateTable()`, `CreateFooter()`. Сделайте две конкретные фабрики: `PdfReportFactory` и `HtmlReportFactory`, каждая из которых возвращает согласованное семейство компонентов (например, для PDF — `PdfHeader`, `PdfTable`, `PdfFooter`; для HTML — аналогичные с HTML-разметкой). Убедитесь, что компоненты разных фабрик не смешиваются: клиент получает только одну фабрику и только её продукты.
7. В `Program.cs` соберите демо через top-level statements. Постройте `ReportRequest` через билдер, выберите источник через фабричный метод по аргументу командной строки (например, `dotnet run --source db --format pdf`), выберите формат-фабрику, прочитайте данные, отрендерите заголовок, таблицу, футер и выведите результат в консоль. Используйте `CancellationToken` и `await`.
8. Зарегистрируйте фабрики в `Microsoft.Extensions.DependencyInjection`: создайте `ServiceCollection`, зарегистрируйте `IReportFactory` по ключу формата (можно через фабричную функцию `Func<string, IReportFactory>`), а `ReportSourceFactory` — по строке источника. Это покажет, как паттерны сочетаются с DI и избавляют клиента от знания конкретных типов.
9. Добавьте unit-тесты через `dotnet new xunit -n AcmeReports.Tests -o AcmeReports.Tests --framework net8.0` и ссылку на основной проект `dotnet add reference ../AcmeReports/AcmeReports.csproj`. Напишите тесты: билдер выбрасывает исключение без Title; `Build()` корректно заполняет поля; PdfReportFactory возвращает именно `PdfHeader`; смешивание компонентов двух фабрик невозможно на уровне типов (через интерфейсы).
10. Запустите `dotnet test` и убедитесь, что все тесты зелёные. Затем запустите `dotnet run -- --source db --format html` и проверьте, что вывод содержит HTML-разметку и корректный футер с датой. Зафиксируйте вывод в файл `sample-output.txt` через перенаправление `dotnet run -- --source db --format pdf > sample-output.txt`.

#### Требования к решению
Решение должно компилироваться под .NET 8 без предупреждений (включая nullable-анализ) и проходить все тесты. Используйте C# 12: первичные конструкторы для классов фабрик и билдеров, `record` для неизменяемых результатов, `init`-свойства, collection expressions для инициализации списков (`[row1, row2]`), pattern matching для выбора фабрики по строке формата (`switch` expression), raw string literals для шаблонов разметки там, где есть кавычки и фигурные скобки. Все фабрики должны возвращать интерфейсы (`IReportSource`, `IReportFactory`, `IReportHeader` и т. д.), а не конкретные типы — клиентский код зависит только от абстракций.

Структура проекта должна быть чистой: каждый класс в отдельном файле, имена файлов соответствуют именам типов, папки отражают слои. Код должен быть готов к расширению: добавление нового формата (например, Markdown) или нового источника (например, Kafka) не должно требовать правок в существующих классах — только добавление нового класса-фабрики и регистрация в DI. Обязательна валидация в `Build()`: отсутствие `Title` или `Format` выбрасывает исключение с двуязычным сообщением. Источники данных должны быть асинхронными (`Task`-based) и принимать `CancellationToken`. В тестах используйте xUnit и, при необходимости, простые stub-реализации интерфейсов (без Moq, чтобы не усложнять зависимости) — это допустимо, потому что паттерны как раз упрощают подмену.

#### Тонкости и подводные камни
Главная ловушка — перепутать паттерны. Если у вас один продукт (источник данных) — это Factory Method, не Abstract Factory; если целое семейство согласованных компонентов (заголовок + таблица + футер) — это Abstract Factory, и попытка собрать семейство из нескольких фабричных методов разной природы приведёт к рассогласованию. Урок явно предостерегает: «Abstract Factory для одного продукта вместо Factory Method → выбирайте по числу продуктов». Следите за этим разделением в коде.

Вторая тонкость — изменяемость. Билдер по своей природе изменяем (он накапливает состояние), но результат `Build()` должен быть неизменяемым. Не возвращайте из `Build()` тот же объект, что и билдер, и не давайте сеттерам менять уже собранный `ReportRequest`. Используйте `record` с `init`-свойствами, как в уроке с `ReportConfig`. Если клиент попытается изменить поле после сборки — компилятор не даст (`init` доступен только при инициализации). Третья тонкость — валидация. Урок требует проверять обязательные поля в `Build()`, а не в каждом fluent-методе: это концентрирует проверки и даёт понятное исключение в одной точке. Не полагайтесь на «автоматически правильно» от первичного конструктора — он не валидирует.

Четвёртая тонкость — первичные конструкторы. Они убирают boilerplate, но помните, что параметры primary ctor неявно становятся полями и доступны во всём классе. Для фабрик это удобно (`IUiFactory factory` доступен в `RenderAll`), но не злоупотребляйте: если параметр нужно валидировать, делайте это явно в фабричном методе или конструкторе. Пятая тонкость — DI и конкретные типы. Клиент не должен зависеть от `PdfReportFactory` напрямую; регистрируйте по интерфейсу `IReportFactory` и выбирайте реализацию по ключу формата через `Func<string, IReportFactory>`. Шестая тонкость — nullable: помечайте необязательные поля как `string?`, обязательные — как `required` или валидируйте в `Build()`. И, наконец, не заводите фабрику ради одного класса: если в текущей итерации нужен только один источник, оставьте прямой `new` или статический метод, пока полиморфизм реально понадобится.

#### Критерии приёмки
- [ ] Проект `AcmeReports` создан под .NET 8, `dotnet build` проходит без ошибок и предупреждений.
- [ ] Структура каталогов: `Abstractions`, `Sources`, `Formats`, `Builders`.
- [ ] `IReportSource` и три компонента `IReportHeader`/`IReportTable`/`IReportFooter` объявлены как интерфейсы в `Abstractions`.
- [ ] `ReportRequest` — неизменяемый `record` с `init`-свойствами и обязательным `Title`.
- [ ] `ReportBuilder` — изменяемый класс с fluent-методами, каждый возвращает `this`.
- [ ] `Build()` валидирует обязательные поля и выбрасывает `InvalidOperationException` с двуязычным сообщением.
- [ ] `ReportSourceFactory` — абстрактный класс с primary ctor и абстрактным `Create()`, возвращающим `IReportSource`.
- [ ] Минимум три конкретных источника и три конкретные фабрики источников.
- [ ] `IReportFactory` (Abstract Factory) объявлен с методами `CreateHeader/CreateTable/CreateFooter`.
- [ ] Минимум две конкретные фабрики форматов (Pdf, Html), каждая возвращает согласованное семейство.
- [ ] Использованы C# 12: первичные конструкторы, collection expressions, pattern matching (switch expression), raw string literals.
- [ ] Источники асинхронны и принимают `CancellationToken`.
- [ ] Фабрики зарегистрированы в `Microsoft.Extensions.DependencyInjection`, клиент не знает конкретных типов.
- [ ] `Program.cs` — top-level statements, демо собирает отчёт и выводит результат.
- [ ] Тесты xUnit зелёные: валидация билдера, корректность `Build()`, тип компонентов фабрики.
- [ ] `sample-output.txt` зафиксирован, вывод содержит корректную разметку выбранного формата.

#### Подсказки (без прямого ответа)
- Для выбора фабрики по строке формата используйте `switch` expression над строкой — это идиоматичный pattern matching C# 12, который заменил цепочки `if/else`.
- Для регистрации нескольких реализаций по ключу в DI удобно использовать `Func<TKey, TService>` — зарегистрируйте словарь и фабричную функцию, которая по ключу возвращает нужный экземпляр.
- В raw string literals (`"""..."""`) можно встраивать интерполяцию, добавив минимум соответствующих `$` — это удобно для HTML/PDF-шаблонов с кавычками.
- Не забудьте `await` для асинхронных источников и `using` для `CancellationTokenSource`, чтобы демонстрация корректно отменяла операции.
- Разделяйте билдер и результат: билдер изменяемый, результат — `record` с `init`. Не давайте билдеру возвращать `this` из `Build()`.
- Для тестов stub-реализации интерфейсов — это и есть проверка того, что паттерн упрощает подмену; Moq не обязателен.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — AcmeReports: Factory Method + Abstract Factory + Fluent Builder
// Полное рабочее решение, демонстрирующее все три паттерна из урока M16-L06.
// A complete working solution demonstrating all three patterns from lesson M16-L06.

using System;
using System.Collections.Generic;
using Microsoft.Extensions.DependencyInjection;

// ── Abstractions / Абстракции ────────────────────────────────
namespace AcmeReports.Abstractions;

public sealed record ReportRow(string Category, string Metric, double Value);

public interface IReportSource
{
    // Асинхронное чтение строк / Async row reading
    Task<IReadOnlyList<ReportRow>> ReadAsync(CancellationToken ct = default);
}

public interface IReportHeader { string Render(ReportRequest req); }
public interface IReportTable   { string Render(IReadOnlyList<ReportRow> rows); }
public interface IReportFooter  { string Render(DateTime generatedAt); }

// ── Builders / Билдеры ────────────────────────────────────────
namespace AcmeReports.Builders;

using AcmeReports.Abstractions;

// Неизменяемый результат сборки / Immutable build result
public sealed record ReportRequest
{
    public required string   Title    { get; init; }
    public string?           Subtitle { get; init; }
    public string?           Author   { get; init; }
    public DateOnly?         From     { get; init; }
    public DateOnly?         To       { get; init; }
    public int               PageSize { get; init; } = 50;
    public required string   Format   { get; init; }   // pdf | html
    public IReadOnlyList<string> Tags { get; init; } = [];
}

// Fluent Builder с валидацией в Build() / Fluent Builder with validation in Build()
public sealed class ReportBuilder
{
    private string? _title;
    private string? _subtitle;
    private string? _author;
    private DateOnly? _from;
    private DateOnly? _to;
    private int _pageSize = 50;
    private string? _format;
    private List<string> _tags = [];

    public ReportBuilder WithTitle(string title)        { _title = title; return this; }
    public ReportBuilder WithSubtitle(string? s)        { _subtitle = s; return this; }
    public ReportBuilder WithAuthor(string? a)          { _author = a; return this; }
    public ReportBuilder WithPeriod(DateOnly from, DateOnly to)
    { _from = from; _to = to; return this; }
    public ReportBuilder WithPageSize(int size)         { _pageSize = size; return this; }
    public ReportBuilder WithFormat(string format)      { _format = format; return this; }
    public ReportBuilder WithTags(params string[] tags) { _tags = [..tags]; return this; }

    public ReportRequest Build()
    {
        // Валидация обязательных полей в одной точке / Validate required fields in one place
        if (string.IsNullOrWhiteSpace(_title))
            throw new InvalidOperationException(
                "Title is required / Заголовок обязателен");
        if (string.IsNullOrWhiteSpace(_format))
            throw new InvalidOperationException(
                "Format is required / Формат обязателен");

        return new ReportRequest
        {
            Title = _title,
            Subtitle = _subtitle,
            Author = _author,
            From = _from,
            To = _to,
            PageSize = _pageSize,
            Format = _format,
            Tags = _tags
        };
    }
}

// ── Sources / Источники (Factory Method) ──────────────────────
namespace AcmeReports.Sources;

using AcmeReports.Abstractions;

public sealed class DatabaseSource : IReportSource
{
    public async Task<IReadOnlyList<ReportRow>> ReadAsync(CancellationToken ct = default)
    {
        await Task.Delay(20, ct); // имитация запроса к БД / DB call imitation
        return
        [
            new("Sales",  "Revenue", 1500.0),
            new("Sales",  "Orders",  42.0),
            new("HR",     "Hires",   5.0),
        ];
    }
}

public sealed class RestApiSource : IReportSource
{
    public async Task<IReadOnlyList<ReportRow>> ReadAsync(CancellationToken ct = default)
    {
        await Task.Delay(30, ct);
        return [ new("API", "Requests", 999.0) ];
    }
}

public sealed class CsvFileSource : IReportSource
{
    public Task<IReadOnlyList<ReportRow>> ReadAsync(CancellationToken ct = default)
        => Task.FromResult<IReadOnlyList<ReportRow>>(
            [ new("CSV", "Rows", 12.0) ]);
}

// Базовый создатель с фабричным методом / Creator base with factory method
public abstract class ReportSourceFactory(string sourceKey)
{
    // Primary ctor хранит sourceKey без явного поля / Primary ctor keeps sourceKey without a field
    public string SourceKey => sourceKey;
    public abstract IReportSource Create();   // Factory Method
}

public sealed class DatabaseSourceFactory()  : ReportSourceFactory("db")  { public override IReportSource Create() => new DatabaseSource(); }
public sealed class RestApiSourceFactory()   : ReportSourceFactory("api") { public override IReportSource Create() => new RestApiSource(); }
public sealed class CsvFileSourceFactory()   : ReportSourceFactory("csv") { public override IReportSource Create() => new CsvFileSource(); }

// ── Formats / Форматы (Abstract Factory) ─────────────────────
namespace AcmeReports.Formats;

using AcmeReports.Abstractions;
using AcmeReports.Builders;

// Контракт на согласованное семейство / Contract for a consistent family
public interface IReportFactory
{
    IReportHeader CreateHeader();
    IReportTable  CreateTable();
    IReportFooter CreateFooter();
}

// PDF-семейство / PDF family
public sealed class PdfHeader : IReportHeader
{
    public string Render(ReportRequest req) =>
        $"<pdf-header>{req.Title} — {req.Author ?? "Anonymous"}</pdf-header>";
}
public sealed class PdfTable : IReportTable
{
    public string Render(IReadOnlyList<ReportRow> rows) =>
        "<pdf-table>" + string.Join("|", rows.Select(r => $"{r.Category}:{r.Value}")) + "</pdf-table>";
}
public sealed class PdfFooter : IReportFooter
{
    public string Render(DateTime generatedAt) => $"<pdf-footer>Generated {generatedAt:O}</pdf-footer>";
}
public sealed class PdfReportFactory : IReportFactory
{
    public IReportHeader CreateHeader() => new PdfHeader();
    public IReportTable  CreateTable()  => new PdfTable();
    public IReportFooter CreateFooter() => new PdfFooter();
}

// HTML-семейство / HTML family
public sealed class HtmlHeader : IReportHeader
{
    public string Render(ReportRequest req) =>
        $"<h1>{req.Title}</h1><p>{req.Subtitle ?? ""}</p>";
}
public sealed class HtmlTable : IReportTable
{
    public string Render(IReadOnlyList<ReportRow> rows) =>
        "<table>" + string.Join("", rows.Select(r => $"<tr><td>{r.Category}</td><td>{r.Value}</td></tr>")) + "</table>";
}
public sealed class HtmlFooter : IReportFooter
{
    public string Render(DateTime generatedAt) => $"<footer>{generatedAt:O}</footer>";
}
public sealed class HtmlReportFactory : IReportFactory
{
    public IReportHeader CreateHeader() => new HtmlHeader();
    public IReportTable  CreateTable()  => new HtmlTable();
    public IReportFooter CreateFooter() => new HtmlFooter();
}

// ── Program (top-level statements) ───────────────────────────
using AcmeReports.Abstractions;
using AcmeReports.Builders;
using AcmeReports.Formats;
using AcmeReports.Sources;
using Microsoft.Extensions.DependencyInjection;

// Парсинг аргументов / Argument parsing
string source = args.Length > 0 ? args[0] : "db";
string format = args.Length > 1 ? args[1] : "pdf";

// DI-регистрация фабрик / DI registration of factories
var services = new ServiceCollection();
services.AddSingleton<ReportSourceFactory, DatabaseSourceFactory>();   // по умолчанию
services.AddSingleton<IReportFactory, PdfReportFactory>();             // по умолчанию

// Фабричная функция выбора по ключу / Factory function for keyed lookup
services.AddSingleton<Func<string, IReportFactory>>(sp => key => key switch
{
    "pdf"  => new PdfReportFactory(),
    "html" => new HtmlReportFactory(),
    _      => throw new InvalidOperationException($"Unknown format: {key}")
});

using var sp = services.BuildServiceProvider();
var factorySelector = sp.GetRequiredService<Func<string, IReportFactory>>();
var sourceFactory   = sp.GetRequiredService<ReportSourceFactory>();

// Pattern matching для источника / Pattern matching for source
ReportSourceFactory srcFactory = source switch
{
    "db"  => new DatabaseSourceFactory(),
    "api" => new RestApiSourceFactory(),
    "csv" => new CsvFileSourceFactory(),
    _     => throw new InvalidOperationException($"Unknown source: {source}")
};

IReportFactory fmtFactory = factorySelector(format);

// Fluent Builder — пошаговая сборка / Fluent Builder — step-by-step assembly
ReportRequest request = new ReportBuilder()
    .WithTitle("Q3 Sales Report / Отчёт по продажам Q3")
    .WithSubtitle("Sales department / Отдел продаж")
    .WithAuthor("Alice")
    .WithPeriod(new DateOnly(2024, 7, 1), new DateOnly(2024, 9, 30))
    .WithPageSize(100)
    .WithFormat(format)
    .WithTags("sales", "q3", "internal")
    .Build();

// Чтение данных и рендеринг / Reading data and rendering
using var cts = new CancellationTokenSource();
IReportSource src = srcFactory.Create();
IReadOnlyList<ReportRow> rows = await src.ReadAsync(cts.Token);

string header = fmtFactory.CreateHeader().Render(request);
string table  = fmtFactory.CreateTable().Render(rows);
string footer = fmtFactory.CreateFooter().Render(DateTime.UtcNow);

Console.WriteLine(header);
Console.WriteLine(table);
Console.WriteLine(footer);
```

**Разбор по строкам.** Решение демонстрирует все три паттерна из урока в одном проекте. `ReportSourceFactory` — классический Factory Method: абстрактный создатель с primary ctor `(string sourceKey)` и абстрактным `Create()`, возвращающим `IReportSource`. Это «полиморфное создание одного продукта» из теории урока: каждый подкласс (`DatabaseSourceFactory`, `RestApiSourceFactory`, `CsvFileSourceFactory`) решает, какой конкретный источник вернуть, а клиент работает только с `IReportSource` и не знает, БД это или REST. Primary ctor убирает boilerplate — `sourceKey` доступен через свойство `SourceKey` без явного поля, как в уроке с `NotificationSender(string defaultTarget)`.

`IReportFactory` — это Abstract Factory: контракт на целое семейство (`CreateHeader/CreateTable/CreateFooter`), а не на один продукт. Конкретные `PdfReportFactory` и `HtmlReportFactory` производят согласованные наборы (`PdfHeader/PdfTable/PdfFooter` и `HtmlHeader/HtmlTable/HtmlFooter`), и клиент, получивший одну фабрику, гарантированно не смешивает компоненты разных семейств. Это прямо иллюстрирует тезис урока: «Abstract Factory создаёт семейство, а не один продукт, и обычно через композицию». Выбор фабрики по строке формата сделан через switch expression (pattern matching C# 12) внутри `Func<string, IReportFactory>` — идиоматичная замена цепочкам `if/else`.

`ReportBuilder` — Fluent Builder. Он изменяемый (`private`-поля, fluent-методы возвращают `this`), а результат `Build()` — неизменяемый `record ReportRequest` с `init`-свойствами и `required`-полями. Валидация обязательных полей (`Title`, `Format`) сосредоточена в `Build()` и выбрасывает `InvalidOperationException` с двуязычным сообщением — ровно как требует урок: «Валидируйте обязательные поля в `Build()`, а не в каждом сеттере». Collection expression `[]` для пустого списка тегов и `[..tags]` для копирования массива — это C# 12 collection expressions из теории. Источники асинхронны с `CancellationToken`, DI регистрирует фабрики по интерфейсу, а клиентский код зависит только от абстракций — применены все best practices из урока и избегнуты все частые ошибки (нет фабрики ради фабрики, нет mutable builder-результата, нет сильной связности с конкретными типами).

#### Задания на углубление (бонус)
1. Добавьте третий формат — Markdown. Реализуйте `MarkdownReportFactory` и три компонента. Убедитесь, что для этого не пришлось править существующие классы — только добавить новый. Зарегистрируйте в switch expression и в DI. Это проверит, насколько система расширяема по сценарию Open/Closed.
2. Добавьте направляющий билдер (staged builder): типобезопасный билдер, где `WithTitle` возвращает интерфейс `IWithSubtitleStage`, который не позволяет вызвать `Build()` без `Title`. Используйте интерфейсы-маркеры стадий. Это усложнённый вариант fluent builder, который на уровне типов гарантирует обязательные шаги.
3. Реализуйте кэширующий декоратор для `IReportSource`, который кэширует результат `ReadAsync` на 30 секунд. Используйте `IMemoryCache` из `Microsoft.Extensions.Caching.Memory`. Это познакомит с композицией декоратора поверх фабричного продукта и подготовит к следующему уроку (Strategy, Decorator, Adapter).
4. Добавьте статический фабричный метод `ReportRequest.CreateDefault(string title, string format)` как альтернативу билдеру для простых случаев. Сравните читаемость двух подходов и опишите в комментариях, когда какой применять — это закрепит тезис урока о выборе между фабричным методом и билдером.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you work on a team building an internal analytics service called "AcmeReports". The service must accept raw events from different sources (a database, a REST API, a local CSV file), transform them into a single domain object `ReportRequest`, and then render a final report in one of several formats: PDF, HTML, Markdown, or an Excel-compatible CSV. Each format requires its own rendering engine with its own consistent set of components: a header, a data table, and a footer. Meanwhile, the data source is selected by environment configuration (dev/prod), while the report format is selected by the user request.

This task maps cleanly onto the three creational patterns from the lesson. The data source is a polymorphic creation of a single product (`IReportSource`), which fits Factory Method perfectly: each creator subclass decides which concrete source to return. The report format is a whole family of consistent components (`IReportHeader`, `IReportTable`, `IReportFooter`), and here you need Abstract Factory so the client gets a guaranteed-compatible set for the chosen format. Finally, `ReportRequest` itself has many optional fields (subtitle, author, tags, period, filters, pagination parameters), and it is far easier to assemble it through a Fluent Builder than to pass through a god-constructor.

The motivation is not academic: in practice, a direct `new` for every format and source quickly leads to scattered `switch` and `if` statements that grow with every new variant. The creational patterns isolate the client from concrete types, make extension safe (a new format is just a new factory, existing code is untouched), and simplify testing by substituting factories in DI. In this assignment you will build exactly such an extensible system and confirm that adding a new format or source requires only a new class, not edits to client code.

#### What to do step by step
1. Create a new .NET 8 console application with `dotnet new console -n AcmeReports -o AcmeReports --framework net8.0`. Move into the folder with `cd AcmeReports` and confirm that `dotnet build` succeeds without errors. Open `Program.cs` and remove the template code — you will write top-level statements.
2. Inside the project folder create subfolders `Sources`, `Formats`, `Builders`, and `Abstractions`. The folder split is deliberate: it reflects the domain layers and simplifies navigation once the number of types grows.
3. In `Abstractions` define the product interfaces. First the data source contract: `IReportSource` with `Task<IReadOnlyList<ReportRow>> ReadAsync(CancellationToken ct)`, where `ReportRow` is a record with fields `Category`, `Metric`, `Value`. Then the report component contracts: `IReportHeader` with `string Render(ReportRequest req)`, `IReportTable` with `string Render(IReadOnlyList<ReportRow> rows)`, `IReportFooter` with `string Render(DateTime generatedAt)`.
4. In `Builders` implement `ReportRequest` as an immutable `record` with `init`-only properties and a required `Title`, plus `ReportBuilder` — a mutable class with fluent methods `WithTitle`, `WithSubtitle`, `WithAuthor`, `WithTags`, `WithPeriod`, `WithFilters`, `WithPageSize`, `WithFormat`. Each method returns `this`. The `Build()` method returns `ReportRequest` and validates required fields (Title, Format), throwing `InvalidOperationException` with a bilingual message.
5. In `Sources` implement three concrete sources: `DatabaseSource` (simulates a DB read, returns test rows), `RestApiSource` (simulates an HTTP call with `Task.Delay`), `CsvFileSource` (reads an embedded CSV string). Then implement the factory method: an abstract class `ReportSourceFactory` with a primary ctor `ReportSourceFactory(SourceConfig config)` and an abstract method `IReportSource Create()`, plus concrete `DatabaseSourceFactory`, `RestApiSourceFactory`, `CsvFileSourceFactory`.
6. In `Formats` implement Abstract Factory. Define the `IReportFactory` interface with methods `CreateHeader()`, `CreateTable()`, `CreateFooter()`. Build two concrete factories: `PdfReportFactory` and `HtmlReportFactory`, each returning a consistent component family (for PDF — `PdfHeader`, `PdfTable`, `PdfFooter`; for HTML — the same names with HTML markup). Confirm that components from different factories never mix: the client receives only one factory and only its products.
7. In `Program.cs` assemble a demo through top-level statements. Build a `ReportRequest` through the builder, select the source through the factory method based on a command-line argument (e.g., `dotnet run --source db --format pdf`), pick the format factory, read the data, render the header, table, footer, and print the result to the console. Use `CancellationToken` and `await`.
8. Register the factories in `Microsoft.Extensions.DependencyInjection`: create a `ServiceCollection`, register `IReportFactory` by format key (you can use a factory function `Func<string, IReportFactory>`), and `ReportSourceFactory` by source string. This shows how the patterns compose with DI and free the client from knowing concrete types.
9. Add unit tests with `dotnet new xunit -n AcmeReports.Tests -o AcmeReports.Tests --framework net8.0` and a reference to the main project `dotnet add reference ../AcmeReports/AcmeReports.csproj`. Write tests: the builder throws without Title; `Build()` populates fields correctly; `PdfReportFactory` returns exactly a `PdfHeader`; mixing components of two factories is impossible at the type level (through interfaces).
10. Run `dotnet test` and confirm all tests are green. Then run `dotnet run -- --source db --format html` and verify the output contains HTML markup and a correct footer with the date. Capture the output in a file `sample-output.txt` through redirection `dotnet run -- --source db --format pdf > sample-output.txt`.

#### Requirements
The solution must compile under .NET 8 without warnings (including nullable analysis) and pass all tests. Use C# 12: primary constructors for factory and builder classes, `record` for immutable results, `init`-only properties, collection expressions for list initialization (`[row1, row2]`), pattern matching for selecting the factory by format string (`switch` expression), and raw string literals for markup templates where quotes and braces appear. All factories must return interfaces (`IReportSource`, `IReportFactory`, `IReportHeader`, etc.), not concrete types — client code depends only on abstractions.

The project structure must be clean: each class in its own file, file names matching type names, folders reflecting layers. The code must be extension-ready: adding a new format (e.g., Markdown) or a new source (e.g., Kafka) must not require edits to existing classes — only a new factory class and a DI registration. Validation in `Build()` is mandatory: missing `Title` or `Format` throws an exception with a bilingual message. Data sources must be asynchronous (`Task`-based) and accept a `CancellationToken`. In tests use xUnit and, if needed, simple stub implementations of the interfaces (no Moq, to keep dependencies light) — this is acceptable because the patterns are exactly what make substitution easy.

#### Pitfalls
The main trap is confusing the patterns. If you have a single product (the data source) that is Factory Method, not Abstract Factory; if you have a whole family of consistent components (header + table + footer) that is Abstract Factory, and an attempt to assemble a family from several factory methods of different nature will lead to inconsistency. The lesson explicitly warns: "Abstract Factory for a single product instead of Factory Method → choose by product count". Watch this separation in your code.

The second pitfall is mutability. The builder is mutable by nature (it accumulates state), but the `Build()` result must be immutable. Do not return the same object from `Build()` as the builder, and do not let setters mutate an already-built `ReportRequest`. Use a `record` with `init`-only properties, like the lesson's `ReportConfig`. If the client tries to change a field after assembly, the compiler will refuse (`init` is only available during initialization). The third pitfall is validation. The lesson requires validating required fields in `Build()`, not in every fluent method: this concentrates checks and gives a clear exception at a single point. Do not rely on "automatically correct" from the primary constructor — it does not validate.

The fourth pitfall is primary constructors. They remove boilerplate, but remember that primary ctor parameters implicitly become fields and are in scope across the whole class. For factories this is convenient (`IUiFactory factory` is available in `RenderAll`), but do not overuse: if a parameter needs validation, do it explicitly in the factory method or constructor. The fifth pitfall is DI and concrete types. The client must not depend on `PdfReportFactory` directly; register by the `IReportFactory` interface and select the implementation by format key through `Func<string, IReportFactory>`. The sixth pitfall is nullable: mark optional fields as `string?`, required ones as `required` or validate in `Build()`. Finally, do not introduce a factory for a single class: if the current iteration needs only one source, keep a direct `new` or a static method until polymorphism is genuinely needed.

#### Acceptance criteria
- [ ] The `AcmeReports` project is created under .NET 8; `dotnet build` succeeds without errors or warnings.
- [ ] Folder structure present: `Abstractions`, `Sources`, `Formats`, `Builders`.
- [ ] `IReportSource` and the three components `IReportHeader`/`IReportTable`/`IReportFooter` are declared as interfaces in `Abstractions`.
- [ ] `ReportRequest` is an immutable `record` with `init`-only properties and a required `Title`.
- [ ] `ReportBuilder` is a mutable class with fluent methods, each returning `this`.
- [ ] `Build()` validates required fields and throws `InvalidOperationException` with a bilingual message.
- [ ] `ReportSourceFactory` is an abstract class with a primary ctor and an abstract `Create()` returning `IReportSource`.
- [ ] At least three concrete sources and three concrete source factories.
- [ ] `IReportFactory` (Abstract Factory) is declared with `CreateHeader/CreateTable/CreateFooter`.
- [ ] At least two concrete format factories (Pdf, Html), each returning a consistent family.
- [ ] C# 12 features used: primary constructors, collection expressions, pattern matching (switch expression), raw string literals.
- [ ] Sources are asynchronous and accept a `CancellationToken`.
- [ ] Factories are registered in `Microsoft.Extensions.DependencyInjection`; the client does not know concrete types.
- [ ] `Program.cs` uses top-level statements; the demo assembles a report and prints the result.
- [ ] xUnit tests are green: builder validation, `Build()` correctness, component type from the factory.
- [ ] `sample-output.txt` is captured; the output contains correct markup for the chosen format.

#### Hints
- To select a factory by format string, use a `switch` expression over the string — this is the idiomatic C# 12 pattern matching that replaces `if/else` chains.
- To register multiple implementations by key in DI, a `Func<TKey, TService>` is convenient — register a dictionary and a factory function that returns the right instance by key.
- In raw string literals (`"""..."""`) you can embed interpolation by adding at least the matching number of `$` — handy for HTML/PDF templates with quotes.
- Do not forget `await` for async sources and `using` for `CancellationTokenSource` so the demo cancels operations correctly.
- Separate the builder from the result: the builder is mutable, the result is a `record` with `init`. Do not let the builder return `this` from `Build()`.
- For tests, stub implementations of interfaces are themselves a check that the pattern makes substitution easy; Moq is optional.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — AcmeReports: Factory Method + Abstract Factory + Fluent Builder
// A complete working solution demonstrating all three patterns from lesson M16-L06.

using System;
using System.Collections.Generic;
using Microsoft.Extensions.DependencyInjection;

// ── Abstractions ─────────────────────────────────────────────
namespace AcmeReports.Abstractions;

public sealed record ReportRow(string Category, string Metric, double Value);

public interface IReportSource
{
    // Async row reading
    Task<IReadOnlyList<ReportRow>> ReadAsync(CancellationToken ct = default);
}

public interface IReportHeader { string Render(ReportRequest req); }
public interface IReportTable   { string Render(IReadOnlyList<ReportRow> rows); }
public interface IReportFooter  { string Render(DateTime generatedAt); }

// ── Builders ─────────────────────────────────────────────────
namespace AcmeReports.Builders;

using AcmeReports.Abstractions;

// Immutable build result
public sealed record ReportRequest
{
    public required string   Title    { get; init; }
    public string?           Subtitle { get; init; }
    public string?           Author   { get; init; }
    public DateOnly?         From     { get; init; }
    public DateOnly?         To       { get; init; }
    public int               PageSize { get; init; } = 50;
    public required string   Format   { get; init; }   // pdf | html
    public IReadOnlyList<string> Tags { get; init; } = [];
}

// Fluent Builder with validation in Build()
public sealed class ReportBuilder
{
    private string? _title;
    private string? _subtitle;
    private string? _author;
    private DateOnly? _from;
    private DateOnly? _to;
    private int _pageSize = 50;
    private string? _format;
    private List<string> _tags = [];

    public ReportBuilder WithTitle(string title)        { _title = title; return this; }
    public ReportBuilder WithSubtitle(string? s)        { _subtitle = s; return this; }
    public ReportBuilder WithAuthor(string? a)          { _author = a; return this; }
    public ReportBuilder WithPeriod(DateOnly from, DateOnly to)
    { _from = from; _to = to; return this; }
    public ReportBuilder WithPageSize(int size)         { _pageSize = size; return this; }
    public ReportBuilder WithFormat(string format)      { _format = format; return this; }
    public ReportBuilder WithTags(params string[] tags) { _tags = [..tags]; return this; }

    public ReportRequest Build()
    {
        // Validate required fields in one place
        if (string.IsNullOrWhiteSpace(_title))
            throw new InvalidOperationException(
                "Title is required / Заголовок обязателен");
        if (string.IsNullOrWhiteSpace(_format))
            throw new InvalidOperationException(
                "Format is required / Формат обязателен");

        return new ReportRequest
        {
            Title = _title,
            Subtitle = _subtitle,
            Author = _author,
            From = _from,
            To = _to,
            PageSize = _pageSize,
            Format = _format,
            Tags = _tags
        };
    }
}

// ── Sources (Factory Method) ─────────────────────────────────
namespace AcmeReports.Sources;

using AcmeReports.Abstractions;

public sealed class DatabaseSource : IReportSource
{
    public async Task<IReadOnlyList<ReportRow>> ReadAsync(CancellationToken ct = default)
    {
        await Task.Delay(20, ct); // DB call imitation
        return
        [
            new("Sales",  "Revenue", 1500.0),
            new("Sales",  "Orders",  42.0),
            new("HR",     "Hires",   5.0),
        ];
    }
}

public sealed class RestApiSource : IReportSource
{
    public async Task<IReadOnlyList<ReportRow>> ReadAsync(CancellationToken ct = default)
    {
        await Task.Delay(30, ct);
        return [ new("API", "Requests", 999.0) ];
    }
}

public sealed class CsvFileSource : IReportSource
{
    public Task<IReadOnlyList<ReportRow>> ReadAsync(CancellationToken ct = default)
        => Task.FromResult<IReadOnlyList<ReportRow>>(
            [ new("CSV", "Rows", 12.0) ]);
}

// Creator base with the factory method
public abstract class ReportSourceFactory(string sourceKey)
{
    // Primary ctor keeps sourceKey without an explicit field
    public string SourceKey => sourceKey;
    public abstract IReportSource Create();   // Factory Method
}

public sealed class DatabaseSourceFactory()  : ReportSourceFactory("db")  { public override IReportSource Create() => new DatabaseSource(); }
public sealed class RestApiSourceFactory()   : ReportSourceFactory("api") { public override IReportSource Create() => new RestApiSource(); }
public sealed class CsvFileSourceFactory()   : ReportSourceFactory("csv") { public override IReportSource Create() => new CsvFileSource(); }

// ── Formats (Abstract Factory) ───────────────────────────────
namespace AcmeReports.Formats;

using AcmeReports.Abstractions;
using AcmeReports.Builders;

// Contract for a consistent family
public interface IReportFactory
{
    IReportHeader CreateHeader();
    IReportTable  CreateTable();
    IReportFooter CreateFooter();
}

// PDF family
public sealed class PdfHeader : IReportHeader
{
    public string Render(ReportRequest req) =>
        $"<pdf-header>{req.Title} — {req.Author ?? "Anonymous"}</pdf-header>";
}
public sealed class PdfTable : IReportTable
{
    public string Render(IReadOnlyList<ReportRow> rows) =>
        "<pdf-table>" + string.Join("|", rows.Select(r => $"{r.Category}:{r.Value}")) + "</pdf-table>";
}
public sealed class PdfFooter : IReportFooter
{
    public string Render(DateTime generatedAt) => $"<pdf-footer>Generated {generatedAt:O}</pdf-footer>";
}
public sealed class PdfReportFactory : IReportFactory
{
    public IReportHeader CreateHeader() => new PdfHeader();
    public IReportTable  CreateTable()  => new PdfTable();
    public IReportFooter CreateFooter() => new PdfFooter();
}

// HTML family
public sealed class HtmlHeader : IReportHeader
{
    public string Render(ReportRequest req) =>
        $"<h1>{req.Title}</h1><p>{req.Subtitle ?? ""}</p>";
}
public sealed class HtmlTable : IReportTable
{
    public string Render(IReadOnlyList<ReportRow> rows) =>
        "<table>" + string.Join("", rows.Select(r => $"<tr><td>{r.Category}</td><td>{r.Value}</td></tr>")) + "</table>";
}
public sealed class HtmlFooter : IReportFooter
{
    public string Render(DateTime generatedAt) => $"<footer>{generatedAt:O}</footer>";
}
public sealed class HtmlReportFactory : IReportFactory
{
    public IReportHeader CreateHeader() => new HtmlHeader();
    public IReportTable  CreateTable()  => new HtmlTable();
    public IReportFooter CreateFooter() => new HtmlFooter();
}

// ── Program (top-level statements) ───────────────────────────
using AcmeReports.Abstractions;
using AcmeReports.Builders;
using AcmeReports.Formats;
using AcmeReports.Sources;
using Microsoft.Extensions.DependencyInjection;

// Argument parsing
string source = args.Length > 0 ? args[0] : "db";
string format = args.Length > 1 ? args[1] : "pdf";

// DI registration of factories
var services = new ServiceCollection();
services.AddSingleton<ReportSourceFactory, DatabaseSourceFactory>();   // default
services.AddSingleton<IReportFactory, PdfReportFactory>();             // default

// Factory function for keyed lookup
services.AddSingleton<Func<string, IReportFactory>>(sp => key => key switch
{
    "pdf"  => new PdfReportFactory(),
    "html" => new HtmlReportFactory(),
    _      => throw new InvalidOperationException($"Unknown format: {key}")
});

using var sp = services.BuildServiceProvider();
var factorySelector = sp.GetRequiredService<Func<string, IReportFactory>>();
var sourceFactory   = sp.GetRequiredService<ReportSourceFactory>();

// Pattern matching for source
ReportSourceFactory srcFactory = source switch
{
    "db"  => new DatabaseSourceFactory(),
    "api" => new RestApiSourceFactory(),
    "csv" => new CsvFileSourceFactory(),
    _     => throw new InvalidOperationException($"Unknown source: {source}")
};

IReportFactory fmtFactory = factorySelector(format);

// Fluent Builder — step-by-step assembly
ReportRequest request = new ReportBuilder()
    .WithTitle("Q3 Sales Report / Отчёт по продажам Q3")
    .WithSubtitle("Sales department / Отдел продаж")
    .WithAuthor("Alice")
    .WithPeriod(new DateOnly(2024, 7, 1), new DateOnly(2024, 9, 30))
    .WithPageSize(100)
    .WithFormat(format)
    .WithTags("sales", "q3", "internal")
    .Build();

// Reading data and rendering
using var cts = new CancellationTokenSource();
IReportSource src = srcFactory.Create();
IReadOnlyList<ReportRow> rows = await src.ReadAsync(cts.Token);

string header = fmtFactory.CreateHeader().Render(request);
string table  = fmtFactory.CreateTable().Render(rows);
string footer = fmtFactory.CreateFooter().Render(DateTime.UtcNow);

Console.WriteLine(header);
Console.WriteLine(table);
Console.WriteLine(footer);
```

**Walk-through.** The solution demonstrates all three patterns from the lesson in a single project. `ReportSourceFactory` is a classic Factory Method: an abstract creator with a primary ctor `(string sourceKey)` and an abstract `Create()` returning `IReportSource`. This is the "polymorphic creation of a single product" from the lesson theory: each subclass (`DatabaseSourceFactory`, `RestApiSourceFactory`, `CsvFileSourceFactory`) decides which concrete source to return, while the client works only with `IReportSource` and does not know whether it is a DB or REST. The primary ctor removes boilerplate — `sourceKey` is reachable through the `SourceKey` property without an explicit field, exactly like the lesson's `NotificationSender(string defaultTarget)`.

`IReportFactory` is the Abstract Factory: a contract for a whole family (`CreateHeader/CreateTable/CreateFooter`), not a single product. The concrete `PdfReportFactory` and `HtmlReportFactory` produce consistent sets (`PdfHeader/PdfTable/PdfFooter` and `HtmlHeader/HtmlTable/HtmlFooter`), and a client that received one factory is guaranteed not to mix components from different families. This directly illustrates the lesson thesis: "Abstract Factory creates a family, not a single product, and usually through composition". Selecting the factory by format string is done through a switch expression (C# 12 pattern matching) inside `Func<string, IReportFactory>` — the idiomatic replacement for `if/else` chains.

`ReportBuilder` is the Fluent Builder. It is mutable (`private` fields, fluent methods return `this`), while the `Build()` result is an immutable `record ReportRequest` with `init`-only properties and `required` fields. Validation of required fields (`Title`, `Format`) is concentrated in `Build()` and throws `InvalidOperationException` with a bilingual message — exactly as the lesson demands: "Validate required fields in `Build()`, not in every setter". The collection expression `[]` for an empty tag list and `[..tags]` for copying an array are the C# 12 collection expressions from the theory. Sources are asynchronous with `CancellationToken`, DI registers factories by interface, and client code depends only on abstractions — all the lesson's best practices are applied and all the common mistakes avoided (no factory-for-factory's-sake, no mutable builder result, no tight coupling to concrete types).

#### Going deeper (bonus)
1. Add a third format — Markdown. Implement `MarkdownReportFactory` and its three components. Confirm that this required no edits to existing classes — only a new addition. Register it in the switch expression and in DI. This will check how Open/Closed your system really is.
2. Add a staged builder: a type-safe builder where `WithTitle` returns an `IWithSubtitleStage` interface that does not allow `Build()` without `Title`. Use stage-marker interfaces. This is an advanced fluent builder that guarantees required steps at the type level.
3. Implement a caching decorator for `IReportSource` that caches the `ReadAsync` result for 30 seconds. Use `IMemoryCache` from `Microsoft.Extensions.Caching.Memory`. This will introduce decorator composition over a factory product and prepare you for the next lesson (Strategy, Decorator, Adapter).
4. Add a static factory method `ReportRequest.CreateDefault(string title, string format)` as an alternative to the builder for simple cases. Compare the readability of the two approaches and describe in comments when to use which — this reinforces the lesson thesis on choosing between a factory method and a builder.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект собирается под .NET 8 без ошибок и предупреждений.
- [ ] Все три паттерна реализованы и разделены по смыслу: Factory Method — источники, Abstract Factory — форматы, Builder — запрос отчёта.
- [ ] Использованы первичные конструкторы C# 12 в фабриках и билдерах.
- [ ] Результат `Build()` — неизменяемый `record` с `init`-свойствами.
- [ ] Валидация обязательных полей сосредоточена в `Build()`.
- [ ] Фабрики возвращают интерфейсы, клиент не зависит от конкретных типов.
- [ ] Фабрики зарегистрированы в DI, выбор по ключу через `Func<string, IReportFactory>`.
- [ ] Источники асинхронны с `CancellationToken`.
- [ ] Тесты xUnit зелёные и покрывают валидацию, сборку и тип компонентов фабрики.
- [ ] Зафиксирован `sample-output.txt` с корректной разметкой.
- [ ] The project builds under .NET 8 without errors or warnings.
- [ ] All three patterns are implemented and semantically separated: Factory Method — sources, Abstract Factory — formats, Builder — report request.
- [ ] C# 12 primary constructors are used in factories and builders.
- [ ] The `Build()` result is an immutable `record` with `init`-only properties.
- [ ] Validation of required fields is concentrated in `Build()`.
- [ ] Factories return interfaces; the client does not depend on concrete types.
- [ ] Factories are registered in DI; selection by key through `Func<string, IReportFactory>`.
- [ ] Sources are asynchronous with `CancellationToken`.
- [ ] xUnit tests are green and cover validation, assembly, and factory component type.
- [ ] `sample-output.txt` is captured with correct markup.

#### Ресурсы / Resources
- [Microsoft Learn — Patterns: Factory Method](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- [Microsoft Learn — Dependency injection in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [Microsoft Learn — Records (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record)
- [Microsoft Learn — Primary constructors (C# 12)](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12#primary-constructors)
- [Microsoft Learn — Collection expressions (C# 12)](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12#collection-expressions)
- [Refactoring Guru — Creational Patterns](https://refactoring.guru/design-patterns/creational-patterns)
