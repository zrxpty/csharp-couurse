[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M15-L02: xUnit: факты, теории, InlineData/MemberData / xUnit: facts, theories, InlineData/MemberData

**Модуль / Module:** M15
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

xUnit — это современный фреймворк для модульного тестирования в экосистеме .NET, пришедший на смену MSTest и NUnit во многих командах. Его философия проста: тест — это метод, а его поведение определяется атрибутами. Два столпа xUnit — это `[Fact]` и `[Theory]`.

`[Fact]` помечает тест, который всегда выполняется одинаково: один вход — один ожидаемый результат. Это аналог утверждения «данная истина неизменна». Если метод `Add(2, 3)` должен вернуть `5`, это факт. Факты идеальны для проверки инвариантов, не зависящих от входных данных.

`[Theory]` — это параметризованный тест. Один и тот же метод прогоняется с разными наборами данных. Подумайте об этом как о матрице: строки — наборы аргументов, столбцы — параметры метода. Вместо того чтобы копировать тест десять раз, вы пишете его один раз и подаёте данные через атрибуты-провайдеры.

`[InlineData]` — самый простой провайдер. Вы прямо в атрибуте перечисляете значения: `[InlineData(2, 3, 5)] [InlineData(10, -2, 8)]`. Это удобно для небольшого количества случаев, но значения должны быть константами времени компиляции — нельзя подставить экземпляр класса или результат вычисления.

`[MemberData]` берёт данные из статического свойства или поля, возвращающего `IEnumerable<object[]>`. Это открывает дверь к сложным объектам, вычисляемым значениям и даже данным, загружаемым из файла или базы. Сравните: `InlineData` — это список покупок на салфетке, `MemberData` — таблица в Excel, которую вы можете редактировать отдельно от кода.

`[ClassData]` идёт ещё дальше: целый класс, реализующий `IEnumerable<object[]>` или `TheoryData<>`, становится источником данных. Это позволяет переиспользовать наборы данных между разными тестовыми классами и хранить их в одном месте.

Жизненный цикл тестового класса в xUnit отличается от NUnit. Для каждого теста xUnit создаёт **новый экземпляр** класса — это гарантирует изоляцию, но означает, что конструктор работает как `SetUp`. Очистка ресурсов делается через интерфейс `IDisposable`: xUnit вызывает `Dispose()` после каждого теста автоматически. Аналогия: каждый тест — это гость в гостинице, конструктор — заселение (комната готова), `Dispose` — выселение (уборка перед следующим гостем).

Когда тяжёлую инициализацию (например, запуск контейнера, загрузку большого набора данных) нежелательно повторять для каждого теста, используют `IClassFixture<T>`. xUnit создаёт один экземпляр fixture на весь класс, разделяя его между всеми тестами, но при этом по-прежнему создаёт новый экземпляр самого тестового класса на каждый тест. Fixture также может реализовывать `IDisposable` для очистки «при выезде всего тура».

Ключевое правило: тесты должны быть детерминированными и независимыми. Порядок выполнения не гарантирован, общее состояние между тестами — путь к хаосу. Именно поэтому xUnit намеренно не предоставляет `SetUp`/`TearDown` на уровне класса: философия диктует, что каждый тест самодостаточен.

#### Theory (EN)

xUnit is the modern unit-testing framework in the .NET ecosystem, widely adopted as a successor to MSTest and NUnit. Its philosophy is lean: a test is a method, and attributes decide how it runs. The two pillars are `[Fact]` and `[Theory]`.

`[Fact]` marks a test that always runs the same way: one input, one expected outcome. Think of it as asserting an invariant truth. If `Add(2, 3)` must return `5`, that is a fact. Facts are perfect for verifying behaviour that does not vary with input.

`[Theory]` is a parameterised test. The same method runs against multiple data sets. Picture a matrix: rows are argument sets, columns are method parameters. Instead of copying the test ten times, you write it once and feed data through provider attributes.

`[InlineData]` is the simplest provider. You list values directly in the attribute: `[InlineData(2, 3, 5)] [InlineData(10, -2, 8)]`. Handy for a handful of cases, but the values must be compile-time constants — you cannot pass a class instance or a computed result.

`[MemberData]` pulls data from a static property or field returning `IEnumerable<object[]>`. This unlocks complex objects, computed values, and even data loaded from a file or database. Compare: `InlineData` is a shopping list scribbled on a napkin, `MemberData` is a spreadsheet you can edit independently of the code.

`[ClassData]` goes further: an entire class implementing `IEnumerable<object[]>` or `TheoryData<>` becomes the data source. This lets you reuse data sets across test classes and keep them in a single location.

The test-class lifecycle in xUnit differs from NUnit. For every test, xUnit creates a **new instance** of the class — this guarantees isolation, but it means the constructor acts like `SetUp`. Cleanup is handled through `IDisposable`: xUnit calls `Dispose()` after each test automatically. Analogy: each test is a hotel guest, the constructor is check-in (the room is prepared), `Dispose` is check-out (housekeeping before the next guest arrives).

When heavy initialisation — spinning up a container, loading a large data set — should not be repeated per test, you reach for `IClassFixture<T>`. xUnit creates a single fixture instance shared across all tests in the class, while still instantiating the test class itself per test. The fixture can also implement `IDisposable` for teardown “when the whole tour checks out”.

The golden rule: tests must be deterministic and independent. Execution order is not guaranteed, and shared mutable state between tests is a path to chaos. That is exactly why xUnit deliberately omits class-level `SetUp`/`TearDown`: the philosophy insists that every test stands on its own.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — xUnit 2.6+
// Примеры: факты, теории, источники данных, fixture и dispose
// Examples: facts, theories, data sources, fixture and dispose

using Xunit;

namespace Course.M15.Tests;

// --- Тестируемая система / System under test ---
public sealed class Calculator
{
    public int Add(int a, int b) => a + b;              // Сложение / addition
    public bool IsPositive(int value) => value > 0;     // Проверка / check
}

// --- [Fact]: один вход, один выход / single input, single output ---
public class CalculatorFactTests
{
    // Конструктор вызывается ПЕРЕД каждым тестом — аналог SetUp
    // Constructor runs BEFORE each test — acts like SetUp
    private readonly Calculator _calc = new();

    // Простой факт / plain fact
    [Fact]
    public void Add_TwoPlusThree_ReturnsFive()
    {
        Assert.Equal(5, _calc.Add(2, 3));
    }
}

// --- [Theory] + [InlineData]: константы времени компиляции / compile-time constants ---
public class CalculatorInlineTests
{
    [Theory]
    [InlineData(2, 3, 5)]        // простые случаи / simple cases
    [InlineData(-1, 1, 0)]       // ноль / zero
    [InlineData(10, -2, 8)]      // отрицательный аргумент / negative arg
    [InlineData(int.MaxValue, 0, int.MaxValue)] // граница / boundary
    public void Add_VariousInputs_ReturnsExpected(int a, int b, int expected)
    {
        Assert.Equal(expected, new Calculator().Add(a, b));
    }
}

// --- [MemberData]: источник из статического свойства / source from static property ---
public class CalculatorMemberDataTests
{
    // IEnumerable<object[]> — каждый элемент это набор аргументов метода
    // IEnumerable<object[]> — each element is a set of method arguments
    public static IEnumerable<object[]> AddCases => new[]
    {
        new object[] { 1, 1, 2 },
        new object[] { 100, 200, 300 },
        new object[] { -5, 5, 0 },
    };

    [Theory]
    [MemberData(nameof(AddCases))]
    public void Add_FromMemberData_ReturnsExpected(int a, int b, int expected)
    {
        Assert.Equal(expected, new Calculator().Add(a, b));
    }
}

// --- [ClassData]: переиспользуемый источник данных / reusable data source ---
public sealed class CalculatorAddData : TheoryData<int, int, int>
{
    public CalculatorAddData()
    {
        Add(4, 6, 10);
        Add(0, 0, 0);
        Add(-3, -7, -10);
    }
}

public class CalculatorClassDataTests
{
    [Theory]
    [ClassData(typeof(CalculatorAddData))]
    public void Add_FromClassData_ReturnsExpected(int a, int b, int expected)
    {
        Assert.Equal(expected, new Calculator().Add(a, b));
    }
}

// --- IClassFixture: общая инициализация на класс / shared setup per class ---
public sealed class DatabaseSimulator : IDisposable
{
    public string ConnectionString { get; } = "simulated-connection";

    public void Dispose()
    {
        // Очистка один раз — после всех тестов класса / cleanup once after all tests
    }
}

public class CalculatorFixtureTests : IClassFixture<DatabaseSimulator>
{
    private readonly DatabaseSimulator _db;

    // Fixture инъектируется через конструктор / fixture injected via constructor
    public CalculatorFixtureTests(DatabaseSimulator db) => _db = db;

    [Fact]
    public void Database_IsAvailable()
    {
        Assert.False(string.IsNullOrEmpty(_db.ConnectionString));
    }
}

// --- Конструктор + IDisposable: setup/teardown на каждый тест / per-test setup & teardown ---
public sealed class PerTestLifecycleTests : IDisposable
{
    private readonly Calculator _calc;

    public PerTestLifecycleTests()
    {
        // Подготовка перед каждым тестом / prepare before each test
        _calc = new Calculator();
    }

    [Fact]
    public void IsPositive_PositiveValue_ReturnsTrue()
    {
        Assert.True(_calc.IsPositive(42));
    }

    public void Dispose()
    {
        // Освобождение ресурсов после каждого теста / release resources after each test
        // xUnit вызовет этот метод автоматически / xUnit calls this method automatically
    }
}
```

#### Best Practices

- Один тест — одно утверждение по возможности; если логика ветвится, разбейте на несколько `[Theory]`. / One test, one assertion where possible; if logic branches, split into several `[Theory]`.
- Именуйте тесты по схеме `Method_Scenario_ExpectedResult` — это читается как спецификация. / Name tests as `Method_Scenario_ExpectedResult` — it reads like a spec.
- Используйте `[InlineData]` для тривиальных констант, `[MemberData]`/`[ClassData]` — для вычисляемых и сложных данных. / Use `[InlineData]` for trivial constants, `[MemberData]`/`[ClassData]` for computed or complex data.
- Не полагайтесь на порядок выполнения тестов — каждый тест должен быть независимым. / Do not rely on test execution order — each test must be independent.
- Выносите тяжёлую инициализацию в `IClassFixture`, а не дублируйте её в конструкторе. / Move heavy setup into `IClassFixture`, do not duplicate it in the constructor.
- Реализуйте `IDisposable` для освобождения внешних ресурсов (файлы, соединения, контейнеры). / Implement `IDisposable` to release external resources (files, connections, containers).
- Предпочитайте `TheoryData<...>` необработанному `IEnumerable<object[]>` ради типобезопасности. / Prefer `TheoryData<...>` over raw `IEnumerable<object[]>` for type safety.

#### Частые ошибки / Common Mistakes

- `[InlineData]` со ссылочными типами или вычисляемыми значениями → используйте `[MemberData]` с `IEnumerable<object[]>` или `TheoryData`. (RU)
- Общее изменяемое поле между тестами (ожидание порядка) → не делитесь состоянием; каждый тест получает новый экземпляр класса. (RU)
- Забытый `IDisposable` при работе с `HttpClient`/`FileStream` → реализуйте `Dispose` или используйте `using`. (RU)
- `MemberData` указан на нестатическом члене → свойство или поле должны быть `static`. (RU)
- `ClassData` без публичного конструктора без параметров → добавьте публичный конструктор по умолчанию. (RU)
- Using `[InlineData]` with reference types or computed values → switch to `[MemberData]` with `IEnumerable<object[]>` or `TheoryData`. (EN)
- Shared mutable field between tests (relying on order) → do not share state; each test gets a fresh class instance. (EN)
- Forgetting `IDisposable` with `HttpClient`/`FileStream` → implement `Dispose` or use `using`. (EN)
- `MemberData` pointing at a non-static member → the property or field must be `static`. (EN)
- `ClassData` type without a public parameterless constructor → add a public default constructor. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я использую `[Fact]` для инвариантов и `[Theory]` для параметризованных проверок. (RU)
- [ ] Я выбрал правильный провайдер данных: `InlineData` — константы, `MemberData`/`ClassData` — сложные данные. (RU)
- [ ] `MemberData` ссылается на `static` свойство/поле, возвращающее `IEnumerable<object[]>`. (RU)
- [ ] `ClassData`-класс имеет публичный конструктор без параметров. (RU)
- [ ] Я реализую `IDisposable`, когда тест создаёт освобождаемые ресурсы. (RU)
- [ ] Тяжёлая инициализация вынесена в `IClassFixture<T>`, а не дублируется в конструкторе. (RU)
- [ ] Тесты независимы и не зависят от порядка выполнения. (RU)
- [ ] I use `[Fact]` for invariants and `[Theory]` for parameterised checks. (EN)
- [ ] I picked the right data provider: `InlineData` for constants, `MemberData`/`ClassData` for complex data. (EN)
- [ ] `MemberData` points at a `static` property/field returning `IEnumerable<object[]>`. (EN)
- [ ] The `ClassData` class has a public parameterless constructor. (EN)
- [ ] I implement `IDisposable` when a test creates disposable resources. (EN)
- [ ] Heavy setup is moved into `IClassFixture<T>`, not duplicated in the constructor. (EN)
- [ ] Tests are independent and do not rely on execution order. (EN)

#### Ресурсы / Resources

- [xUnit — Getting started with .NET CLI — https://xunit.net/docs/getting-started/netcore/cmdline](https://xunit.net/docs/getting-started/netcore/cmdline)

---

[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
