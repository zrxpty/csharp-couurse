---
[← К уроку M05-L09](lesson-M05-L09-composition-vs-inheritance.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M05-L09: Composition vs inheritance (вступление к SOLID) / Homework M05-L09: Composition vs inheritance (intro to SOLID)

**Урок / Lesson:** M05-L09
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться перепроектировать «наследовательный ад» в композицию с интерфейсами, делегированием и декораторами; закрепить связь композиции с буквами SOLID (SRP, OCP, LSP, ISP, DIP) и получить дизайн, удобный для unit-тестирования. (EN) Learn to refactor an "inheritance hell" design into composition with interfaces, delegation and decorators; internalise how composition supports the SOLID letters (SRP, OCP, LSP, ISP, DIP) and yields a test-friendly design.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит принцип «Prefer composition over inheritance» (Gang of Four) и показывает композицию + делегирование на примере `Car`, содержащего `IEngine`, `IDrivetrain`, `ILogger`, а также декоратор `TimestampedLogger`. ДЗ берёт ту же модель (has-a через интерфейсы, инъекция через конструктор, sealed-классы, private readonly-поля) и переносит её в новую предметную область — обработку документов, — требуя от студента самостоятельно пройти путь от плохого наследовательного дизайна к хорошему композиционному.
(EN) The lesson introduces "Prefer composition over inheritance" (Gang of Four) and demonstrates composition + delegation through a `Car` that *has-an* `IEngine`, `IDrivetrain`, `ILogger`, plus a `TimestampedLogger` decorator. This homework reuses the same model (has-a through interfaces, constructor injection, sealed classes, private readonly fields) and moves it into a new domain — document processing — asking the student to walk the full path from a bad inheritance-based design to a good compositional one.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

В небольшом стартапе «DocFlow» пишут конвейер обработки текстовых документов: документ нужно проверить, отформатировать, экспортировать в один из форматов (HTML, PDF) и залогировать каждый шаг. Первый разработчик, недавно окончивший курс ООП, написал решение «в лоб» через наследование: абстрактный `DocumentBase` с виртуальными методами `Validate`, `Format`, `Export`, а от него — `MarkdownDocument`, `HtmlDocument`, а потом появились `PdfMarkdownDocument`, `HtmlMarkdownDocument`, и иерархия стала расти вглубь и вширь. Добавление каждого нового формата или правила проверки заставляет плодить подклассы по схеме «декартова произведения» (формат × правило × логгер), что в точности повторяет «наследовательный ад» `ManagerEmployeeWithBonusAndStockOptions` из урока.

Тебя нанимают как Senior-инженера, чтобы перепроектировать систему в стиле урока M05-L09: композиция + делегирование через маленькие интерфейсы (ISP), инъекция зависимостей через конструктор (DIP), sealed-классы по умолчанию, private readonly-поля-компоненты, декоратор для сквозной функциональности (логирование с таймстемпом). Ты должен не просто «переписать код», а аргументированно показать, почему новая структура лучше поддерживает SRP (каждый компонент делает одно дело), OCP (новый формат = новый компонент, а не новый подкласс), LSP (мы вообще убираем иерархию, в которой легко нарушить подстановку) и тестируемость (компоненты мокаются через интерфейсы).

Эта задача — мостик к следующим урокам SOLID: ты на практике почувствуешь, почему «наследование определяет, что объект есть, а композиция — что он делает», и почему современный C# 12 (primary constructors, init-свойства, default interface methods, records) делает композицию удобнее, чем когда-либо.

#### Что нужно сделать (пошагово)

1. **Создай проект.** Выполни команды (используй .NET 8 SDK):
   ```
   dotnet new console -n DocFlow -o DocFlow --framework net8.0
   cd DocFlow
   dotnet new sln -n DocFlow
   dotnet sln add DocFlow.csproj
   ```
   Добавь тестовый проект (для пункта о тестируемости):
   ```
   dotnet new xunit -n DocFlow.Tests -o DocFlow.Tests --framework net8.0
   dotnet sln add DocFlow.Tests/DocFlow.Tests.csproj
   dotnet add DocFlow.Tests/DocFlow.Tests.csproj reference DocFlow/DocFlow.csproj
   ```

2. **Сохрани «плохой» стартовый код** в файл `DocFlow/Legacy.cs` — это наследовательный дизайн, который ты будешь критиковать и переписывать:
   ```csharp
   namespace DocFlow.Legacy;

   public abstract class DocumentBase
   {
       public string Title { get; init; } = "";
       public string Body { get; init; } = "";
       public virtual bool Validate() => Body.Length > 0;
       public virtual string Format() => Body;
       public virtual byte[] Export() => System.Text.Encoding.UTF8.GetBytes(Format());
   }

   public sealed class MarkdownDocument : DocumentBase
   {
       public override string Format() => $"# {Title}\n\n{Body}";
   }

   public sealed class HtmlDocument : DocumentBase
   {
       public override string Format() => $"<h1>{Title}</h1><p>{Body}</p>";
   }

   // «Наследовательный ад» начинается здесь:
   public sealed class PdfMarkdownDocument : MarkdownDocument
   {
       public override byte[] Export() => System.Text.Encoding.UTF8.GetBytes($"PDF>>{Format()}");
   }
   ```
   В файле `DocFlow/Notes.md` выпиши **минимум 5 конкретных проблем** этого дизайна, опираясь на урок: глубокая иерархия, жёсткая связь с реализацией базового класса, декартово произведение подклассов, риск нарушения LSP, невозможность комбинировать формат и экспортёр независимо, отсутствие тестируемости без создания реального объекта.

3. **Спроектируй компоненты и интерфейсы** в `DocFlow/Composition.cs`. Определи маленькие интерфейсы (ISP): `IValidator`, `IFormatter`, `IExporter`, `ILogger`. У каждого — по одному методу с ясным контрактом. Тип документа сделай `record` с `init`-свойствами: `public sealed record Document(string Title, string Body);`.

4. **Реализуй компоненты** как `sealed` классы с primary constructors:
   - `LengthValidator(int minLength) : IValidator` — проверяет, что `Body.Length >= minLength`.
   - `MarkdownFormatter : IFormatter` — оборачивает в `# {Title}` + тело.
   - `HtmlFormatter : IFormatter` — оборачивает в `<h1>`/`<p>`.
   - `HtmlExporter : IExporter` — кодирует в UTF-8 с префиксом `HTML>>`.
   - `PdfExporter : IExporter` — кодирует с префиксом `PDF>>`.
   - `ConsoleLogger : ILogger` — пишет `[log] {message}`.
   - `TimestampedLogger(ILogger inner) : ILogger` — **декоратор**, добавляет `[UTC]`-метку и делегирует внутреннему логгеру (в точности как `TimestampedLogger` из урока).

5. **Реализуй главный объект** `DocumentProcessor` через композицию: он **содержит** `IValidator`, `IFormatter`, `IExporter`, `ILogger` как `private readonly` поля, принимает их через конструктор (или primary constructor), в конструкторе зовёт `ArgumentNullException.ThrowIfNull(...)` для каждого (как в `Car` из урока). Метод `Process(Document doc)` делегирует: validate → format → export → логирует шаги.

6. **Напиши `Program.cs`** с top-level statements: собери два разных процессора из одних и тех же компонентов, меняя только `IExporter` и `IFormatter`, и обработай документ. Вывод должен показать, что одна и та же оболочка `DocumentProcessor` работает с разными компонентами без новой иерархии классов — как `Car` с `GasEngine` и `ElectricEngine` в уроке.

7. **Покрой тестами** (`DocFlow.Tests/ProcessorTests.cs`): внедри `FakeLogger` (или используй mock-библиотеку) и проверь, что `Process` вызывает логгер нужное число раз и что смена `IExporter` меняет префикс в результате. Тест не должен создавать реальный `ConsoleLogger` — это и есть доказательство тестируемости через интерфейсы.

8. **Запусти и проверь:**
   ```
   dotnet build
   dotnet run --project DocFlow
   dotnet test
   ```
   Убедись, что вывод содержит строки с `HTML>>` и `PDF>>` и что тесты зелёные.

#### Требования к решению

- Целевой фреймворк — `net8.0`, язык — C# 12: используй file-scoped namespaces, primary constructors, `init`/`required`-свойства где уместно, collection expressions для инициализации, pattern matching в проверках, `record` для неизменяемого `Document`.
- Все компоненты и сам `DocumentProcessor` — `sealed` (урок явно рекомендует запечатывать классы по умолч, чтобы поощрять композицию).
- Компоненты хранятся как `private readonly` поля; внутренние детали не утечки наружу (нет публичных mutable-полей — частая ошибка из урока).
- Зависимости инъектируются **только** через конструктор; внутри конструктора бизнес-класса нет `new` для зависимостей (урок называет это типичной ошибкой, мешающей тестированию и замене реализации).
- Декоратор `TimestampedLogger` реализует `ILogger`, хранит внутренний логгер и делегирует ему вызов, добавляя метку — это композиция + делегирование, основа паттернов Decorator/Adapter/Strategy.
- Никаких классов, наследующих `DocumentBase`-подобный базовый класс в финальном дизайне; иерархия не глубже одного уровня (только `object`).
- Код компилируется без предупреждений уровня error и проходит `dotnet test`.

#### Тонкости и подводные камни

- **Не путай «is-a» и «has-a».** `MarkdownFormatter` **не является** документом, он **содержит логику** форматирования и применяется к документу. Если потянешься написать `class MarkdownDocument : DocumentBase` — остановись: ты делаешь наследование ради переиспользования кода, а не ради истинного *is-a*. Урок прямо предостерегает от этого.
- **LSP-ловушка Square/Rectangle.** В наследовательном дизайне легко переопределить `Validate` или `Format` так, что подкласс перестаёт удовлетворять контракту базового (например, `PdfMarkdownDocument.Export` игнорирует `Format`-логику предка). Композиция убирает эту ловушку вовсе, потому что нет иерархии, в которой можно «переопределить с сюрпризом».
- **`sealed` по умолчанию.** Урок рекомендует запечатывать классы: это сигнал «расширяй меня композицией, а не наследованием». Если ты оставляешь класс незапечатанным без причины — это антипаттерн по умолчанию.
- **`new` в конструкторе — запах.** Если `DocumentProcessor` сам создаёт `new ConsoleLogger()`, ты не сможешь подменить логгер в тестах. Инъектируй через интерфейс (DIP). Это та же ошибка, что и создание `new GasEngine()` внутри `Car` — но урок намеренно инъектирует двигатель снаружи.
- **Декоратор делегирует, а не дублирует.** `TimestampedLogger` не должен переписывать логику форматирования; он добавляет метку и зовёт `_inner.Log(...)`. Дублирование — частая ошибка: нарушает DRY и ломает цепочку декораторов (ведь внутренний логгер тоже может быть декоратором).
- **Маленькие интерфейсы (ISP).** Не делай один `IDocumentService` с `Validate+Format+Export+Log`. Раздели на четыре интерфейса — тогда компоненты переиспользуются независимо и тестируются изолированно.
- **`ArgumentNullException.ThrowIfNull`** — идиоматичный guard в .NET 8; он бросает `ArgumentNullException` с именем параметра автоматически. Не пиши ручные `if (x == null) throw new ArgumentNullException(nameof(x))`.

#### Критерии приёмки

- [ ] Проект `DocFlow` (net8.0) собирается командой `dotnet build` без ошибок.
- [ ] Файл `Legacy.cs` содержит наследовательный дизайн, а `Notes.md` — минимум 5 аргументированных проблем со ссылкой на концепции урока.
- [ ] Определены интерфейсы `IValidator`, `IFormatter`, `IExporter`, `ILogger` (по одному методу, ISP-friendly).
- [ ] `Document` — `sealed record` с `init`/позиционными свойствами.
- [ ] Все компоненты — `sealed` классы с primary constructors.
- [ ] `TimestampedLogger` — декоратор над `ILogger`, добавляет UTC-метку и делегирует внутреннему логгеру.
- [ ] `DocumentProcessor` содержит компоненты как `private readonly` поля, инъектируемые через конструктор, с `ArgumentNullException.ThrowIfNull`-проверками.
- [ ] Внутри конструктора `DocumentProcessor` нет `new` для зависимостей.
- [ ] `Program.cs` (top-level) собирает два процессора с разными `IExporter`/`IFormatter` и обрабатывает документ; иерархия классов не глубже одного уровня.
- [ ] Тесты (`dotnet test`) зелёные и используют фейковый/моковый `ILogger`, не `ConsoleLogger`.
- [ ] Использованы C# 12-фичи: file-scoped namespace, primary constructor, collection expression или pattern matching хотя бы в одном месте.
- [ ] В коде нет TODO, заглушек, закомментированного кода.
- [ ] Вывод `dotnet run` содержит `HTML>>` и `PDF>>` строки.
- [ ] В `Notes.md` указано, какие буквы SOLID поддерживает новый дизайн (SRP, OCP, LSP, ISP, DIP).

#### Подсказки (без прямого ответа)

- Подумай, что унаследовал бы `PdfHtmlDocument` от `HtmlDocument` в Legacy-дизайне, и почему это уже тупик — это поможет сформулировать проблему №5 в `Notes.md`.
- Вспомни, как `Car` из урока принимает `IEngine engine, IDrivetrain drivetrain, ILogger logger` в конструкторе — `DocumentProcessor` устроён так же.
- Декоратор `TimestampedLogger` один-в-один повторяет `TimestampedLogger(ILogger inner)` из урока — просто другая метка или тот же формат `DateTime.UtcNow:O`.
- Для тестов достаточно ручного `FakeLogger : ILogger`, записывающего сообщения в `List<string>` — это мок без библиотеки, как `ConsoleLogger` в уроке, только с памятью.
- Collection expression пригодится для инициализации `List<string>` в фейковом логгере: `private readonly List<string> _entries = [];`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Composition vs inheritance (DocFlow)
// Композиция + делегирование через интерфейсы; sealed-компоненты; декоратор.
// Composition + delegation through interfaces; sealed components; decorator.

namespace DocFlow;

using System;
using System.Text;

// --- Маленькие контракты (ISP) / Small contracts (ISP) ---
public interface IValidator
{
    bool Validate(Document document); // true, если документ допустим
}

public interface IFormatter
{
    string Format(Document document); // возвращает строковое представление
}

public interface IExporter
{
    byte[] Export(string formatted); // превращает формат в байты
}

public interface ILogger
{
    void Log(string message);
}

// --- Неизменяемый документ / Immutable document ---
public sealed record Document(string Title, string Body);

// --- Конкретные компоненты (composable, swappable) ---

public sealed class LengthValidator(int minLength) : IValidator
{
    public bool Validate(Document document) =>
        document.Body.Length >= minLength && document.Title.Length > 0;
}

public sealed class MarkdownFormatter : IFormatter
{
    public string Format(Document document) =>
        $"# {document.Title}\n\n{document.Body}";
}

public sealed class HtmlFormatter : IFormatter
{
    public string Format(Document document) =>
        $"<h1>{document.Title}</h1><p>{document.Body}</p>";
}

public sealed class HtmlExporter : IExporter
{
    public byte[] Export(string formatted) =>
        Encoding.UTF8.GetBytes($"HTML>>{formatted}");
}

public sealed class PdfExporter : IExporter
{
    public byte[] Export(string formatted) =>
        Encoding.UTF8.GetBytes($"PDF>>{formatted}");
}

public sealed class ConsoleLogger : ILogger
{
    public void Log(string message) =>
        Console.WriteLine($"[log] {message}");
}

// --- Декоратор: композиция + делегирование (как TimestampedLogger из урока) ---
public sealed class TimestampedLogger(ILogger inner) : ILogger
{
    private readonly ILogger _inner = inner;
    public void Log(string message) =>
        _inner.Log($"[{DateTime.UtcNow:O}] {message}");
}

// --- Главный объект: СОДЕРЖИТ компоненты (has-a), не ЯВЛЯЕТСЯ ими ---
public sealed class DocumentProcessor(
    IValidator validator,
    IFormatter formatter,
    IExporter exporter,
    ILogger logger)
{
    private readonly IValidator _validator = validator;
    private readonly IFormatter _formatter = formatter;
    private readonly IExporter _exporter = exporter;
    private readonly ILogger _logger = logger;

    public DocumentProcessor(IValidator v, IFormatter f, IExporter e, ILogger l)
        : this(v, f, e, l)
    {
        ArgumentNullException.ThrowIfNull(v);
        ArgumentNullException.ThrowIfNull(f);
        ArgumentNullException.ThrowIfNull(e);
        ArgumentNullException.ThrowIfNull(l);
    }

    public byte[] Process(Document document)
    {
        _logger.Log($"Начинаем обработку: {document.Title}");
        if (!_validator.Validate(document))
        {
            _logger.Log("Документ не прошёл валидацию");
            return [];
        }
        var formatted = _formatter.Format(document);
        var bytes = _exporter.Export(formatted);
        _logger.Log($"Готово, байт: {bytes.Length}");
        return bytes;
    }
}
```

**Разбор по строкам.** `IValidator`/`IFormatter`/`IExporter`/`ILogger` — четыре маленьких интерфейса вместо одного «толстого» `IDocumentService`; это **ISP** из урока: каждый компонент переиспользуется независимо, как `IEngine`/`IDrivetrain`/`ILogger` у `Car`. `Document` — `sealed record` с позиционными свойствами: неизменяемость и value-семантика, без иерархии. `LengthValidator` и `MarkdownFormatter` — `sealed` классы с primary constructors (`LengthValidator(int minLength)`); запечатывание по умолчанию — прямая рекомендация урока, поощряющая композицию. `HtmlExporter`/`PdfExporter` — два независимых экспортёра; в Legacy-дизайне их пришлось бы множить через `PdfMarkdownDocument`/`HtmlMarkdownDocument`, а здесь это просто два компонента, которые можно комбинировать с любым форматером — **OCP**: новый формат или экспортёр = новый класс, а не правка существующих. `TimestampedLogger(ILogger inner)` — точная копия декоратора из урока: он реализует `ILogger`, хранит `_inner` как `private readonly`, добавляет метку и делегирует; это и есть композиция + делегирование, основа Decorator/Adapter/Strategy. `DocumentProcessor` принимает все четыре зависимости через конструктор и зовёт `ArgumentNullException.ThrowIfNull` — идентично `Car` из урока с его проверками `engine`/`drivetrain`/`logger`; внутри нет `new`, поэтому в тестах подменяем любой компонент (**DIP** + тестируемость). Метод `Process` ничего не знает о конкретных классах — он делегирует контрактам, поэтому смена `HtmlExporter` на `PdfExporter` не требует правки `DocumentProcessor` (полиморфизм через интерфейсы, а не через иерархию). Заметь отсутствие `DocumentBase`: иерархия не глубже одного уровня (`object`), «наследовательного ада» нет, **LSP**-риска нет — потому что нет иерархии, в которой можно переопределить метод с сюрпризом. В тестах подставляем `FakeLogger : ILogger` со счётчиком вызовов — доказательство, что дизайн поддерживает изолированное тестирование. Так одна задача иллюстрирует все пять букв SOLID, которые урок связывает с композицией: SRP (один компонент — одна ответственность), OCP (расширяем новыми компонентами), LSP (нет иерархии — нет нарушения), ISP (маленькие интерфейсы), DIP (инъекция через абстракции).

#### Задания на углубление (бонус)

1. **LSP-кейс через композицию.** Реализуй классический пример `Square`/`Rectangle`, где наследование ломает инвариант (установка `Width` у `Square` меняет `Height`). Перепиши через композицию: `Rectangle` имеет `IResizeBehavior`, а `Square` использует другой компонент, сохраняющий равенство сторон. Опиши, почему композиция снимает LSP-проблему.
2. **Цепочка декораторов.** Добавь `RedactingLogger(ILogger inner)`, маскирующий секреты (заменяет `password=...` на `password=***`), и собери цепочку `new TimestampedLogger(new RedactingLogger(new ConsoleLogger()))`. Покажи, что порядок декораторов меняет поведение.
3. **Runtime-смена компонента.** Сделай `DocumentProcessor` с внутренним `IExporter`, заменяемым в рантайме через метод `WithExporter(IExporter)` (возвращающий новый процессор или меняющий поле). Объясни, почему это невозможно в наследовательном дизайне без правки иерархии.
4. **Default interface Methods.** Добавь в `ILogger` default-метод `LogError(string)` с дефолт-реализацией, зовущей `Log($"ERROR: {message}")`. Покажи, что существующие компоненты не нужно менять — ещё один плюс композиции с интерфейсами в C# 12.

---

## Statement in English / Постановка на английском

#### Context & motivation

A small startup, "DocFlow", is building a text-document processing pipeline: a document must be validated, formatted, exported into one of several formats (HTML, PDF), and every step must be logged. The first developer, fresh out of an OOP course, wrote the solution the straightforward way, through inheritance: an abstract `DocumentBase` with virtual `Validate`, `Format`, `Export` methods, then `MarkdownDocument` and `HtmlDocument` deriving from it, and soon `PdfMarkdownDocument`, `HtmlMarkdownDocument`, with the hierarchy growing deeper and wider. Each new format or validation rule forces a Cartesian product of subclasses (format × rule × logger), which is exactly the "inheritance hell" of `ManagerEmployeeWithBonusAndStockOptions` described in the lesson.

You are hired as a Senior engineer to redesign the system in the spirit of lesson M05-L09: composition + delegation through small interfaces (ISP), dependency injection through the constructor (DIP), sealed classes by default, private readonly component fields, and a decorator for cross-cutting concerns (timestamped logging). You must do more than "rewrite the code" — you must argue why the new structure better supports SRP (each component does one thing), OCP (a new format is a new component, not a new subclass), LSP (you remove the hierarchy in which substitution is easy to break), and testability (components are mockable through interfaces).

This task is a bridge to the upcoming SOLID lessons: you will feel in practice why "inheritance defines what an object is; composition defines what it does", and why modern C# 12 (primary constructors, init properties, default interface methods, records) makes composition more ergonomic than ever.

#### What to do step by step

1. **Create the project.** Run the following (use the .NET 8 SDK):
   ```
   dotnet new console -n DocFlow -o DocFlow --framework net8.0
   cd DocFlow
   dotnet new sln -n DocFlow
   dotnet sln add DocFlow.csproj
   ```
   Add a test project (for the testability step):
   ```
   dotnet new xunit -n DocFlow.Tests -o DocFlow.Tests --framework net8.0
   dotnet sln add DocFlow.Tests/DocFlow.Tests.csproj
   dotnet add DocFlow.Tests/DocFlow.Tests.csproj reference DocFlow/DocFlow.csproj
   ```

2. **Keep the "bad" starter code** in `DocFlow/Legacy.cs` — this is the inheritance-based design you will critique and replace:
   ```csharp
   namespace DocFlow.Legacy;

   public abstract class DocumentBase
   {
       public string Title { get; init; } = "";
       public string Body { get; init; } = "";
       public virtual bool Validate() => Body.Length > 0;
       public virtual string Format() => Body;
       public virtual byte[] Export() => System.Text.Encoding.UTF8.GetBytes(Format());
   }

   public sealed class MarkdownDocument : DocumentBase
   {
       public override string Format() => $"# {Title}\n\n{Body}";
   }

   public sealed class HtmlDocument : DocumentBase
   {
       public override string Format() => $"<h1>{Title}</h1><p>{Body}</p>";
   }

   // "Inheritance hell" starts here:
   public sealed class PdfMarkdownDocument : MarkdownDocument
   {
       public override byte[] Export() => System.Text.Encoding.UTF8.GetBytes($"PDF>>{Format()}");
   }
   ```
   In `DocFlow/Notes.md` list **at least 5 concrete problems** of this design, grounded in the lesson: deep hierarchy, tight coupling to the base class implementation, a Cartesian product of subclasses, LSP risk, the inability to combine a formatter and an exporter independently, and the lack of testability without instantiating a real object.

3. **Design components and interfaces** in `DocFlow/Composition.cs`. Define small interfaces (ISP): `IValidator`, `IFormatter`, `IExporter`, `ILogger`. Each should have a single method with a clear contract. Make the document a `record` with `init` properties: `public sealed record Document(string Title, string Body);`.

4. **Implement components** as `sealed` classes with primary constructors:
   - `LengthValidator(int minLength) : IValidator` — checks that `Body.Length >= minLength`.
   - `MarkdownFormatter : IFormatter` — wraps as `# {Title}` + body.
   - `HtmlFormatter : IFormatter` — wraps as `<h1>`/`<p>`.
   - `HtmlExporter : IExporter` — encodes to UTF-8 with an `HTML>>` prefix.
   - `PdfExporter : IExporter` — encodes with a `PDF>>` prefix.
   - `ConsoleLogger : ILogger` — writes `[log] {message}`.
   - `TimestampedLogger(ILogger inner) : ILogger` — a **decorator** that adds a `[UTC]` timestamp and delegates to the inner logger (exactly like the `TimestampedLogger` in the lesson).

5. **Implement the main object** `DocumentProcessor` through composition: it **contains** `IValidator`, `IFormatter`, `IExporter`, `ILogger` as `private readonly` fields, receives them through a constructor (or a primary constructor), and calls `ArgumentNullException.ThrowIfNull(...)` for each in the constructor (as `Car` does in the lesson). The `Process(Document doc)` method delegates: validate → format → export → log the steps.

6. **Write `Program.cs`** with top-level statements: assemble two different processors from the same components, changing only `IExporter` and `IFormatter`, and process a document. The output should show that the same `DocumentProcessor` shell works with different components without a new class hierarchy — just as `Car` works with `GasEngine` and `ElectricEngine` in the lesson.

7. **Cover with tests** (`DocFlow.Tests/ProcessorTests.cs`): inject a `FakeLogger` (or a mocking library) and verify that `Process` calls the logger the expected number of times and that swapping `IExporter` changes the prefix in the result. The test must not instantiate a real `ConsoleLogger` — this is the proof of testability through interfaces.

8. **Run and verify:**
   ```
   dotnet build
   dotnet run --project DocFlow
   dotnet test
   ```
   Make sure the output contains lines with `HTML>>` and `PDF>>` and that the tests are green.

#### Requirements

- Target framework `net8.0`, language C# 12: use file-scoped namespaces, primary constructors, `init`/`required` properties where appropriate, collection expressions for initialisation, pattern matching in checks, and a `record` for the immutable `Document`.
- All components and the `DocumentProcessor` itself are `sealed` (the lesson explicitly recommends sealing classes by default to encourage composition).
- Components are stored as `private readonly` fields; internals do not leak (no public mutable fields — a common mistake from the lesson).
- Dependencies are injected **only** through the constructor; there is no `new` for dependencies inside the business class constructor (the lesson calls this a typical mistake that hurts testing and swapping).
- The `TimestampedLogger` decorator implements `ILogger`, holds an inner logger, and delegates the call to it while adding a timestamp — this is composition + delegation, the foundation of Decorator/Adapter/Strategy.
- No classes inherit a `DocumentBase`-like base class in the final design; the hierarchy is no deeper than one level (only `object`).
- The code compiles without error-level warnings and passes `dotnet test`.

#### Pitfalls

- **Do not confuse "is-a" and "has-a".** `MarkdownFormatter` **is not** a document; it **holds formatting logic** and is applied to a document. If you reach for `class MarkdownDocument : DocumentBase`, stop: you are inheriting to reuse code, not to express a genuine *is-a*. The lesson warns against exactly this.
- **The LSP Square/Rectangle trap.** In an inheritance design it is easy to override `Validate` or `Format` so that a subclass stops honouring the base contract (for instance, `PdfMarkdownDocument.Export` ignores the ancestor's `Format` logic). Composition removes the trap entirely, because there is no hierarchy in which a method can be "overridden with a surprise".
- **`sealed` by default.** The lesson recommends sealing classes: it signals "extend me through composition, not inheritance". Leaving a class unsealed without reason is an anti-pattern by default.
- **`new` in the constructor is a smell.** If `DocumentProcessor` itself creates `new ConsoleLogger()`, you cannot replace the logger in tests. Inject through an interface (DIP). This is the same mistake as creating `new GasEngine()` inside `Car` — but the lesson injects the engine from outside on purpose.
- **A decorator delegates, it does not duplicate.** `TimestampedLogger` must not reimplement the formatting logic; it adds a timestamp and calls `_inner.Log(...)`. Duplication is a common mistake: it breaks DRY and breaks decorator chains (the inner logger may itself be a decorator).
- **Small interfaces (ISP).** Do not build a single `IDocumentService` with `Validate+Format+Export+Log`. Split into four interfaces so components are reused and tested independently.
- **`ArgumentNullException.ThrowIfNull`** is the idiomatic guard in .NET 8; it throws `ArgumentNullException` with the parameter name automatically. Do not write manual `if (x == null) throw new ArgumentNullException(nameof(x))`.

#### Acceptance criteria

- [ ] The `DocFlow` project (net8.0) builds with `dotnet build` without errors.
- [ ] `Legacy.cs` contains the inheritance design, and `Notes.md` lists at least 5 argued problems referencing lesson concepts.
- [ ] Interfaces `IValidator`, `IFormatter`, `IExporter`, `ILogger` are defined (one method each, ISP-friendly).
- [ ] `Document` is a `sealed record` with `init`/positional properties.
- [ ] All components are `sealed` classes with primary constructors.
- [ ] `TimestampedLogger` is a decorator over `ILogger`, adds a UTC timestamp, and delegates to the inner logger.
- [ ] `DocumentProcessor` holds components as `private readonly` fields, injected through the constructor, with `ArgumentNullException.ThrowIfNull` checks.
- [ ] There is no `new` for dependencies inside the `DocumentProcessor` constructor.
- [ ] `Program.cs` (top-level) assembles two processors with different `IExporter`/`IFormatter` and processes a document; the class hierarchy is no deeper than one level.
- [ ] Tests (`dotnet test`) are green and use a fake/mock `ILogger`, not `ConsoleLogger`.
- [ ] C# 12 features are used: file-scoped namespace, primary constructor, a collection expression or pattern matching in at least one place.
- [ ] No TODOs, stubs, or commented-out code.
- [ ] The `dotnet run` output contains `HTML>>` and `PDF>>` lines.
- [ ] `Notes.md` states which SOLID letters the new design supports (SRP, OCP, LSP, ISP, DIP).

#### Hints (no direct answer)

- Think about what `PdfHtmlDocument` would inherit from `HtmlDocument` in the Legacy design, and why that is already a dead end — this helps formulate problem №5 in `Notes.md`.
- Recall how `Car` in the lesson takes `IEngine engine, IDrivetrain drivetrain, ILogger logger` in its constructor — `DocumentProcessor` is structured the same way.
- The `TimestampedLogger` decorator mirrors `TimestampedLogger(ILogger inner)` from the lesson one-to-one — just a different label, or the same `DateTime.UtcNow:O` format.
- For tests a manual `FakeLogger : ILogger` that records messages into a `List<string>` is enough — a mock without a library, like the lesson's `ConsoleLogger` but with memory.
- A collection expression is handy to initialise the `List<string>` in the fake logger: `private readonly List<string> _entries = [];`.

#### Reference solution (walk-through)

```csharp
// C# 12 / .NET 8 — Composition vs inheritance (DocFlow)
// Composition + delegation through interfaces; sealed components; decorator.

namespace DocFlow;

using System;
using System.Text;

// --- Small contracts (ISP) ---
public interface IValidator
{
    bool Validate(Document document); // true when the document is acceptable
}

public interface IFormatter
{
    string Format(Document document); // returns a string representation
}

public interface IExporter
{
    byte[] Export(string formatted); // turns the format into bytes
}

public interface ILogger
{
    void Log(string message);
}

// --- Immutable document ---
public sealed record Document(string Title, string Body);

// --- Concrete components (composable, swappable) ---

public sealed class LengthValidator(int minLength) : IValidator
{
    public bool Validate(Document document) =>
        document.Body.Length >= minLength && document.Title.Length > 0;
}

public sealed class MarkdownFormatter : IFormatter
{
    public string Format(Document document) =>
        $"# {document.Title}\n\n{document.Body}";
}

public sealed class HtmlFormatter : IFormatter
{
    public string Format(Document document) =>
        $"<h1>{document.Title}</h1><p>{document.Body}</p>";
}

public sealed class HtmlExporter : IExporter
{
    public byte[] Export(string formatted) =>
        Encoding.UTF8.GetBytes($"HTML>>{formatted}");
}

public sealed class PdfExporter : IExporter
{
    public byte[] Export(string formatted) =>
        Encoding.UTF8.GetBytes($"PDF>>{formatted}");
}

public sealed class ConsoleLogger : ILogger
{
    public void Log(string message) =>
        Console.WriteLine($"[log] {message}");
}

// --- Decorator: composition + delegation (like the lesson's TimestampedLogger) ---
public sealed class TimestampedLogger(ILogger inner) : ILogger
{
    private readonly ILogger _inner = inner;
    public void Log(string message) =>
        _inner.Log($"[{DateTime.UtcNow:O}] {message}");
}

// --- Main object: HAS-A components, not IS-A them ---
public sealed class DocumentProcessor(
    IValidator validator,
    IFormatter formatter,
    IExporter exporter,
    ILogger logger)
{
    private readonly IValidator _validator = validator;
    private readonly IFormatter _formatter = formatter;
    private readonly IExporter _exporter = exporter;
    private readonly ILogger _logger = logger;

    public DocumentProcessor(IValidator v, IFormatter f, IExporter e, ILogger l)
        : this(v, f, e, l)
    {
        ArgumentNullException.ThrowIfNull(v);
        ArgumentNullException.ThrowIfNull(f);
        ArgumentNullException.ThrowIfNull(e);
        ArgumentNullException.ThrowIfNull(l);
    }

    public byte[] Process(Document document)
    {
        _logger.Log($"Start processing: {document.Title}");
        if (!_validator.Validate(document))
        {
            _logger.Log("Document failed validation");
            return [];
        }
        var formatted = _formatter.Format(document);
        var bytes = _exporter.Export(formatted);
        _logger.Log($"Done, bytes: {bytes.Length}");
        return bytes;
    }
}
```

**Line-by-line walk-through.** `IValidator`/`IFormatter`/`IExporter`/`ILogger` are four small interfaces instead of one fat `IDocumentService`; this is **ISP** from the lesson: each component is reused independently, like `IEngine`/`IDrivetrain`/`ILogger` on `Car`. `Document` is a `sealed record` with positional properties: immutability and value semantics, with no hierarchy. `LengthValidator` and `MarkdownFormatter` are `sealed` classes with primary constructors (`LengthValidator(int minLength)`); sealing by default is a direct recommendation of the lesson, nudging the design toward composition. `HtmlExporter`/`PdfExporter` are two independent exporters; in the Legacy design they would have to be multiplied as `PdfMarkdownDocument`/`HtmlMarkdownDocument`, while here they are just two components combinable with any formatter — **OCP**: a new format or exporter is a new class, not an edit to existing ones. `TimestampedLogger(ILogger inner)` is an exact copy of the lesson's decorator: it implements `ILogger`, stores `_inner` as `private readonly`, adds a timestamp, and delegates; this is composition + delegation, the foundation of Decorator/Adapter/Strategy. `DocumentProcessor` takes all four dependencies through the constructor and calls `ArgumentNullException.ThrowIfNull` — identical to `Car` from the lesson with its `engine`/`drivetrain`/`logger` checks; there is no `new` inside, so any component can be substituted in tests (**DIP** + testability). The `Process` method knows nothing about concrete classes — it delegates to contracts, so swapping `HtmlExporter` for `PdfExporter` requires no change to `DocumentProcessor` (polymorphism through interfaces, not through a hierarchy). Notice the absence of `DocumentBase`: the hierarchy is no deeper than one level (`object`), there is no inheritance hell, and there is no **LSP** risk — because there is no hierarchy in which a method can be overridden with a surprise. In tests we inject a `FakeLogger : ILogger` with a call counter — proof that the design supports isolated testing. A single task thus illustrates all five SOLID letters that the lesson ties to composition: SRP (one component, one responsibility), OCP (extend with new components), LSP (no hierarchy, no violation), ISP (small interfaces), DIP (injection through abstractions).

#### Going deeper (bonus)

1. **LSP case via composition.** Implement the classic `Square`/`Rectangle` example where inheritance breaks an invariant (setting `Width` on `Square` also changes `Height`). Rewrite it through composition: `Rectangle` has an `IResizeBehavior`, and `Square` uses a different component that keeps the sides equal. Explain why composition removes the LSP problem.
2. **Decorator chain.** Add a `RedactingLogger(ILogger inner)` that masks secrets (replaces `password=...` with `password=***`) and assemble a chain `new TimestampedLogger(new RedactingLogger(new ConsoleLogger()))`. Show that the order of decorators changes behaviour.
3. **Runtime component swap.** Make `DocumentProcessor` hold an `IExporter` that is replaceable at runtime through a `WithExporter(IExporter)` method (returning a new processor or mutating the field). Explain why this is impossible in an inheritance design without editing the hierarchy.
4. **Default interface methods.** Add a default method `LogError(string)` to `ILogger` with a default implementation that calls `Log($"ERROR: {message}")`. Show that existing components need no changes — another win of composition with interfaces in C# 12.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `DocFlow` собирается (`dotnet build`), тесты зелёные (`dotnet test`).
- [ ] `Legacy.cs` + `Notes.md` с 5+ проблемами наследовательного дизайна.
- [ ] `Composition.cs`: 4 интерфейса (ISP), sealed-компоненты, декоратор `TimestampedLogger`.
- [ ] `DocumentProcessor` с инъекцией через конструктор, `ArgumentNullException.ThrowIfNull`, без `new` для зависимостей.
- [ ] `Program.cs` (top-level) демонстрирует смену компонентов без новой иерархии.
- [ ] Тесты с фейковым `ILogger`; использованы C# 12-фичи.
- [ ] В `Notes.md` указаны поддерживаемые буквы SOLID.
- [ ] The `DocFlow` project builds (`dotnet build`), tests are green (`dotnet test`).
- [ ] `Legacy.cs` + `Notes.md` with 5+ problems of the inheritance design.
- [ ] `Composition.cs`: 4 interfaces (ISP), sealed components, `TimestampedLogger` decorator.
- [ ] `DocumentProcessor` with constructor injection, `ArgumentNullException.ThrowIfNull`, no `new` for dependencies.
- [ ] `Program.cs` (top-level) demonstrates swapping components without a new hierarchy.
- [ ] Tests with a fake `ILogger`; C# 12 features used.
- [ ] `Notes.md` lists the supported SOLID letters.

#### Ресурсы / Resources
- Microsoft Learn — Interfaces and default interface methods (C#): https://learn.microsoft.com/dotnet/csharp/fundamentals/types/interfaces
- Microsoft Learn — Microservices DDD/CQRS patterns (composition over inheritance in domain design): https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/
- Gang of Four, *Design Patterns* (1994) — original "Favor object composition over class inheritance" guidance.
- Robert C. Martin, *Agile Principles, Patterns, and Practices in C#* — SOLID foundation.
- Microsoft Learn — `ArgumentNullException.ThrowIfNull`: https://learn.microsoft.com/dotnet/api/system.argumentnullexception.throwifnull
